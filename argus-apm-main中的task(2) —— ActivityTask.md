# argus-apm-main中的task(2) —— Activity Task

Activity Task实现对Activity的性能监控。我们在“开始”一篇里提到过，Activity的性能采集实现方式有两种，Instrumentation和AOP，而ActivitTask的start方法主要就做了一个判断：如果使能了ApmTask.FLAG_COLLECT_ACTIVITY_INSTRUMENTATION，也就是开启了Instrumentation方式，但是InstrumentationHooker失败，mIsCanWork要设置为false，也就是mIsCanWork不能work。
我们接下来对两种采集方式分别进行分析。

## Instrumentation
我们先来看看Instrumentation方式。这是一种在运行期进行hook的方式。从ActivityTask的start方法中，我们看到Activity 的监控，并不是在ActivityTask的start方法中开启的。那么，对Activity的监控，是在哪里开启的呢？其实我们在“开始”一篇中，已经分析过了，实际上，这是在TaskManager类的startWorkTasks方法中。这个方法之前分析过了，这里不再详述。这个方法内会决定用Instrumentation方式还是用AOP方式进行Activity信息的收集，并且会启动相应的任务。对于Instrumentation方式，会通过InstrumentationHooker.doHook()方法来启动hook。这个方法又调用本类的hookInstrumentation，如果调用成功，会把isHookSucceed设置为true。hookInstrumentation的代码是这样的：
```
private static void hookInstrumentation() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException, NoSuchFieldException {
    Class<?> c = Class.forName("android.app.ActivityThread");
    Method currentActivityThread = c.getDeclaredMethod("currentActivityThread");
    boolean acc = currentActivityThread.isAccessible();
    if (!acc) {
        currentActivityThread.setAccessible(true);
    }
    Object o = currentActivityThread.invoke(null);
    if (!acc) {
        currentActivityThread.setAccessible(acc);
    }
    Field f = c.getDeclaredField("mInstrumentation");
    acc = f.isAccessible();
    if (!acc) {
        f.setAccessible(true);
    }
    Instrumentation currentInstrumentation = (Instrumentation) f.get(o);
    Instrumentation ins = new ApmInstrumentation(currentInstrumentation);
    f.set(o, ins);
    if (!acc) {
        f.setAccessible(acc);
    }
}
```
通过反射找到android.app.ActivityThread类，再找到类中的currentActivityThread的方法。判断方法是否能访问，如果不能，首先要激活访问。通过invoke调用这个方法之后，如果初始方法是不可访问的，需要恢复为不可访问。找到ActivityThread类的mInstrumentation属性，同样，如果不可访问，需要打开访问权限，不详述。通过Field的get方法，得到currentActivityThread的mInstrumentation，currentInstrumentation。通过currentInstrumentation构建一个ApmInstrumentation的对象ins，通过Field的set方法，设置currentActivityThread的mInstrumentation为新的ApmInstrumentation类型的ins。最后，不要忘了恢复field的可访问状态。

- ApmInstrumentation
那么，问题的关键转到ApmInstrumentation类。这个类都做了些什么呢？从源码看到，这个类扩展了Instrumentation类。Instrumentation类是Android提供的对Application进行监控的类。因为前边我们已经把一个ApmInstrumentation类型的对象hook进了currentActivityThread，所以在系统触发Instrumentation的回调时，就会执行ApmInstrumentation中定义的逻辑。
首先看callApplicationOnCreate回调。这里先把ActivityCore的appAttachTime设置成当前的系统时间。如果mOldInstrumentation不为null，调用mOldInstrumentation的callApplicationOnCreate方法，否则，调用超类的同名方法。
callApplicationOnCreate的第一个参数是当前的Activity，第二个参数是一个bundle，是传递给onCreate的前一个“frozen“状态。如果Activity Task并没有运行，只调用mOldInstrumentation的callActivityOnCreate或者超类的callActivityOnCreate之后就返回了。如果任务在运行，首先获取当前的系统时间startTime，然后再触发原始的callActivityOnCreate回调。根据ActivityCore中isFirst的值，判断Activity的启动类型，并且保存在startType中。调用ActivityCore.onCreateInfo(activity, startTime)。这个方法一方面向Activity的DecorView发送一个FirstFrameRunnable，另一方面保存Activity的信息，activity的启动时间就是当前时间减去startTime。
我们来看看FirstFrameRunnable这个Runnable做了什么。
```
private static class FirstFrameRunnable implements Runnable {
    private Activity activity;//Activity
    private int startType;//启动类型
    private long startTime;//开始时间

    public FirstFrameRunnable(Activity activity, int startType, long startTime) {
        this.startTime = startTime;
        this.activity = activity;
        this.startType = startType;
    }

    @Override
    public void run() {
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "FirstFrameRunnable time:" + (System.currentTimeMillis() - startTime));
        }
        if ((System.currentTimeMillis() - startTime) >= ArgusApmConfigManager.getInstance().getArgusApmConfigData().funcControl.activityFirstMinTime) {
            saveActivityInfo(activity, startType, System.currentTimeMillis() - startTime, ActivityInfo.TYPE_FIRST_FRAME);
        }
        //保存应用冷启动时间
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "FirstFrameRunnable time:" + String.format("[%s, %s]", ActivityCore.isFirst, ActivityCore.appAttachTime));
        }
        if (ActivityCore.isFirst) {
            ActivityCore.isFirst = false;
            if (ActivityCore.appAttachTime <= 0) {
                return;
            }
            int t = (int) (System.currentTimeMillis() - ActivityCore.appAttachTime);
            AppStartInfo info = new AppStartInfo(t);
            ITask task = Manager.getInstance().getTaskManager().getTask(ApmTask.TASK_APP_START);
            if (task != null) {
                task.save(info);
                if (AnalyzeManager.getInstance().isDebugMode()) {
                    AnalyzeManager.getInstance().getParseTask(ApmTask.TASK_APP_START).parse(info);
                }
            } else {
                if (DEBUG) {
                    LogX.d(TAG, SUB_TAG, "AppStartInfo task == null");
                }
            }
        }
    }
}
```
它有三个属性，分别是当前Activity activity， 启动类型startType，和启动时间startTime。重点看一下它的run方法。如果当前系统时间减去startTime不小于activityFirstMinTime，就保存Activity的信息。activityFirstMinTime是设定的Activity第一帧最小时间。如果Activity是第一次启动，还要记录冷启动时间。如果ActivityCore.appAttachTime小于或者等于0（应用没有启动），返回不处理。用当前系统时间减去ActivityCore.appAttachTime，这个就是冷启动时间。获取正在运行的ApmTask.TASK_APP_START，保存task的信息。
其它的几个Activity生命周期的回调方法都是类似的。

## AOP
如果我们没有配置ApmTask.FLAG_COLLECT_ACTIVITY_INSTRUMENTATION，那么默认就会以aop的方式对Activity进行数据采集。具体的采集方式，我们在分析argus-apm-aop方式时，已经讨论过了。