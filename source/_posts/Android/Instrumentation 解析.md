---
title: 关于 Instrumentation
link: instrumentation
cover: https://miro.medium.com/max/3140/1*4QUG8flHphALSTqb06j3ww.png
---

Instrumentation 的英文意思是『仪表』，但在 Android 中的功能可不是『仪表』。

<!-- more -->

它其实是 Android 系统里的一套控制方法，或者叫『钩子』。这些『钩子』可以在正常的生命周期（正常是由操作系统控制的)之外控制Android控件的运行，其实指的就是Instrumentation类提供的各种流程控制方法。我们看一点源码就知道了：

```java
public class Instrumentation {
    private ActivityThread mThread = null;
    private MessageQueue mMessageQueue = null;
    private Context mInstrContext;
    private Context mAppContext;
    private ComponentName mComponent;
    private Thread mRunner;
    ...

    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }

    ...
    public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
}
```

看起来像不像静态代理？

![](http://img.doutula.com/production/uploads/image/2019/11/17/20191117993281_HImBdo.jpg)

它会在 Activity 的生命周期方法执行时加一个钩子，用来做一些监听的工作。

Activity 的实例也是由 Instrumentation 创建的，在 ActivityThread 中：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    activity = mInstrumentation.newActivity(
        cl, component.getClassName(), r.intent);
}
```

Instumentation 说了，我创建的 Activity 实例，我要是不监听着点，那还不完蛋了。所以它一般用来进行测试，用来实现 Activity 生命周期钩子、获取 Activity 中的 View 等等。