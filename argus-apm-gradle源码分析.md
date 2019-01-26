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

AspectJPlugin类详细分析
---------------------
我们再回到AspectJPlugin类，详细的分析一下apply方法的流程，以及涉及的类和方法。
PluginConfig类init的代码如下所示：
```
fun init(project: Project) {
    val hasAppPlugin = project.plugins.hasPlugin(AppPlugin::class.java)
    val hasLibPlugin = project.plugins.hasPlugin(LibraryPlugin::class.java)
    if (!hasAppPlugin && !hasLibPlugin) {
        throw  GradleException("argusapm: The 'com.android.application' or 'com.android.library' plugin is required.")
    }

    Companion.project = project

    getVariants(project).all { variant ->
        val javaCompile = variant.javaCompile as JavaCompile
        encoding = javaCompile.options.encoding
        bootClassPath = getBootClasspath().joinToString(File.pathSeparator)
        sourceCompatibility = javaCompile.sourceCompatibility
        targetCompatibility = javaCompile.targetCompatibility
    }
}
```
init方法一次调用project.plugins.hasPlugin(AppPlugin::class.java)和project.plugins.hasPlugin(LibraryPlugin::class.java)来判断project是否有app plugin和lib plugin。如果两种都不存在，就会抛出GradleException。 接下来把参数project赋值给类的Companion变量project。获取project的所有build variant，对所有variants执行以下操作：获取variant的javaCompile，用javaCompile中的encoding，bootClassPath，sourceCompatibility和targetCompatibility，对类的对应属性进行初始化。其中，bootClassPath要通过调用`getBootClasspath().joinToString(File.pathSeparator)`获取。getBootClasspath的代码如下：
```
private fun getBootClasspath(): List<File> {
    val hasAppPlugin = project.plugins.hasPlugin(AppPlugin::class.java)
    val plugin = project.plugins.getPlugin(if (hasAppPlugin) {
        AppPlugin::class.java
    } else {
        LibraryPlugin::class.java
    })
    val extAndroid = if (hasAppPlugin) {
        project.extensions.getByType(AppExtension::class.java)
    } else {
        project.extensions.getByType(LibraryExtension::class.java)
    }
    return extAndroid.bootClasspath
            ?: plugin::class.java.getMethod("getRuntimeJarList").invoke(plugin) as List<File>
}
```
首先判断project是否有app plugin,如果存在，则通过project.plugins.getPlugin获取一个AppPlugin，否则，获取一个LibraryPlugin。如果存在app plugin，则从project extensions（ExtensiionContainer）中，获取AppExtensioin，赋值给变量extAndroid;否则，获取LibraryExtension，赋值给extAndroid。如果extAndroidgetBootClasspath（）返回列表非空，就返回这个列表；否则，通过反射，找到plugin(Plugin类型)的getRuntimeJarList方法，调用该方法，并且把该方法的返回转换为List<File>作为getBootClasspat（）的返回。
回到AspectJPlugin的apply方法, `project.gradle.addListener(ArgusDependencyResolutionListener(project))`添加自定义依赖哭管理，这对应着使用argus-apm-gradle插件的项目中的argusapm.gradle文件中的moduleDependencies。ArgusDependencyResolutionListener实现了DependencyResolutionListener接口，重写了beforeResolve方法。该方法主要是决定使用网络上的lib还是本地的module。

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
transform方法第一步
-----------------
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
这里实际上指明了gradle编译过程中各个路径，basePath就位于我们build中常见的imtermediates目录下，它可能是这样的：`{project root dir}/{module dir}/build/imtermediates/argus_apm_ajx/debug/`。后面的几个path都位于这个路径下。

回到AspectJTransform::transform()中的代码，定义并且初始化了inputSourceFileStatus之后，实例化InputSourceCutter类并且调用它的startCut()方法：`InputSourceCutter(transformInvocation, fileFilter, inputSourceFileStatus).startCut()`。
下面是一个目录的实际的例子：
![Argus Build Path](ArgusBuildPathBase64)
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

FileFilter源码
-------------
下面我们来分析FileFilter的源码。
构造函数和init方法我们之前已经分析过了。我们先来看看前边在InputSourceCutter中用到的几个方法。
先来看whenAJClassChangedOfDir方法。
```
fun whenAJClassChangedOfDir(dirInput: DirectoryInput, file: File, status: Status, inputSourceFileStatus: InputSourceFileStatus) {
    if (isAspectClassFile(file)) {
        log("aj class changed ${file.absolutePath}")
        inputSourceFileStatus.isAspectChanged = true
        val path = file.absolutePath
        val subPath = path.substring(dirInput.file.absolutePath.length)
        val cacheFile = File(aspectPath + subPath)

        when (status) {
            Status.REMOVED -> {
                FileUtils.deleteQuietly(cacheFile)
            }
            Status.CHANGED -> {
                FileUtils.deleteQuietly(cacheFile)
                cache(file, cacheFile)
            }
            Status.ADDED -> {
                cache(file, cacheFile)
            }
            else -> {
            }
        }
    }
}
```
whenAJClassChangedOfDir首先调用isAspectClassFile(file)这是不是一个aspect class文件。isAspectClassFile(file)首先判断文件是不是一个classfile，进一步通过`isAspectClass(FileUtils.readFileToByteArray(file))`读入文件内容并转换为byte array，判断这是不是一个aspect class文件。isAspectClass方法位于Utils类，其代码如下：
```
fun isAspectClass(bytes: ByteArray): Boolean {
    if (bytes.isEmpty()) {
        return false
    }

    try {
        val classReader = ClassReader(bytes)
        val classWriter = ClassWriter(classReader, ClassWriter.COMPUTE_MAXS or ClassWriter.COMPUTE_FRAMES)
        val aspectJClassVisitor = AspectJClassVisitor(classWriter)
        classReader.accept(aspectJClassVisitor, ClassReader.EXPAND_FRAMES)
        return aspectJClassVisitor.isAspectClass
    } catch (e: Exception) {

    }

    return false
}
```
isAspectClass用到了ASM框架。ASM是一个Java字节码操控框架。它能被用来动态生成类或者增强既有类的功能。ClassReader,ClassWriter类都来自该框架。
ClassReader类可以直接由字节数组或由class文件间接的获得字节码数据，它能正确的分析字节码，构建出抽象的树在内存中表示字节码。它会调用accept方法，这个方法接受一个实现了ClassVisitor接口的对象实例作为参数，然后依次调用 ClassVisitor接口的各个方法。字节码空间上的偏移被转换成 visit 事件时间上调用的先后，所谓 visit 事件是指对各种不同 visit 函数的调用，ClassReader知道如何调用各种visit函数。在这个过程中用户无法对操作进行干涉，所以遍历的算法是确定的，用户可以做的是提供不同的Visitor来对字节码树进行不同的修改。ClassVisitor会产生一些子过程，比如visitMethod会返回一个实现MethordVisitor接口的实例，visitField会返回一个实现 FieldVisitor接口的实例，完成子过程后控制返回到父过程，继续访问下一节点。因此对于ClassReader来说，其内部顺序访问是有一定要求的。实际上用户还可以不通过ClassReader类，自行手工控制这个流程，只要按照一定的顺序，各个 visit 事件被先后正确的调用，最后就能生成可以被正确加载的字节码。当然获得更大灵活性的同时也加大了调整字节码的复杂度。
ClassWriter实现了ClassVisitor接口，而且含有一个toByteArray()函数，返回生成的字节码的字节流，将字节流写回文件即可生产调整后的 class 文件。一般它都作为职责链的终点，把所有visit事件的先后调用（时间上的先后），最终转换成字节码的位置的调整（空间上的前后）。
AspectJClassVisitor扩展自ClassVisitor类，并且重写了visitAnnotation方法.visitAnnotation方法声明：`public AnnotationVisitor visitAnnotation(String desc, boolean visible)`。参数desc表示被注解修饰的class的描述符，参数visible表示注解在运行时是否可见.重写的visitAnnotation会判断类前有没有“org/aspectj/lang/annotation/Aspect”注解。这也是判断是否是Aspect class的依据。
如果类前包含以上注解，那么它就是一个Aspect class，那么回到FiltFilter的isAspectClassFile方法。如果这是一个Aspect class，那么我们先要把inputSourceFileStatus.isAspectChanged设置成true，标记Aspect class 发生了改变，并且设置path和subPath，subPathdirInput的路径除去所有前导目录，cacheFile的路径就是aspectPath再加上subPath。接下来，根据变化的status，采取进一步操作。如果是Status.REMOVED，就删除cacheFile；如果是Status.CHANGED，首先删除cacheFile，然后调用`cache(file, cacheFile)`方法，生成新的cache文件；如果状态是Status.ADDED，就直接调用`cache(file, cacheFile)`生成新的cache文件;其它状态不处理。
接下来我们看whenClassChangedOfDir的代码：
```
fun whenClassChangedOfDir(dirInput: DirectoryInput, file: File, status: Status, inputSourceFileStatus: InputSourceFileStatus) {
	val path = file.absolutePath
    val subPath = path.substring(dirInput.file.absolutePath.length)
    val transPath = subPath.replace(File.separator, ".")

    val isInclude = isIncludeFilterMatched(transPath, PluginConfig.argusApmConfig().includes) && !isExcludeFilterMatched(transPath, PluginConfig.argusApmConfig().excludes)

    if (!inputSourceFileStatus.isIncludeFileChanged && isInclude) {
        inputSourceFileStatus.isIncludeFileChanged = isInclude
    }

    if (!inputSourceFileStatus.isExcludeFileChanged && !isInclude) {
        inputSourceFileStatus.isExcludeFileChanged = !isInclude
    }

    val target = File(if (isInclude) {
        includeDirPath + subPath
    } else {
        excludeDirPath + subPath
    })
    when (status) {
        Status.REMOVED -> {
            logCore("[ Status.REMOVED ] file path is ${file.absolutePath}")
            FileUtils.deleteQuietly(target)
        }
        Status.CHANGED -> {
            logCore("[ Status.CHANGED ] file path is ${file.absolutePath}")
            FileUtils.deleteQuietly(target)
            cache(file, target)
        }
        Status.ADDED -> {
            logCore("[ Status.ADDED ] file path is ${file.absolutePath}")
            cache(file, target)
        }
        else -> {
            logCore("else file path is ${file.absolutePath}")
        }
    }
}
```
whenClassChangedOfDir和whenAJClassChangedOfDir类似。首先获得transform path，和aspect path不同，transform path，就是传入file的绝对路径的父目录。接下来调用isIncludeFilterMatched方法和isExcludeFilterMatched，判断include路径和exclude路径是否匹配。如果匹配include路径并且不匹配exclude路径，并且inputSourceFileStatus.isIncludeFileChanged原来的值为false，就把inputSourceFileStatus.isIncludeFileChanged标记为true；如果匹配exlcue路径，并且inputSourceFileStatus.isExcludeFileChanged与阿里来的值为false，那么就把inputSourceFileStatus.isExcludeFileChanged标记为true。接下来生成一个target文件，根据之前isInclude的值决定生成include还是exclude文件。最后，根据参数status的值，对target文件执行不同的操作。如果是Status.REMOVED，就删除目标文件；如果是Status.CHANGED，就先删除目标文件，然后以参数file的内容，重新生成一个target文件；如果是Status.ADDED，就直接以参数file的内容，生成一个新的目标文件。
接下来，我们来分析whenAJClassChangedOfJar函数。
```
fun whenAJClassChangedOfJar(jarInput: JarInput, inputSourceFileStatus: InputSourceFileStatus) {
    val jarFile = java.util.jar.JarFile(jarInput.file)
    val entries = jarFile.entries()
    while (entries.hasMoreElements()) {
        val jarEntry = entries.nextElement()
        val entryName = jarEntry.name
        if (!jarEntry.isDirectory && isClassFile(entryName)) {
            val bytes = ByteStreams.toByteArray(jarFile.getInputStream(jarEntry))
            val cacheFile = java.io.File(aspectPath + java.io.File.separator + entryName)
            if (isAspectClass(bytes)) {
                inputSourceFileStatus.isAspectChanged = true
                when {
                    jarInput.status == Status.REMOVED -> FileUtils.deleteQuietly(cacheFile)
                    jarInput.status == Status.CHANGED -> {
                        FileUtils.deleteQuietly(cacheFile)
                        cache(bytes, cacheFile)
                    }
                    jarInput.status == Status.ADDED -> {
                        cache(bytes, cacheFile)
                    }
                }
            }
        }
        jarFile.close()
    }
}
```
whenAJClassChangedOfJar获取输入的jar文件，并且解析出jar中的条目。然后遍历这个条目列表。如果该条目不是目录，并且是class文件，就读取该class文件的内容，并且转换为byte array，输出到变量bytes。生成一个cache文件，文件路径是aspect目录的路径+当前这个条目对应的class文件的名字。如果这是一个aspect文件，就把inputSourceFileStatus.isAspectChanged标记为true。然后根据参数jarInput中status的值，对cache文件进行不同的处理：如果status是Status.REMOVED，就删除缓存文件；如果是Status.CHANGED，就先删除cache文件，在用bytes的内容重新生成一个cache文件；如果是Status.ADDED，就直接以bytes中的内容生成一个新的cache文件。最后，记得把jar文件关闭。
接下来，我们来分析whenClassChangedOfJar函数。
```
fun whenClassChangedOfJar(transformInvocation: TransformInvocation, jarInput: JarInput) {
    val outputJar = transformInvocation.outputProvider.getContentLocation(jarInput.name, jarInput.contentTypes, jarInput.scopes, Format.JAR)

    when {
        jarInput.status == Status.REMOVED -> {
            FileUtils.deleteQuietly(outputJar)
        }
        jarInput.status == Status.ADDED -> {
            filterJar(jarInput, PluginConfig.argusApmConfig().includes, PluginConfig.argusApmConfig().excludes, PluginConfig.argusApmConfig().excludeJars)
        }
        jarInput.status == Status.CHANGED -> {
            FileUtils.deleteQuietly(outputJar)
        }
    }

    //将不需要做AOP处理的文件原样copy到输出目录
    if (!filterJar(jarInput, PluginConfig.argusApmConfig().includes, PluginConfig.argusApmConfig().excludes, PluginConfig.argusApmConfig().excludeJars)) {
        FileUtils.copyFile(jarInput.file, outputJar)
    }
}
```
首先，通过transformInvocation.outputProvider.getContentLocation获取输出jar文件outputJar。根据参数jarInput中status的值，决定对ouputJar的动作。如果是Status.REMOVED，就删除outputJar；如果是Status.ADDED，就调用filterJar函数，生成一个新的outputJar文件；如果是Status.CHANGE，就把outputJar直接删掉。对于不需要做AOP处理的文件，调用FileUtils.copyFile(jarInput.file, outputJar)，原样拷贝到outputJar。
方法filterJar位于Utils类中，它的作用就是参数includes或excludes以及excludeJars中有没有和jarInput匹配的路径。代码比较简单，这里省略。

下面，我们分析在InputSourceCutter类中调用的FileFilter的以下几个方法：filterAJClassFromDir，filterClassFromDir，filterAJClassFromJar，filterClassFromJar。
filterAJClassFromDir的代码如下：
```
fun filterAJClassFromDir(dirInput: DirectoryInput, file: File) {
    if (isAspectClassFile(file)) {
        val path = file.absolutePath
        val subPath = path.substring(dirInput.file.absolutePath.length)
        val cacheFile = File(aspectPath + subPath)
        cache(file, cacheFile)
    }
}
```
这个函数首先判断参数file是不是aspect class文件，如果是，就用aspect目录+file的文件名组成一个缓存文件路径cacheFile，调用cache方法，将file文件中的内容，输出到cacheFile中。
filterClassFromDir的代码如下：
```
fun filterClassFromDir(dirInput: DirectoryInput, file: File) {
    if (isClassFile(file)) {
        val path = file.absolutePath
        val subPath = path.substring(dirInput.file.absolutePath.length)
        val transPath = subPath.replace(File.separator, ".")

        val isInclude = isIncludeFilterMatched(transPath, PluginConfig.argusApmConfig().includes) &&
                !isExcludeFilterMatched(transPath, PluginConfig.argusApmConfig().excludes)
        if (isInclude) {
            cache(file, File(includeDirPath + subPath))
        } else {
            cache(file, File(excludeDirPath + subPath))
        }
    }
}
```
首先判断参数file是不是一个class文件，如果是，就从file的绝对路径换算出transPath，然后，判断transPath是不是匹配indluces路径或者excludes路径。如果匹配includes路径，就调用cache方法，把file中的内容，输出到一个新的文件，文件路径是include路径+上文计算出的subPath；如果匹配excludes路径，就调用cache方法，把file中的内容，输出到一个新的文件，文件路径是exclude路径+上文计算出的subPath。
方法filterAJClassFromJar的代码如下：
```
fun filterAJClassFromJar(jarInput: JarInput) {
    val jarFile = JarFile(jarInput.file)
    val entries = jarFile.entries()
    while (entries.hasMoreElements()) {
        val jarEntry = entries.nextElement()
        val entryName = jarEntry.name
        if (!(jarEntry.isDirectory || !isClassFile(entryName))) {
            val bytes = ByteStreams.toByteArray(jarFile.getInputStream(jarEntry))
            val cacheFile = File(aspectPath + File.separator + entryName)
            if (isAspectClass(bytes)) {
                cache(bytes, cacheFile)
            }
        }
    }

    jarFile.close()

}
```
从参数jarInput中解析出jar文件，并且遍历jar文件中的条目，如果该条目不是目录，并且不是class文件，读取文件的内容到Byte Array类型的变量bytes中，用aspect目录+当前条目的名称组成一个路径，并以该路径生成一个cache文件，如果从获取的bytes判断出这是一个aspect文件，就调用cache方法把bytes的内容输出到cacheFile。最后，把jarFile关闭。
filterClassFromJar的代码如下：
```
fun filterClassFromJar(transformInvocation: TransformInvocation, jarInput: JarInput) {
    //如果该Jar包不需要参与到AJC代码织入的话,则直接拷贝到目标文件目录下
    if (!filterJar(jarInput, PluginConfig.argusApmConfig().includes, PluginConfig.argusApmConfig().excludes, PluginConfig.argusApmConfig().excludeJars)) {
        val dest = transformInvocation.outputProvider.getContentLocation(jarInput.name
                , jarInput.contentTypes
                , jarInput.scopes
                , Format.JAR)
        FileUtils.copyFile(jarInput.file, dest)
    }
}
```
首先通过filterJar判断参数jarInput是否匹配includes，excludes或者excludeJars路径。如果都不匹配，说明Jar包不需要参与到AJC代码织入，直接把该jar文件拷贝到目标文件目录下即可。
以上一组函数名中都包含“filter”，在执行目标操作之前都要进行以下“filter”操作。

到目前为止，我们实际上分析完了AspectJTransform类中transform方法的第一步。接下来我们分析该方法的第二步。

transform方法第二步
-----------------
首先判断PluginConfig.argusApmConfig().enabled是否为true，即argus apm有没有使能，同时判断第一步生成的fileFilter是否包含aspect 文件。如果两个条件都满足，就用transformInvocation和inputSourceFileStatus生成一个AjcWeaverManager的对象，并且调用该对象的weaver方法。如果前边的两个条件有一个不满足，调用`outputFiles(transformInvocation)`输出文件的内容。这个方法的具体内容我们稍后分析。首先我们先先来看看类AjcWeaverManager的代码。
类AjcWeaverManager的默认构造函数有两个参数，TransformInvocation类型和InputSourceFileStatus类型，这两个参数会初始化类的属性transformInvocation和inputSourceFileStatus。类中还有以下几个属性：
```
private val threadPool = ThreadPool()
private val aspectPath = arrayListOf<File>()
private val classPath = arrayListOf<File>()
```
threadPool定义了一个线程池，aspectPath和classPath分别用来存储aspect文件路径和class文件路径的列表。
weaver函数的代码如下：
```
fun weaver() {

    System.setProperty("aspectj.multithreaded", "true")

    if (transformInvocation.isIncremental) {
        createIncrementalTask()
    } else {
        createTask()
    }
    log("AjcWeaverList.size is ${threadPool.taskList.size}")

    aspectPath.add(getAspectDir())
    classPath.add(getIncludeFileDir())
    classPath.add(getExcludeFileDir())

    threadPool.taskList.forEach { ajcWeaver ->
        ajcWeaver as AjcWeaver
        ajcWeaver.encoding = PluginConfig.encoding
        ajcWeaver.aspectPath = aspectPath
        ajcWeaver.classPath = classPath
        ajcWeaver.targetCompatibility = PluginConfig.targetCompatibility
        ajcWeaver.sourceCompatibility = PluginConfig.sourceCompatibility
        ajcWeaver.bootClassPath = PluginConfig.bootClassPath
        ajcWeaver.ajcArgs = PluginConfig.argusApmConfig().ajcArgs
    }
    threadPool.startWork()
}
```
首先，设置系统属性aspectj.multithreaded为true，允许多线程运行aspectj。如果transformInvocation.isIncremental为true，也就是支持增量编译，那么调用方法createIncrementalTask，创建增量编译任务；否则，调用方法createTask，创建普通（全量）编译任务。接下来，遍历threadPool的taskList，taskList中的条目通过上文的createIncrementalTask或者createTask添加，这些条目的类型是AjcWeaver。设置每一个条目的encoding，aspectPath，classPath，targetCompatibility,sourceCompatibility,bootClassPath(启动类)，ajcArgs(aspectj的参数)。设置好这些参数之后，调用threadPool.startWork()，启动线程执行。
我们先来看看上文提到的两个create方法。先来看一下createTask：
```
private fun createTask() {

    val ajcWeaver = AjcWeaver()
    val includeJar = transformInvocation.outputProvider.getContentLocation("include", contentTypes as Set<QualifiedContent.ContentType>, scopes, Format.JAR)
    if (!includeJar.parentFile.exists()) {
        FileUtils.forceMkdir(includeJar.parentFile)
    }
    FileUtils.deleteQuietly(includeJar)
    ajcWeaver.outputJar = includeJar.absolutePath
    ajcWeaver.inPath.add(getIncludeFileDir())
    addAjcWeaver(ajcWeaver)

    transformInvocation.inputs.forEach { input ->
        input.jarInputs.forEach { jarInput ->
            classPath.add(jarInput.file)
            //如果该Jar参与AJC织入的话，则进行下面操作
            if (filterJar(jarInput, PluginConfig.argusApmConfig().includes, PluginConfig.argusApmConfig().excludes, PluginConfig.argusApmConfig().excludeJars)) {
                val tempAjcWeaver = AjcWeaver()
                tempAjcWeaver.inPath.add(jarInput.file)

                val outputJar = transformInvocation.outputProvider.getContentLocation(jarInput.name, jarInput.contentTypes,
                        jarInput.scopes, Format.JAR)
                if (!outputJar.parentFile?.exists()!!) {
                    outputJar.parentFile?.mkdirs()
                }

                tempAjcWeaver.outputJar = outputJar.absolutePath
                addAjcWeaver(tempAjcWeaver)
            }

        }
    }

}
```
首先，创建一个AjcWeaver的对象ajcWeaver。获取include jar文件includeJar，如果includeJar的父目录不存在，就创建它的父目录。删除includeJar。ajcWeaver的outputJar设置成includeJar的absolutePath，将Include file目录添加到ajcWeaver的inPath列表。调用addAjcWeaver(ajcWeaver)，实际上把ajcWeaver添加到threadPool的task列表。遍历参数transformInvocation的inputs列表，对每一个item，遍历它的jarInputs列表，把每一个jar文件添加到classPath列表。然后调用filterJar方法，判断该jar文件是否要参与AJC织入。如果要参与，首先生成一个AjcWeaver的对象tempAjcWeaver，获取output jar文件，如果outputJar文件的父目录不存在，首先创建该目录。把tempAjcWeaver的outputJar设成outputJar的绝对路径，然后调用addAjcWeaver把tempAjcWeaver添加到threadPool的taskList。
createIncrementalTask的代码如下：
```
private fun createIncrementalTask() {
    //如果AJ或者Include文件有一个变化的话,则重新织入
    if (inputSourceFileStatus.isAspectChanged || inputSourceFileStatus.isIncludeFileChanged) {
        val ajcWeaver = AjcWeaver()
        val outputJar = transformInvocation.outputProvider?.getContentLocation("include", contentTypes as Set<QualifiedContent.ContentType>, scopes, Format.JAR)
        FileUtils.deleteQuietly(outputJar)

        ajcWeaver.outputJar = outputJar?.absolutePath
        ajcWeaver.inPath.add(getIncludeFileDir())
        addAjcWeaver(ajcWeaver)

        logCore("createIncrementalTask isAspectChanged: [ ${inputSourceFileStatus.isAspectChanged} ]    isIncludeFileChanged:  [ ${inputSourceFileStatus.isIncludeFileChanged} ]")
    }


    transformInvocation.inputs?.forEach { input ->
        input.jarInputs.forEach { jarInput ->
            classPath.add(jarInput.file)
            val outputJar = transformInvocation.outputProvider.getContentLocation(jarInput.name, jarInput.contentTypes, jarInput.scopes, Format.JAR)

            if (!outputJar.parentFile?.exists()!!) {
                outputJar.parentFile?.mkdirs()
            }

            if (filterJar(jarInput, PluginConfig.argusApmConfig().includes, PluginConfig.argusApmConfig().excludes, PluginConfig.argusApmConfig().excludeJars)) {
                if (inputSourceFileStatus.isAspectChanged) {
                    FileUtils.deleteQuietly(outputJar)

                    val tempAjcWeaver = AjcWeaver()
                    tempAjcWeaver.inPath.add(jarInput.file)
                    tempAjcWeaver.outputJar = outputJar.absolutePath
                    addAjcWeaver(tempAjcWeaver)

                    logCore("jar inputSourceFileStatus.isAspectChanged true")
                } else {
                    if (!outputJar.exists()) {
                        val tempAjcWeaver = AjcWeaver()
                        tempAjcWeaver.inPath.add(jarInput.file)
                        tempAjcWeaver.outputJar = outputJar.absolutePath
                        addAjcWeaver(tempAjcWeaver)
                        logCore("jar inputSourceFileStatus.isAspectChanged false && outputJar.exists() is false")
                    }
                }
            }
        }
    }
}
```
首先判断aspect文件或者include文件有没有变化，只要有一个发生变化，就需要重新进行“织入”。生成一个新的AjcWeaver对象ajcWeaver，从output provider中获取输出jar文件，然后删除这个jar，把ajcWeaver的outputJar设置成这个jar文件的绝对路径，把include目录的路径，加入到ajcWeaver的inPath列表中。调用addAjcWeaver(ajcWeaver)把ajcWeaver加入到threadPool的taskList中。之后的操作和createTask()方法类似，遍历transformInvocation重的inputs列表，对inputs中的每一个item，遍历它的jarInputs列表，把每一个jarInput的file属性（File类型）添加到classpath中。从transformInvocation的output provider中获取outputJar文件，如果outputJar的父路径不存在，首先创建它的父目录。调用filterJar判断是否有匹配的includes，excludes path或者exclude jar包。如果存在，并且有aspectj文件发生改变，首先删除前边获取的outputJar文件，生成一个AjcWeaver的对象tempAjcWeaver，把jarInputs中当前的条目的file（File类型）属性，添加到tempAjcWeaver的inPath列表中，outputJar的绝对路径absolutePath，然后调用addAjcWeaver(tempAjcWeaver)把tempAjcWeaver添加到threadPool的taskList列表中。如果没有aspectj文件发生改变，并且outputJar不存在，同样先生成一个AjcWeaver的对象tempAjcWeaver，然后把jarInput的file属性，添加到tempAjcWeaver的inPath列表。tempAjcWeaver的outputJar设置成outputJar的绝对路径absolutePath，然后调用addAjcWeaver(tempAjcWeaver)把tempAjcWeaver添加到threadPool的taskList列表中。

回到AspectJTransform的transform中来。前面讨论的是如果有aspectj文件需要织入的情况。如果没有，那么直接调用`outputFiles(transformInvocation)`将输入源输出到目录下。outputFiles方法的实现很简单，如果transformInvocation支持增量编译，就调用outputChangeFiles(transformInvocation)；否则，调用outputAllFiles(transformInvocation)。我们先看一下outputChangeFiles的代码。
```
fun outputChangeFiles(transformInvocation: TransformInvocation) {
    transformInvocation.inputs.forEach { input ->
        input.directoryInputs.forEach { dirInput ->
            if (dirInput.changedFiles.isNotEmpty()) {
                val excludeJar = transformInvocation.outputProvider.getContentLocation("exclude", dirInput.contentTypes, dirInput.scopes, Format.JAR)
                mergeJar(dirInput.file, excludeJar)
            }
        }

        input.jarInputs.forEach { jarInput ->
            val target = transformInvocation.outputProvider.getContentLocation(jarInput.name, jarInput.contentTypes, jarInput.scopes, Format.JAR)
            when {
                jarInput.status == Status.REMOVED -> {
                    FileUtils.forceDelete(target)
                }

                jarInput.status == Status.CHANGED -> {
                    FileUtils.forceDelete(target)
                    FileUtils.copyFile(jarInput.file, target)
                }

                jarInput.status == Status.ADDED -> {
                    FileUtils.copyFile(jarInput.file, target)
                }

            }
        }
    }
}
```
outputChangeFiles方法遍历参数transformInvocation中的inputs列表。对列表中的每一个条目input，首先遍历它的directoryInputs列表，对其中的每一个条目dirInput，判断其中的changedFiles（map）如果不为空，从outputProvider中获取excludeJar，调用mergeJar(dirInput.file, excludeJar)将dirInput.file中的内容，合并到excludeJar中。然后便利input中的jarInputs，对每一个条目jarInput，从outputProvider中获取目标jar包target，判断jarInput.status：如果是Status.REMOVED，直接把target删除；如果是Status.CHANGED，先删除target，再把jarInput.file中的内容，拷贝到target中；如果是Status.ADDED，直接把jarInput.file中的内容，拷贝到target中。

outputAllFiles的代码如下：
```
fun outputAllFiles(transformInvocation: TransformInvocation) {
    transformInvocation.outputProvider.deleteAll()

    transformInvocation.inputs.forEach { input ->
        input.directoryInputs.forEach { dirInput ->
            val outputJar = transformInvocation.outputProvider.getContentLocation("output", dirInput.contentTypes, dirInput.scopes, Format.JAR)

            mergeJar(dirInput.file, outputJar)
        }

        input.jarInputs.forEach { jarInput ->
            val dest = transformInvocation.outputProvider.getContentLocation(jarInput.name
                    , jarInput.contentTypes
                    , jarInput.scopes
                    , Format.JAR)
            FileUtils.copyFile(jarInput.file, dest)
        }
    }
}
```
outputAllFiles首先删除outputProvider中的所有内容。遍历参数transformInvocation中的inputs列表，对列表中每一个item input，遍历它的directoryInputs列表，对每一个条目dirInput，从outProvider中根据那么“output”获取outputJar，调用mergeJar，把dirInput.file中的内容，合并到outputJar中。然后遍历input中的jarInputs列表，对每一个条目jarInput，根据jarInput.name，从outputProvider中获取目标文件dest，调用`FileUtils.copyFile(jarInput.file, dest)`，把jarInput.file，拷贝到dest。

到目前为止，我们已经分析完成了transform方法，以及其中涉及的主要类和方法。

下面，我们来看看经常被用到的mergeJar函数：
```
fun mergeJar(sourceDir: File, targetJar: File) {
    if (!targetJar.parentFile.exists()) {
        FileUtils.forceMkdir(targetJar.parentFile)
    }

    FileUtils.deleteQuietly(targetJar)
    val jarMerger = JarMerger(targetJar)
    try {
        jarMerger.setFilter(object : JarMerger.IZipEntryFilter {
            override fun checkEntry(archivePath: String): Boolean {
                return archivePath.endsWith(SdkConstants.DOT_CLASS)
            }
        })
        jarMerger.addFolder(sourceDir)
    } catch (e: Exception) {
        e.printStackTrace()
    } finally {
        jarMerger.close()
    }
}
```
mergeJar首先判断参数targetJar的父目录是否存在，如果不存在，首先创建targetJar的父目录。删除targetJar文件。以targetJar为参数创建一个JarMerger的对象jarMerger。调用jarMerger的setFilter方法，设置filter，filter是一个JarMerger.IZipEntryFilter的具体实现，重写方法checkEntry，判断传入的文件路径是不是以".class"结尾。调用jarMerger.addFolder(sourceDir)，最终会触发对文件的读写。
接下来我们逐个分析一下mergeJar中用到的JarMerger中的几个方法。
JarMerger构造函数有一个File类型的参数，jarFile属性代表一个jar文件。JarMerger的init方法还要进行以下初始化操作：
```
private fun init() {
    if (this.closer == null) {
        FileUtils.forceMkdir(jarFile.parentFile)
        this.closer = Closer.create()
        val fos = this.closer!!.register(FileOutputStream(jarFile))
        jarOutputStream = this.closer!!.register(JarOutputStream(fos))
    }
}
```
创建jarFile的父目录，创建一个新的Closer对象，并赋值给closer属性。打开jarFile输出流，同时注册到closer，并且返回给jarOutputStream。
addFolder方法首先调用init方法，然后尝试调用addFolderWithPath(folder, "")。addFolderWithPath方法的代码如下：
```
@Throws(IOException::class, IZipEntryFilter.ZipAbortException::class)
private fun addFolderWithPath(folder: File, path: String) {
    folder.listFiles()?.forEach { file ->

        if (file.isFile) {
            val entryPath = path + file.name
            if (filter == null || filter!!.checkEntry(entryPath)) {
                // new entry
                this.jarOutputStream!!.putNextEntry(JarEntry(entryPath))

                // put the file content
                val localCloser = Closer.create()
                localCloser.use { localCloser ->
                    val fis = localCloser.register(FileInputStream(file))
                    var count = -1
                    while ({ count = fis.read(buffer);count }() != -1) {
                        jarOutputStream!!.write(buffer, 0, count)
                    }
                }

                // close the entry
                jarOutputStream!!.closeEntry()
            }
        } else if (file.isDirectory) {
            addFolderWithPath(file, path + file.name + "/")
        }
    }
}
```
addFolderWithPath列出参数folder目录下的所有文件，并且进行遍历。如果当前条目file是一个目录，那么对该目录继续调用addFolderWithPath方法。如果当前条目是一个文件，用参数path再加上file的name属性组成一个新的path服之给entryPath。如果filter为null，或者entryPath应该包含在jar归档文件中，就根据entryPath生成一个JarEntry对象，并且添加到jarOutputStream。创建一个closer的对象localCloser，创建一个file的FileInputStream，并且注册到localCloser，返回输入流给fis。读取file的内容，输出到jarOutputStream绑定的entryPath对应的文件中。输出完成之后，关闭jarOutputStream。


[ArgusBuildPathBase64]:data:image/jpeg;base64,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iiivPOgKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKcEcjIBNBVh1BFK47DaKKKYgooooAKKKKACiiun8H+GLrxh4gtvD9k22a53YPH8Klu5Hp61UIOTshSkkrs5iius8a+E73wT4hufD1+26a225PH8Shh0JHf1rk6iMk1dFOLW4UUV0/g/wxdeMPEFt4fsm2zXO7B4/hUt3I9PWrhBydkTKSSuzmKK9gt/g/rE3xDPw+eUJcr958LxmPzBxux0965bx/wCCb3wF4gm0G9k81otuH4GcqG6An19al6KMns9ikndrscRRRUkUUk8giiG5m6CmlfRCbI6K09T0fUdHkWLUYTCzdASD/L61mUgCiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKAP/9P4booorzzoCiiigAooooAKKKKACiiigAooooAKKKKAPSPhL4d03xX4903QdWj8y1ufN3Lkj7sbMOhB6j1r0X4gfs9eIfDWb3Qj/aNmMljhY9g4xwzknJJrmP2f/wDkq+i/9t//AES9fpJqOoWOm2r3V/II4k6k8/pW9OCcdSJOx+PDxvGxSRSpHY0yvoH42+MvAnijUN/hq0Es/wDFdBnXOAv8DADsRXz9WMlZloKKKKQHufg9EPh61JUH73b/AGjWrqekWmqW5gmUex9PyrM8Hf8AIvWv/A//AEI109fkmOqyhipyi7NSf5n7zllCFTBUoTV04r8jzE/Dr/p+/wDIf/16y9Z8F/2Tp0t/9q8zy8fLsxnJx1zXsVcx4x/5F66/4B/6EK9TA59ip1oQlPRtdF39Dxcz4XwNPD1KkKeqTa1fb1PDERpHCLyWOBX1XoH7MNzeaTFqXiXXF0dpRkRmES98dVk+nbvXzJpFpeX2ow22njM7n5en9eK+49d+GNmtnZv8UfFpaVQ3lx/ZiuzpnmJueMda/UYQj7Pma6/18z8bbbnZdjwD4pfAzVvhxbR6lDd/2np75zOFWPH3R93cx6titLwT+z7qvjfwta+JdOvton3bo9i/Ltcr94uM9M9K+hPiDZaJZfAK4tdBuzf2UW3y5CjIebgE8Nz1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Dk8LaPpQtJACGnEgbdlt33Sn4da1VX3ZyfxOPL/XoEVa0Fte7Lvwe8S/DrwPeah4j8RX27VVP+jweXJ8pO9W+ZQV5BB5FdB8PPj7ZXGt6rB8QJtul6lsAJUnyxGrdo0ycnFfILsXYuerHNNqVXel1srW6DnG7bWl3c6vxrF4ch8R3S+Fbj7VppKmN9rJnKgnh+eua5SiiueMbKxcpXdwoooqiQooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooA//9D4booorzzoCiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACilCs33Rml2OOqn8qVx2G0UUUxBRRRQAUV2HgfwbqHjnXoNB087ZJt3PHG1S3cj09aoeKvD1x4V1250K6bdLbbcngfeUN2J9aqUWrX6iTve3Q56iinxoZJFjHViB+dKMW3ZDYyivT/iB8L9W+H9vp9zqL711AOU4A+5tz0Y/3q8wpdWg6J9wooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooA//R+G6KKK886AooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigDoPC2kWWu65baXqN7/Z8E+4NNsMm3Ckj5QQTk8V9Vr+yYHUMviXIPP/AB7f/bK+NkdkcOpwVORX6U/A7x+vjTwusF1L5mo2IxPxj77Nt6ADoO1a0knoyZNo8J1z9mSy8P6XPq2o+J/LggALH7KT1OB0cnqa+THAV2UHIBIzX2J+0t8Qd8q+CtOmwEz9rUDrnY6ckfyNfHNKpa9kEb9QooorMo9N+Haq327cAf8AV/1r0ia2gnjMUiAq3WvOPh1/y/f9s/616dX5hxDJrGTt5fkj9q4Tgnl1NNd/zZ5vcfD2KSZnhuvKQ9F2Zx+Oagb4d4Un7d0H/PP/AOvXp9Nf7jfQ1MOIcWrLn/Bf5Fz4UwDu/Z/i/wDM+ZmGGI9K92+FfwL1j4kQtqMtz/Z2nr0nKq4P3gfl3Keq4rw3GZsHu39a+6PGV/e+G/2f9PtdEHkxTh/MIwcYuAR1z6mv1aNo0pVGrtWX3n4fK7moLz/A2/ht8Bz4L8aWfiDSdXGp2kIk8zEYj25jZR1cnkn0r5b+KGk3+u/FfUNM02PzbiYxhVyBnESk8njpXa/sya1qkHj2OwilY292G80HnO2NyOvTmvTvAtna3H7Q+oTzKGeHZtPpm3Oa6JUlOdJN6WkY+1sqllrdfqc7a/soAWsDax4lWwupgT5Jtw+Mf7Qkx0rw3x98Mdb+G+uQWmojzbeZh5M3ygPjaW4DNjG7HNfVfj7wz8MdZ8VXV5rviUwXgKZTyJvlwoxypA5AFc58bPFvhDVfCWm6Ro+o/b7i1YjcYnj+9Ih/iGOgrOla8J7O608mzpUbNweuj18z0X4mfCm6+JFjoLPejTrOzWXzJiofG/bj5dynquK8H8bfsz3uhaE2v+G9UGsRL95BGIsfMF6s5Pr27V3H7SuuapaeFNA0u2cx21ysvm4x820xlecZGDWZ+ytrOqXMuq6NO5lstqfKccfLI3Xr1qaqjKdRR0acn912Qm4wpylqml/keKfD34ST+O9K1TVBffYxpoTK+Xv3byw67hjG2vOND0c6zrMWkCTyzKxXfjOMAnpx6V9pfBa2WLS/GUFsvy/6PtAPu+a+TvAUMsvjizijXLmR+PwNVCMZVqcbaNRf37kTbVObvqnJfcdP43+D2q+FfFdv4U0+f+0p7r/VttEecIHPBY9AfXtXsS/sorDaQNqniVbS8nBxCbcN0/2hJjpXvNza28vxss5pVDSQq23PbNvzXxP8bNb1u9+I1+b+RlaJo9q8DH7tfSslaKimr3vr5J2NWndtbK33tXNzx38A9U8B+GJPEWpXu5kxiLYOcuF+8HPrnpXz7X3N8QNQ1DVP2ebO81Ni07hsk4HS4UDp7V8M0q0eWrOHZ2/BCpzUoRn3/wA2FFFFQUFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAf/0vhuiiivPOgKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAK7z4f+PdS8A6wNSsl82JgRJFkDf8AKwXkg4xnNcHRTTsBo6tqt7rWoS6lqD+ZPMQWbAHQYHT2rOoopAFFFFAHdeC9Z07SftX2+Xy/M2beCc4znpXd/wDCY+Hv+fr/AMcb/CvCqK8HG8O0a9R1Zt3fp/kfU5dxbiMNRjQpxVl3v3v3Pdf+Ex8Pf8/X/jjf4U1/GHh4owFz1B/gb/CvDKK5Vwlh/wCZ/h/kdr47xf8ALH7n/mOY5csPWvr/AOE/xW8Iah4RPw8+IvyWa/ckO87su0h4jXIwcd6+PqM46V9XCdouLV0z4eUbtSW6Pvrw14u+A/wtv/N0O5Est1nfLtnGzaDjhg2c7iK+eL34mQaL8W5vG+gP9rtcrjqm4GLYfvKSMZPavDSSetJTjVkpRl1V/wAegnBOLi+p94atefs+/EC5XxNql39mvZeZl23ByVAUdMDoOwryH4vfELwTq8djongm3CWVqTuly/zZKt0cA9Qe9fNwJHQ0lLmSa5VZJ3LTe8tXsfoNrXxS+CfjTRbPwzr955ke0gybJx5ZBDdFUE5Ix1rHufiP8K/hP4XuLP4eOLvUbsAE4lXO1uP9YGHRjXwjSkk9aqrVcubpzbmcaaXLfW2x9E/A74paX4O1y/8A+EmOLTVCvmSc/LsV8cKpJ5YV7Bb3n7O3gvWV8TWNyJ7kHMSbbgbcja3JyOhz0r4VpSSepzV/WGuVpapWv5D5PiTejPsDxl8UbXV/ivpuv/DyT+03i34TBizmEKeZF7YPau78X+Jvgh4g1Rb7xzYmLWIcGSHNwckgBfmjAXoAa+IfDPiC98L61b63p5xPbk7en8QKnqCOh9K+s1/aN8HawkV34h8OK96mct57D2/hTHQCqg4qC11TfyvroTK7m3bovn6nbfGnVrTW/gpHqNjbfZLeTISPJOAsyjuAe1fnxXunxX+Nuo/ElIrGG1/s+whziEMHznaeu1T1XNeF1yyd5ykurNkrRjF7pBRRRQIKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooA//9P4booorzzoCiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACilAJOBzS7H/ALp/KlcdhtFHSimIKKKKACiiigAooooAKKKKACiiigAooooAKKkiiknkEUQ3M3QVf1PR9R0eRYtRhMLN0BIP8vrRbqFzMooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooA//1PhuiiivPOgKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAK+rPCfwJ0/wAcfDXStf06b7JqMnneYdpfzMSFV6uAMAelfKdfpl+z/wAfCjRf+2//AKOetaUU3qTJ6H58+KfBPiLwdd/Y9dtTA5zjlWBxg/wk+ork6/TH4qeO/AWi6ZLpfiPbdSzD5YPnG/BU/fQHGMg1+bmpT2lzeyzWMH2aBj8se4tj8TSqRSegRdyjRRRWZR03g8A+IbUEZ+9/6Ca9z8uP+6Pyrw3wd/yMNr/wP/0E17rX55xY/wDaI+n6s/WeBF/skv8AE/yRw2reCLbULj7RBN9nJ6jbuz+tZX/Cuv8Ap+/8h/8A169OorzaWfYuEVGM9F6f5HsVuF8DUm5yp6vza/JnztrGnf2TqMthv8zy8fNjGcjPSu5+Gvwx1j4k6jJZ6e3kwwY82U7Ts3BiOCy5ztrmvGP/ACMN1/wD/wBBFe7/AAF8G+Ltetr+507V/wCxtNGzzX8pJt4O8DgkEYI/Wv1PKZe0oxqVNXyp9vvPxfOKSp4idOnolJpfedRN+yjDLBP/AGT4oW9u4QD5Itwuc+5kx0r5rs/Auv3nitfB6QY1AkjZley7+ucdOetfbPw78G/DjQPGFrPo/iE3WpDfhPJlAY7GB5JK9Ca8t8S+K7PwR+0Lca3eRbreHbvAz/FbhewJ7+ldcYQVSHNs7/h5nG+bkqW3VrFyL9lO3zHaXXilYb9wf3H2fOO/3vMx0ryGX4M6xZ/EGDwLqM3kG53bJ8K2dsfmH5Q34da+ntU8LfDH4n6y3iLwxr5stQuMHcYpm5VQv3XKjop7VxUGheMNB+Neh2/iu+/tD/X+TLsSPIEHPCE+oHNaUKSdWEZq17+mxNSdoScei+fzRmQfsqTQyOuveIBpy8eWTAH3+vAk4xxXmPxT+CWtfDVY7sz/AG+xkzicKqdNo+7uY9WxWv8AtHa5qt38RryzmmbyLTy/KUcY3RoT06817Rpd5eeKP2dJzrw8wxbNjnAJzc89MegrjbTourH7NvmtvvOhw5aipy6/hpc8D+FvwO1n4kRvfSXH9naeuP35VXB+8Pu7lPVcV3fiP9l+9sdHl1PwxrQ1uSIZMQiEXcDq0n1/KvoCy0zw2nwWsdP1S+/suxcPucI0nSbI+7z1rnfh/L8KPh5eT3mn+JDOs4+ZPs83OAwHJ3Y+9XVUoxUpQ2t18/NHPSm+WM3rfp5ep8aeA/h1rvj3W/7H0yMrsOJX4+TgkcEjOcV9FSfsnQ/PbW/ilZb5AP3P2cDrz97zMdK9G+Gl5bWujeK/EHh1fMmk+zlT0J5ZTw3TjPavhi28VeIdP8Qf27aXLLfIxw2Ae237pGOntWScLxhJbpNv1/yLs/eku7S+R7R8OPhJexfESXRtduxp0+mlcgqJN3mRlh0bHTHrXtH7R3w3tdTEvic6osDWgXEHlZ3bgifeyPTPSvm218a+IfG3xAsdR16RjLk4+UJ0TH8IX0FehftTuy+LrcKxAK9M/wDTOOorJ+wgut3+W/3dCqVvaza2svz2+85b4b/AXUfHOlHXr+/GlacfuzFBJuwzKfl3gjkYrc8Zfs33ugeH5PEOgasNZhjxkCMRdWC/xOT3PbtXe/Dnx38PPEPw3h+HvjGY2ci52nbKd2ZTJ1QDGMD+Kr+q/CzWtF8M3l78OvEf2iwcKZLfyB0DYHzSsT1JNbYuMUpOC0Wz/wAzLDybsnu3t/kfDBBBIPUUlPcEOwbqCc0yuY1CiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooA//1fhuiiivPOgKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAK9l0/40eI9F8FWPg/Qx9jFp5m+X5X8ze+8fKynGMkda8aopptbBYmnuJ7qQyzuXdupNQ0UUgCiiigDd8NXlvYazBdXTbIk3ZOCeqkdq9Y/wCEx8Pf8/X/AI43+FeFUV42Y5HSxM1Obd7W0Posp4lr4Om6VJJpu+t/8/I91/4THw9/z9f+ON/hR/wmPh7/AJ+v/HG/wrwqiuD/AFSw/wDM/wAP8j1P9e8X/LH7n/mbviW8t7/WZ7q1bfE+3BwR0UDvX0b8B/ib4R0LTL7wp4z/AHNnebB5nzn7pduiAnqR3r5Vor6bCxVKn7Jaq1vkfG4uq61V1ZaNu/zZ96eHvEPwB+G2sJqmmXnn3Mm4iTbcDy+COhDA5DV5Ne/E3wmPjLc+KSBf6PPtyxDJnEOzpt3fe9q+ZSSetJW6qvmjLtf7mY8q5ZR72/A+65k/Z21LVB4qXUvJDcmMJc9QNvXj+VYsPxI0/wCIPxt0OTSY8Wln5wjfJ+bdBg8EAjBWvi/JxjNdb4G8VP4L8S2niKOD7Q1rvwm7bncpXrg+vpW+FrKNSLlpFf5EVYNwklu1Y+6viVJ8FfEfiiWx8YzC01Kx27jtmbfvRSPuYAwAK8g+MnxZ8KSeGY/AHgAbtOGd7jeMfMsg4kXPXPevnjx34vl8beJbrxFLD9ma52fJu3Y2qF64Hp6VxtcXNJw5Xp39TdpKfNF+h9cfCz4qeDtQ8In4efEcbLFPuS5c5y7SHiNcjBx3rrbfWfgH8Orae90iT+0b2YfIhFwnTg8tuHRq+GqUknqc1tUq8131fUyjGy5ei6H0f8J/jTF4R8S3smqxf8S3Uyvmc/cCKwHRSTkntXsMTfs36Tqp8XQXgdk5WPZcjkjaeT/hXwdS5OMZojUslpqlZPrYHG7fZ7nt3jv4pW/iP4hJ4mtLcLZ25Hlxg44KKp52g9s9K9c+LPjL4M+PtD/tiK+xrsYXZH5c3qqnnAX7or40orJ/w/Z+d79bs0T9/n/rQ+w/BPi/4R+JPBcPhXxmBZXibsSfvWJ+ct/yzAA4A71vXvjz4V/DPwpfaV4KuTqN5fBQwxMn3GyP9YGHRjXw9nHSlJJ61rVquV+l9/MzpwUbeWwrtvdn/vEn86bRRWRYUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFAH/1vhuiiivPOgKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKUAk4HNLsf8Aun8qVx2G0UdKKYgooooAKKKKACiiigAoop8aGSRYx1YgfnTjFt2QMZRXp/xA+F+rfD+30+51F966gHKcAfc256Mf71eYUurQdE+4UUUUAFFdh4H8G6h4516DQdPO2SbdzxxtUt3I9PWqHirw9ceFddudCum3S223J4H3lDdifWqlFq1+ok73t0OeoooqRhRRRQAUUDnivUrz4WaxY+BIvHc74tpc4XC9pPL67s9faq5XyuXT/MV1e3U8tor2pPgtrD/Dz/hP1nHl/wDPLC/89PL67vx6V4qRg4pTi4ycHuhx1ipLYKKKKQBRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAf/X+G6KKK886AooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigDpvB4B8Q2oIz97/ANBNe5+XH/dH5V4b4O/5GG1/4H/6Ca91r884sf8AtEfT9WfrPAi/2SX+J/kjhtW8EW2oXH2iCb7OT1G3dn9ayv8AhXX/AE/f+Q//AK9enUV5tLPsXCKjGei9P8j2K3C+BqTc5U9X5tfkz521jTv7J1GWw3+Z5ePmxjORnpXVfD/4fa18QtYGl6Snyj/WSfL8nBI4JGc4rJ8Y/wDIw3X/AAD/ANBFfZ/7MFpYQ+D9YvJH8hpBFvkwWIw0gHFfqWVt1KCqT1fKn67H4vm1NU8TOlT0XM0vJXZydx+ydCokgsvFC3F5GB+5+zhck8/e8zHSvGfCnwjuNZ8ayeB/EF9/Y98v3QYxLu+Qv1VgOgHfvX0LouhfCXRdei8QW/iom5iZj/x7z85BXoSR39K8t+O/jLTNS8c22t+E7kyOh++EZT9xF6MPY10tRi4uXXRr5bo5GnyyS3WzOA1v4ReIdH8cL4KAMssh/dyYUBgEDnjcQMA+tanjz4N3PhHW7Dw7p1//AGpqF5uzEIxHs2qrdSxB4P6V9++HLi81nwbZeM9T00f29EkmxPMGRlinUYX7oz0r5L+BN7d+Jfi3LrWvfvb4FuTxj9069BgdAKuND94qEt1dt90r2X4bkyn+7dWOzskvN9S9a/spBLeFtc8SLp9zMDiIwB+nuJMdMV4z8QvhRrXw01a1ivm8+2uG/dTYUbtu3PAZsctjmpPjR4h1vUviBfvfyMjRFNq8Db8i+mPSs/W/iZ4w8UaZp+ia3OZbe2YgExqudzA9QoPYd6nDzU3CcdHdaeRo48t4y10/E9//AGj7Rr+18J2inBkE4z6f6uvB/il8L5PhpcWdvJffbTdhj/q9m3aFP95s/er6F+PwJfwcB/03/wDadc9+1cjjUdFcj5SsmD/wGOniYpJyW7nJCg2+WP8AcT/E8o8I/CObxX4O1DxXFf8AlGxCHyfLzu3uV+9uGMYz0rU+FvwK1j4jRvfzXP8AZ2noeJyquDywPy71PVcV7J8GIZF+D+tyOuEk8jB9cTNmtTxvf3vhv4CWVtog8mKfzPMYYOMXAI65Pc0VoqnzSSvZRsvNmdK8lFX3cvuRr/Db4DnwX40s/EGk6uNTtIRJ5mIxHtzGyjq5PJPpXy38UNJv9d+K+oaZpsfm3ExjCrkDOIlJ5PHSu1/Zk1rVIPHsdhFKxt7sN5oPOdsbkdenNeneBbO1uP2h9QnmUM8OzafTNuc1rKkpzpJvS0jP2tlUstbr9TnbX9lAC1gbWPEq2F1MCfJNuHxj/aEmOleC/En4Y638N9TW01EebbzZ8mb5QHwAW4DNjG7HNfWfj7wz8MdZ8VXV5rviUwXgKZTyJvlwoxypA5AFcp8d/FvhDVfBFlpGj6j9vuLUONxieP7zof4gB0FclS3Ip7O+3kzrhG0nF66b+Z8WVqaNo9/r2pQ6Vpkfm3E5IVcgZwMnrgdBWXX0r+zBZ20/j6O4lTdJCG2HPTMb5rXDU1Kdnt/lqYVpuMbo7WH9lFIbCJ9V8RLa3swyIjAGxg+okx0rqvil4avfCHwHg0G/O6a33ZPHO64Vh0J7H1r5s+MviDWtT+IN9JfyMjRNHtXgY+RfTHpX0J4+1S/1j9nizvtSYvO4bJIA6XAA6ewqefnwzktNY6fPQrl5ayg97PX5Hj8XhfxO/wAGD4hXWSNMX/l08pe8237+c9eelWfAn7PWo+OvDNt4jtNR8pZ925DGDt2uV6lxnOK761/5Ngl/D/0qq9o2pXul/szy3NjJ5UgxyAD1uSO9dWIhGMq0mr25fxsOlFyjTS6to8J+K/wrs/hs9kltrI1U3W/OIvL2bNv+02c7v0rxupJJZJWJkYsfc5qOvPSNJNaWQUUUUyQooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooA/9D4booorzzoCiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKAN3w1eW9hrMF1dNsiTdk4J6qR2r1j/hMfD3/P1/443+FeFUV42Y5HSxM1Obd7W0Posp4lr4Om6VJJpu+t/wDPyPdf+Ex8Pf8AP1/443+FH/CY+Hv+fr/xxv8ACvCqK4P9UsP/ADP8P8j1P9e8X/LH7n/mbviW8t7/AFme6tW3xPtwcEdFA71698EPirbeAb+40/WU8zTNQ2iU5I2bA2OFUk5Ldq8Eor6fCL2MFCOqStr2PjcZVdapKrLdu+nfc+6o4P2bdJvz4kivPMVORFtuepG3qc9z6V5b/wAJh8OvFfxLGt6/KNM0W2PyR7ZJd+Y8dVAYYYD86+aMnGM0laxqcsoyj02MZR5k1LrufYGr/tFn/hZFtqGmP/xILTIC4xvDRBe6bhhqyPGfxE8EaF8Q7Hx58Pbr7W77/tEQR4wP3YjXmQe5PAr5WoojVa5bbpv8d0/IHFO99mrWPvXWdV/Z/wDiPcx+JdauxBfNzMmy4OSAFHK7R0XsK8o+M3xT8L+ILfTvDng6MLptkW3SDd8wJRujqG4IPevmIEjoaSlzJW5VZJ3t5lRvu99j6W+N/wAQ/C/i208Pr4du/tL2Hm+aNjpt3bMfeA9DXrUnjT4O/FXw5ZR+Mrj7JfWAYbdsz7dzeqBQchRXwfSgkdDir9s2pJrd39H5Cs7xaeyt8j71l+Kfwb0TwPqHhDwzd+QqhBD8kzeYd+8/eU4xz1Ncb8Kvix4P1Lwq/wAPviJ8lmCfLkO87su0h4jXIwcd6+O6M46UnVbcnPW6SfyFypW5dLO6+Z99eGvF3wH+Ft/5uh3Illus75ds42bQccMGzncRXzxe/EyDRfi3N430B/tdrlcdU3AxbD95SRjJ7V4aST1pKUaslKMuqv8Aj0E4JxcX1PvDVrz9n34gXK+JtUu/s17LzMu24OSoCjpgdB2FebfEnXNC8f8A2Xwn8LtP86KDcC+9huztbpKAR0PevlkEjoa6Pw14t1/whef2h4fufs0/97Yr9iOjAjoTUtQejVlvp3Kcpb7vz/rsdX/wpz4if9Ao/wDfyP8A+Krc8N2fjb4Pa7aeLNR08xwxFsjzE+YEFO27pu9Kp/8AC9/ij/0GP/IEX/xFc74l+JnjTxfaJY+IL/7TCmcL5aJ1IPVVB7CqVTlalDcTgpK0tj6417Uf2e/Hk0finWboQ3TjMq7bg5KgKPu4Hb0rlfiz8WvAHif4dt4d8OzeVMmAkO2Q/wDLRWPzMoHQE9a+NcnGM0lTJ3i4pWX/AAblK9+Z6s+l7f4g+FU+BMngxrv/AImxx+52P/z33/ext+770sHxB8Kp8CZPBjXmNWbGIdj/APPxv+9jb933r5norSrXc+e/2rX+Vv8AIdN8vLb7Lb+8KKKKxEFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQB//R/LX/AISvXf8An5/8dX/Cj/hK9d/5+f8Ax1f8K52ivrPqdL+RfcfLfWqn8z+86L/hK9d/5+f/AB1f8KP+Er13/n5/8dX/AArnaKPqdL+RfcH1qp/M/vOi/wCEr13/AJ+f/HV/wo/4SvXf+fn/AMdX/Cudoo+p0v5F9wfWqn8z+86L/hK9d/5+f/HV/wAKP+Er13/n5/8AHV/wrnaKPqdL+RfcH1qp/M/vOi/4SvXf+fn/AMdX/Cj/AISvXf8An5/8dX/Cudoo+p0v5F9wfWqn8z+86L/hK9d/5+f/AB1f8KP+Er13/n5/8dX/AArnaKPqdL+RfcH1qp/M/vOi/wCEr13/AJ+f/HV/wo/4SvXf+fn/AMdX/Cudoo+p0v5F9wfWqn8z+86L/hK9d/5+f/HV/wAKP+Er13/n5/8AHV/wrnaKPqdL+RfcH1qp/M/vOi/4SvXf+fn/AMdX/Cj/AISvXf8An5/8dX/Cudoo+p0v5F9wfWqn8z+86L/hK9d/5+f/AB1f8KP+Er13/n5/8dX/AArnaKPqdL+RfcH1qp/M/vOi/wCEr13/AJ+f/HV/wo/4SvXf+fn/AMdX/Cudoo+p0v5F9wfWqn8z+86L/hK9d/5+f/HV/wAKP+Er13/n5/8AHV/wrnaKPqdL+RfcH1qp/M/vOi/4SvXf+fn/AMdX/Cj/AISvXf8An5/8dX/Cudoo+p0v5F9wfWqn8z+86L/hK9d/5+f/AB1f8KP+Er13/n5/8dX/AArnaKPqdL+RfcH1qp/M/vOi/wCEr13/AJ+f/HV/wo/4SvXf+fn/AMdX/Cudoo+p0v5F9wfWqn8z+86L/hK9d/5+f/HV/wAKP+Er13/n5/8AHV/wrnaKPqdL+RfcH1qp/M/vOi/4SvXf+fn/AMdX/Cj/AISvXf8An5/8dX/CueAJOAMmn+VL/cP5UvqlH+VfchrEVf5mb3/CV67/AM/P/jq/4Uf8JXrv/Pz/AOOr/hXOkEcGij6pS/kX3C+tVP5n950X/CV67/z8/wDjq/4Uf8JXrv8Az8/+Or/hXO1798G/gJrHxftdTu7G7FomnBD91W3b93q64+7SnhaMYubirLfQccRVbUVJ3fmeR/8ACV67/wA/P/jq/wCFH/CV67/z8/8Ajq/4Vm6rptxpF/Np1yMSQnB6H+Waz6I4ajJJqKt6FTrVoycZSd15nRf8JXrv/Pz/AOOr/hR/wleu/wDPz/46v+Fc7RVfU6X8i+4j61U/mf3nRf8ACV67/wA/P/jq/wCFH/CV67/z8/8Ajq/4VgxRmWVIhwXIH517N8VPgprnwrstLvdVl8xNUEhThRjy9uejN/e9qieGoRtzRWrstOpdOrWldRk9Ndzzf/hK9d/5+f8Ax1f8KP8AhK9d/wCfn/x1f8K52ir+p0v5F9xH1qp/M/vOi/4SvXf+fn/x1f8ACj/hK9d/5+f/AB1f8K52u/8Aht8PtU+JXii28M6Udks+75uDjajP0JX+760LBUv5F9wfWqn8z+8wv+Er13/n5/8AHV/wo/4SvXf+fn/x1f8ACrfjnwldeB/E954YvX8yaz2bmwBneoboCfX1rkqinh6EoqUYqz8ip160ZOMpO68zov8AhK9d/wCfn/x1f8KP+Er13/n5/wDHV/wrnaKv6nS/kX3E/Wqn8z+86L/hK9d/5+f/AB1f8KP+Er13/n5/8dX/AArnaBzxR9TpfyL7g+tVP5n950X/AAleu/8APz/46v8AhR/wleu/8/P/AI6v+FelX/wS17TvhhF8TbmXbay5xHhe0vldQ2ep9K6GD9nTX5/hR/wtBLseV18jCf8APXyvvb8+/SsqlLDwUnKK912enU2pyrzcYxbvJXWu6PFP+Er13/n5/wDHV/wo/wCEr13/AJ+f/HV/wrniMEg9qStPqlL+RfcY/Wan8z+86L/hK9d/5+f/AB1f8KP+Er13/n5/8dX/AArnaKf1Ol/IvuD61U/mf3nRf8JXrv8Az8/+Or/hR/wleu/8/P8A46v+Fc7RR9TpfyL7g+tVP5n950X/AAleu/8APz/46v8AhR/wleu/8/P/AI6v+Fc7RR9TpfyL7g+tVP5n950X/CV67/z8/wDjq/4Uf8JXrv8Az8/+Or/hXO0UfU6X8i+4PrVT+Z/edF/wleu/8/P/AI6v+FH/AAleu/8APz/46v8AhXO0UfU6X8i+4PrVT+Z/edF/wleu/wDPz/46v+FH/CV67/z8/wDjq/4VztFH1Ol/IvuD61U/mf3nRf8ACV67/wA/P/jq/wCFH/CV67/z8/8Ajq/4VztFH1Ol/IvuD61U/mf3nRf8JXrv/Pz/AOOr/hR/wleu/wDPz/46v+Fc7RR9TpfyL7g+tVP5n950X/CV67/z8/8Ajq/4Uf8ACV67/wA/P/jq/wCFc7RR9TpfyL7g+tVP5n950X/CV67/AM/P/jq/4Uf8JXrv/Pz/AOOr/hXO0UfU6X8i+4PrVT+Z/edF/wAJXrv/AD8/+Or/AIUf8JXrv/Pz/wCOr/hXO0UfU6X8i+4PrVT+Z/edF/wleu/8/P8A46v+FH/CV67/AM/P/jq/4VztFH1Ol/IvuD61U/mf3nRf8JXrv/Pz/wCOr/hR/wAJXrv/AD8/+Or/AIVztFH1Ol/IvuD61U/mf3nRf8JXrv8Az8/+Or/hR/wleu/8/P8A46v+Fc7RR9TpfyL7g+tVP5n950X/AAleu/8APz/46v8AhR/wleu/8/P/AI6v+Fc7RR9TpfyL7g+tVP5n950X/CV67/z8/wDjq/4Uf8JXrv8Az8/+Or/hXO0UfU6X8i+4PrVT+Z/edF/wleu/8/P/AI6v+FH/AAleu/8APz/46v8AhXO0UfU6X8i+4PrVT+Z/edF/wleu/wDPz/46v+FH/CV67/z8/wDjq/4VztFH1Ol/IvuD61U/mf3n/9L8k6KKK+0PjwooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAr7Q8GfsxR/ET4QaP4x0K58nVZ/P8yPbu83bMUHLOFXCj0r4vr9l/2VmC/Arw6WOAPtPX/rvJXDmFaUIKUe53YClGcmpdj8lvFvgfxL4I1BtM8RWZtpl7ZVh0B6qSOhHeuRr9dvj58U/hPpWlz6D4lhXVrwgYtgZI8nKt/rEUjgEHr2xX5K381rPdyS2cP2eFj8qbi2PxNaYWvKpG8lYyxVBQlZO5TooorqOY7r4bgN4ysAwyP3n/oDV9XGGEjGxfyr5S+Gv/I5WH/bT/0Bq+sa/IePm/rkP8K/Nn7d4br/AGGf+J/kjynxF8LbLWbz7ZZ3H2Mt94bS+eAB1Yelc/8A8KWP/QV/8g//AGVe7UV41DirH04KEami8k/zR7mI4Py2rN1J0tX5tfkz4w8S6L/wj2s3Gk+b5/kbfnxtzuUHpk+tfRH7N/g/xf4sj15fC+vnRBbLEZR5CTeZuD4+8RjGD+deK/Er/kcr/wD7Z/8AoC19ffsSdPFv+5b/APoMtfs2FrynlzrSfvcl/nZfI/DcbhoU8wlRgrRU2l6XseHfCT4Faj8YtQ1m2t9S+zXGmtGP9WH8wybvV1Axtr3mz/YbSdBaz+MVi1TnNr9lDY7j5xLjkc1ofscyyQav40liO1l8jB/CavmHwN4k1mX4xWuqy3LNcmWQFu3+rK9OnStqinKsqMHb3Yv5smvShCE6sle0mtzC8UfCDxZ4a8eN8P2tzNqBICDKjd8gkP8AEQMA+tfVNl+w6EtLdvEXjBdLu7gEiE2ok6f7Sy46Yr6gv9Osrr9onTb2eMNNCkhQ57m1Ga/OL9ozxX4j1f4qamdTlZDbtH5aDA25jT+6BnpXHh8VUn7OF9Wm2/R20HXwsKbnO2iskvNq5B8WPgZ4i+D2s2Ueov8AarO6f9zOAqh9u3PyhmIwWxzX0v8Atc2L6np/gixRtplW5APXH+qr5Z8SfGXx/wCNND0zw34iuGmtbRiATGi7tzq3UIDxgd6+uP2ogS3gHH/Tz/7TrtdOf7iFV3/ePXurBTnT/eypK37t/efJ/wAa/gtL8HbnTbaXU/7ROoBz/qvL27Ah/vNn71afgH4CT+OvAOqeN4dV+znThGfI8rdv3uU+9vGMYz0r3D9uKKT+0vDku35Nk3P/AACKuv8A2dLeVPgN4glkXEcot9p9cTuDXJ9Zm8JOrf3lf/0q35EQwsfrNOm1o7f+k3/M+d/gr+zNr/xagl1S4vP7I0tD8twUWUNyyn5d6kfMuPxr67+EX7Mn/CuviHY+KtD15dYsoBL5qiERbMxMg6yMTkk9u1YnxG1PUPBv7MFja+Gh9nin83zGXDYxdAjrk85NeEfsa+ItatvidHpcU7G1vw/nKfmzsilK8npz6V0KpVqTqqDso3Xrprf5bGEo06dODqK7lZ+muh598a9C1TxN8dNW0XR4fPu7hoQiZAziBSeTgdBXvdl+w2q2Vs+v+MV0y9uAT5BtRJjH+0JcdMGuv+HGnWV3+1hq9zcqGktzHsJ7brRs8V0XxQ8G/BrxD41vb/xN4taDUEMZaP7PP8mEXHKMAcgCuGnXcKdCnF2vFNvd9tEdssOp1K1SXSVl/wAE+EfjB8F/Efwg1dLLVf39pcZ8i4G0CTaFLfKrMRgtjmvG6/Rn9pvxz4E1r4a2Og6Fqv8AaV3ZhgGMMked0iH+IAdB61+c1d2CrSnFqXR2v38zjx9KEZRcNLrbezNvw94f1TxRq9voejQ+fd3JIRcgZwCTySB0Ffdtv+w7Fb6ZE+teLFstQuBlYTbBipB6bhLjkYrzr9jDTrO6+J8d3PGHltgxjJPTdDLnivPv2iPFfiLV/itqR1KZkNu0WxBgbcxp/dxnOKrE1Je1hRg7XTbfle2hGGpx9nKrNXSaVvlfU+vvjV4P1DwH+zFD4X1J/MntM5bgZ33SsOASOh9a+crXwX40l/Z4bxVH4iK6Ov8Ay4eQn/Pzs/1ud33vm6e1e8/FDWtT1/8AZPstS1dzJcyBgzEAZ23agdAB0Fc/pv8AyZjN+H/paa5p88addyd5e0j+ieh6GEjCdSgo6Jxf6nlnwx/ZS1T4meDbPxdY6wIFut+6MxKdmx2T7xkXOduelcV8cfghY/CBtPW08QLrbXvmbgIPK2bNv+22c7v0r6b8NaxqOi/sdzXmmTGGYcBgAet4R39jX5yyzTTMWlcuck8nPWumpzvE1IRdoxf36HLyQhQjOUbuV/lZ7kVFFFdZ5wUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFABRRRQAUUUUAFFFFAH//T/JOiiivtD48KKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAK9zs/j34x0j4d6f8AD3Q5PsdtZ+aJHwj+Z5knmDhkyMEnvzXhlFROCl8SLhNx2ZNPPNcyGWdy7tySTmoaKKsgKKKKAOs8EalZaR4mtNQ1CTyoIt+5sE4ypA4GT1r6F/4WX4N/5/8A/wAhv/8AE18nUV85nHDGHxtRVaraaVtLf5M+qyPi7E4Ck6NGMWm7638uzXY+sf8AhZfg3/n/AP8AyG//AMTR/wALL8G/8/8A/wCQ3/8Aia+TqK8r/UDB/wA0vvX+R7P/ABErHfyR+5/5nWeN9SstX8TXeoafJ5sEuza2CM4UA8HB619G/sufE/wX8Oh4hHi6/wDsX25YRD+7d9xQSZ+4px94da+RqK+vo4aMMP8AVl8PLy+dj4ivjZVMQ8TJat83le9z7I/Zw+K3gbwFf+KJvFGofY01HyvIPlyPu2iTP3FOPvDrXzx4S1zTNM8eW+tXkvl2aSOxfBOAVIHAGf0rz6itowSqqr1sl9xNbFSnCVN7Nt/Nn3V8VP2iPD8Pxa0jx14Du/7Tt7EOHG1os74Vj/5aJnjntXrXiDW/2Wvi/dxeLvEN6LfUm5nTZdHJUBV5UKOi9hX5d0oZh0OK5fqEOWKW8b2fXXf5FvHSc5SaWtrr02PtD9oX43eCvFNjpfhH4fxKNJsWbfKAwLZZH6SIG4IPeqn7SHxX8E+OtP8AC8fhO/8AtkumCbzh5cibdxjxy6jOdp6V8c0VpTwqiopN3Uua/W9rajePlduy1XL8j9PH+IfwB+N3hHToPiBdCy1HS1cbdtw+ze3rGFByEH0qxJ8a/wBnvwz8O9T8B+D7/wCzIgjEA8u4bzSZN7ffU4wSepr8vASOhxSVnWwMZ80btKWtl3KpZhKPI7Xcdmz9Afgf8c/AWr+CpfhX8VvksAT5Up8w78yPKeIlyMHb3r0fwj46/Zi+Cmpmbw3diWe9z5kuy6Hl7QccMGzncRxX5bgkcilJJ6nNVVwkZTc07X38/wCuplTxbjBQte23kfS+p/GW18PfHi5+I/hZ/t1iWXbwY94MHln76kjBJ7V9Sa5f/ssfFW8TxjrV99l1GYZuE2XbZKAKvK7R0XsK/MOlDMOhIoeDjyQinrFWT62HHGyUpy6S3R9v/F7xN4a+Kcdn4F+CumfaYLbeGk8xl3Z2uOJwp/hbvXg//DPnxa/6An/keH/4uuH8HePPFHgHUP7U8K3n2O5P8WxH7EdHBHQmvTv+GnvjX/0Hx/4DQf8AxFKGHlTVqet9Xfe4SrxqO9TS21uxf8IaZ8SPgD4jsvHWraV5VtAW3L5sZ3BlMfbeeN/pX1v4m1b9lb4oTxeNvEN6Ib1xmZdl0clQEH3do/h7Cvgzxh8ZviL4909NL8U6p9stkzhfJiTqQeqKD1Ary/c2MZ4pTw86iTqaSWzXYccRGndU9U90z9Cvjp8dfhZ4y+E0vhLwnceVcx7RHbhJT/y2Vj8zoB0BPWvO7H4q+B4f2ZpPh9Lf411sYg8uT/n68z723b93nrXxxRVrCRVOVO7tJpv1Vv8AIccfNVI1El7qsvx/zPsiy+K3gaL9meX4fSX+NdbGLfy5Of8ASvM+/t2/d5618bmiit1TXtJ1Osnd/dYwniHKnGm9o3/EKKKKswCiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooA//U/Pn/AIRTQv8An2/8eb/Gj/hFNC/59v8Ax5v8a6KisPrlX+d/eP6rT/lX3HO/8IpoX/Pt/wCPN/jR/wAIpoX/AD7f+PN/jXRUUfXKv87+8PqtP+Vfcc7/AMIpoX/Pt/483+NH/CKaF/z7f+PN/jXRUUfXKv8AO/vD6rT/AJV9xzv/AAimhf8APt/483+NH/CKaF/z7f8Ajzf410VFH1yr/O/vD6rT/lX3HO/8IpoX/Pt/483+NH/CKaF/z7f+PN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/AI83+NH/AAimhf8APt/483+NdFRR9cq/zv7w+q0/5V9xzv8Awimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/wA7+8PqtP8AlX3HO/8ACKaF/wA+3/jzf40f8IpoX/Pt/wCPN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/zv7w+q0/5V9xzv/CKaF/z7f8Ajzf40f8ACKaF/wA+3/jzf410VFH1yr/O/vD6rT/lX3HO/wDCKaF/z7f+PN/jR/wimhf8+3/jzf410VFH1yr/ADv7w+q0/wCVfcc7/wAIpoX/AD7f+PN/jR/wimhf8+3/AI83+NdFRR9cq/zv7w+q0/5V9xzv/CKaF/z7f+PN/jR/wimhf8+3/jzf410saGSRYx1YgfnXpvxA+F+r/D620+51J9y6gHKcAfc256Mf71N4qslzObttuCw1JvlUVf0PDf8AhFNC/wCfb/x5v8aP+EU0L/n2/wDHm/xroq9O+Hfwx1b4ii+/st9v2EKW4B+9uPdh/doWKrNN8z013E8PSW8V9x4f/wAIpoX/AD7f+PN/jR/wimhf8+3/AI83+NdPPE0Erwt1Q4qKpWMq/wA7+8p4SmnZxX3HO/8ACKaF/wA+3/jzf40f8IpoX/Pt/wCPN/jXRUU/rlX+d/eL6rT/AJV9xzv/AAimhf8APt/483+NH/CKaF/z7f8Ajzf410VFH1yr/O/vD6rT/lX3HO/8IpoX/Pt/483+NH/CKaF/z7f+PN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/AI83+NH/AAimhf8APt/483+NdFRR9cq/zv7w+q0/5V9xzv8Awimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/wA7+8PqtP8AlX3HO/8ACKaF/wA+3/jzf40f8IpoX/Pt/wCPN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/zv7w+q0/5V9xzv/CKaF/z7f8Ajzf40f8ACKaF/wA+3/jzf410VFH1yr/O/vD6rT/lX3HO/wDCKaF/z7f+PN/jR/wimhf8+3/jzf410VFH1yr/ADv7w+q0/wCVfcc7/wAIpoX/AD7f+PN/jR/wimhf8+3/AI83+NdFRR9cq/zv7w+q0/5V9xzv/CKaF/z7f+PN/jR/wimhf8+3/jzf410VFH1yr/O/vD6rT/lX3HO/8IpoX/Pt/wCPN/jR/wAIpoX/AD7f+PN/jXRUUfXKv87+8PqtP+Vfcc7/AMIpoX/Pt/483+NH/CKaF/z7f+PN/jXRUUfXKv8AO/vD6rT/AJV9xzv/AAimhf8APt/483+NH/CKaF/z7f8Ajzf410VFH1yr/O/vD6rT/lX3HO/8IpoX/Pt/483+NH/CKaF/z7f+PN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/AI83+NH/AAimhf8APt/483+NdFRR9cq/zv7w+q0/5V9xzv8Awimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/wA7+8PqtP8AlX3HO/8ACKaF/wA+3/jzf40f8IpoX/Pt/wCPN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/zv7w+q0/5V9xzv/CKaF/z7f8Ajzf40f8ACKaF/wA+3/jzf410VFH1yr/O/vD6rT/lX3HO/wDCKaF/z7f+PN/jR/wimhf8+3/jzf410VFH1yr/ADv7w+q0/wCVfcc7/wAIpoX/AD7f+PN/jR/wimhf8+3/AI83+NdFRR9cq/zv7w+q0/5V9xzv/CKaF/z7f+PN/jR/wimhf8+3/jzf410VFH1yr/O/vD6rT/lX3HO/8IpoX/Pt/wCPN/jR/wAIpoX/AD7f+PN/jXRUUfXKv87+8PqtP+Vfcc7/AMIpoX/Pt/483+NH/CKaF/z7f+PN/jXRUUfXKv8AO/vD6rT/AJV9xzv/AAimhf8APt/483+NH/CKaF/z7f8Ajzf410VFH1yr/O/vD6rT/lX3HO/8IpoX/Pt/483+NH/CKaF/z7f+PN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/AI83+NH/AAimhf8APt/483+NdFRR9cq/zv7w+q0/5V9xzv8Awimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/wA7+8PqtP8AlX3HO/8ACKaF/wA+3/jzf40f8IpoX/Pt/wCPN/jXRUUfXKv87+8PqtP+Vfcc7/wimhf8+3/jzf40f8IpoX/Pt/483+NdFRR9cq/zv7w+q0/5V9xzv/CKaF/z7f8Ajzf40f8ACKaF/wA+3/jzf410VFH1yr/O/vD6rT/lX3H/2Q==