---
title: 关于 Leak Canary
link: leakcanary
tag: [Android, leakcanary, 源码解析]
category: [Android]
cover: https://d3h2k7ug3o5pb3.cloudfront.net/image/2020-12-04/a881a490-360f-11eb-bcbe-7daa1ab28fd4.jpg
---

> “A small leak will sink a great ship.” - Benjamin Franklin

这篇文章来讲讲著名的第三方内存泄漏检测工具 - LeakCanary。

> 注：本文源代码基于 leakcanary-android:2.3

<!-- more -->

LeakCanary 的使用非常简单，我们一笔带过：

```
dependencies {
  // debugImplementation because LeakCanary should only run in debug builds.
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.3'
}
```

配置完 gradle 依赖后，任何代码都不用写，就能直接生效。

神奇吗？神奇就对了。

官方文档上提到，LeakCanary 能够自动检测下面几种对象：

- 销毁之后的 Activity 实例
- 销毁之后的 Fragment 实例
- 销毁之后的 Fragment View 实例
- 清除之后的 ViewModel 实例

在这几种行为发生后，LeanCanary 会做四个步骤来帮助我们分析是否有内存泄漏：

1. 检查是否有被保留下的对象
2. Dump Heap
3. 分析 HPROF 文件
4. 对泄漏的部分进行分类

我们来按步骤分析分析它的源码，看它究竟是如何做到的。

## 检查要被保留的对象

LeanCanary 会监测 Android 组件（并不是全部）的生命周期，在它们销毁的时候，会检测它们是否出现了内存泄漏。

这些对象会被传递到一个 ObjectWatcher 类，它使用 WeakReference 来持有这些实例。当然，你也可以使用这个类来自己观察某个对象是否出现了泄漏。

它的代码如下所示：

```kotlin
/**
 * [ObjectWatcher] can be passed objects to [watch]. It will create [KeyedWeakReference] instances
 * that reference watches objects, and check if those references have been cleared as expected on
 * the [checkRetainedExecutor] executor. If not, these objects are considered retained and
 * [ObjectWatcher] will then notify the [onObjectRetainedListener] on that executor thread.
 *
 * [checkRetainedExecutor] is expected to run its tasks on a background thread, with a significant
 * to give the GC the opportunity to identify weakly reachable objects.
 *
 * [ObjectWatcher] is thread safe.
 */
// Thread safe by locking on all methods, which is reasonably efficient given how often
// these methods are accessed.
class ObjectWatcher constructor(
  private val clock: Clock,
  private val checkRetainedExecutor: Executor,
  /**
   * Calls to [watch] will be ignored when [isEnabled] returns false
   */
  private val isEnabled: () -> Boolean = { true }
) {

  private val onObjectRetainedListeners = mutableSetOf<OnObjectRetainedListener>()

  /**
   * References passed to [watch].
   */
  private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()

  private val queue = ReferenceQueue<Any>()

  ...

  /**
   * Returns the objects that are currently considered retained. Useful for logging purposes.
   * Be careful with those objects and release them ASAP as you may creating longer lived leaks
   * then the one that are already there.
   */
  val retainedObjects: List<Any>
    @Synchronized get() {
      removeWeaklyReachableObjects()
      val instances = mutableListOf<Any>()
      for (weakReference in watchedObjects.values) {
        if (weakReference.retainedUptimeMillis != -1L) {
          val instance = weakReference.get()
          if (instance != null) {
            instances.add(instance)
          }
        }
      }
      return instances
    }

  ...

  /**
   * Watches the provided [watchedObject].
   *
   * @param description Describes why the object is watched.
   */
  @Synchronized fun watch(
    watchedObject: Any,
    description: String
  ) {
    if (!isEnabled()) {
      return
    }
    removeWeaklyReachableObjects()
    val key = UUID.randomUUID()
        .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
          (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
          (if (description.isNotEmpty()) " ($description)" else "") +
          " with key $key"
    }

    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }

  ...
}
```

可以看到，ObjectWatcher 内部有这么几个重要的东西：

- `onObjectRetainedListeners`，使用者可以手动添加 OnObjectRetainedListener，当有对象被保留时，可以接收到回调；
- `watchedObjects`，持有对要观察的对象的 WeakReference。可以看到 LeakCanary 使用了 KeyedWeakReference，它是 WeakReference 的子类，添加了一个字段用于保存对象被观测的时长；
- `queue`，[ReferenceQueue](https://docs.oracle.com/javase/8/docs/api/java/lang/ref/ReferenceQueue.html)，这是 Java 中的特性，当内存中对象的可达性发生变化时，GC 会将变化的对象添加到这个队列中；
- `retainedObjects`，可能会发生内存泄漏的对象就保存在这个 List 中，供后续分析使用。

默认情况下，LeakCanary 会将应用中的 Activity、Fragment 等组件注册到 ObjectWatcher 中，那么问题来了，是在何时注册的？

通过扒拉代码，我发现了位于 leakcanary/internal/ 中的这个类：AppWatcherInstaller。我们看一下它的代码：

```kotlin
/**
 * Content providers are loaded before the application class is created. [AppWatcherInstaller] is
 * used to install [leakcanary.AppWatcher] on application start.
 */
internal sealed class AppWatcherInstaller : ContentProvider() {
  ...
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    InternalAppWatcher.install(application)
    return true
  }
  ...
}
```

厉害了，利用了 Content Provider 会早于 Application 类加载的这个特性，在 `onCreate()` 方法中调用了 `InternalAppWatcher.install()` 方法进行初始化的工作：

```kotlin
internal object InternalAppWatcher {

  private val onAppWatcherInstalled: (Application) -> Unit

  ...

  init {
    val internalLeakCanary = try {
      val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
      leakCanaryListener.getDeclaredField("INSTANCE").get(null)
    } catch (ignored: Throwable) {
      NoLeakCanary
    }
    @kotlin.Suppress("UNCHECKED_CAST")
    onAppWatcherInstalled = internalLeakCanary as (Application) -> Unit
  }

  ...

  val objectWatcher = ObjectWatcher(
      clock = clock,
      checkRetainedExecutor = checkRetainedExecutor,
      isEnabled = { AppWatcher.config.enabled }
  )

  fun install(application: Application) {
    SharkLog.logger = DefaultCanaryLog()
    checkMainThread()
    if (this::application.isInitialized) {
      return
    }
    InternalAppWatcher.application = application

    val configProvider = { AppWatcher.config }
    ActivityDestroyWatcher.install(application, objectWatcher, configProvider)
    FragmentDestroyWatcher.install(application, objectWatcher, configProvider)
    onAppWatcherInstalled(application)
  }

  ...

  private fun checkMainThread() {
    if (Looper.getMainLooper().thread !== Thread.currentThread()) {
      throw UnsupportedOperationException(
          "Should be called from the main thread, not ${Thread.currentThread()}"
      )
    }
  }

  ...
}
```

`InternalAppWatcher.install()` 方法中，首先检查了当前代码是否运行在主线程上，如果不是，则抛出异常；然后分别调用了 ActivityDestroyWatcher 和 FragmentDestroyWatcher 的 `install()` 方法，当这些事情都做完后，用了一个秀儿的方法，给 `internalLeakCanary` 这个成员变量赋值。

![](https://s3.ax1x.com/2020/12/11/rAhsB9.md.jpg)

来看 ActivityDestroyWatcher：

```kotlin
internal class ActivityDestroyWatcher private constructor(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
          objectWatcher.watch(
              activity, "${activity::class.java.name} received Activity#onDestroy() callback"
          )
        }
      }
    }

  companion object {
    fun install(
      application: Application,
      objectWatcher: ObjectWatcher,
      configProvider: () -> Config
    ) {
      val activityDestroyWatcher = ActivityDestroyWatcher(objectWatcher, configProvider)
      application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
    }
  }
}
```

它的功能非常简单，直接创建 ActivityDestroyWatcher 的实例，并利用 Application 中的 ActivityLifecycleCallbacks 接口，接收 Activity 的 `onActivityDestroyed()` 回调，然后将 Activity 交给 ObjectWatcher。

> kotlin 中的 by 一般用于实现委托机制，在这里的功能相当于可以省略掉其他的回调的代码。
> noOpDelegate 的代码如下：
> ```kotlin
> inline fun <reified T : Any> noOpDelegate(): T {
    val javaClass = T::class.java
    val noOpHandler = InvocationHandler { _, _, _ ->
      // no op
    }
    return Proxy.newProxyInstance(
        javaClass.classLoader, arrayOf(javaClass), noOpHandler
    ) as T
  }
> ```

FragmentDestroyWatcher 稍微复杂一些：

```kotlin
internal object FragmentDestroyWatcher {

  private const val ANDROIDX_FRAGMENT_CLASS_NAME = "androidx.fragment.app.Fragment"
  private const val ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME = "leakcanary.internal.AndroidXFragmentDestroyWatcher"

  // Using a string builder to prevent Jetifier from changing this string to Android X Fragment
  @Suppress("VariableNaming", "PropertyName")
  private val ANDROID_SUPPORT_FRAGMENT_CLASS_NAME = StringBuilder("android.").append("support.v4.app.Fragment").toString()
  private const val ANDROID_SUPPORT_FRAGMENT_DESTROY_WATCHER_CLASS_NAME = "leakcanary.internal.AndroidSupportFragmentDestroyWatcher"

  fun install(
    application: Application,
    objectWatcher: ObjectWatcher,
    configProvider: () -> AppWatcher.Config
  ) {
    val fragmentDestroyWatchers = mutableListOf<(Activity) -> Unit>()

    if (SDK_INT >= O) {
      fragmentDestroyWatchers.add(
          AndroidOFragmentDestroyWatcher(objectWatcher, configProvider)
      )
    }

    getWatcherIfAvailable(
        ANDROIDX_FRAGMENT_CLASS_NAME,
        ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
        objectWatcher,
        configProvider
    )?.let {
      fragmentDestroyWatchers.add(it)
    }

    getWatcherIfAvailable(
        ANDROID_SUPPORT_FRAGMENT_CLASS_NAME,
        ANDROID_SUPPORT_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
        objectWatcher,
        configProvider
    )?.let {
      fragmentDestroyWatchers.add(it)
    }

    if (fragmentDestroyWatchers.size == 0) {
      return
    }

    application.registerActivityLifecycleCallbacks(object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
        for (watcher in fragmentDestroyWatchers) {
          watcher(activity)
        }
      }
    })
  }

  private fun getWatcherIfAvailable(
    fragmentClassName: String,
    watcherClassName: String,
    objectWatcher: ObjectWatcher,
    configProvider: () -> AppWatcher.Config
  ): ((Activity) -> Unit)? {

    return if (classAvailable(fragmentClassName) &&
        classAvailable(watcherClassName)
    ) {
      val watcherConstructor = Class.forName(watcherClassName)
          .getDeclaredConstructor(ObjectWatcher::class.java, Function0::class.java)
      @Suppress("UNCHECKED_CAST")
      watcherConstructor.newInstance(objectWatcher, configProvider) as (Activity) -> Unit

    } else {
      null
    }
  }

  private fun classAvailable(className: String): Boolean {
    return try {
      Class.forName(className)
      true
    } catch (e: Throwable) {
      // e is typically expected to be a ClassNotFoundException
      // Unfortunately, prior to version 25.0.2 of the support library the
      // FragmentManager.FragmentLifecycleCallbacks class was a non static inner class.
      // Our AndroidSupportFragmentDestroyWatcher class is compiled against the static version of
      // the FragmentManager.FragmentLifecycleCallbacks class, leading to the
      // AndroidSupportFragmentDestroyWatcher class being rejected and a NoClassDefFoundError being
      // thrown here. So we're just covering our butts here and catching everything, and assuming
      // any throwable means "can't use this". See https://github.com/square/leakcanary/issues/1662
      false
    }
  }
}
```

对 Fragment 的监控是主要采用了两个类：`leakcanary.internal.AndroidXFragmentDestroyWatcher` 和 `leakcanary.internal.AndroidSupportFragmentDestroyWatcher`。这两个类的实现方式相同，都是 lambda 表达式，我们只看其中一个：

```kotlin
internal class AndroidXFragmentDestroyWatcher(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) : (Activity) -> Unit {

  private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

    override fun onFragmentCreated(
      fm: FragmentManager,
      fragment: Fragment,
      savedInstanceState: Bundle?
    ) {
      ViewModelClearedWatcher.install(fragment, objectWatcher, configProvider)
    }

    override fun onFragmentViewDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      val view = fragment.view
      if (view != null && configProvider().watchFragmentViews) {
        objectWatcher.watch(
            view, "${fragment::class.java.name} received Fragment#onDestroyView() callback " +
            "(references to its views should be cleared to prevent leaks)"
        )
      }
    }

    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      if (configProvider().watchFragments) {
        objectWatcher.watch(
            fragment, "${fragment::class.java.name} received Fragment#onDestroy() callback"
        )
      }
    }
  }

  override fun invoke(activity: Activity) {
    if (activity is FragmentActivity) {
      val supportFragmentManager = activity.supportFragmentManager
      supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
      ViewModelClearedWatcher.install(activity, objectWatcher, configProvider)
    }
  }
}
```

利用了 FragmentManager 中的 FragmentLifecycleCallbacks 接口实现了对 Fragment 生命周期的监听，同时还对 ViewModel 的销毁事件做了监听。

当 Activity 走完自己的 `onDestroy()` 方法后，Watcher 中的 watch() 方法会被执行（上方代码），然后，利用了 Executor （注意，该 Executor 是在主线程上跑的）开始执行`moveToRetained(key)`代码：

```kotlin
@Synchronized private fun moveToRetained(key: String) {
  removeWeaklyReachableObjects()
  val retainedRef = watchedObjects[key]
  if (retainedRef != null) {
    retainedRef.retainedUptimeMillis = clock.uptimeMillis()
    onObjectRetainedListeners.forEach { it.onObjectRetained() }
  }
}
```

首先移除了所有在 GC 后依旧有可达性的对象，如果全部都移除了，`retainedRef` 就是 `null`，则不会触发回调，证明没有内存泄漏；如果移除不了，说明有内存泄漏，开始进行下一步：Dump Heap。

## Dump Heap

书接上文，Watcher 开始回调所有的 `onObjectRetainedListeners`。这些 Listeners 是在何时被添加的呢？

我们回到初始化的部分，也即`InternalAppWatcher.install()`，其中有这样的代码：

```kotlin
init {
    val internalLeakCanary = try {
      val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
      leakCanaryListener.getDeclaredField("INSTANCE").get(null)
    } catch (ignored: Throwable) {
      NoLeakCanary
    }
    @kotlin.Suppress("UNCHECKED_CAST")
    onAppWatcherInstalled = internalLeakCanary as (Application) -> Unit
  }
```

当 `onAppWatcherInstalled` 这个 Lambda 表达式被执行时，`leakcanary.internal.InternalLeakCanary` 这个类会被初始化，我们看一下这个类：

```kotlin
internal object InternalLeakCanary : (Application) -> Unit, OnObjectRetainedListener {
  private lateinit var heapDumpTrigger: HeapDumpTrigger
  lateinit var application: Application

  override fun invoke(application: Application) {
    this.application = application

    checkRunningInDebuggableBuild()

    AppWatcher.objectWatcher.addOnObjectRetainedListener(this)

    val heapDumper = AndroidHeapDumper(application, leakDirectoryProvider)

    val gcTrigger = GcTrigger.Default

    val configProvider = { LeakCanary.config }

    val handlerThread = HandlerThread(LEAK_CANARY_THREAD_NAME)
    handlerThread.start()
    val backgroundHandler = Handler(handlerThread.looper)

    heapDumpTrigger = HeapDumpTrigger(
        application, backgroundHandler, AppWatcher.objectWatcher, gcTrigger, heapDumper,
        configProvider
    )
    application.registerVisibilityListener { applicationVisible ->
      this.applicationVisible = applicationVisible
      heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
    }
    registerResumedActivityListener(application)
    addDynamicShortcut(application)

    disableDumpHeapInTests()
  }

  ...

  override fun onObjectRetained() {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.onObjectRetained()
    }
  }

  fun onDumpHeapReceived(forceDump: Boolean) {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.onDumpHeapReceived(forceDump)
    }
  }

  ...

}
```

好家伙，我 TM 直接好家伙，这居然又是个 Lambda 表达式，话不多说，直接看 `invoke()` 回调部分中，有 `AppWatcher.objectWatcher.addOnObjectRetainedListener(this)` 这样一句，将自己做为被回调方添加到了 watcher 中，那么上面的 `onObjectRetainedListener` 就找到了实现者。

可以看到，调用了 `heapDumpTrigger.onObjectRetained()` 方法，我们去看看这里做了什么。

```kotlin
fun onObjectRetained() {
    scheduleRetainedObjectCheck(
        reason = "found new object retained",
        rescheduling = false
    )
}
```

可以看到只是很简单地调用了 `scheduleRetainedObjectCheck()` 方法：

```kotlin
private fun scheduleRetainedObjectCheck(
  reason: String,
  rescheduling: Boolean,
  delayMillis: Long = 0L
) {
  val checkCurrentlyScheduledAt = checkScheduledAt
  if (checkCurrentlyScheduledAt > 0) {
    val scheduledIn = checkCurrentlyScheduledAt - SystemClock.uptimeMillis()
    SharkLog.d { "Ignoring request to check for retained objects ($reason), already scheduled in ${scheduledIn}ms" }
    return
  } else {
    val verb = if (rescheduling) "Rescheduling" else "Scheduling"
    val delay = if (delayMillis > 0) " in ${delayMillis}ms" else ""
    SharkLog.d { "$verb check for retained objects${delay} because $reason" }
  }
  checkScheduledAt = SystemClock.uptimeMillis() + delayMillis
  backgroundHandler.postDelayed({
    checkScheduledAt = 0
    checkRetainedObjects(reason)
  }, delayMillis)
}
```

记录了时间戳之后，开始在子线程中执行 `checkRetainedObjects(reason)` 方法：

```kotlin
private fun checkRetainedObjects(reason: String) {
  ...

  val now = SystemClock.uptimeMillis()
  val elapsedSinceLastDumpMillis = now - lastHeapDumpUptimeMillis
  if (elapsedSinceLastDumpMillis < WAIT_BETWEEN_HEAP_DUMPS_MILLIS) {
    onRetainInstanceListener.onEvent(DumpHappenedRecently)
    showRetainedCountNotification(
        objectCount = retainedReferenceCount,
        contentText = application.getString(R.string.leak_canary_notification_retained_dump_wait)
    )
    scheduleRetainedObjectCheck(
        reason = "previous heap dump was ${elapsedSinceLastDumpMillis}ms ago (< ${WAIT_BETWEEN_HEAP_DUMPS_MILLIS}ms)",
        rescheduling = true,
        delayMillis = WAIT_BETWEEN_HEAP_DUMPS_MILLIS - elapsedSinceLastDumpMillis
    )
    return
  }

  SharkLog.d { "Check for retained objects found $retainedReferenceCount objects, dumping the heap" }
  dismissRetainedCountNotification()
  dumpHeap(retainedReferenceCount, retry = true)

  ...
}
```

首次运行时，`lastHeapDumpUptimeMillis` 为 `0`，所以代码会执行到 `dumpHeap(retainedReferenceCount, retry = true)`，而后续执行时，如果两次执行的间隔过小（60秒），则直接展示通知，不再 dump heap。

```kotlin
private fun dumpHeap(retainedReferenceCount: Int, retry: Boolean) {
  saveResourceIdNamesToMemory()
  val heapDumpUptimeMillis = SystemClock.uptimeMillis()
  KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
  val heapDumpFile = heapDumper.dumpHeap()
  if (heapDumpFile == null) {
    if (retry) {
      SharkLog.d { "Failed to dump heap, will retry in $WAIT_AFTER_DUMP_FAILED_MILLIS ms" }
      scheduleRetainedObjectCheck(
          reason = "failed to dump heap",
          rescheduling = true,
          delayMillis = WAIT_AFTER_DUMP_FAILED_MILLIS
      )
    } else {
      SharkLog.d { "Failed to dump heap, will not automatically retry" }
    }
    showRetainedCountNotification(
        objectCount = retainedReferenceCount,
        contentText = application.getString(
            R.string.leak_canary_notification_retained_dump_failed
        )
    )
    return
  }
  lastDisplayedRetainedObjectCount = 0
  lastHeapDumpUptimeMillis = SystemClock.uptimeMillis()
  objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
  HeapAnalyzerService.runAnalysis(application, heapDumpFile)
}
```

可以看到，这里关键的一步是 `heapDumper.dumpHeap()` 方法，这里 dump 出来的内存数据会直接供后面的 HeapAnalyzerService 使用。此处的 `heapDumper` 是一个 AndroidHeapDumper 的实例，我们看一下它的代码：

```kotlin
internal class AndroidHeapDumper(
  context: Context,
  private val leakDirectoryProvider: LeakDirectoryProvider
) : HeapDumper {

  private val context: Context = context.applicationContext
  private val mainHandler: Handler = Handler(Looper.getMainLooper())

  override fun dumpHeap(): File? {
    val heapDumpFile = leakDirectoryProvider.newHeapDumpFile() ?: return null

    val waitingForToast = FutureResult<Toast?>()
    showToast(waitingForToast)

    if (!waitingForToast.wait(5, SECONDS)) {
      SharkLog.d { "Did not dump heap, too much time waiting for Toast." }
      return null
    }

    val notificationManager =
      context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    if (Notifications.canShowNotification) {
      val dumpingHeap = context.getString(R.string.leak_canary_notification_dumping)
      val builder = Notification.Builder(context)
          .setContentTitle(dumpingHeap)
      val notification = Notifications.buildNotification(context, builder, LEAKCANARY_LOW)
      notificationManager.notify(R.id.leak_canary_notification_dumping_heap, notification)
    }

    val toast = waitingForToast.get()

    return try {
      Debug.dumpHPROFData(heapDumpFile.absolutePath)
      if (heapDumpFile.length() == 0L) {
        SharkLog.d { "Dumped heap file is 0 byte length" }
        null
      } else {
        heapDumpFile
      }
    } catch (e: Exception) {
      SharkLog.d(e) { "Could not dump heap" }
      // Abort heap dump
      null
    } finally {
      cancelToast(toast)
      notificationManager.cancel(R.id.leak_canary_notification_dumping_heap)
    }
  }

  private fun showToast(waitingForToast: FutureResult<Toast?>) {
    mainHandler.post(Runnable {
      val resumedActivity = InternalLeakCanary.resumedActivity
      if (resumedActivity == null) {
        waitingForToast.set(null)
        return@Runnable
      }
      val toast = Toast(resumedActivity)
      val iconSize = resumedActivity.resources.getDimensionPixelSize(
          R.dimen.leak_canary_toast_icon_size
      )
      toast.setGravity(Gravity.CENTER_VERTICAL, 0, -iconSize)
      toast.duration = Toast.LENGTH_LONG
      // Inflating with application context: https://github.com/square/leakcanary/issues/1385
      val inflater = LayoutInflater.from(context)
      toast.view = inflater.inflate(R.layout.leak_canary_heap_dump_toast, null)
      toast.show()

      val toastIcon = toast.view.findViewById<View>(R.id.leak_canary_toast_icon)
      toastIcon.translationY = -iconSize.toFloat()
      toastIcon
          .animate()
          .translationY(0f)
          .setListener(object : AnimatorListenerAdapter() {
            override fun onAnimationEnd(animation: Animator) {
              waitingForToast.set(toast)
            }
          })
    })
  }

  private fun cancelToast(toast: Toast?) {
    if (toast == null) {
      return
    }
    mainHandler.post { toast.cancel() }
  }
}

```

dump 的过程比较简单，可以分为四步：

1. 创建一个 HPROF 文件，用于存储内存信息；
2. 展示一个通知方便开发者点击察看；
3. 展示一个 toast 告诉开发者有内存泄漏并请等待；
4. 调用 `Debug.dumpHPROFData()` 方法写入文件。

`Debug.dumpHPROFData()` 方法是 Android 提供的一个方法，会将 HPROF 信息存储到指定的路径中，同时可能会引起一次 GC。

生成 HPROF 文件之后，就来到了重头戏，第三步——分析 HPROF 文件。

## 分析 HPROF 文件

上面提到了，分析 HPROF 文件使用的是 HeapAnalyzerService 这个服务，我们来看一下它的实现方法。

HeapAnalyzerService 本质上是一个 IntentService，它继承自 leakcanary.internal.ForegoundService，而 ForegroundService 又继承自 IntentService。

```kotlin
internal class HeapAnalyzerService : ForegroundService(
    HeapAnalyzerService::class.java.simpleName,
    R.string.leak_canary_notification_analysing,
    R.id.leak_canary_notification_analyzing_heap
), OnAnalysisProgressListener {

    override fun onHandleIntentInForeground(intent: Intent?) {
    if (intent == null || !intent.hasExtra(HEAPDUMP_FILE_EXTRA)) {
      SharkLog.d { "HeapAnalyzerService received a null or empty intent, ignoring." }
      return
    }

    // Since we're running in the main process we should be careful not to impact it.
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)
    val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File

    val config = LeakCanary.config
    val heapAnalysis = if (heapDumpFile.exists()) {
      analyzeHeap(heapDumpFile, config)
    } else {
      missingFileFailure(heapDumpFile)
    }
    onAnalysisProgress(REPORTING_HEAP_ANALYSIS)
    config.onHeapAnalyzedListener.onHeapAnalyzed(heapAnalysis)
  }

  ...

  companion object {
    private const val HEAPDUMP_FILE_EXTRA = "HEAPDUMP_FILE_EXTRA"
    private const val PROGUARD_MAPPING_FILE_NAME = "leakCanaryObfuscationMapping.txt"

    fun runAnalysis(
      context: Context,
      heapDumpFile: File
    ) {
      val intent = Intent(context, HeapAnalyzerService::class.java)
      intent.putExtra(HEAPDUMP_FILE_EXTRA, heapDumpFile)
      startForegroundService(context, intent)
    }

    private fun startForegroundService(
      context: Context,
      intent: Intent
    ) {
      if (SDK_INT >= 26) {
        context.startForegroundService(intent)
      } else {
        // Pre-O behavior.
        context.startService(intent)
      }
    }
  }
}
```

在调用了 `HeapAnalyzerService.runAnalysis()` 方法之后，就启动了这个前台服务，最终会调用回 `onHandleIntentInForeground()` 方法，紧接着，在调低进程优先级后，调用了 `analyzeHeap()` 方法；

```kotlin
private fun analyzeHeap(
  heapDumpFile: File,
  config: Config
): HeapAnalysis {
  val heapAnalyzer = HeapAnalyzer(this)

  val proguardMappingReader = try {
    ProguardMappingReader(assets.open(PROGUARD_MAPPING_FILE_NAME))
  } catch (e: IOException) {
    null
  }
  return heapAnalyzer.analyze(
      heapDumpFile = heapDumpFile,
      leakingObjectFinder = config.leakingObjectFinder,
      referenceMatchers = config.referenceMatchers,
      computeRetainedHeapSize = config.computeRetainedHeapSize,
      objectInspectors = config.objectInspectors,
      metadataExtractor = config.metadataExtractor,
      proguardMapping = proguardMappingReader?.readProguardMapping()
  )
}
```

此处新建了 HeapAnalyzer 的实例并进行 HPROF 文件的处理。

Leakcanary 在分析 HPROF 文件方面进行了一次变革，之前使用的是第三方库 [HAHA](https://github.com/square/haha)，后来又着手开发了自己的库叫 [Shark](https://square.github.io/leakcanary/shark/)。接下来就开始使用到自己的 Shark 库了，HeapAnalyzer 就是这个库中的类。

```kotlin
class HeapAnalyzer constructor(
  private val listener: OnAnalysisProgressListener
) {
  ...
  /**
   * Searches the heap dump for leaking instances and then computes the shortest strong reference
   * path from those instances to the GC roots.
   */
  fun analyze(
    heapDumpFile: File,
    leakingObjectFinder: LeakingObjectFinder,
    referenceMatchers: List<ReferenceMatcher> = emptyList(),
    computeRetainedHeapSize: Boolean = false,
    objectInspectors: List<ObjectInspector> = emptyList(),
    metadataExtractor: MetadataExtractor = MetadataExtractor.NO_OP,
    proguardMapping: ProguardMapping? = null
  ): HeapAnalysis {

    ...

    return try {
      listener.onAnalysisProgress(PARSING_HEAP_DUMP)
      Hprof.open(heapDumpFile)
          .use { hprof ->
            val graph = HprofHeapGraph.indexHprof(hprof, proguardMapping)
            val helpers = FindLeakInput(graph, referenceMatchers, computeRetainedHeapSize, objectInspectors)
            helpers.analyzeGraph(
                metadataExtractor, leakingObjectFinder, heapDumpFile, analysisStartNanoTime
            )
          }
    } catch (exception: Throwable) {
      HeapAnalysisFailure(
          heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
          HeapAnalysisException(exception)
      )
    }
  }
}
```

在这篇文章中就不具体展开 Shark 对于 HPROF 文件的解析了，有空单开一篇文章讲一讲。

总之，在分析完 HRPOF 文件后，返回分析结果，并交给 `config.onHeapAnalyzedListener.onHeapAnalyzed(heapAnalysis)`。

这个 Config 有必要提一下，它是个贯穿始终的角色，里面存储了 N 多的全局变量，是一个 data 类，开发者可以使用 `copy()` 方法修改 Leakcanary 的默认配置，或者自定义解析器、各种 Listener 等等。此处的 `onHeapAnalyzedListener` 是在 config 中已经被预定义好的：

```kotlin
/**
  * Called on a background thread when the heap analysis is complete.
  * If you want leaks to be added to the activity that lists leaks, make sure to delegate
  * calls to a [DefaultOnHeapAnalyzedListener].
  *
  * Defaults to [DefaultOnHeapAnalyzedListener]
  */
val onHeapAnalyzedListener: OnHeapAnalyzedListener = DefaultOnHeapAnalyzedListener.create(),
```

我们看一下它的 `onHeapAnalyzed()` 方法：

```kotlin
class DefaultOnHeapAnalyzedListener(private val application: Application) : OnHeapAnalyzedListener {
  private val mainHandler = Handler(Looper.getMainLooper())

  override fun onHeapAnalyzed(heapAnalysis: HeapAnalysis) {
    SharkLog.d { "$heapAnalysis" }

    val id = LeaksDbHelper(application).writableDatabase.use { db ->
      HeapAnalysisTable.insert(db, heapAnalysis)
    }

    val (contentTitle, screenToShow) = when (heapAnalysis) {
      is HeapAnalysisFailure -> application.getString(
          R.string.leak_canary_analysis_failed
      ) to HeapAnalysisFailureScreen(id)
      is HeapAnalysisSuccess -> {
        val retainedObjectCount = heapAnalysis.allLeaks.sumBy { it.leakTraces.size }
        val leakTypeCount = heapAnalysis.applicationLeaks.size + heapAnalysis.libraryLeaks.size
        application.getString(
            R.string.leak_canary_analysis_success_notification, retainedObjectCount, leakTypeCount
        ) to HeapDumpScreen(id)
      }
    }

    if (InternalLeakCanary.formFactor == TV) {
      showToast(heapAnalysis)
      printIntentInfo()
    } else {
      showNotification(screenToShow, contentTitle)
    }
  }
}
```

不考虑我们是 TV 的情况，会展示一个通知：

```kotlin
private fun showNotification(
  screenToShow: Screen,
  contentTitle: String
) {
  val pendingIntent = LeakActivity.createPendingIntent(
      application, arrayListOf(HeapDumpsScreen(), screenToShow)
  )

  val contentText = application.getString(R.string.leak_canary_notification_message)

  Notifications.showNotification(
      application, contentTitle, contentText, pendingIntent,
      R.id.leak_canary_notification_analysis_result,
      LEAKCANARY_MAX
  )
}
```

点击这个通知，就会进入 LeakActivity。LeakActivity 就会完整展示内存泄漏的 Reference 路径，方便开发者查找问题。
