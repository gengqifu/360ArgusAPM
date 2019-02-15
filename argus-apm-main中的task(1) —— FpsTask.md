# argus-apm-main中的task(1) —— FpsTask

argus-apm-main中定义和实现了各个具体的监控task，包括Activity，网络，fps等等。这里我们先从fps作为例子入手分析。

## FpsTask类
FpsTask类定义了fps监控的task，相当于是fps task的入口。先来看一下这个类的代码。
```
public class FpsTask extends BaseTask implements Choreographer.FrameCallback {
    private final String SUB_TAG = ApmTask.TASK_FPS;

    private long mLastFrameTimeNanos = 0; //最后一次时间
    private long mFrameTimeNanos = 0; //本次的当前时间
    private int mCurrentCount = 0; //当前采集条数
    private int mFpsCount = 0;
    private FpsInfo fpsInfo = new FpsInfo();
    private JSONObject paramsJson = new JSONObject();
    //定时任务
    private Runnable runnable = new Runnable() {
        @Override
        public void run() {
            if (!isCanWork()) {
                mCurrentCount = 0;
                return;
            }
            calculateFPS();
            mCurrentCount++;
            //实现分段采集
            if (mCurrentCount < ArgusApmConfigManager.getInstance().getArgusApmConfigData().onceMaxCount) {
                AsyncThreadTask.executeDelayed(runnable, TaskConfig.FPS_INTERVAL);
            } else {
                AsyncThreadTask.executeDelayed(runnable, ArgusApmConfigManager.getInstance().getArgusApmConfigData().pauseInterval > TaskConfig.FPS_INTERVAL ? ArgusApmConfigManager.getInstance().getArgusApmConfigData().pauseInterval : TaskConfig.FPS_INTERVAL);
                mCurrentCount = 0;
            }
        }
    };

    private void calculateFPS() {
        if (mLastFrameTimeNanos == 0) {
            mLastFrameTimeNanos = mFrameTimeNanos;
            return;
        }
        float costTime = (float) (mFrameTimeNanos - mLastFrameTimeNanos) / 1000000.0F;
        if (mFpsCount <= 0 && costTime <= 0.0F) {
            return;
        }
        int fpsResult = (int) (mFpsCount * 1000 / costTime);
        if (fpsResult < 0) {
            return;
        }
        if (fpsResult <= TaskConfig.DEFAULT_FPS_MIN_COUNT) {
            fpsInfo.setFps(fpsResult);
            try {
                paramsJson.put(FpsInfo.KEY_STACK, CommonUtils.getStack());
            } catch (JSONException e) {
                e.printStackTrace();
            }
            fpsInfo.setParams(paramsJson.toString());
            fpsInfo.setProcessName(ProcessUtils.getCurrentProcessName());
            save(fpsInfo);
        }
        if (AnalyzeManager.getInstance().isDebugMode()) {
            if (fpsResult > TaskConfig.DEFAULT_FPS_MIN_COUNT) {
                fpsInfo.setFps(fpsResult);
            }
            AnalyzeManager.getInstance().getParseTask(ApmTask.TASK_FPS).parse(fpsInfo);
        }
        mLastFrameTimeNanos = mFrameTimeNanos;
        mFpsCount = 0;
    }

    @Override
    protected IStorage getStorage() {
        return new FpsStorage();
    }

    @Override
    public void start() {
        super.start();
        AsyncThreadTask.executeDelayed(runnable, (int) (Math.round(Math.random() * TaskConfig.TASK_DELAY_RANDOM_INTERVAL)));
        Choreographer.getInstance().postFrameCallback(this);
    }

    @Override
    public void stop() {
        super.stop();
    }

    @Override
    public String getTaskName() {
        return ApmTask.TASK_FPS;
    }

    @Override
    public void doFrame(long frameTimeNanos) {
        mFpsCount++;
        mFrameTimeNanos = frameTimeNanos;
        if (isCanWork()) {
            //注册下一帧回调
            Choreographer.getInstance().postFrameCallback(this);
        } else {
            mCurrentCount = 0;
        }
    }
}
```
fps task从start方法启动。至于任务如何被启动，我们在“开始”一篇里已经分析过，这里不重复了。start方法首先调用超类的start方法。所有task类的超类是BaseTask类，其实这个类里的start方法，只是记了一行log。回到FpsTask的start方法，这里启动类内定义的runnable（稍后分析）。AsyncThreadTask创建并维护了一个线程池。executeDelayed的第二个参数，要求任务延时指定的毫秒数执行。启动任务之后，向下一帧post一个frame callback，call就是FpsTask本身，这个类实现了Choreographer.FrameCallback接口。
我们先看一下要启动执行的runnable。run方法首先要判断任务是不是可以执行，如果不能执行，就把当前采集条数设置为0，并且返回。如果可以执行fps任务，调用calculateFPS计算fps，并且把calculateFPS加1。如果当前采集的条数小于设定的最大值，那么，TaskConfig.FPS_INTERVALms之后，继续执行这个runnable；如果采集条数已经达到最大值，就取pauseInterval和TaskConfig.FPS_INTERVAL之间的大值，在这个间隔之后，再次执行runnable,并且要把mCurrentCount清空。这样做的目的是实现fps的分段采集。
calculateFPS函数计算帧率。如果mLastFrameTimeNanos等于0，说明之前没有记录frame信息，那么就把mFrameTimeNanos的值赋给mLastFrameTimeNanos，直接返回。如果mLastFrameTimeNanos不等于0，mFrameTimeNanos减去mLastFrameTimeNanos，再除以1000000（换算成豪秒），作为costTime。如果mFpsCount小于等于0，或者costTime小于等于0，函数直接返回。mFpsCount乘以1000再除以costTime，作为fpsResult。mFpsCount的值在doFrame回调方法中累加，稍后分析这个函数。乘以1000是因为costTime的单位是毫秒，计算fps要换算成秒。如果这里计算出的fpsResult小于0，直接返回，因为帧率不可能小于0。如果fpsResult小于TaskConfig.DEFAULT_FPS_MIN_COUNT的值，设置fpsInfo中的各个字段：包括fps，就是上面计算出的fpsResult；调用栈，通过CommonUtils.getStack()获取；processName，通过ProcessUtils.getCurrentProcessName()获取当前进程的名称。save(fpsInfo)保存fpsInfo中信息到数据库。
save方法会调用mStorage中的save，在FpsTask中，getStorage方法会返回一个新的FpsStorage实例，不过它并没有重载save方法，所以这里的save实际上就是TableStorage中的save。
最后看一下重载的Choreographer.FrameCallback的回调方法doFrame。代码比较简单，首先实现mFpsCount的自增。然后把参数frameTimeNanos的值赋值给mFrameTimeNanos。如果现在的canWork标记还是true，继续注册下一帧的回调；否则的话，就把mCurrentCount清空。