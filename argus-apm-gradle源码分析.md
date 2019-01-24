# argus-apm-gradle源码分析
argus-apm-gradle工程定义了一个gradle plugin，主要有以下两个作用：
1. 支持AOP编程，方便ArgusAPM能够在编译期织入一些性能采集的代码；
2. 通过Gradle插件来管理依赖库，使用户接入ArgusAPM更简单。

argus-apm-gradle使用kotlin语言开发。这里我们假定大家已经熟悉gradle plugin的开发。
resources/META-INF/gradle-plugins/目录下的argusapm.properties指明了plugin的入口：
```
implementation-class=com.argusapm.gradle.AspectJPlugin
```
这是个java properties文件，指明了plugin的入口是com.argusapm.gradle.AspectJPlugin。
我们看一下com.argusapm.gradle.AspectJPlugin类的代码。
```
internal class AspectJPlugin : Plugin<Project> {
    private lateinit var mProject: Project
    override fun apply(project: Project) {
        mProject = project
        project.extensions.create(AppConstant.USER_CONFIG, ArgusApmConfig::class.java)

        //公共配置初始化,方便获取公共信息
        PluginConfig.init(project)

        //自定义依赖库管理
        project.gradle.addListener(ArgusDependencyResolutionListener(project))

        project.repositories.mavenCentral()
        project.compatCompile("org.aspectj:aspectjrt:1.8.9")


        if (project.plugins.hasPlugin(AppPlugin::class.java)) {
            project.gradle.addListener(BuildTimeListener())

            val android = project.extensions.getByType(AppExtension::class.java)
            android.registerTransform(AspectJTransform(project))
        }
    }
}
```
类AspectJPlugin实现接口org.gralde.api.Plugin，有一个org.gradle.api.Project的范型参数。接口Plugin只有一个方法：apply(T var1)。AspectJPlugin中定义了一个mProject属性，并且在apply方法中使用类型参数project进行初始化。`project.extensions.create(AppConstant.USER_CONFIG, ArgusApmConfig::class.java)`接收ArgusApmConfig中定义的参数，而create的第一个参数，是在使用插件的项目的build.gradle 文件中的进行参数配置的dsl的名字。接下来调用PluginConfig.init(project)进行初始化。`project.gradle.addListener(ArgusDependencyResolutionListener(project))`实现了自定义依赖库管理。`project.repositories.mavenCentral()`将中央仓库指向默认的maven central。`project.compatCompile("org.aspectj:aspectjrt:1.8.9")`添加aspectjrt依赖。compatCompile是通过kotlin的扩展功能为Project临时扩展的函数。ArgusDependencyResolutionListener类实现了DependencyResolutionListener接口，用来监听构建过程中依赖的关系。最后，如果使用plugin的工程中有AppPlugin（`apply plugin: 'com.android.application'`),  再添加BuildTimeListener，该类实现了TaskExecutionListener, BuildListener两个接口，监控build的时间。然后，获取Extension的具体类型，赋值给android，并为android注册AspectJTransform。

##AspectJTransform类
AspectJTransform类继承自com.android.build.api.transform.Transform，主要的逻辑都在override的方法transform中：
```
override fun transform(transformInvocation: TransformInvocation) {
    val transformTask = transformInvocation.context as TransformTask
    LogStatus.logStart(transformTask.variantName)

    //第一步：对输入源Class文件进行切割分组
    val fileFilter = FileFilter(project, transformTask.variantName)
    val inputSourceFileStatus = InputSourceFileStatus()
    InputSourceCutter(transformInvocation, fileFilter, inputSourceFileStatus).startCut()

    //第二步：如果含有AspectJ文件,则开启织入;否则,将输入源输出到目标目录下
    if (PluginConfig.argusApmConfig().enabled && fileFilter.hasAspectJFile()) {
        AjcWeaverManager(transformInvocation, inputSourceFileStatus).weaver()
    } else {
        outputFiles(transformInvocation)
    }

    LogStatus.logEnd(transformTask.variantName)
}
```
transform方法首先定义一个TransformTask类型的变量transformTask，并且用参数transformInvocation的context属性对它进行初始化。然后开始输出log。
接下来，对输入源class文件进行切割分组。我们分析FileFilter的源代码。
FileFilter构造函数有两个参数：private val project: Project, private val variantName: String。前一个参数是Project信息，后一个参数，指明编译的变体，即debug或release等。FileFilter的init代码块如下：
```
init {
    init()
}

private fun init() {
    basePath = project.buildDir.absolutePath + File.separator + AndroidProject.FD_INTERMEDIATES + "/${AppConstant.TRANSFORM_NAME}/" + variantName
    aspectPath = basePath + File.separator + "aspects"
    includeDirPath = basePath + File.separator + "include_dir"
    excludeDirPath = basePath + File.separator + "exclude_dir"
}
```
这里实际上指明了gradle编译过程中各个路径，basePath就位于我们build中常见的imtermediates目录下，它可能是是这样的：`{project root dir}/{module dir}/build/imtermediates/argus_apm_ajx/debug/`。后面的几个path都位于这个路径下。
回到AspectJTransform::transform()中的代码，定义并且初始化了inputSourceFileStatus之后，实例化InputSourceCutter类并且调用它的startCut()方法：`InputSourceCutter(transformInvocation, fileFilter, inputSourceFileStatus).startCut()`。

##InputSourceCutter类
类InputSourceCutter的构造函数有三个参数, 分别是transformInvocation，前边定义的fileFilter和inputSourceFileStatus，它的init方法如下：
```
init {
    if (transformInvocation.isIncremental) {
        LogStatus.isIncremental("true")
        LogStatus.cutStart()

        transformInvocation.inputs.forEach { input ->
            input.directoryInputs.forEach { dirInput ->
                whenDirInputsChanged(dirInput)
            }

            input.jarInputs.forEach { jarInput ->
                whenJarInputsChanged(jarInput)
            }
        }

        LogStatus.cutEnd()
    } else {
        LogStatus.isIncremental("false")
        LogStatus.cutStart()

        transformInvocation.outputProvider.deleteAll()

        transformInvocation.inputs.forEach { input ->
            input.directoryInputs.forEach { dirInput ->
                cutDirInputs(dirInput)
            }

            input.jarInputs.forEach { jarInput ->
                cutJarInputs(jarInput)
            }
        }
        LogStatus.cutEnd()
    }
}
```
init方法中，先判断transformInvocation是不是incremental的，如果是，就遍历transformInvocation中的input列表，接着遍历该列表下每一个input的directoryInputs和jarInputs列表，分别调用whenDirInputsChanged和whenJarInputsChanged方法。如果transformInvocation不是incremental的，就删除transformInvocation的outputProvider中的所有内容。然后遍历transformInvocation中的input列表，接着遍历该列表下每一个input的directoryInputs和jarInputs列表，分表调用cutDirInputs和cutJarInputs。
InputSourceCutter中定义了一个ThreadPool类型的属性taskManager，ThreadPool类维护了一个定长线程池。
首先我们看whenDirInputsChanged的代码
```
private fun whenDirInputsChanged(dirInput: DirectoryInput) {
    taskManager.addTask(object : ITask {
        override fun call(): Any? {
            dirInput.changedFiles.forEach { (file, status) ->
                fileFilter.whenAJClassChangedOfDir(dirInput, file, status, inputSourceFileStatus)
                fileFilter.whenClassChangedOfDir(dirInput, file, status, inputSourceFileStatus)
            }

            //如果include files 发生变化，则删除include输出jar
            if (inputSourceFileStatus.isIncludeFileChanged) {
                logCore("whenDirInputsChanged include")
                val includeOutputJar = transformInvocation.outputProvider.getContentLocation("include", contentTypes as Set<QualifiedContent.ContentType>, scopes, Format.JAR)
                FileUtils.deleteQuietly(includeOutputJar)
            }

            //如果exclude files发生变化，则重新生成exclude jar到输出目录
            if (inputSourceFileStatus.isExcludeFileChanged) {
                logCore("whenDirInputsChanged exclude")
                val excludeOutputJar = transformInvocation.outputProvider.getContentLocation("exclude", contentTypes as Set<QualifiedContent.ContentType>?, scopes, Format.JAR)
                FileUtils.deleteQuietly(excludeOutputJar)
                mergeJar(getExcludeFileDir(), excludeOutputJar)
            }
            return null
        }
    })
}
```
whenDirInputsChanged往taskManager中添加一个新的task。这个task的工作内容如下：遍历whenDirInputsChanged的参数dirInput目录中发生变化的文件，然后调用fileFilter的whenAJClassChangedOfDir方法和whenClassChangedOfDir方法。如果属性inputSourceFileStatus中的isIncludeFileChanged变成true，即include files 发生变化，则删除include输出jar；如果inputSourceFileStatus的isExcludeFileChanged为true，即exclude files发生变化，则重新生成exclude jar到输出目录。最后，调用mergeJar生成新的jar文件。
whenJarInputsChanged的file如下：
```
private fun whenJarInputsChanged(jarInput: JarInput) {
    if (jarInput.status != Status.NOTCHANGED) {
        taskManager.addTask(object : ITask {
            override fun call(): Any? {
                fileFilter.whenAJClassChangedOfJar(jarInput, inputSourceFileStatus)
                fileFilter.whenClassChangedOfJar(transformInvocation, jarInput)
                return null
            }
        })
    }
}
```
如果参数jarInput的status不等于Status.NOTCHANGED，就是jarInput表示的jar文件发生了变化，whenJarInputsChanged向taskManager中添加一个新的task。这个task依次调用fileFilterwhenAJClassChangedOfJar方法和whenClassChangedOfJar方法。
cutDirInputs方法的代码如下：
```
private fun cutDirInputs(dirInput: DirectoryInput) {
    taskManager.addTask(object : ITask {
        override fun call(): Any? {
            dirInput.file.eachFileRecurse { file ->
                //过滤出AJ文件
                fileFilter.filterAJClassFromDir(dirInput, file)
                //过滤出class文件
                fileFilter.filterClassFromDir(dirInput, file)
            }

            //put exclude files into jar
            if (countOfFiles(getExcludeFileDir()) > 0) {
                val excludeJar = transformInvocation.outputProvider.getContentLocation("exclude", contentTypes as Set<QualifiedContent.ContentType>, scopes, Format.JAR)
                mergeJar(getExcludeFileDir(), excludeJar)
            }
            return null
        }
    })
}
```
cutDirInputs也向taskManager中添加一个task。这个task的工作内容是：遍历参数dirInput中的每一个文件，调用fileFilter的filterAJClassFromDir和filterClassFromDir。如果exclude File Dir中的文件数大于0，就调用mergeJar，把这些exclude files放入jar文件。
方法cutJarInputs的代码如下：
```
private fun cutJarInputs(jarInput: JarInput) {
    taskManager.addTask(object : ITask {
        override fun call(): Any? {
            fileFilter.filterAJClassFromJar(jarInput)
            fileFilter.filterClassFromJar(transformInvocation, jarInput)
            return null
        }
    })
}
```
cutJarInputs向taskManager中添加一个task。task调用fileFilter的filterAJClassFromJar和filterClassFromJar方法，从jar文件中过滤掉AJ文件和class文件。
最后，我们回到AspectJTransform的transform方法，它调用了InputSourceCutter的starCut方法，这个方法调用了taskManager.startWork()，开启taskManager中的所有task。taskManager中的task，是通过前面分析的几个方法加入的，最初的入口是InputSourceCutter的init方法。