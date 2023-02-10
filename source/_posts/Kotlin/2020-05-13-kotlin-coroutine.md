---
title: Kotlin 协程
link: kotlin-coroutine
date: 2020-05-13
category: 
    - Development
tag: 
    - kotlin
    - coroutine
    - 协程
---

Kotlin 语言发展至今，也有9个年头了。当初发明之后一直不愠不火，直到近年 Google 宣布将支持使用 Kotlin 语言支持 Android 开发之后，Kotlin 才算是真正火起来。

在 Android 开发中，Kotlin 语言与 Java 语言可以无缝衔接，其实背后还是 Android 虚拟机的功劳，它会将 Kotlin 文件转换为 class 文件，然后加载到虚拟机中运行。在本文写作时，Kotlin 版本已经到了 [1.3.72](https://github.com/JetBrains/kotlin/releases/tag/v1.3.72)，[1.4.0](https://blog.jetbrains.com/kotlin/2020/03/kotlin-1-4-m1-released/) 正处于 preview 阶段。关于 Kotlin 语言的学习，我会单开一篇文章来讲，这篇文章我们来讲讲 Kotlin 中一个非常重要的特性——协程（Coroutine）。

<!-- more -->

## 协程的概念

协程的概念由来已久，当初邓超代言广告的时候。。。喂喂喂，此协程非彼携程啊！

好，重新来。

协程的概念由来已久，它又被称为『**微线程**』、『**纤程**』。从名字上可以看出，它是一种非常轻量级的概念。它很早就被提出来了，进程下有线程，那线程下是不是也得分一分？好，就叫协程吧。虽然概念的提出很早，但也只是近几年才得到广泛应用。

![](/img/coroutine-1589349773.png)

<div style="text-align:center">

*进程下有多个线程*

</div>

![](/img/coroutine-1589349908.png)

<div style="text-align:center">

*线程下有多个协程*

</div>

我们可以给它下一个定义：**协程是能够在单个线程下，还能够并发运行的一种机制**。它最重要的特点是，它**不会被操作系统内核所管理**，而**完全由用户程序来控制**，也就是在**用户态**执行，而非内核态。这样带来的好处是**可以省去传统多线程机制中线程切换时的性能损耗**，大幅度提高并发性能。

在 Python 中很早就引入了协程的使用，我们可以简单看一下一个很简单的**生产者-消费者的例子**：

```python
# 消费者
def consume():
    while True:
        number = yield  # 在这停顿
        print('Start Consuming', number)

consumer = consume()

# 初始化协程，并在 yield 处停止
next(consumer)

# 生产者
for num in range(0, 100):
    print('Start Producing', num)
    consumer.send(num)
```

上面的代码创建了一个协程`consumer`，然后在主线程中生产数据，并交给协程去消费。`yield`是 Python 的语法，指的是程序执行到这儿的时候，会停顿，等待主线线使用`send()`发送消息时，才会接收消息并继续执行。

上面的代码运行如下：

```
Start Producing 0
Start Consuming 0
Start Producing 1
Start Consuming 1
Start Producing 2
Start Consuming 2
Start Producing 3
Start Consuming 3
...
...
Start Producing 97
Start Consuming 97
Start Producing 98
Start Consuming 98
Start Producing 99
Start Consuming 99
```

各位同学们看出来了没有，实际上这是一种『障眼法』，两段代码完全是在程序的控制下交替执行，虽然我们使用线程 + 同步机制也可以做到这一点，但是在线程切换的资源消耗以及性能上，就比这种做法要差上那么『一 nai nai』。

> 在 Python 3.5+ 上，使用`async/await`来替代了`yield/send()`。

还有其他的语言也使用了协程的概念，比如 Lua、Go、C# 等等。

## 协程与线程的比较

上面我们提到了，协程是比线程还要低一级的并发机制，那么它们两个有什么相同点和不同之处呢？理论上来说，这两种概念不太应该用来比较，但是我们还是列举一下，也方便我们理解协程的优缺点。

### 线程

线程拥有独立的栈、局部变量、可以共享进程的内存。在使用共享数据时，为了避免造成数据错误，需要控制线程的状态。但这种控制是由内核来做的，所以程序会不断地在用户态和内核态之间切换，这种切换的消耗和代价是比较大的。

### 协程

协程也拥有自己的栈、局部变量，可以共享线程内的内存。在一个线程上可以同时跑多个协程，同一时间只有一个协程被执行，本质上是在单线程上模拟多线程并发。协程的运行和暂停，全部是由开发者自己决定的，不需要切换至内核态，大大减少了消耗，提高了性能。

协程默认是非阻塞式的（也可以阻塞），一个协程在**进入阻塞后不会阻塞当前线程**，当前线程会去执行其他协程任务。

## Kotlin 中的协程

Kotlin 的协程的中心思想是：**暂停计算**。简单来说，就是一个方法在执行到某处的时候，可以暂停，然后在到达某一个时间节点时，继续运行。把这种概念应用到编程中，就是可以**直接编写无阻塞的代码**，能达到与阻塞代码相同的效果。官网给出了一个例子：

```kotlin
fun postItem(item: Item) {
    launch {
        val token = preparePost()
        val post = submitPost(token, item)
        processPost(post)
    }
}

suspend fun preparePost(): Token {
    // 执行请求，暂停协程
    return suspendCoroutine { /* ... */ } 
}
```

这块代码并不会阻塞主线程，`preparePost()`被称为『**可中止方法**』。与上面讲过的中心思想一致：方法被执行 → 暂停执行 → 在某个时间节点继续执行。

使用 Kotlin 的协程有下面几个优点：

1. 方法本身不变，唯一的变化是加上了`suspend`关键字；
2. 写代码时直接采用同步写法，看起来就是『由上至下』的执行方式，不用任何特殊的语法、特殊的处理。我们只需要使用`launch`关键字来启动协程并执行可中止方法即可；
3. Kotlin 中其他的 API 等照常使用，你可以随便使用循环、异常处理之类的，不需要再学习什么新的 API；
4. 平台无关性。无论你的代码是运行在 JVM 上，Javascipt 平台或者是其他平台，代码是一样一样的，让编译器来帮你做这些适配的工作。

## Android 中的 Kotlin 协程

要在 Android 中使用 Kotlin 协程，要分为两步：

1. 将 Kotlin 集成到项目中。

首先在项目的 build.gradle 文件中，修改成如下代码：

```gradle
buildscript {
    ext.kotlin_version = '1.3.72'  // kotlin 的版本
    repositories {
        google()
        jcenter()
        
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.6.3'  // 要使用 kotlin，gradle 版本必须在 3.0.0 以上
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"  

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
...
```

然后在模块的 build.gradle 文件中，修改代码：

```gradle
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'

android {
    ...
    kotlinOptions {
        jvmTarget = "1.8"
    }
}

dependencies {
    ...
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
    ...
}
```

同步 gradle，Kotlin 就集成完毕了。

2. 加入协程的依赖

在模块的 buidl.gradle 文件中，做出如下修改：

```gradle
dependencies {
    ...
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.3.72"
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.3.6"
}
```

就完成了协程的集成工作。

然后我们使用协程来写一个简单的倒计时的小例子，我们设计一个 textView 和一个 button，当点击 button 时，textView 就会显示 10 秒钟的倒计时，精确到毫秒。

```kotlin
button.setOnClickListener {
    GlobalScope.launch(Dispatchers.Main) {
        println(Looper.myLooper() == Looper.getMainLooper())

        for (i in 1000 downTo 0) {
            textView.text = formatTime(i)
            delay(1)
        }
    }
}
```

可以看出，与上面简单介绍的 suspend、launch 关键字不同，这里又多了一些新的东西。

- GlobalScope

GlobalScope 用来启动顶级的协程，生命周期与 Application 一致。当然，还有其他的 Scope，我们回头慢慢介绍。

- Dispatchers

用来指定协程运行在哪个线程里。毕竟网络请求还是要在子线程中，更新 UI 还是要在主线程哇。后面会讲一个简单的网络请求的例子。

Dispatcher 有几种类型，如下表所示：

| 常量 | 作用 |
| --  |  -- |
| Default | |
| Main | 让协程运行在主线程 |
| 
