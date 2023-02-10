---
title: Flutter 的 Isolate 与事件循环机制
date: 2020-07-23
link: flutter-isolate-and-event-loop
category:
    - Development
    - Flutter
tags:
    - Flutter
---

在 Dart 中，没有『多线程』的概念，在绝大多数的开发中，只会用到 UI 线程，也就是 Android 中所谓的『主线程』。

但是 Dart 给我们提供了异步编程的方式，来让我们『同步』地编写『异步』的代码。也即`await/async`关键字。在运行一些代码时，UI 线程还会继续渲染 Widget Tree，不会卡死。

那么单线程的 Dart 是如何实现这种看起来像『多线程』的机制的呢？这里我们要先介绍一个概念：Isolate。

<!-- more -->

## Isolate

每一个 Isolate 都有自己的内存空间，以及一个单独的线程来运行 event loop。

绝大多数情况下，一个 Flutter 应用只需要一个 Isolate 就可以了，但如果你有庞大的计算需求，大到有可能会阻碍到 UI 线程掉帧，那你就得创建一个新的 Isolate 来进行这些计算。

创建新的 Isolate 包括但不限于以下方式：

- `Isolate.spawn()`
- `compute()`

创建完成的新 Isolate 有它自己独立的内存空间，即便是它的创建者也无法访问这部分空间。这也是『隔离』的意义所在。

实际上，要让多个 Isolate 合作，也是可以的，这需要用到它的 SendPort 和 ReceivePort。举例代码如下：

```dart
ReceivePort receivePort = ReceivePort();
// 创建新的 Isolate
await Isolate.spawn(complexCompute, receivePort.sendPort);
SendPort sendPort = await receivePort.first;
// 流的第一个元素被收到后监听会关闭，所以需要新打开一个ReceivePort以接收传入的消息
ReceivePort response = ReceivePort();

int toBeComputed = 1;

// 以数组的形式将参数和发送方传入，方便进行数据回传
sendPort.send([toBeComputed, response.sendPort]);
int result = await response.first;

setState(() {
    ...
});
```

`complexCompute()`方法必须是个顶级方法或者静态方法：

```dart
void complexCompute(SendPort sendPort) async {
  ReceivePort port = ReceivePort();
  sendPort.send(port.sendPort);

  await for (var param in port) {
    int toBeComputed = param[0];
    SendPort replyTo = param[1];

    toBeComputed++;

    replyTo.send(toBeComputed);
  }
}
```

当然，上面的计算也可以直接用 `compute()` 来解决：

```dart
int result = await compute(anotherComplexCompute, 1);

setState(( {
    ...
}));
```

同样地，`anotherComplexCompute()` 也必须是顶级方法：

```dart
int anotherComplexCompute(int value) {
    return value + 1;
}
```

## Event Loop

上面讲完了 Isolate 机制，下面就该讲讲异步代码是如何被运行起来的。

Event Loop（事件循环）与 Android 中的 Looper 类似。