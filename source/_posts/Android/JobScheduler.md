---
title: JobScheduler 和 WorkManager
link: jobscheduler-and-workmanager
cover: http://blog.cigis-cloud.com/android-1593080864.jpg
toc: true
category: 
    - Android
    - Development
tag: 
    - Android
    - Jetpack
date: 2020-04-26
---

这篇文章我们来讲讲 JobScheduler 和 WorkManager。

<!-- more -->

## JobScheduler

JobScheduler 是 Google 在 API 21（Android 5.0）中推出的一种后台执行任务的方式。因为普通的 Service 会在后台一直运行，只有系统资源不足时才会将其回收，哪怕这个 Service 现在没有在做任何事情，这显然会对系统资源造成相当大的浪费，毕竟启动一个 Service 对开发者来说没什么成本是吧？

JobScheduler 提供的是**根据某个条件来执行任务**，而不是在某个时间来执行。它能确保把你的任务执行了，但是不能告诉你会在什么时候执行。<img class="sticker" src="http://img.doutula.com/production/uploads/image/2017/11/12/20171112500197_UKzaih.jpg" />

这让我想起了一句名言：

> 我会还你钱的，但我没说什么时候还你。

当然，情况没有那么严重。系统只是会**智能分配**目前所有的用 JobScheduler 方式提交的任务，并尝试批量执行，并且能推迟多久就推迟多久。这么说吧，你要是不给这个 Job 指定一个 deadline，那它真的可能会在很久之后才会执行。

那这玩意这么不靠谱，我们为什么还要用它呢？

因为随着 Android 版本的不断更迭，系统也逐渐变得流畅，这背后是对 App 更多的限制：限制后台进程的运行时间、限制反射可调用的接口、限制某些耗电行为、限制你可以接收到的广播。。。Android 必须也要推出一种行之有效的方案，让 App 能够在某种条件下，完成某种任务。JobScheduler 就应运而生了。

### JobScheduler 的使用

要使用 JobScheduler，必须要先了解它的一家人：

- JobInfo：提交任务时要传递的参数，包含了任务概要、执行条件、deadline 等。
- JobService：当系统调度到你这个任务时，要如何处理任务，你要自己去实现。
- JobScheduler：用于向系统提交任务。

它的使用很简单，可以分为三步：

### 1. 创建 JobService 的子类

JobService 继承自 Service，也是抽象类，所以我们必须要自己写一个子类：

> 注意！这是一个 Sevice，意味着它是在**主线程**中被回调的！所以，这里如果要访问网络进行密集、耗时的操作，还是需要另起线程来做。

```java
public class CustomJobService extends JobService {
    @Override
    public boolean onStartJob(JobParameters params) {
        jobFinished(params, true);
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        return false;
    }
}
```

它只需要实现两个方法：`onStartJob(JobParameters)`和`onStopJob(JobParameters)`。顾名思义，就是任务开始的回调和任务停止的回调。还有一个重要的方法是`jobFinished(JobParameters, boolean)`，我们解释一下这三个方法。

- `onStartJob()`：传入了 JobParameters 类型的变量，它包含了`jobId`和提交任务时创建 JobInfo 时传入的 Bundle 等数据。在任务执行完成时，需要调用`jobFinished()`来告诉系统：我的任务执行完成了。它的返回值是一个 boolean：
  - 返回`true`就是告诉系统，我还在执行任务，一会任务完成了，我会调用`jobFinished()`来告诉你的。但是，如果在任务执行期间，任务的执行环境发生了变更，比如在提交任务时指定『充电时执行』，但用户这时候把充电线拔掉了，那就会**立刻结束任务**，并调用`onStopJob()`；
  - 返回`false`则告诉系统，我执行完了。系统也不会再去调用`onStopJob()`方法。
  
- `jobFinished()`：只能在`onStartJob()`中调用，这个方法是被 final 修饰的，不能被覆写。它有两个参数，第一个参数是 JobParameters，第二个是 boolean 类型的参数，传入`true`是希望系统能根据创建任务时的标准**再次将这个任务提上日程**，传入`false`则相反。

- `onStopJob()`：当这个方法被调用时，表示任务要被强制中断了。这通常发生在任务执行条件不满足时。这个方法执行完毕后，系统就会释放 Wakelock。这个方法的返回值也是一个 boolean：如果返回`true`表示『教练，我想再试试🏀』，教练会在合适的时间再安排你打篮球（大误）；返回`false`表示『我不行了☠️』，教练扭头就走。但不管你返回什么，任务是肯定会被终止的。

在`onStartJob()`被调用之前，系统会给 App 绑定一个 Wakelock，以防止系统进入休眠状态，这个 Wakelock 会在你调用`jobFinished()`或者系统调用`onStopJob()`时被释放掉。

### 2. 创建 JobInfo 

JobInfo 是一个 Parcelable，显然是用于 IPC 的。它采用了 Builder 设计模式来创建实例，使用起来也比较简单：

```java
public static final int JOB_ID = 0;

JobInfo jobInfo = new JobInfo.Builder(
    JOB_ID,
    new ComponentName(context, CustomJobService.class))
        .setRequiredNetWork()
        .setRequireCharging()
        .build();
```

可以看到，给这个 JobInfo 设置了要处理任务的类，设置了 jobId，设置了执行任务的条件等。

它的 Builder 中还有很多其他的方法，下面是它所有的方法：

![](/img/job-1589006675.png)

### 3. 提交任务

JobScheduler 是系统服务，不能直接被实例化，要通过`Context.getSystemService()`来获取。

```java
JobScheduler js = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
js.schedule(jobInfo);
```

至此，一个任务就被『安排上了』。<img class="sticker" src="/img/anpai.jpg" />


JobScheduler 中最常用的是`schedule()`方法，用于提交任务。多次调用的话，如果系统中已经有相同 jobId 的任务，则会用新的直接替换掉旧任务，如果旧任务当前正在运行，那它**会被终止**。

它还有一个方法是`enqueue(JobInfo, JobWorkItem)`，用来给某个任务（新任务或者已存在任务皆可）插入工作内容（JobWorkItem）。如果系统中已经有相同 jobId 的任务，则会用新的直接替换旧任务，如果旧任务当前正在运行，传入的 JobWorkItem 会被**插入任务的工作队列**，任务**并不会被终止**。虽然不会终止任务，但还是**强烈建议使用同一个 JobInfo**来插入工作内容，这样，系统就不会分配额外的资源去更改 JobInfo，从而能更专注于执行任务。

## JobScheduler 实现原理

既然是系统服务，那必然是遵循老套路，得有个类似 JobSchedulerService 之类的东西，并且由 system_server 进程启动，我们直接扒 SystemServer 类的代码：

```java
// android.server.SystemServer.java

private void startOtherServices() {
    ...
    traceBeginAndSlog("StartJobScheduler");
    mSystemServiceManager.startService(JobSchedulerService.class);
    traceEnd();
    ...
}
```

✌️✌️✌️果然如此，接下来就分析一下 JobSchedulerService 的代码：

```java

```

## WorkManager

WorkManager 是 Android Jetpack 中的一个组件。它提供了更为强大的功能，包括：

- 向下兼容到 API 14
- 可以添加工作条件约束：比如有 Wifi 连接、正在充电等等（这很合适在后台下载 App 的更新包）
- 可以执行一次性的或者周期性的任务
- 能够监听和管理已计划的任务
- 将多个任务链接起来（妙啊）
- 保证任务一定会被执行，哪怕 App 重启甚至设备重启
- 遵循手机的省电模式

WorkManager 适合运行那种**非即时的、确定要执行**的任务，比如说在某个时间段向后台发送日志、同步数据，或者定期检查 App 版本情况并下载新版本等等。



While a job is running, the system holds a wakelock on behalf of your app. For this reason, you do not need to take any action to guarantee that the device stays awake for the duration of the job.

You do not instantiate this class directly; instead, retrieve it through Context.getSystemService(Context.JOB_SCHEDULER_SERVICE).