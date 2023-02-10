---
title: Activity 启动流程分析
link: activity-start-process
cover: http://blog.cigis-cloud.com/android-1593080864.jpg
toc: true
tag: 
    - Android
    - Activity
category: 
    - Android
    - Development
date: 2020-05-27
---

很多人对 Activity 启动过程不甚了解，我们来捋一下它的启动流程。

<!-- more -->

启动 Activity 无非是下面几种场景：

1. 从应用中启动自身 Activity
2. 从应用中启动其他应用中的 Activity
3. 从桌面点击应用图标启动 Activity

理论上来说，桌面是一个**单独进程**，点击应用图标打开新的 Activity 属于上述第2种情况。
所以，上面的3种情况，可以归结为两类：

1. 启动自身进程内的 Activity
2. 启动其他进程内的 Activity

我们来看看 Android 系统是如何实现的。了解启动过程，需要理解 ActvitiyTaskManagerService 和 ActivityThread 的功能，以及 Binder 通讯机制，我们在下面会慢慢讲到。

> 注：本文源代码基于 Android API 29

## 源进程中的流程

启动自身进程的的 Activity 就是在代码中调用`startActivity()`或`startActivityForResult()`来启动目标 Activity。我们来看看`startActivity()`和`startActivityForResult()`都做了什么。

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```

可以看到，它调用了`startActivityForResult()`方法，只不过传入了`requestCode`为`-1`，表示『我不要结果』。

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
            // activity hidden during this time, to avoid flickering.
            // This can only be done when a result is requested because
            // that guarantees we will get information back when the
            // activity is finished, no matter what happens to it.
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            // Note we want to go through this method for compatibility with
            // existing applications that may have overridden it.
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

`startActivityForResult()`方法会先判断`mParent`的值是否为空，这个`mParent`也是一个 Activity 类型的变量，在`Activity.attach()`时被赋值，第一次启动 Activity 时必然为空。接下来就将任务转交给了`Instrumentation.execStartActivity()`方法。

mInstrumentation 也是在`Activity.attach()`中被初始化，由 ActivityThread 传入。我们来看看它的`execStartActivity()`方法：

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    try {
        ...
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

在这里，Instumentation 将任务通过 Binder 通信的方式交给了 ActivityTaskManagerService（下称ATMS）。盲猜这里是要先判断要启动的 Activity 是否已经有了相应任务栈，如果没有，就要创建新的任务栈，我们通过代码来看看它是否是这么做的。

## ActivityTaskManagerService 中的流程

```java
// com.android.server.wm.ActivityTaskManagerService.java

@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

@Override
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}

int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
        boolean validateIncomingUser) {
    enforceNotIsolatedCaller("startActivityAsUser");

    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setMayWait(userId)
            .execute();
}

ActivityStartController getActivityStartController() {
    return mActivityStartController;
}
```

ActivityStartController 是一个控制器，用来代理的 Activity 启动。它的主要目的是把 Activity 的启动请求进行整理，并交给 ActivityStarter 类。它的`obtainStarter()`方法就是获取一个 ActivityStarter 的实例。可见 ActivityStarter 使用的是 Builder 模式，最终的执行在它的`execute()`方法中：

```java
int execute() {
    try {
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                    mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup,
                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup,
                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        }
    } finally {
        onExecutionComplete();
    }
}
```

在 ActivityStarter 中有个内部类 Request，用来存储一次启动请求，包括传递的 Intent、请求者的身份、启动模式的 FLAG、requestCode 等等。这里单独讲一下`Request.mayWait`变量，这个变量表示是否要等待启动结果。当调用`Request.setMayWait()`方法时，这个值就会被置为`true`。在这个例子里面，我们在上面的代码里看到，它的值为`true`，所以会继续调用`startActivityMayWait()`方法：

```java
// com.android.server.wm.ActivityStarter.java

private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
    // 一系列判断 + 初始化
    ...
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
    ...
    synchronized (mService.mGlobalLock) {
        ...
        int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                allowBackgroundActivityStart);
        ...

        return res;
    }
}
```
`mSupervisor`是 ActivityStackSupervisor 类，从命名上看，是管理 Activity 栈的一个类，但其实这个类在不久的版本里，可能会被废除掉，把它的功能都移动到 ActivityLifeCycle、AMS 中。这里我们不用太在意，继续向下看`ActivityStarter.startActivity()`方法：

```java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        SafeActivityOptions options,
        boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
        TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
    ...
    // 各种判断与初始化
    ....
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, checkedOptions, sourceRecord);
    if (outActivity != null) {
        outActivity[0] = r;
    }
    ...
    final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
            true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);
    mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outActivity[0]);
    return res;
}

private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity, boolean restrictedBgActivity) {
    int result = START_CANCELED;
    final ActivityStack startedActivityStack;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
    } finally {
        ...
    }

    return result;
}

private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity, boolean restrictedBgActivity) {
    ...
    // 各种判断与初始化
    reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);
    ...
    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
            mOptions);
    ...
    return START_SUCCESS;
}
```

上面最后的代码中，`mTargetStack`是 ActivityStack 类型的变量，是在`setTargetStackAndMoveToFrontIfNeeded()`方法中被赋值的。我们继续来看看`ActivityStack.startActivityLocked()`：

```java
void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
        boolean newTask, boolean keepCurTransition, ActivityOptions options) {
    TaskRecord rTask = r.getTaskRecord();
    final int taskId = rTask.taskId;
    final boolean allowMoveToFront = options == null || !options.getAvoidMoveToFront();
    // mLaunchTaskBehind tasks get placed at the back of the task stack.
    if (!r.mLaunchTaskBehind && allowMoveToFront
            && (taskForIdLocked(taskId) == null || newTask)) {
        // Last activity in task had been removed or ActivityManagerService is reusing task.
        // Insert or replace.
        // Might not even be in.
        insertTaskAtTop(rTask, r);
    }
    TaskRecord task = null;
    if (!newTask) {
        // 如果不是新的任务栈，那就找到它所属的任务栈
        boolean startIt = true;
        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            task = mTaskHistory.get(taskNdx);
            if (task.getTopActivity() == null) {
                // All activities in task are finishing.
                continue;
            }
            if (task == rTask) {
                // 找到了对应的 TaskRecord，如果当前任务栈还未对用户可见，那就只添加，不启动
                if (!startIt) {
                    if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to task "
                            + task, new RuntimeException("here").fillInStackTrace());
                    r.createAppWindowToken();
                    ActivityOptions.abort(options);
                    return;
                }
                break;
            } else if (task.numFullscreen > 0) {
                startIt = false;
            }
        }
    }

    // 在栈顶放目标 Activity，下一个与用户交互的 Activity 就是它

    // 如果不把目标 Activity 放到最前，就不要给实际的当前最前的 Activity 发送 onUserLeaving 消息
    final TaskRecord activityTask = r.getTaskRecord();
    if (task == activityTask && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
        mStackSupervisor.mUserLeaving = false;
        if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                "startActivity() behind front, mUserLeaving=false");
    }

    task = activityTask;

    // Slot the activity into the history stack and proceed
    if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
            new RuntimeException("here").fillInStackTrace());
    // TODO: Need to investigate if it is okay for the controller to already be created by the
    // time we get to this point. I think it is, but need to double check.
    // Use test in b/34179495 to trace the call path.
    if (r.mAppWindowToken == null) {
        r.createAppWindowToken();
    }

    // 将 Activity 移到 task 最前
    task.setFrontOfTask();

    // 如果 allowMoveToFront 为 false，则不需要切换动画，因为 Activity 不可见
    if ((!isHomeOrRecentsStack() || numActivities() > 0) && allowMoveToFront) {
        final DisplayContent dc = getDisplay().mDisplayContent;
        if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                "Prepare open transition: starting " + r);
        if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
            dc.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
            mStackSupervisor.mNoAnimActivities.add(r);
        } else {
            int transit = TRANSIT_ACTIVITY_OPEN;
            if (newTask) {
                if (r.mLaunchTaskBehind) {
                    transit = TRANSIT_TASK_OPEN_BEHIND;
                } else if (getDisplay().isSingleTaskInstance()) {
                    transit = TRANSIT_SHOW_SINGLE_TASK_DISPLAY;
                } else {
                    // If a new task is being launched, then mark the existing top activity as
                    // supporting picture-in-picture while pausing only if the starting activity
                    // would not be considered an overlay on top of the current activity
                    // (eg. not fullscreen, or the assistant)
                    if (canEnterPipOnTaskSwitch(focusedTopActivity,
                            null /* toFrontTask */, r, options)) {
                        focusedTopActivity.supportsEnterPipOnTaskSwitch = true;
                    }
                    transit = TRANSIT_TASK_OPEN;
                }
            }
            dc.prepareAppTransition(transit, keepCurTransition);
            mStackSupervisor.mNoAnimActivities.remove(r);
        }
        boolean doShow = true;
        if (newTask) {
            // Even though this activity is starting fresh, we still need
            // to reset it to make sure we apply affinities to move any
            // existing activities from other tasks in to it.
            // If the caller has requested that the target task be
            // reset, then do so.
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                resetTaskIfNeededLocked(r, r);
                doShow = topRunningNonDelayedActivityLocked(null) == r;
            }
        } else if (options != null && options.getAnimationType()
                == ActivityOptions.ANIM_SCENE_TRANSITION) {
            doShow = false;
        }
        if (r.mLaunchTaskBehind) {
            // Don't do a starting window for mLaunchTaskBehind. More importantly make sure we
            // tell WindowManager that r is visible even though it is at the back of the stack.
            r.setVisibility(true);
            // 这里继续深入
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
            // Figure out if we are transitioning from another activity that is
            // "has the same starting icon" as the next one.  This allows the
            // window manager to keep the previous window it had previously
            // created, if it still had one.
            TaskRecord prevTask = r.getTaskRecord();
            ActivityRecord prev = prevTask.topRunningActivityWithStartingWindowLocked();
            if (prev != null) {
                // We don't want to reuse the previous starting preview if:
                // (1) The current activity is in a different task.
                if (prev.getTaskRecord() != prevTask) {
                    prev = null;
                }
                // (2) The current activity is already displayed.
                else if (prev.nowVisible) {
                    prev = null;
                }
            }
            r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
        }
    } else {
        // If this is the first activity, don't do any fancy animations,
        // because there is nothing for it to animate on top of.
        ActivityOptions.abort(options);
    }
}
```

这里的代码我没有精简，这段代码展示了对于 TaskRecord 及内部 Activity 顺序的处理。Activity 到现在还未进行创建，我们继续向下走，看`ensureActivitiesVisibleLocked()`方法：

```java
final void ensureActivitiesVisibleLocked(ActivityRecord starting, int configChanges,
        boolean preserveWindows, boolean notifyClients) {
    ...
    try {
        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            final TaskRecord task = mTaskHistory.get(taskNdx);
            final ArrayList<ActivityRecord> activities = task.mActivities;
            for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
                ...
                if (reallyVisible) {
                    ...
                    if (!r.attachedToProcess()) {
                        if (makeVisibleAndRestartIfNeeded(starting, configChanges, isTop,
                                resumeNextActivity, r)) {
                            if (activityNdx >= activities.size()) {
                                // Record may be removed if its process needs to restart.
                                activityNdx = activities.size() - 1;
                            } else {
                                resumeNextActivity = false;
                            }
                        }
                    } else if (r.visible) {
                        ...
                    } else {
                        ...
                    }
                    // Aggregate current change flags.
                    configChanges |= r.configChangeFlags;
                }
                ...
            }
            ...
        }
        ...
    } finally {
        ...
    }
}

private boolean makeVisibleAndRestartIfNeeded(ActivityRecord starting, int configChanges,
        boolean isTop, boolean andResume, ActivityRecord r) {
    if (isTop || !r.visible) {
        ...
        if (r != starting) {
            // We should not resume activities that being launched behind because these
            // activities are actually behind other fullscreen activities, but still required
            // to be visible (such as performing Recents animation).
            mStackSupervisor.startSpecificActivityLocked(r, andResume && !r.mLaunchTaskBehind,
                    true /* checkConfig */);
            return true;
        }
    }
    return false;
}
```

这里又出现了一个`mStackSupervisor`的变量，它是 ActivityStackSupervisor 类型，上面我们介绍过，我们看它的`startSpecificActivityLocked()`方法：

```java
// com.android.server.wm.ActivityStackSupervisor.java

void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Activity 所属的应用是否正在运行？
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    if (wpc != null && wpc.hasThread()) { // 判断它的 ActivityThread 是否为 null
        // 如果已经在运行，则开始创建 Activity 实例
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }

    // Suppress transition until the new activity becomes ready, otherwise the keyguard can
    // appear for a short amount of time before the new process with the new activity had the
    // ability to set its showWhenLocked flags.
    if (getKeyguardController().isKeyguardLocked()) {
        r.notifyUnknownVisibilityLaunched();
    }

    final boolean isTop = andResume && r.isTopRunningActivity();
    mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
}
```

好，接下来是重头戏，`realStartActivityLocked()`：

```java
 boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    // 一系列检查
    ...
    try {
        ...
        try {
            ...
            // 检查是否要输出 Transaction too large 的日志，面熟吗？
            logIfTransactionTooLarge(r.intent, r.icicle);

            // 创建启动 Activity 的事务
            final ClientTransaction clientTransaction = ClientTransaction.obtain(
                    proc.getThread(), r.appToken);

            final DisplayContent dc = r.getDisplay().mDisplayContent;
            clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                    System.identityHashCode(r), r.info,
                    // TODO: Have this take the merged configuration instead of separate global
                    // and override configs.
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                    r.icicle, r.persistentState, results, newIntents,
                    dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                            r.assistToken));

            // 设置最终的生命周期状态
            final ActivityLifecycleItem lifecycleItem;
            // 创建下一个生命周期事务
            if (andResume) {
                lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
            } else {
                lifecycleItem = PauseActivityItem.obtain();
            }
            clientTransaction.setLifecycleStateRequest(lifecycleItem);

            // 安排上了。
            mService.getLifecycleManager().scheduleTransaction(clientTransaction);
            ...
        } catch (RemoteException e) {
            ...
        }
    } finally {
        ...
    }
    ...
    return true;
}
```

从 Android 26 之后，引入了事务机制，并将 Activity 的生命周期管理都交给一个名为 ClientLifecycleManager 的工具类来进行辅助管理。ClientTransaction 指定要执行的任务类型（如 LaunchActivityItem）及所在进程，ClientLifecycleManager 负责传递消息。

上面代码中，mService 是 ATMS 的实例，在构造 ActivityStackSupervisor 实例时由外部传入。`ActivityTaskManagerService.getLifecycleManager()`返回的即是 ATMS 中 ClientLifecycleManager 的实例，我们来看看`ClientLifecycleManager.scheduleTransaction()`都做了些什么：

```java
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

哦？又交还给 ClientTransaction 自己去 schedule 了：

```java
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

`mClient` 自然就是 ApplicationThread 的实例，通过 Binder 方式远程调用了它的`scheduleTransaction()`方法。

至此，Activity 启动的请求就交还给了目标进程的 ApplicationThread。

## ActivityThread 中的流程

接下来的部分就比较好理解了，我们继续看看`ApplicationThread.scheduleTransaction()`：

```java
// android.app.ActivityThread.ApplicationThread.java

private class ApplicationThread extends IApplicationThread.Stub {
    public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
    }
}

// android.app.ActivityThread.java

public final class ActivityThread extends ClientTransactionHandler {
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
}
```

H 是 ActivityThread 中的一个内部类，它继承自 Handler。这里调用了`sendMessage()`方法，企图向这个 Handler 发送消息：

```java
void sendMessage(int what, Object obj) {
    sendMessage(what, obj, 0, 0, false);
}

private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}
```

`mH`自然就是 H 的实例。我们来看看它是如何处理`EXECUTE_TRANSACTION`消息的：

```java
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
```

又蹦出个东西叫 TransactionExecutor，它的功能是让事务能够按照正确的顺序执行。

```java
// android.app.servertransaction.TransactionExecutor.java

public void execute(ClientTransaction transaction) {
    ...
    executeCallbacks(transaction);
    // 如果指定的最终的生命周期状态，那就再继续执行事务直到指定状态
    executeLifecycleState(transaction);
    ...
}

public void executeCallbacks(ClientTransaction transaction) {
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
    ...
    final IBinder token = transaction.getActivityToken();
    ...
    final int size = callbacks.size();
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        ...
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
        if (r == null) {
            // Launch activity request will create an activity record.
            r = mTransactionHandler.getActivityClient(token);
        }

        if (postExecutionState != UNDEFINED && r != null) {
            // Skip the very last transition and perform it by explicit state request instead.
            final boolean shouldExcludeLastTransition =
                    i == lastCallbackRequestingState && finalState == postExecutionState;
            cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
        }
    }
}
```

这里看到了，是拿到事务的 callback，并进行回调。调用它的`execute()`方法来真正执行操作。

那么这个 callback 是什么时候设置的呢？向上翻翻代码，在 ClientTransaction 创建时，有这样的一句代码：

```java
clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
        System.identityHashCode(r), r.info,
        // TODO: Have this take the merged configuration instead of separate global
        // and override configs.
        mergedConfiguration.getGlobalConfiguration(),
        mergedConfiguration.getOverrideConfiguration(), r.compat,
        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
        r.icicle, r.persistentState, results, newIntents,
        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                r.assistToken));
```

也就是说，在『启动 Activity』这个事务中，callback 是 LaunchActivityItem 的实例。我们来看看这个类：

```java
public class LaunchActivityItem extends ClientTransactionItem {
    private Intent mIntent;
    ...
    private ActivityInfo mInfo;
    ...
    private Bundle mState;
    private PersistableBundle mPersistentState;
    private List<ResultInfo> mPendingResults;
    ...

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        ...
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        ...
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        client.countLaunchingActivities(-1);
    }
}
```

在`execute()`方法中，首先构造了一个 ActivityClientRecord 类型的变量`r`，然后将它交给了 ClientTransactionHandler。这个类是啥呢？它是 ActivityThread 的父类。这个`client`也即`TransactionExecutor.mTransactionHandler`，而`mTransactionHander`是在 TransactionExecutor 构造时传入的，我们看一眼：

```java
public final class ActivityThread extends ClientTransactionHandler {
    ...
    private final TransactionExecutor mTransactionExecutor = new TransactionExecutor(this);
    ...
}
```

转来转去，又回到了 ActivityThread 中。

```java
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
    // 初始化 WindowManagerGlobal
    // 这个类用来与 WindowManagerService 进行通讯
    WindowManagerGlobal.initialize();
    ...
    final Activity a = performLaunchActivity(r, customIntent);
    ...
    return a;
}

/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    ContextImpl appContext = createBaseContextForActivity(r);
    // 创建 Activity 实例
    Activity activity = null;
    try {
        ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
        // 如果应用未启动，则要创建应用的 Application
        // 如果创建过了，则直接返回已创建的实例
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
        if (localLOGV) Slog.v(
                TAG, r + ": app=" + app
                + ", appName=" + app.getPackageName()
                + ", pkg=" + r.packageInfo.getPackageName()
                + ", comp=" + r.intent.getComponent().toShortString()
                + ", dir=" + r.packageInfo.getAppDir());

        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            // 调用 Activity.attach() 方法，对 Activity 进行初始化，比如创建 PhoneWindow，绑定 ActivityThread 等
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            checkAndBlockForNetworkAccess();
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            // 调用 Activity 生命周期方法，完成创建
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
        }
        r.setState(ON_CREATE);

        // updatePendingActivityConfiguration() reads from mActivities to update
        // ActivityClientRecord which runs in a different thread. Protect modifications to
        // mActivities to avoid race.
        synchronized (mResourcesManager) {
            mActivities.put(r.token, r);
        }

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to start activity " + component
                + ": " + e.toString(), e);
        }
    }

    return activity;
}
```

上面的代码分为三大步：

1. 利用 Instrumentation 反射创建 Activity 的实例；
2. 调用`Activity.attach()`对 Activity 进行 Context 绑定、创建 PhoneWindow 实例等；
3. 调用`Instrumentation.callActivityOnCreate()`从而调用到 Activity 的`onCreate()`生命周期方法。

那么，Activity 的`onCreate()`被调用完成后，还会继续调用`onStart()`和`onResume()`呢，它们是在哪里被调用的呢？

还记得上面有这样一段吗？

```java
// 如果指定的最终的生命周期状态，那就再继续执行事务直到指定状态
executeLifecycleState(transaction);
```

这里就执行了创建 ClientTransactionItem 请求时，顺带创建的 ActivityLifecycleItem 请求。在这个例子里，`transaction`是 ResumeActivityItem 的实例。

经过与上面相同的步骤，最终会来到 ActivityThread 的`handleResumeActivity()`方法中：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...
    final Activity a = r.activity;
    ...
    // If the window hasn't yet been added to the window manager,
    // and this guy didn't finish itself or start another activity,
    // then go ahead and add the window.
    boolean willBeVisible = !a.mStartedActivity;
    if (!willBeVisible) {
        try {
            willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(
                    a.getActivityToken());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        if (r.mPreserveWindow) {
            a.mWindowAdded = true;
            r.mPreserveWindow = false;
            // Normally the ViewRoot sets up callbacks with the Activity
            // in addView->ViewRootImpl#setView. If we are instead reusing
            // the decor view we have to notify the view root that the
            // callbacks may have changed.
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }

        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }

    // Get rid of anything left hanging around.
    cleanUpPendingRemoveWindows(r, false /* force */);

    // The window is now visible if it has been added, we are not
    // simply finishing, and we are not starting another activity.
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        if (r.newConfig != null) {
            performConfigurationChangedForActivity(r, r.newConfig);
            if (DEBUG_CONFIGURATION) {
                Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                        + r.activity.mCurrentConfig);
            }
            r.newConfig = null;
        }
        if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
        WindowManager.LayoutParams l = r.window.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                != forwardBit) {
            l.softInputMode = (l.softInputMode
                    & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                    | forwardBit;
            if (r.activity.mVisibleFromClient) {
                ViewManager wm = a.getWindowManager();
                View decor = r.window.getDecorView();
                wm.updateViewLayout(decor, l);
            }
        }

        r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        if (r.activity.mVisibleFromClient) {
            // 让 Activity 可见的重要方法
            r.activity.makeVisible();
        }
    }
}
```

这个方法也是比较重要的一个方法，它会调用`performResumeActivity()`方法，然后初始化一些 Window 相关的数据，并最终通过调用 `Activity.makeVisible()`方法让 Activity 可见。

```java
public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
        String reason) {
    final ActivityClientRecord r = mActivities.get(token);
    ...
    try {
        ...
        r.activity.performResume(r.startsNotResumed, reason);
    } catch (Exception e) {
        ...
    }
}
```

Activity 的`performResume()`如下：

```java
final void performResume(boolean followedByPause, String reason) {
    ...
    performRestart(true /* start */, reason); // 
    ...
    // mResumed is set by the instrumentation
    mInstrumentation.callActivityOnResume(this);
    ...
    onPostResume();
    ...
    dispatchActivityPostResumed();
}

final void performRestart(boolean start, String reason) {
    ...
    if (mStopped) {
        ...
        mInstrumentation.callActivityOnRestart(this);
        ...
        if (start) {
            performStart(reason);
        }
    }
}

final void performStart(String reason) {
    ...
    mInstrumentation.callActivityOnStart(this);
    ...
    dispatchActivityPostStarted();
}
```

Activity 生命周期流程得到了佐证，会由`onCreate()`→`onRestart()`→`onStart()`→`onResume()`

真的是很长的一个流程。我们简单总结一下：

1. 源进程调用`startActivity()`，最终会调用到`Instrumentation.execStartActivity()`方法；
2. 通过 Binder 调用 ActivityTaskManagerService 的`startActivity()`方法；
3. ATMS 会使用 ActivityStack 类和 ActivityStackSupervisor 类来处理 Task 与 Activity 的入栈操作；
4. 接着在`ActivityStackSupervisor.realStartActivityLocked()`方法中，使用提交事务的方法，创建一个启动 Activity 的事务，并提交给 ClientLifecycleManager；
5. ClientLifecycleManager 会通过 Binder 调用 ActivityThread 的`scheduleTransaction()`方法，该方法会向 ActivityThread 中的 Handler 发送一个消息；
6. Handler 接收消息后，利用 TransactionExecutor 按顺序执行事务；
7. 在这个『启动 Activity』的事务回调中，又调用了 ActivityThread 的`handleLaunchActivity()`方法，进而又调用了`performLaunchActivity()`方法；
8. 在这个方法中，利用反射创建了 Activity 的实例，如果需要的话，还利用反射创建了 Application 的实例，并调用`Activity.attach()`方法，创建 PhoneWindow、绑定 Context；
9. 最后调用了`Instrumentation.callActivityOnCreate()`方法，执行了 Activity 的`onCreate()`生命周期方法；
10. 接着 TransactionExecutor 又开始执行生命周期事务，最后依次调用到`onStart()`、`onResume()`。


## 注意事项

本文代码分析基于 Android 10.0，在 10.0 中，出现了 ATMS，AMS 中原来的`startActivity()`全部由 ATMS 来执行。

可以从下面链接中观察两者的不同：
[10.0.0_r30 中的 ActivityManagerService](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=3521;bpv=0;bpt=1)
[9.0.0_r34 中的 ActivityManagerService](https://cs.android.com/android/platform/superproject/+/android-9.0.0_r34:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=5088;bpv=0;bpt=1)