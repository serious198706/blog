---
title: Jetpack Compose Side Effects
date: 2023-06-15
toc: true
link: jetpack-compose-side-effects
cover: http://blog.cigis-cloud.com/java-dynamic-proxy-1593752525.png
category: 
    - Development
tag:
    - Jetpack Compose
    - Kotlin
---

#### `LaunchedEffect`

它会在第一次重组时运行，之后重组时不会再重新运行。但是，我们可以通过指定 `key1` 的方式来重新运行。**它运行在一个协程上**。但是想要它一直运行也可以，写个死循环呗（大雾）。

#### `SideEffect`

它会在每次重组时运行。**它不运行在协程上**。

举个例子。我们来做一个计时器。

```kotlin
@Composable
fun TryWithoutSideEffect() {
    var timer by remember { mutableStateOf(0) }
    Box(modifier = Modifier.fillMaxSize(), 
        contentAlignment = Alignment.Center) {
        Text("Time $timer")
    }

    Thread.sleep(1000)
    timer++
}
```

1秒钟之后，`timer` 的值的变化，会引起一次重组吗？答案是**不会**。因为它的值是在重组时发生了变化，并不会引起下一次重组。如果想达到我们的目的，可以使用 `SideEffect`：

```kotlin
@Composable
fun TrySideEffect() {
    var timer by remember { mutableStateOf(0) }
    Box(modifier = Modifier.fillMaxSize(), 
        contentAlignment = Alignment.Center) {
        Text("Time $timer")
    }

    SideEffect {
        Thread.sleep(1000)
        timer++
    }
}
```

#### DisposableEffect

与 [LaunchedEffect](#LaunchedEffect) 功能相同，但多了一个特性，就是它有被取消时的**回调**，使用场景基本上是当该 `Effect` 被取消时，需要做些资源清理之类的。举例如下：

```kotlin
@Composable
fun DisposableEffectTest() {
    val context = LocalContext.current

    DisposableEffect(key1 = Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context?, intent: Intent?) {
                // do something
            }
        }
        context.registerReceiver(receiver, IntentFilter(BatteryManager.ACTION_CHARGING))
        Log.d("DisposableEffectTest", "register receiver")

        onDispose {
            context.unregisterReceiver(receiver)
            Log.d("DisposableEffectTest", "unregister receiver")
        }
    }
}
```

#### `produceState()`

该方法用来将 non-composable state 转换为 composable state。

看一段代码：

```kotlin
@Composable
fun ProduceStateTestScreen() {
    val timer by produceState(initialValue = 0) {
        delay(1000L)
        value++
    }

    Box(
        modifier = Modifier.fillMaxSize()
    ) {
        Text("$timer")
    }
}
```

timer 本是一个 non-compose 的变量，但是通过 `produceState` 方法，它变成了一个 compose 变量。

它有一个 `awaitDispose` 的回调，与 `DisposableEffect.onDispose` 类似。


#### `rememberCoroutineScope()`

这个 `Effect` 应该是使用频率比较高的。它允许你在 composable 函数之外启动一个协程，在该协程中操作变量可以引起重组。那么有人要问了，那它和 `LaunchedEffect` 有啥区别？都是启动了一个新的协程？

区别如下：
- `LaunchedEffect` 必须在 composable 函数之内运行，但 `rememberCoroutineScope()` 可以在 composable 函数之外运行。
- `LaunchedEffect` 在第一次重组时一定会运行，但使用 `rememberCoroutineScope()` 创建的协程是需要我们手动控制启动的。

```kotlin
@Composable
fun MoviesScreen(snackbarHostState: SnackbarHostState) {

    val scope = rememberCoroutineScope()

    Scaffold(
        snackbarHost = {
            SnackbarHost(hostState = snackbarHostState)
        }
    ) { contentPadding ->
        Column(Modifier.padding(contentPadding)) {
            Button(
                onClick = {
                    // Create a new coroutine in the event handler to show a snackbar
                    scope.launch {
                        snackbarHostState.showSnackbar("Something happened!")
                    }
                }
            ) {
                Text("Press me")
            }
        }
    }
}
```

上述例子只有在点击时，才会启动协程，弹出 `Snackbar`（`Snackbar`必须要在协程中弹出，你想在 `LaunchedEffect` 中弹出也行）。

#### `rememberUpdatedState()`

`LaunchedEffect` 的特点是当 `key` 变化时会重运行，那么有些时候吧你想反其道而行之，你不想要 `LaunchedEffect` 重运行，比如说 `LaunchedEffect` 中有些耗时/重量级的操作。那么此时在运行到某个阶段，我需要获取某个参数的最新值，能获取到吗？。 

```kotlin
@Composable
fun RememberUpdatedStateTest() {
    var parameter by remember {
        mutableStateOf("Unknown")
    }

    Column() {
        Button(
            onClick = { parameter = "Apple" }) {
            Text("Apple")
        }
        Button(
            onClick = { parameter = "Orange" }) {
            Text("Orange")
        }
        Timer((parameter))
    }
}

@Composable
fun Timer(parameter: String) {

    LaunchedEffect(true) {   // 传入 never-changing constant，可以保证重组时该 LaunchEffect 不会重启
        delay(5000L)
        Log.d("Timer", parameter)
    }

    Box(modifier = Modifier.size(300.dp)) {
        Text(parameter, fontSize = 24.sp, color = Color.White, modifier = Modifier.align(Alignment.Center))
    }
}

```
上面的例子，用户点击两个 `Button`，`parameter` 的值会发生变化，同时 `Timer` 也会不断重组，屏幕上的 `Text` 也会更新 `parameter` 的最新值，但是 5 秒钟后，我们看到 Logcat 仿佛不受控制般输出了 `Unknown`。

`LaunchedEffect` 我不能频繁重启，因为它有重量级操作，此时我的需求就是在使用 `parameter` 时，我希望它的值是最新的，那么就要用上 `rememberUpdatedState()`：

```kotlin
@Composable
fun Timer(parameter: String) {
    val parameterNew by rememberUpdatedState(parameter)

    LaunchedEffect(true) {
        delay(5000L)
        Log.d("Timer", parameterNew)
    }

    Box(modifier = Modifier.size(300.dp)) {
        Text(parameter, fontSize = 24.sp, color = Color.White, modifier = Modifier.align(Alignment.Center))
    }
}
```

等待 5 秒，我们如愿从 Logcat 中看到了 `"Apple"`。

#### `derivedStateOf()`

它可以将一个或多个 state 对象转换为其他 state。如果某个 state 是由其他的 state 对象计算或派生出来的，我们可以使用 `derivedStateOf()`。举个例子：

```kotlin

val tasks = arrayOf(
    "Task1",
    "Task2",
    "ReviewTask3",
    "Task4",
    "UnblockTask5",
    "ComposeTask6",
    "Task7",
)

@Composable
fun TodoList(highPriorityKeywords: List<String> = listOf("Review", "Unblock", "Compose")) {
    var index = 0

    val todoTasks = remember { mutableStateListOf<String>() }

    val highPriorityTasks by remember(highPriorityKeywords) {
        derivedStateOf {
            todoTasks.filter { task ->
                highPriorityKeywords.any { keyword ->
                    task.contains(keyword)
                }
            }
        }
    }

    Box(Modifier.fillMaxSize()) {
        LazyColumn {
            items(highPriorityTasks) {
                Text(it, color = Color.White)
            }
            
            if (highPriorityTasks.isNotEmpty()) {
                item {
                    Divider(color = Color.White)
                }
            }
            
            items(todoTasks) {
                Text(it, color = Color.White)
            }
        }

        FloatingActionButton(
            modifier = Modifier.align(Alignment.BottomEnd),
            onClick = {
                todoTasks.add(tasks[index++])
            }) {
            Icon(Icons.Default.Add, contentDescription = null)
        }
    }
}
```

上述例子展示了一个 todo task 列表，当且仅当 `todoTasks` 发生变化时，`highPriorityTasks` 才会发生变化。如果 `highPriorityKeywords` 发生变化， `remember` 代码块会重新生成一个新的对象以代替旧的对象。

`derivedStateOf` 生成的状态不会导致整个 composable 重组，而是仅重组使用它的那个部分，也即 LazyColumn 中的那部分。

#### `snapShotFlow` 

它的作用是将一个 composable state 转换为 Flows。

Flow 是 java 和 kotlin 中的一个重要概念，它在响应式编程中起到了很重要的作用，日后写一篇单独分析 Flow。

它可以在某个 state 对象发生变化时，以 Flow 的形式产生新的值。

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
    // ...
}

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it == true }
        .collect {
            MyAnalyticsService.sendScrolledPastFirstItemEvent()
        }
}
```
上述例子中，当 listState 发生变化时（比如滚动了列表），`snapshotFlow` 会立刻触发，将列表的第一个元素的最新内容获取并上报日志。