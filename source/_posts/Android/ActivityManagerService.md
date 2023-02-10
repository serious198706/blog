---
title: 关于 ActivityManagerService
link: activity-manager-service
toc: true
tag: 
    - Android
    - Android Framework
category: 
    - Android
    - Development
cover: http://blog.cigis-cloud.com/android-1593080864.jpg
date: 2020-04-18
---

ActivityManagerService 是 Android 提供的**管理 Activity 运行状态的系统进程**，其实大家别被名字迷惑了，ActivityManagerService（后称AMS）其实也兼任管理其他组件运行状态。

<!-- more -->

## AMS概述
### AMS启动流程

根据[](/android-boot/)这篇文章中提到的，init 进程是 Android 系统中的初始化进程，init 生成 Zygote 进程，Android 中大多数应用进程和系统进程都是通过 Zygote 进程生成的。

![](/img/ams-1587379033.jpg)

上面的流程图展示了 AMS 的代码执行流程。我们接下来详细讲讲。

AMS这种系统级别的服务，一般都是在启动的时候触发，我们可以看一下在 system_server 进程中，是如何启动 AMS 的：

```java
// com.android.server.SystemServer.java;
public final class SystemServer {
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }

    private void run() {
        ...
        // Start services.
        ...
        startBootstrapServices();
        ...
    }

    private void startBootstrapServices() {
        ...
        // Activity manager runs the show.
        traceBeginAndSlog("StartActivityManager");
        // TODO: Might need to move after migration to WM.
        ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
        // 这里真正启动了 AMS
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        mWindowManagerGlobalLock = atm.getGlobalLock();
        ...
        // 为 system 进程设置 Application 实例
        mActivityManagerService.setSystemProcess();
    }
}
```

可以看到，启动 AMS 使用的是 AMS 自身的一个静态内部类的`startService()`方法，我们来看看：

[查看ActivityManagerService.java](https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/am/ActivityManagerService.java)

```java
// ActivityManagerService.java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;
    ...df
    public static ActivityManagerService startService(
            SystemServiceManager ssm, ActivityTaskManagerService atm) {
        sAtm = atm;
        return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
    }

    @Override
    public void onStart() {
        mService.start();
    }
    ...
}
```

这里又调用了`SystemServiceManager.startService(Class)`方法，我们来看一下：

```java
// com.android.server.SystemServiceManager.java
/**
    * Creates and starts a system service. The class must be a subclass of
    * {@link com.android.server.SystemService}.
    *
    * @param serviceClass A Java class that implements the SystemService interface.
    * @return The service instance, never null.
    * @throws RuntimeException if the service fails to start.
    */
@SuppressWarnings("unchecked")
public <T extends SystemService> T startService(Class<T> serviceClass) {
    ...
    // 通过反射的方式实例化该 service
    Constructor<T> constructor = serviceClass.getConstructor(Context.class);
    service = constructor.newInstance(mContext);
    ...
    startService(service);
    return service;
}

public void startService(@NonNull final SystemService service) {
    // 注册
    mServices.add(service);
    ...
    // 调用 Lifecycle 的 onStart() 方法
    service.onStart();
    ...
}
```

可以看到，最后调用到了`ActivityManagerService.Lifecycle.onStart()`方法，而`ActivityManagerService.Lifecycle.onStart()`方法又调用了`mService.start()`方法，进行了一些初始化的工作：

```java
private void start() {
    removeAllProcessGroups();
    mProcessCpuThread.start();
    mBatteryStatsService.publish();
    mAppOpsService.publish(mContext);
    Slog.d("AppOps", "AppOpsService published");
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    mActivityTaskManager.onActivityManagerInternalAdded();
    mUgmInternal.onActivityManagerInternalAdded();
    mPendingIntentController.onActivityManagerInternalAdded();
    // Wait for the synchronized block started in mProcessCpuThread,
    // so that any other access to mProcessCpuTracker from main thread
    // will be blocked during mProcessCpuTracker initialization.
    try {
        mProcessCpuInitLatch.await();
    } catch (InterruptedException e) {
        Slog.wtf(TAG, "Interrupted wait during start", e);
        Thread.currentThread().interrupt();
        throw new IllegalStateException("Interrupted wait during start");
    }
}
```

初始化完成后，就会调用`mActivityManagerService.setSystemProcess()`方法：

```java
public void setSystemProcess() {
    ...
    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
    ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
    ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
            DUMP_FLAG_PRIORITY_HIGH);
    ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
    ServiceManager.addService("dbinfo", new DbBinder(this));
    if (MONITOR_CPU_USAGE) {
        ServiceManager.addService("cpuinfo", new CpuBinder(this),
                /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
    }
    ServiceManager.addService("permission", new PermissionController(this));
    ServiceManager.addService("processinfo", new ProcessInfoService(this));
    ...
}
```

可以看出，通过`ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true)`方法注册了当前 AMS 的实例。AMS 是一个典型的 Binder Server，ServiceManager 还注册了与 AMS 有关的其他服务（如 CPU、PermissionController等），说明后面 AMS 会与这些服务有很多交互。

刚才我们跳过了 AMS 的实例化过程，现在我们来看一下它的实例化中，具体都做了哪些工作：

```java
    // Note: This method is invoked on the main thread but may need to attach various
    // handlers to other threads.  So take care to be explicit about the looper.
    public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
        LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
        mInjector = new Injector();
        mContext = systemContext;

        mFactoryTest = FactoryTest.getMode();

        // 第一步
        mSystemThread = ActivityThread.currentActivityThread();
        mUiContext = mSystemThread.getSystemUiContext();

        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        mUiHandler = mInjector.getUiHandler(this);

        mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
                THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
        mProcStartHandlerThread.start();
        mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());

        mConstants = new ActivityManagerConstants(mContext, this, mHandler);
        final ActiveUids activeUids = new ActiveUids(this, true /* postChangesToAtm */);
        mPlatformCompat = (PlatformCompat) ServiceManager.getService(
                Context.PLATFORM_COMPAT_SERVICE);
        mProcessList.init(this, activeUids, mPlatformCompat);
        mLowMemDetector = new LowMemDetector(this);
        mOomAdjuster = new OomAdjuster(this, mProcessList, activeUids);

        // 第二步
        // Broadcast policy parameters
        final BroadcastConstants foreConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_FG_CONSTANTS);
        foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;

        final BroadcastConstants backConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_BG_CONSTANTS);
        backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;

        final BroadcastConstants offloadConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
        offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
        // by default, no "slow" policy in this queue
        offloadConstants.SLOW_TIME = Integer.MAX_VALUE;

        mEnableOffloadQueue = SystemProperties.getBoolean(
                "persist.device_config.activity_manager_native_boot.offload_queue_enabled", false);
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", foreConstants, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", backConstants, true);
        mOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
                "offload", offloadConstants, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
        mBroadcastQueues[2] = mOffloadBroadcastQueue;

        // 第三步
        mServices = new ActiveServices(this);
        mProviderMap = new ProviderMap(this);
        mPackageWatchdog = PackageWatchdog.getInstance(mUiContext);
        mAppErrors = new AppErrors(mUiContext, this, mPackageWatchdog);

        // 第四步
        final File systemDir = SystemServiceManager.ensureSystemDir();

        // TODO: Move creation of battery stats service outside of activity manager service.
        mBatteryStatsService = new BatteryStatsService(systemContext, systemDir,
                BackgroundThread.get().getHandler());
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);
        mOomAdjProfiler.batteryPowerChanged(mOnBattery);

        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);

        mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);

        // 第五步
        mUserController = new UserController(this);

        mPendingIntentController = new PendingIntentController(
                mHandlerThread.getLooper(), mUserController);
                
        if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
            mUseFifoUiScheduling = true;
        }

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);

        mActivityTaskManager = atm;

        mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper());
        mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);

        // 第六步
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                synchronized (mProcessCpuTracker) {
                    mProcessCpuInitLatch.countDown();
                    mProcessCpuTracker.init();
                }
                while (true) {
                    try {
                        try {
                            synchronized(this) {
                                final long now = SystemClock.uptimeMillis();
                                long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                                long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                                //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                                //        + ", write delay=" + nextWriteDelay);
                                if (nextWriteDelay < nextCpuDelay) {
                                    nextCpuDelay = nextWriteDelay;
                                }
                                if (nextCpuDelay > 0) {
                                    mProcessCpuMutexFree.set(true);
                                    this.wait(nextCpuDelay);
                                }
                            }
                        } catch (InterruptedException e) {
                        }
                        updateCpuStatsNow();
                    } catch (Exception e) {
                        Slog.e(TAG, "Unexpected exception collecting process stats", e);
                    }
                }
            }
        };

        mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);

        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);

        // bind background threads to little cores
        // this is expected to fail inside of framework tests because apps can't touch cpusets directly
        // make sure we've already adjusted system_server's internal view of itself first
        updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
        try {
            Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
            Process.setThreadGroupAndCpuset(
                    mOomAdjuster.mAppCompact.mCompactionThread.getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
        } catch (Exception e) {
            Slog.w(TAG, "Setting background thread cpuset failed");
        }
    }
```

我总共将 AMS 的实例化过程总体分为了六个步骤：

1. 构造一些 Context、Handler 和 Thread，如 uiHandler 等，用来处理 UI 相关的工作；
2. 定义了容纳前台和后台的广播队列，这也说明了 AMS 不仅仅关注 Activity，也负责其他组件状态的管理；
3. 管理 Service 和 Provider 的对象数组；
4. 初始化system下面需要的一系列文件目录。例如权限文件、进程状态信息文件等等；
5. 管理 ActivityStack，也管理 Activity 启动时用到的 Intent 和 flag；
6. 启动一个线程专门跟进 cpu 当前状态信息，AMS 对当前 cpu 状态了如指掌，可以更加高效的安排其他工作。

## Activity 状态管理

先提出几个问题，我们根据问题去看下面的部分：

> Activity 是如何被创建的？
> 
> Android 是如何管理 Activity 状态的？
> 
> 一个 Task 包含一个或者多个 Activity，一个 Stack 包含一个或者多个 Task，这儿引入 ActivityStack，还有ActivityStackSupervisor 负责管理所有的 Stack。那么这些对象都是如何管理的？

### Activity 的创建

我们在[关于 Activity 的一切](/about-activity/)这篇文章中讲过，Activity 是由 ActivityThread 创建的实例，那么 ActivityThread 是由谁创建的？AMS 又是如何介入的呢？我们来看一下。

在前文中讲过，system_server 在它的`main()`方法中创建了 SystemServer 的实例，并调用了它的`run()`方法，在这里有这么一段：

```java
private void run() {
    ...
    // Initialize the system context.
    createSystemContext();
    ...
}

private void createSystemContext() {
    ActivityThread activityThread = ActivityThread.systemMain();
    mSystemContext = activityThread.getSystemContext();
    mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
    final Context systemUiContext = activityThread.getSystemUiContext();
    systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
}
```

system_server 进程似乎是创建了 ActivityThread 的实例，我们看看`ActivityThread.systemMain()`方法做了什么：

```java
final ApplicationThread mAppThread = new ApplicationThread();

@UnsupportedAppUsage
public static ActivityThread systemMain() {
    ...
    ActivityThread thread = new ActivityThread();
    thread.attach(true, 0);
    return thread;
}

private void attach(boolean system, long startSeq) {
    ...
    // 找到了，AMS 在这里介入了
    final IActivityManager mgr = ActivityManager.getService();
    try {
        mgr.attachApplication(mAppThread, startSeq);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    ...
}
```

ApplicationThread 是 IApplicationThread.Stub 的子类，而 IApplicationThread 是 AMS 用来与应用进程通讯的接口。

IActivityManager 是 AMS 的 Binder 接口，应用进程会通过这个接口，用 Binder 机制与 AMS 进行通讯。

所以，`attachApplication()`方法一经调用，就将应用进程与 AMS 联系了起来，AMS 就可以管理应用进程中的 Activity 状态了。

实际上我们仔细看一下 ActivityThread 的代码，我们会发现，AMS 在很多地方都介入了管理，比如下面几处：

```java
private void handleReceiver(ReceiverData data) {
    ...
    IActivityManager mgr = ActivityManager.getService();
    ...
    data.sendFinished(mgr);
    ...
}

private void handleCreateService(CreateServiceData data) {
    ...
    service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
    ...

}

private void handleBindService(BindServiceData data) {
    ...
    if (!data.rebind) {
        IBinder binder = s.onBind(data.intent);
        ActivityManager.getService().publishService(
                data.token, data.intent, binder);
    } else {
        s.onRebind(data.intent);
        ActivityManager.getService().serviceDoneExecuting(
                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
    }
    ...
}
```

可见，AMS 真的如开头所说，是个多面手，管理了许多 Android 组件的状态与数据。

### ActivityRecord、TaskRecord、ActivityStack

先来一张图，挑明这三者的关系：

![](/img/ams-1587440232.png)

- 一个 ActivityStack 中包含多个 TaskRecord
- 一个 TaskRecord 中包含一个或多个 ActivityRecord，
- 一个 ActivityRecord 就对应着一个 Activity，保存了它的所有信息；同时一个 Activity 可能会生成多个 ActivityRecord，因为由于其启动模式决定了，它有可能被多次启动，会存在多个实例

那么我们从下往上分析，先来看看 ActivityRecord。

#### ActivityRecord

> An entry in the history stack, representing an activity.

如代码注释中所言，ActivityRecord 是 Activity 历史栈中的一个条目，代表了一个 Activity。我们看看它比较重要的部分代码：

```java
// package com.android.server.wm.ActivityRecord.java

final class ActivityRecord extends ConfigurationContainer {
    final ActivityTaskManagerService mAtmService; // owner
    final IApplicationToken.Stub appToken; // window manager token
    final ActivityInfo info; // all about me
    ApplicationInfo appInfo; // information about activity's app
    final Intent intent;    // the original intent that generated us
    final ComponentName mActivityComponent;  // the intent component, or target of an alias.
    final String taskAffinity; // as per ActivityInfo.taskAffinity
    private TaskRecord task;        // the task this is in.
    ActivityRecord resultTo; // who started this entry, so will get our reply
    final int requestCode;  // code given by requester (resultTo)
    int launchMode;         // the launch mode activity attribute.
    final ActivityStackSupervisor mStackSupervisor;
}
```

### Android 如何管理 Activity 状态

先来一张图，我们看看 Activity 的各种生命周期方法是在何时被调用：

![](/img/ams-1587435208.jpg)

这儿写明了回调各个流程的时机，其中包含这对 Activity 状态的处理，这一点非常重要，Android 系统处理的 Activity 很多，我们准确指示当前 Activity 的状态，可以保证 Activity 调用的正确性。

我们来仔细分析一个事件：Activity 的`onPause()`事件是如何从触摸屏幕开始触发的。因为触发`onPause()`的途径有很多种，我们选择一种 —— **按下了 HOME 键**。这是一个非常复杂的过程，我们得一步步来。

我们在[View事件传递机制](/view-event-dispatch/)中提到过，触摸作为一个 InputEvent，由 InputManagerService 来进行处理。

InputManagerService 此时会收到来自 Native 层的调用：

```cpp
void NativeInputManager::onPointerDownOutsideFocus(const sp<IBinder>& touchedToken) {
    ATRACE_CALL();
    JNIEnv* env = jniEnv();
    ScopedLocalFrame localFrame(env);
    jobject touchedTokenObj = javaObjectForIBinder(env, touchedToken);
    env->CallVoidMethod(mServiceObj, gServiceClassInfo.onPointerDownOutsideFocus, touchedTokenObj);
    checkAndClearExceptionFromCallback(env, "onPointerDownOutsideFocus");
}
```

调用了它的`onPointerDownOutsideFocus()`方法：

```java
// Native callback. 
private void onPointerDownOutsideFocus(IBinder touchedToken) {
    /**
     * Notifies window manager that a {@link android.view.MotionEvent#ACTION_DOWN} pointer event
     * occurred on a window that did not have focus.
     *
     * @param touchedToken The token for the window that received the input event.
     */
    mWindowManagerCallbacks.onPointerDownOutsideFocus(touchedToken);
}
```

此处的`mWindowManagerCallback`是 system_server 在`startOtherServices()`方法中赋值的，代码如下：

```java
// com.android.server.SystemServer.java

private void startOtherServices() {
    ...
    inputManager = new InputManagerService(context);
    ...
    wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
                    new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
    ...
    inputManager.setWindowManagerCallbacks(wm.getInputManagerCallback());
}
```

所以此处的`onPointerDownOutsideFocus()`方法将由 InputManagerCallback 来实现：

```java
// com.android.server.wm.InputManagerCallback.java

@Override
public void onPointerDownOutsideFocus(IBinder touchedToken) {
    // mService 也即 WindowManagerService 的实例，它的 mH 是一个自定义的 Handler 实例
    mService.mH.obtainMessage(ON_POINTER_DOWN_OUTSIDE_FOCUS, touchedToken).sendToTarget();
}
```

```java
// com.android.server.wm.WindowManagerService.java

class H extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch(msg.what) {
            ...
            case ON_POINTER_DOWN_OUTSIDE_FOCUS: {
                synchronized (mGlobalLock) {
                    final IBinder touchedToken = (IBinder) msg.obj;
                    onPointerDownOutsideFocusLocked(touchedToken);
                }
                break;
            }
            ...
        }
    }
}

private void onPointerDownOutsideFocusLocked(IBinder touchedToken) {
    ...
    handleDisplayFocusChange(touchedWindow);
}

private void handleDisplayFocusChange(WindowState window) {
    final DisplayContent displayContent = window.getDisplayContent();
    ...
    // For compatibility, only the topmost activity is allowed to be resumed for pre-Q
    // app. Ensure the topmost activities are resumed whenever a display is moved to top.
    // TODO(b/123761773): Investigate whether we can move this into
    // RootActivityContainer#updateTopResumedActivityIfNeeded(). Currently, it is risky
    // to do so because it seems possible to resume activities as part of a larger
    // transaction and it's too early to resume based on current order when performing
    // updateTopResumedActivityIfNeeded().
    displayContent.mAcitvityDisplay.ensureActivitiesVisible(null /* starting */,
            0 /* configChanges */, !PRESERVE_WINDOWS, true /* notifyClients */);
}
```

DisplayContent.mAcitvityDisplay 是一个 ActivityDisplay 类型的变量。一个 ActivityDisplay 表示一块屏幕，一般情况下，ActivityDisplay 在 Android 系统中，都只有一个实例。下面是 ActivityDisplay 的简介，可以回头再看。

::: tip

**关于 ActivityDisplay**

ActivityDisplay 维护了一个 ActivityStack 的栈，并且有添加和移除的方法
```java
/**
 * Exactly one of these classes per Display in the system. Capable of holding zero or more
 * attached {@link ActivityStack}s.
 */
class ActivityDisplay {
    ...
    /**
     * All of the stacks on this display. Order matters, topmost stack is in front of all other
     * stacks, bottommost behind. Accessed directly by ActivityManager package classes. Any calls
     * changing the list should also call {@link #onStackOrderChanged()}.
     */
	final ArrayList<ActivityStack> mStacks = new ArrayList<ActivityStack>();
    ...
    void addChild(ActivityStack stack, int position) {
        if (position == POSITION_BOTTOM) {
            position = 0;
        } else if (position == POSITION_TOP) {
            position = mStacks.size();
        }
        addStackReferenceIfNeeded(stack);
        positionChildAt(stack, position);
    }

    void removeChild(ActivityStack stack) {
        mStacks.remove(stack);
        ...
        onStackOrderChanged(stack);
    }

    private void positionChildAt(ActivityStack stack, int position, boolean includingParents,
            String updateLastFocusedStackReason) {
        ...
        final int insertPosition = getTopInsertPosition(stack, position);
        mStacks.add(insertPosition, stack);
        ...
        onStackOrderChanged(stack);
    }
}
```
:::

我们来看`ActivityDisplay.ensureActivitiesVisible()`方法的代码：

```java
void ensureActivitiesVisible(ActivityRecord starting, int configChanges,
        boolean preserveWindows, boolean notifyClients) {
    for (int stackNdx = getChildCount() - 1; stackNdx >= 0; --stackNdx) {
        final ActivityStack stack = getChildAt(stackNdx);
        stack.ensureActivitiesVisibleLocked(starting, configChanges, preserveWindows,
                notifyClients);
    }
}
```

循环调用了栈内的 ActivityStack 的`ensureActivitiesVisibleLocked()`方法：

```java
// com.android.server.vm.ActivityStack.java
/**
 * Ensure visibility with an option to also update the configuration of visible activities.
 * @see #ensureActivitiesVisibleLocked(ActivityRecord, int, boolean)
 * @see RootActivityContainer#ensureActivitiesVisible(ActivityRecord, int, boolean)
 */
// TODO: Should be re-worked based on the fact that each task as a stack in most cases.
// 由这个 TODO 可以看出，日后 Android 必定会变成一个 task 中只有一个 stack，毕竟大多数情况下是这样的
final void ensureActivitiesVisibleLocked(ActivityRecord starting, int configChanges,
        boolean preserveWindows, boolean notifyClients) {
    ...
        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            ...
            final TaskRecord task = mTaskHistory.get(taskNdx);
            final ArrayList<ActivityRecord> activities = task.mActivities;
            for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
                final ActivityRecord r = activities.get(activityNdx);
                ...
                r.makeClientVisible();
                ...
            }
}
```

来到了`ActivityRecord.makeClientVisible()`：

```java
// com.android.server.wm.ActivityRecord.java

/** Send visibility change message to the client and pause if needed. */
void makeClientVisible() {
    ...
    makeActiveIfNeeded(null /* activeActivity*/);
    ...
}

/**
 * Make activity resumed or paused if needed.
 * @param activeActivity an activity that is resumed or just completed pause action.
 *                       We won't change the state of this activity.
 */
boolean makeActiveIfNeeded(ActivityRecord activeActivity) {
    if (shouldResumeActivity(activeActivity)) {
        return getActivityStack().resumeTopActivityUncheckedLocked(activeActivity /* prev */,
                null /* options */);
    } else if (shouldPauseActivity(activeActivity)) {
        ...
    }
    return false;
}
```

又调用了 ActivityStack 的`resumeTopActivityUncheckedLocked()`方法：

```java
// com.android.server.wm.ActivityStack.java

/**
 * Ensure that the top activity in the stack is resumed.
 *
 * @param prev The previously resumed activity, for when in the process
 * of pausing; can be null to call from elsewhere.
 * @param options Activity options.
 *
 * @return Returns true if something is being resumed, or false if
 * nothing happened.
 *
 * NOTE: It is not safe to call this method directly as it can cause an activity in a
 *       non-focused stack to be resumed.
 *       Use {@link RootActivityContainer#resumeFocusedStacksTopActivities} to resume the
 *       right activity for the current system state.
 */
@GuardedBy("mService")
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    result = resumeTopActivityInnerLocked(prev, options);
    ...
    return result;
}

@GuardedBy("mService")
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    pausing |= startPausingLocked(userLeaving, false, next, false);
    ...
}

```

<!--上一节中我们讲过 ActivityRecord、ActivityClientRecord、Activity 的关系，那么Android系统又是如何利用这层对应关系的呢。比如启动 Activity 时需要令上一个 Activity 执行 onPause() 方法。这时会经历以下方法调用链：

 ```java
ActivityStack.startPausingLocked() 
IApplicationThread.schudulePauseActivity() 
ActivityThread.sendMessage() 
ActivityThread.H.sendMessage(); 
ActivityThread.H.handleMessage() 
ActivityThread.handlePauseActivity() 
ActivityThread.performPauseActivity() 
Activity.performPause() 
Activity.onPause() 
ActivityManagerNative.getDefault().activityPaused(token) 
ActivityManagerService.activityPaused() 
ActivityStack.activityPausedLocked() 
ActivityStack.completePauseLocked()
``` -->

```java
// com.android.server.wm.ActivityStack.java
ActivityTaskManagerService mService;

final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
    ...
    ActivityRecord prev = mResumedActivity;
    ...
    if (prev.attachedToProcess()) {
        ...
        try {
            ...
            //prev 就是当前获得焦点的 ActivityRecord，现在的目的是令该 ActivityRecord 对应的 Activity 调用 onPause() 方法
            ...
            mService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                    prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,
                            prev.configChangeFlags, pauseImmediately));
        } catch (Exception e) {
            ...
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
    } else {
        mPausingActivity = null;
        mLastPausedActivity = null;
        mLastNoHistoryActivity = null;
    }
    ...
}
```

`mService`也即 ActivityTaskManagerService，它的`getLifecycleManager()`获取了一个 ClientLifecycleManager 的内部实例，然后调用它的`schedultTransaction()`方法。PauseActivityItem 是 ActivityLifecycleItem 的派生类：

```java
// com.android.server.wm.ActivityStack.java
void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
        @NonNull ActivityLifecycleItem stateRequest) throws RemoteException {
    final ClientTransaction clientTransaction = transactionWithState(client, activityToken,
            stateRequest);
    scheduleTransaction(clientTransaction);
}

void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    transaction.schedule();
    if (!(client instanceof Binder)) {
        // If client is not an instance of Binder - it's a remote call and at this point it is
        // safe to recycle the object. All objects used for local calls will be recycled after
        // the transaction is executed on client in ActivityThread.
        transaction.recycle();
    }
}
```

ClientTransaction 类是一个 Parcelable，看命名方式，就知道这又涉及到 Binder 通讯了：

```java
// android.app.servertransaction.ClientTransaction
public class ClientTransaction implements Parcelable, ObjectPoolItem {
    ...
    private IApplicationThread mClient;
    ...
    // 这里的处理分为3步：
    // 1. client 端调用 preExecute(ClientTransactionHandler)，将会触发一系列的准备工作，包括设置回调，生命周期管理等等
    // 2. 准备传递的消息
    // 3. client 端调用 TransactionExecutor.execute(ClientTransaction)，将会执行所有的回调和生命周期方法
    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }
}
```

也即将 pauseActivity 的工作交给了交给了应用进程的 ActivityThread：

```java
@Override
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

但是 ActivityThread 并没有直接处理这个 transaction，而是又调用了`scheduleTransaction()`这个方法，这个方法来自于它的**抽象父类**ClientTransactionHandler：

```java
/**
 * Defines operations that a {@link android.app.servertransaction.ClientTransaction} or its items
 * can perform on client.
 * @hide
 */
public abstract class ClientTransactionHandler {
    // Schedule phase related logic and handlers.

    /** Prepare and schedule transaction for execution. */
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        // H extends Handler， 它定义了 ActivityThread 所需要处理的所有事件的 code
        // EXECUTE_TRANSACTION 就是其中之一，代表要转换 Activity 的状态了
        // 而 sendMessage 方法由 ActivityThread 方法实现，最终将消息发送到 ActivityThread 的成员变量 mH 中（H类的实例）
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
}
```

我们来看看这个 H 做了什么：

```java
switch(msg.what) {
    ...
    case EXECUTE_TRANSACTION:
        final ClientTransaction transaction = (ClientTransaction) msg.obj;
        mTransactionExecutor.execute(transaction);
        if (isSystem()) {
            // Client transactions inside system process are recycled on the client side
            // instead of ClientLifecycleManager to avoid being cleared before this
            // message is handled.
            transaction.recycle();
        }
        // TODO(lifecycler): Recycle locally scheduled transactions.
        break;
    ...
}
```

啊。。绕得还真远，又出现了一个 TransactionExecutor 类。这个类的作用是**保证一个 transaction 转换过程能够按照正确的顺序执行**。

经过一系列调用，最终会运行到它的这一句：

```java
performLifecycleSequence(r, path, transaction);
```

这个方法的定义如下：

```java
private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
                                                  ClientTransaction transaction) {
    final int size = path.size();
        for (int i = 0, state; i < size; i++) {   
            state = path.get(i);
            switch (state) {  
                ...
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                ...
            }                                               
```

绕啊绕啊绕，最终还是绕回了`ActivityThread.handlePauseActivity()`方法：

```java
@Override
public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
        int configChanges, PendingTransactionActions pendingActions, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        if (userLeaving) {
            performUserLeavingActivity(r);
        }

        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(r, finished, reason, pendingActions);

        // Make sure any pending writes are now committed.
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }
        mSomeActivitiesChanged = true;
    }
}

final Bundle performPauseActivity(ActivityClientRecord r, boolean finished, String reason,
            PendingTransactionActions pendingActions) {
    ...
    // 之前启动Activity时会调用 performLaunchActivity()，并在最后以 token 为 Key，ActivityClientRecord 为 value 保存 ActivityClientRecord。
    // 然后到了 performPauseActivity() 中又会根据 token 取出对应 ActivityClientRecord。
    // 再调用 ActivityClientRecord 中保存的 activity 的 onPause() 方法
    performPauseActivityIfNeeded(r, reason);
    ...
}

private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
	...
	mInstrumentation.callActivityOnPause(r.activity);
	...
}

public class Instrumentation {
	public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
}
```

至此，才完成了 Activity 的 pause 过程。