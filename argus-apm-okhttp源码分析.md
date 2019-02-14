# argus-apm-okhttp源码分析

- OkHttp3Aspect类
OkHttp3Aspect类是OKHTTP3的切面文件。
```
@Aspect
public class OkHttp3Aspect {

    @Pointcut("call(public okhttp3.OkHttpClient build())")
    public void build() {

    }

    @Around("build()")
    public Object aroundBuild(ProceedingJoinPoint joinPoint) throws Throwable {
        Object target = joinPoint.getTarget();

        if (target instanceof OkHttpClient.Builder && Client.isTaskRunning(ApmTask.TASK_NET)) {
            OkHttpClient.Builder builder = (OkHttpClient.Builder) target;
            builder.addInterceptor(new NetWorkInterceptor());
        }

        return joinPoint.proceed();
    }
}
```
相对于TraceNetTrafficMonitor，OkHttp3Aspect的代码明显要简单很多。这是否从另一方面印证了框架的强大？要对它做些手脚都这么方便。后边还会看到，对OKHttp还能收集到比HttpClient和HttpURLconnection更多的数据。

build（）定义了切点，很简单，就是okhttp3.OkHttpClient的build方法被调用的点。
aroundBuild上的注解是Around，意味着要在切点前后都织入代码。首先获得切点的target，如果target是OkHttpClient.Builde类型，并且ApmTask.TASK_NET类型的任务正在运行，要给target添加一个NetWorkInterceptor的拦截器。当然，不要忘了joinPoint.proceed()执行原始代码。发现只有切点前需要织入代码，貌似应该用before就够了呀？但是我们这里需要ProceedingJoinPoint参数来获取target。
NetWorkInterceptor类都做了些什么事呢？
```
public class NetWorkInterceptor implements Interceptor {

    private static final String TAG = Env.TAG;

    private OkHttpData mOkHttpData;

    public NetWorkInterceptor() {
    }

    @Override
    public Response intercept(Chain chain) throws IOException {

        long startNs = System.currentTimeMillis();

        mOkHttpData = new OkHttpData();
        mOkHttpData.startTime = startNs;

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp request 开始时间：" + mOkHttpData.startTime);
        }

        Request request = chain.request();

        recordRequest(request);

        Response response;

        try {
            response = chain.proceed(request);
        } catch (IOException e) {
            if (Env.DEBUG) {
                e.printStackTrace();
                Log.e(TAG, "HTTP FAILED: " + e);
            }
            throw e;
        }

        mOkHttpData.costTime = System.currentTimeMillis() - startNs;

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp chain.proceed 耗时：" + mOkHttpData.costTime);
        }

        recordResponse(response);

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp chain.proceed end.");
        }

        DataRecordUtils.recordUrlRequest(mOkHttpData);
        return response;
    }

    /**
     * request
     */
    private void recordRequest(Request request) {
        if (request == null || request.url() == null || TextUtils.isEmpty(request.url().toString())) {
            return;
        }

        mOkHttpData.url = request.url().toString();

        RequestBody requestBody = request.body();
        if (requestBody == null) {
            mOkHttpData.requestSize = request.url().toString().getBytes().length;
            if (Env.DEBUG) {
                Log.d(TAG, "okhttp request 上行数据，大小：" + mOkHttpData.requestSize);
            }
            return;
        }

        long contentLength = 0;
        try {
            contentLength = requestBody.contentLength();
        } catch (IOException e) {
            e.printStackTrace();
        }

        if (contentLength > 0) {
            mOkHttpData.requestSize = contentLength;
        } else {
            mOkHttpData.requestSize = request.url().toString().getBytes().length;
        }
    }

    /**
     * 设置 code responseSize
     */
    private void recordResponse(Response response) {
        if (response == null) {
            return;
        }

        mOkHttpData.code = response.code();

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp chain.proceed 状态码：" + mOkHttpData.code);
        }

        if (!response.isSuccessful()) {
            return;
        }

        ResponseBody responseBody = response.body();
        if (responseBody == null) {
            return;
        }

        long contentLength = responseBody.contentLength();

        if (contentLength > 0) {
            if (Env.DEBUG) {
                Log.d(TAG, "直接通过responseBody取到contentLength:" + contentLength);
            }
        } else {
            BufferedSource source = responseBody.source();
            if (source != null) {
                try {
                    source.request(Long.MAX_VALUE);
                } catch (IOException e) {
                    e.printStackTrace();
                }

                Buffer buffer = source.buffer();
                contentLength = buffer.size();

                if (Env.DEBUG) {
                    Log.d(TAG, "通过responseBody.source()才取到contentLength:" + contentLength);
                }
            }
        }

        mOkHttpData.responseSize = contentLength;

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp 接收字节数：" + mOkHttpData.responseSize);
        }
    }
}
```
我们看一下拦截器的intercept方法。首先记录一下startNs，mOkHttpData是OkHttpData的一个实例，这个类里会记录请求的url，请求体积，响应体积，请求开始时间，请求消耗时间，以及http返回码。mOkHttpData的startTime就是startNs。通过recordRequest方法对原始的请求做处理之后，发送请求，返回结果保存到response。mOkHttpData的costTime就是此时的时间减去startNs。通过recordResponse处理response。最后，DataRecordUtils.recordUrlRequest(mOkHttpData)保存所有请求相关的数据。
recordRequest函数首先从request中获取请求的url，保存到mOkHttpData的url中。从request中获取requestBody，如果requestBody为空，就把请求的url的length作为requestBody的requestSize，然后直接返回。如果requestBody不为空，获取rqeustBody的contentLength，如果conetentLength大于0，就把contentLength作为mOkHttpData的requestSize，否则，还是把请求url的长度作为mOkHttpData的requestSzie的值。
recordResponse方法首先获取response的code，保存到mOkHttpData的code中。如果请求不成功（code不是2xx），到这里就直接返回了。如果请求成功，获取响应的reqsponseBody，如果responseBody为空，就直接返回；如果不为空，获取responseBody的contentLength，如果contentLength大于0的话，contentLength的长度作为mOkHttpData中responseSize的值。如果contentLength的长度等于0，获取responseBody的BufferedSource,source.request发送请求之后，获取source的buffer，并且把buffer的size作为mOkHttpData的responseSzie。
DataRecordUtils的recordUrlRequest函数只有依据，就是调用QOKHttp.recordUrlRequest(okHttpData.url, okHttpData.code, okHttpData.requestSize,
                okHttpData.responseSize, okHttpData.startTime, okHttpData.costTime)
这个方法，生成一个NetInfo的对象，设置好各个字段之后，保存到本地数据库之中。