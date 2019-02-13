# argus-apm-aop源码分析

argus-apm-aop主要实现了面向Activity的切面和面向HttpClient和URLConnection的切片。阅读代码需要有AOP和aspectj的基础知识。

- TraceActivity
TraceActivity类实现对Activity（还有Application）的切面。
```
@Aspect
public class TraceActivity {

    @Pointcut("!within(com.argusapm.android.aop.*) && !within(com.argusapm.android.core.job.activity.*)")
    public void baseCondition() {
    }

    @Pointcut("execution(* android.app.Application.onCreate(android.content.Context)) && args(context)")
    public void applicationOnCreate(Context context) {

    }

    @After("applicationOnCreate(context)")
    public void applicationOnCreateAdvice(Context context) {
        AH.applicationOnCreate(context);
    }

    @Pointcut("execution(* android.app.Application.attachBaseContext(android.content.Context)) && args(context)")
    public void applicationAttachBaseContext(Context context) {
    }

    @Before("applicationAttachBaseContext(context)")
    public void applicationAttachBaseContextAdvice(Context context) {
        AH.applicationAttachBaseContext(context);
    }

    @Pointcut("execution(* android.app.Activity.on**(..)) && baseCondition()")
    public void activityOnXXX() {
    }

    @Around("activityOnXXX()")
    public Object activityOnXXXAdvice(ProceedingJoinPoint proceedingJoinPoint) {
        Object result = null;
        try {
            Activity activity = (Activity) proceedingJoinPoint.getTarget();
            //        Log.d("AJAOP", "Aop Info" + activity.getClass().getCanonicalName() +
            //                "\r\nkind : " + thisJoinPoint.getKind() +
            //                "\r\nargs : " + thisJoinPoint.getArgs() +
            //                "\r\nClass : " + thisJoinPoint.getClass() +
            //                "\r\nsign : " + thisJoinPoint.getSignature() +
            //                "\r\nsource : " + thisJoinPoint.getSourceLocation() +
            //                "\r\nthis : " + thisJoinPoint.getThis()
            //        );
            long startTime = System.currentTimeMillis();
            result = proceedingJoinPoint.proceed();
            String activityName = activity.getClass().getCanonicalName();

            Signature signature = proceedingJoinPoint.getSignature();
            String sign = "";
            String methodName = "";
            if (signature != null) {
                sign = signature.toString();
                methodName = signature.getName();
            }

            if (!TextUtils.isEmpty(activityName) && !TextUtils.isEmpty(sign) && sign.contains(activityName)) {
                invoke(activity, startTime, methodName, sign);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        return result;
    }

    public void invoke(Activity activity, long startTime, String methodName, String sign) {
        AH.invoke(activity, startTime, methodName, sign);
    }
}
```
@Aspect注解表明这是一个aspectj的切面类。
第一个PointCut要排除com.argusapm.android.aop和com.argusapm.android.core.job.activity下的所有切点。第二个PointCut指定android.app.Application的onCreate方法作为切点，并且要获取方法的参数。applicationOnCreateAdvice方法上有After注解，切点事前面定义的applicationOnCreate，就是在android.app.Application.onCreate执行的最后，织入代码：AH.applicationOnCreate(context)。该方法实际上只打印了一行log。applicationAttachBaseContextAdvice方法上有Before注解，它的切点在android.app.Application.attachBaseContext方法上，所以，在这个方法运行的开始，要调用AH.applicationAttachBaseContext(context)。这个方法获取当前的系统时间，保存在ActivityCore的appAttachTime属性中，也打印一行log。
activityOnXXXAdvice方法上有Around注解，它的切点在android.app.Activity的以on开头的系列回调方法上，并且要满足前面定义的baseCondition，也即不在指定的那两个包下的类。因为是Around注解，所以会在切点前后都进行织入操作。通过proceedingJoinPoint.getTarget()获取target Activity的对象activity，获取当前的系统时间startTime。然后执行切点的原始代码。获取Activity的canonical name和切点函数的签名，进一步得到函数的名字。如果activity的canonical name和函数的签名都不是空字符串，并且函数签名的字符串里包含activity的canonical name，调用AH.invoke(activity, startTime, methodName, sign)。
AH.invoke(activity, startTime, methodName, sign)函数的代码如下：
```
public static void invoke(Activity activity, long startTime, String lifeCycle, Object... extars) {
    boolean isRunning = isActivityTaskRunning();
    if (Env.DEBUG) {
        LogX.d(Env.TAG, SUB_TAG, lifeCycle + " isRunning : " + isRunning);
    }
    if (!isRunning) {
        return;
    }

    if (TextUtils.equals(lifeCycle, ActivityInfo.TYPE_STR_ONCREATE)) {
        ActivityCore.onCreateInfo(activity, startTime);
    } else {
        int lc = ActivityInfo.ofLifeCycleString(lifeCycle);
        if (lc <= ActivityInfo.TYPE_UNKNOWN || lc > ActivityInfo.TYPE_DESTROY) {
            return;
        }
        ActivityCore.saveActivityInfo(activity, ActivityInfo.HOT_START, System.currentTimeMillis() - startTime, lc);
    }
}
```
首先要判断Activity task有没有运行。Activity task运行的标准是：ApmTask.FLAG_COLLECT_ACTIVITY_AOP标记被设置，TaskManager列表中有ApmTask.TASK_ACTIVITY类型的task，并且该task的mIsCanWork的标记为true。如果Activity task没有运行，直接返回。如果第三个参数lifeCycle是“onCreate"，也就是现在执行的是Activity的onCreate回调函数，调用ActivityCore.onCreateInfo(activity, startTime)；对其它生命周期，需要从参数lifeCycle转换出对应的life cycle的整型常量，获取到的值必须在合理范围内（大于ActivityInfo.TYPE_UNKNOWN，小于等于ActivityInfo.TYPE_DESTROY）。对于处在合理生命周期的方法回调，调用ActivityCore.saveActivityInfo(activity, ActivityInfo.HOT_START, System.currentTimeMillis() - startTime, lc）。
以下是ActivityCore.onCreateInfo的代码：
```
public static void onCreateInfo(Activity activity, long startTime) {
    startType = isFirst ? ActivityInfo.COLD_START : ActivityInfo.HOT_START;
    activity.getWindow().getDecorView().post(new FirstFrameRunnable(activity, startType, startTime));
    //onCreate 时间
    long curTime = System.currentTimeMillis();
    saveActivityInfo(activity, startType, curTime - startTime, ActivityInfo.TYPE_CREATE);
}
```
首先判断是冷启动还是热启动。向Activity的DecorView发送一个新的FirstFrameRunnable。记录当前时间curTime，这个就是onCreate的时间，调用saveActivityInfo(activity, startType, curTime - startTime, ActivityInfo.TYPE_CREATE)，保存onCreate的相关信息。
以下是saveActivityInfo的代码：
```
public static void saveActivityInfo(Activity activity, int startType, long time, int lifeCycle) {
    if (activity == null) {
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "saveActivityInfo activity == null");
        }
        return;
    }
    if (time < ArgusApmConfigManager.getInstance().getArgusApmConfigData().funcControl.activityLifecycleMinTime) {
        return;
    }
    String pluginName = ExtraInfoHelper.getPluginName(activity);
    String activityName = activity.getClass().getCanonicalName();
    activityInfo.resetData();
    activityInfo.activityName = activityName;
    activityInfo.startType = startType;
    activityInfo.time = time;
    activityInfo.lifeCycle = lifeCycle;
    activityInfo.pluginName = pluginName;
    activityInfo.pluginVer = ExtraInfoHelper.getPluginVersion(pluginName);
    if (DEBUG) {
        LogX.d(TAG, SUB_TAG, "apmins saveActivityInfo activity:" + activity.getClass().getCanonicalName() + " | lifecycle : " + activityInfo.getLifeCycleString() + " | time : " + time);
    }
    ITask task = Manager.getInstance().getTaskManager().getTask(ApmTask.TASK_ACTIVITY);
    boolean result = false;
    if (task != null) {
        result = task.save(activityInfo);
    } else {
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "saveActivityInfo task == null");
        }
    }
    if (DEBUG) {
        LogX.d(TAG, SUB_TAG, "activity info:" + activityInfo.toString());
    }
    if (AnalyzeManager.getInstance().isDebugMode()) {
        AnalyzeManager.getInstance().getActivityTask().parse(activityInfo);
    }
    if (Env.DEBUG) {
        LogX.d(TAG, SUB_TAG, "saveActivityInfo result:" + result);
    }
}
```
第一个参数是当前的Activity，第二个参数是启动类型（冷启动还是热启动），第三个是生命周期的耗时，第四个是当前所处的生命周期。time的值如果小于Activity生命周期最小数据收集时间间隔单位，就不做记录，忽略该条数据。获取插件的名称pluginName和activity的名字activityName。重置activityInfo的数据，然后依次设置activityInfo的activityName，startType，time，lifecycle等属性的值。获取ApmTask.TASK_ACTIVITY的task，如果task不为null，调用task.save(activityInfo)保存activityInfo中的信息。如果当前以debug模式运行，需要通过ActivityParseTask的parse方法，解析activityInfo的内容并且通过广播发送到悬浮窗。
ITask是个接口，它有一个直接实现类，BaseTask。BaseTask中的save方法，最终需要通过调用IStorage的save方法，实现对传入参数IInfo的保存。这里保存的机会是ActivityInfo类型的数据。ActivityInfo扩展自BaseInfo，当然，BaseInfo是接口IInfo的具体实现。IStorage本身也是一个接口，它有一个直接实现类TableStorage，这是一个抽象类，ActivityStroage扩展了这个类。不过ActivityStorage并没有复写save方法，所以我们可以直接看TableStorage中save的代码：
```
@Override
public boolean save(IInfo value) {
    ContentValues values = value.toContentValues();
    if (!values.containsKey(BaseInfo.KEY_TIME_RECORD)) {
        values.put(BaseInfo.KEY_TIME_RECORD, System.currentTimeMillis());
    }
    try {
        return null != Manager.getInstance().getConfig().appContext.getContentResolver().insert(getTableUri(), values);
    } catch (Exception e) {
        if (Env.DEBUG) {
            LogX.d(Env.TAG, "save ex : " + Log.getStackTraceString(e));
        }
    }
    return false;
}
```
首先需要把参数value转换成ContentValues类型，这里实际调用的是ActivityInfo类中该方法的实现。如果values中不包含BaseInfo.KEY_TIME_RECORD这个key，就把当前的系统时间保存为BaseInfo.KEY_TIME_RECORD。向应用的ContentResolver中指定的Uri插入values。这里的Uri是：content://{package name}.apm.storage/activity。通过ApmProvider类中的insert方法可以看到，这里会调用saveDataToDB(new DbCache.InfoHolder(values, table.getTableName()))把数据写入数据库。数据存储和读取等操作我们后面会详细分析。

- TraceNetTrafficMonitor
代码这里不贴了。
baseCondition定义切点，要排除以下的包和类：com.argusapm.android.aop， com.argusapm.android， com.argusapm.android.core.job.net.i，com.argusapm.android.core.job.net.impl， com.qihoo360.mobilesafe.mms.transaction.MmsHttpClient，com.qihoo360.mobilesafe.mms.transaction.MmsHttpClient及其子类。
httpClientExecuteOne定义切点:org.apache.http.HttpResponse org.apache.http.client.HttpClient.execute(org.apache.http.client.methods.HttpUriRequest)的调用点，第一参数是HttpClient及其子类，第二参数是HttpUriRequest类型；同时要符合baseCondition的切点要求。
httpClientExecuteOneAdvice上的注解是Around，在符合条件的切点上，调用QHC.execute(httpClient, request)。
QHC中的execute是一组函数，它们的实现大同小异。比如这里用到的代码如下：
```
public static HttpResponse execute(HttpClient client, HttpUriRequest request) throws IOException {
return isTaskRunning()
        ? AopHttpClient.execute(client, request)
        : client.execute(request);
}
```
首先要确认ApmTask.TASK_NET有没有在运行，如果没有运行，直接调用client的execute；如果在运行，调用AopHttpClient.execute(client, request)，它的代码是这样的：
```
public static HttpResponse execute(HttpClient httpClient, HttpUriRequest request) throws IOException {
    NetInfo data = new NetInfo();
    HttpResponse response = httpClient.execute(handleRequest(request, data));
    handleResponse(response, data);
    return response;
}
```
方法的核心在handleRequest和handleResponse。首先看一下handleRequest的代码：
```
private static HttpUriRequest handleRequest(HttpUriRequest request, NetInfo data) {
    data.setURL(request.getURI().toString());
    if (request instanceof HttpEntityEnclosingRequest) {
        HttpEntityEnclosingRequest entityRequest = (HttpEntityEnclosingRequest) request;
        if (entityRequest.getEntity() != null) {
            entityRequest.setEntity(new AopHttpRequestEntity(entityRequest.getEntity(), data));
        }
        return (HttpUriRequest) entityRequest;
    }
    return request;
}
```
从request获取URI，来设置data中URL。如果request的类型不是HttpEntityEnclosingRequest，直接就把request返回。如果是HttpEntityEnclosingRequest，先转换为HttpEntityEnclosingRequest类型，转换后的结果保存在HttpEntityEnclosingRequest，通过原始的Entity和data生成一个新的AopHttpRequestEntity，作为entityRequest的新的entity。对于handleRequest方法，这个data还只是一个新生成的空的NetInfo对象。通过handleRequest方法得到一个新的HttpUriRequest以后，httpClient.execute执行这个请求，这一方面会得到请求响应response，接下来handleResponse(response, data)对响应数据处理。
```
private static HttpResponse handleResponse(HttpResponse response, NetInfo data) {
    data.setStatusCode(response.getStatusLine().getStatusCode());
    Header[] headers = response.getHeaders("Content-Length");
    if ((headers != null) && (headers.length > 0)) {
        try {
            long l = Long.parseLong(headers[0].getValue());
            data.setReceivedBytes(l);
            if (DEBUG) {
                LogX.d(TAG, SUB_TAG, "-handleResponse--end--1");
            }
            data.end();
        } catch (NumberFormatException e) {
            if (Env.DEBUG) {
                LogX.d(TAG, SUB_TAG, "NumberFormatException ex : " + e.getMessage());
            }
        }
    } else if (response.getEntity() != null) {
        response.setEntity(new AopHttpResponseEntity(response.getEntity(), data));
    } else {
        data.setReceivedBytes(0);
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "----handleResponse--end--2");
        }
        data.end();
    }
    if (DEBUG) {
        LogX.d(TAG, SUB_TAG, "execute:" + data.toString());
    }
    return response;
}
```
代码比较简单明显。从返回的信息中解析返回码设置到data中。如果首部非空并且长度大于0，从首部中解析接收字节数，设置到data中；或者，response的entity不空，通过response的entity和data生成一个新的AopHttpResponseEntity，作为response的新的entity；如果以上两个条件都不满足，就把data中的接收字节数设置为0。
此时，我们会比较好奇，data中的数据，到底是在什么时候设置的呢？实际上，AopHttpResponseEntity扩展自AopHttpEntity，而AopHttpEntity实现了HttpEntity, IStreamCompleteListener两个接口，通过实现接口中的writeTo，onOutputstreamComplete，onOutputstreamError，onInputstreamComplete，onInputstreamError，会写入接收字节数。同样的，AopHttpRequestEntity也继承自AopHttpEntity，重载的以上几个方法会写入发送字节数到data中。

回到TraceNetTrafficMonitor类，后面的方法和前面类似，只是针对不同函数调用设置切点。