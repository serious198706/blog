---
title: Handler 和它的朋友们
toc: true
date: 2020-01-02
link: handler-and-its-friends
cover: https://res.cloudinary.com/practicaldev/image/fetch/s--Nj15dkTo--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://thepracticaldev.s3.amazonaws.com/i/953lp2lvq3f0mrq9jsx5.png
category:
    - Android
    - Development
tags:
    - Android
    - Handler
    - Android Framwork
---

Android 中的进程间通信，采用 Binder 机制，那线程间通信呢？Android 给出了一个机制 —— Handler。

一提起 Handler，就不得不提到它的**全家**：

- Message
- MessageQueue
- Looper
- Handler

我们来一个个地解释一下。

<!-- more -->

## Message —— 信件

Message 在何时都会代表『消息』的意思。在 Android 中消息还会附带一些其他的数据，供**暂存消息的 MessageQueue** 和**分发消息的 Looper** 以及**处理消息的 Handler**使用。

Message 中包含可供 Handler 使用的三个字段，分别是**两个`int`字段**和**一个`Object`字段**。

我们来看看 Message 的定义：

```java
// android.os.Message

// 注意，虽然 Message 的构造函数是 public的，但最好的获取 Message 的方法还是使用 Message.obtain() 或者是 Handler.obtainMessage() 方法，它会从回收池子中拿出一个对象来
public final class Message implements Parcelable {
    // 消息的 code，可以让接收者辨认这是哪个消息
    // 每一个 Handler 应对 code 时都有自己的命名空间，所以你不用担心会与其他的 Handler 有冲突
    public int what;

    // arg1 和 arg2 相比较 setData(Bundle) 来说是消耗很低，如果你只需要暂存几个 int 值的话，可以用这两个参数
    public int arg1;
    public int arg2;

    // 想存啥就存啥，但是要存储的数据必须是一个 Parcelable 对象
    // 注意在Android 4.0之前不支持 Parcelable 对象
    public Object obj;

    ...

    /*package*/ Bundle data;

    // 存储着要处理消息的 Handler 的实例
    @UnsupportedAppUsage
    /*package*/ Handler target;

    // 要处理消息时，可以回调的方法
    @UnsupportedAppUsage
    /*package*/ Runnable callback;

    // 有时会链式存储下一条消息
    @UnsupportedAppUsage
    /*package*/ Message next;

    // 池子锁
    public static final Object sPoolSync = new Object();

    // 一个池，用来暂存消息，这样如果需要新消息实例的话，就直接使用池里的，而不用再新建实例了
    private static Message sPool;
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50;

    // 从全局池中拿出一个新的消息，在很多情况下可以避免生成一个新的对象
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

    // obtain() 方法的各种重载方法，但最后都会调用上面的方法
    // 比如比较常用的 obtain(Handler)
    // 与 obtain() 相同，但是会把 target 设置为传入的 Handler
    public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;

        return m;
    }
    ...

    // 将一个 Message 实例扔回全局池中
    // 在调用完这个方法后，你绝对不能再动这个 Message，因为它已经被释放了。
    // 回收一个正在队列中或正在被传递给 Handler 的消息是错误的
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    // 回收一个可能正在使用的 Message
    // 该方法会在 MessageQueue 内部使用，也会在 Looper 抛弃 Message 时使用
    void recycleUnchecked() {
        // 标记为 FLAG_IN_USE，并清空它内部的东西
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }

    // 将 Message 发送到 Handler，也即 target 变量
    public void sendToTarget() {
        target.sendMessage(this);
    }
}
```

从上面的代码，我们可以看出几个 Message 的特性：

1. 有一个『回收池』的概念，如果 Message 完成使命之后被抛弃，那这个 Message 实例会被清空数据并扔到回收池中。如果再需要新建 Message 实例的时候，就直接从池中拿，避免了新建 Message 实例。
2. 如果要新建 Message 实例，使用`Message.obtain()`方法，而不是调用`new Message()`。
3. Message 本身包含了一个 Handler 的引用，存储在 target 成员变量中，在发送消息时，会调用`target.sendMessage(this)`将自身传递到 Handler 中。

## MessageQueue —— 邮筒

在 MessageQueue 的类注释里写着：

>  Low-level class holding the list of messages to be dispatched by a Looper.  Messages are not added directly to a MessageQueue, but rather through Handler objects associated with the Looper.
>  它是一个低等级的类，拿着一堆 Message 等待 Looper 来分发。Message 并不是直接就添加到 MessageQueue 中，而是通过与 Looper 关联的 Handler 来操作的。

MessageQueue 最重要的功能当然就是维护本身的队列，提供进、出的方法给 Looper，以及掌管 Message 的生命周期。

我们分别来看看这几种特性是如何完成的。

```java
// android.os.MessageQueue.java

public final class MessageQueue {
    // 如果是 true，表示消息队列能退出
    @UnsupportedAppUsage
    private final boolean mQuitAllowed;
    ...
    @UnsupportedAppUsage
    Message mMessages;

    // 添加消息
    boolean enqueueMessage(Message msg, long when) {
        // 各种判断
        ...

        synchronized (this) {
            // 如果正在退出，就不能添加了
            // 同时还得回收这个消息
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            // 如果 MessageQueue 中没有消息
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                // 循环查看 Message.next，如有，就继续查看 next，如果没有，就把消息添加到队尾 Message.next 中
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg; // 然后把当前 Message 赋值到上一个 Message 的 next 中 
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

    // 获取队头消息
    @UnsupportedAppUsage
    Message next() {
        // 如果 Looper 已经完蛋了，就返回 null 了
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // 还没到发送消息的时候，就设置一个超时，等时间到了再唤醒它
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 到发送消息的时候了，获取它
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
                ...
            }
            ...
        }
    }

    ...
    // 根据 Handler 和 what 移除 Message，同时要回收这个 Message
    void removeMessages(Handler h, int what, Object object) {
        ...
        synchronized (this) {
            Message p = mMessages;
            ...
            p.recycleUnchecked();
        }
        ...
    }
}
```

读完上面的代码，我们可以发现，其实 MessageQueue 并不想我们想象的，有一个真实的『队列』存在，而是用 Message 的 next，建立了一个链表，形成了所谓的『队列』。在『进』的流程里比较简单，就是建立链表的过程；而『出』的流程相对复杂一些，是利用了一个死循环，因为每条 Message 都有自己『要被发送』的时间戳，如果时间没到，那还要等待一下，如果时间已到，就取出，并把其它的 Message 向队头提一位。

可以用两个流程图来展示『进』与『出』：

![](/img/30.png)

## Looper —— 马车

Looper 是一个死循环。它负责不停地读取 MessageQueue 中的消息，如果一旦有消息，就拿出来，扔给 Handler，然后又进入自己的小圈子里不问世事。一旦 MessageQuee 里没有消息了，那它也就结束自己的工作。

我们看看它的代码：

```java
// android.os.Looper.java

/**
  * Class used to run a message loop for a thread.  Threads by default do
  * not have a message loop associated with them; to create one, call
  * {@link #prepare} in the thread that is to run the loop, and then
  * {@link #loop} to have it process messages until the loop is stopped.
  *
  * <p>Most interaction with a message loop is through the
  * {@link Handler} class.
  *
  * <p>This is a typical example of the implementation of a Looper thread,
  * using the separation of {@link #prepare} and {@link #loop} to create an
  * initial Handler to communicate with the Looper.
  *
  * <pre>
  *  class LooperThread extends Thread {
  *      public Handler mHandler;
  *
  *      public void run() {
  *          Looper.prepare();
  *
  *          mHandler = new Handler() {
  *              public void handleMessage(Message msg) {
  *                  // process incoming messages here
  *              }
  *          };
  *
  *          Looper.loop();
  *      }
  *  }</pre>
  */
public final class Looper {
    ...
    @UnsupportedAppUsage
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    @UnsupportedAppUsage
    private static Looper sMainLooper;  // guarded by Looper.class
    private static Observer sObserver;

    @UnsupportedAppUsage
    final MessageQueue mQueue;
    final Thread mThread;

    /** 将当前线程初始化为一个 Looper。
      * 之后，在真正启动这个 Looper 之前，你就可以用这个 Looper 去创建 Handler了。
      * 在调用完 prepare() 之后一定要调用 loop() 方法，如果要停止，则调用 quit()
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    /**
     * 将当前线程初始化为一个 Looper，并将其当做应用的 main looper。
     * 介个方法是由 Android 来调用的，所以理论上来说，你应该压根用不到这个方法。
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    /**
     * 返回应用的 main looper，在应用的主线程里存活的那个
     */
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

    ...

    /**
     * 开始监听 message queue，一定记得要调用 quit() 来停止 Looper
     */
    public static void loop() {
        // 拿到调用此方法的当前线程下的 Looper 实例
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        ...

        // 开启自闭模式
        for (;;) {
            Message msg = queue.next(); // 可能会 block，因为在 MessageQueue.next() 中，
                                        // 调用了 nativePollOnce 方法，可能会引起阻塞。
                                        // 也就是说，没有消息的时候，Looper 就在这儿『等着』。
            if (msg == null) {
                // 木有 message，意味着 message queue 正在 quit
                // 一旦 return，这个 Looper 就结束了
                return;
            }

            // 各种检查
            ...
            
            try {
                // 发送消息
                msg.target.dispatchMessage(msg);
                ...
            } catch (Exception exception) {
                ...
            } finally {
                ...
            }

            ...
        }
    }

    /**
     * 从 sThreadLocal 中获取当前线程下的 Looper
     * 此处利用了 ThreadLocal，某个线程下存储的数据，只有这个线程能读取，其他线程不可以，从而达到
     * 线程与 Looper 的『绑定』效果
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

    /**
     * 返回当前线程下 Looper 的 MessageQueue。
     * 该方法必须从一个正在跑 Looper 的线程下调用，否则就会抛出 NullPointerException。
     */
    public static @NonNull MessageQueue myQueue() {
        return myLooper().mQueue;
    }

    // 初始化
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

    ...

    // 返回当前线程是否是 Looper 的线程
    public boolean isCurrentThread() {
        return Thread.currentThread() == mThread;
    }

    ...

    public interface Observer {
        // Message 分发之前被调用
        Object messageDispatchStarting();

        // Message 被 Handler 处理过了之后调用
        void messageDispatched(Object token, Message msg);

        // Message 处理过程中出现异常时调用
        void dispatchingThrewException(Object token, Message msg, Exception exception);
    }
```

Looper 类比较清晰，职责也相对单一，就是用死循环的方式，一直通过`MessageQueue.next()`方法获取 Message，然后调用`msg.target.dispatchMessage(msg)`方法，将 Message 交给 Handler 处理（`msg.target`是 Handler 类型）。

等任务全部处理完成后，`使用 MessageQueue.quit()` 清空消息，结束 Looper 的循环。

有一个比较经典的但是又很无厘头的问题：主线程的 Looper 死循环为什么不会导致 ANR？

这个问题其实算是偷换概念。

首先，死循环不是造成 ANR 的必然原因，ANR 是因为消息队列中的消息没有得到及时处理才造成的，比如 BroadcastReceiver 的`onReceive()`方法中处理事务超过了10秒，比如`onTouch()`事件超过了5秒，才会导致 ANR。

第二，主线程的 Looper 死循环，最多也就会导致个 OOM，但是主线程在没有消息时也会休眠、进入阻塞状态，当有新消息来临时，再被唤醒，分发消息，实际上对于内存的消耗非常小。

第三，如果主线程的 Looper 不循环了，那`main()`方法就抛出一个异常结束了，就整个应用就退出了。

另外，我们看到，`prepare()`方法中使用到了 ThreadLocal 的机制，该机制的描述是：**A 线程下使用了 `ThreadLocal.set()` 方法存储了某个资源，那么只有在 A 线程下才能通过 `ThreadLocal.get()` 方法拿到这个资源，从而实现了资源隔离**。对 ThreadLocal 的详解，在[这篇文章](/threadlocal/)里。

## Handler —— 邮递员

Handler 顾名思义就是处理者。当接到 Message 的时候，就马不停蹄地开始工作。它的头脑很简单，给我活我就干，没活我就等着。

我们在初始化 Handler 的时候，有几种方式：

1. 直接调用`new Handler()`。

我们来看看在这种情况下 Handler 的构造函数。

```java
// android.os.Handler.java

public Handler() {
    this(null, false);
}

public Handler(@Nullable Callback callback, boolean async) {
    // 检查持有该实例的类是否会导致内存泄漏
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    // 调用了 Looper.myLooper() 方法，来获取 Looper 的实例。
    // 刚才上面讲过了，只能获取当前线程下的 Looper，所以，一般情况下，我们 new 一个 Handler 的时候，都是
    // 在主线程下，那这个 mLooper 其实就是 Main Looper
    mLooper = Looper.myLooper();
    // 如果一个线程被创建了，但是它的 Looper 没有调用过 prepare()，也就是没有启用，那获取到的 Looper 就是空
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

2. 创建 Handler 实例时传入 Looper。

同样地，也看看这种情况下，会调用哪个构造函数：

```java
// android.os.Handler.java

public Handler(@NonNull Looper looper) {
        this(looper, null, false);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

有了 Looper 就可以直接赋值了，Handler 也并不关心当前的 Looper 是在哪个线程下，干就完了。

我们也观察到，Handler 的几个重载的构造函数里，总会有 Callback 参数，言下之意，是 Handler 在某个节点，应该会调用这个 Callback。我们来看看代码是不是这么写的，从 Looper 向 Handler 发送消息开始：

```java
// android.os.Handler.java

public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

// 子类必须要实现这个方法，才能获取 Message
// 但它不是抽象方法，所以可以不实现该方法
public void handleMessage(@NonNull Message msg) {
}
```

是了，如果在 Message 中有 Callback，那就直接回调 Message 中的 Callback，下面的就不走了。

如果给 Handler 设置了 Callback，那就调用，如果该 Callback 的实现类里，返回了`true`，那就不再调用 handleMessage 方法；如果返回了`false`，那就再调用一次 handleMessage 方法。

Callback 的定义是这样的：

```java
// 你可以用 Callback 接口来初始化一个 Handler，这样你就不用去继承 Handler 然后去实现一堆方法了
// 说白了，就是简单地实现一个回调功能
public interface Callback {
    // 返回 true 的话，就不再继续处理
    boolean handleMessage(@NonNull Message msg);
}
```

刚才在 Looper 的代码中看到，`msg.target.diapatch()`是在向 Handler 发送消息，那`msg.target`是什么时候被赋值的呢？

我们来看 Handler 中最著名的`post()`方法。

```java
// android.os.Handler.java

// 把一个 Runnable 的实例添加到 MessageQueue 中。
// 这个 Runnable 将会运行在 Handler 所在的线程上。
// 返回 true 的话，就是正确地放入了 MessageQueue，否则就返回 false，通常是因为 MessageQueue 正在退出
public final boolean post(@NonNull Runnable r) {
    return sendMessageDelayed(getPostMessage(r), 0);
}

// 向 MessageQueue 中添加一个 Message，如果有延时，那就在这个时间段（current time + delayMillis）再添加
// 这个 Message 回头能在 handleMessage 里拿到，当前，所在的线程也是 Hanlder 绑定的线程
// 添加成功后返回 true，添加失败返回 false
// 添加成功不代表一定能被处理
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

// 在一个绝对时间（毫秒）时，将 Message 添加到 MessageQueue 中
// 这个时间是基于 android.os.SystemClock#uptimeMillis
// 这个 Message 回头能在 handleMessage 里拿到，当前，所在的线程也是 Hanlder 绑定的线程
// 添加成功后返回 true，添加失败返回 false
// 添加成功不代表一定能被处理
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    // 赋值了，将 msg.target 赋值为自身的实例
    msg.target = this; 
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

也就是说，Handler 不光只是处理消息，也要负责将消息添加到 MessageQueue 中。

关于这四大天王的关系，可以用一个关系图来展示一下：

![](/img/31.png)

