---
title: 透读源码之 OkHttp
link: okhttp
toc: true
date: 2023-03-14 13:00:00
tag: 
    - Android
    - OkHttp
    - 三方库
category: 
    - Android
    - Development
---

做 Android 开发的同学们几乎没有不知道 OkHttp 这个网络请求库的了。这个库可以帮我们完成几乎所有类型的、涵盖几乎所有需求的网络请求。

它与大家耳熟能详的 LeakCanary、Retrofit 一样来自大名鼎鼎的 [Square 公司](https://square.github.io/)，截至本文写下时，该库在 Github 上已有 43.7k 的 Start 数量。

<!-- more -->

### 一、OkHttp 解决了什么问题

从库的名字不难看出，OkHttp 解决的是网络问题。在 Android 开发中，网络问题自古以来便是开发者比较头疼的问题。

在互联网时代，几乎没有任何应用可以完全脱离互联网使用，良好的网络通信设计，对衡量一款 APP 的质量来说，无疑是尤为重要的加分项。

在上古时期(🙅‍♂️)，Android 开发者们还在使用 `HttpUrlConnection` 来实现网络请求，不必引入三方代码，原始且简单。但也正是因为原始，一些高级功能就要自己实现。甚至在 Android2.2 之前还有 Bug，需要禁用连接池以正常使用 inputstream，所以 Google 接下来引入了 `HttpClient`。它解决了一些使用和维护上的问题，但这个库饱受开发者和维护者诟病，其属于 Apache 组织，API 复杂多变，并且不易扩展，所以被 Google 弃之敝屣。

于是一个易用、可维护、可拓展、稳定性高的网络请求库在此时就应运而生。它就是 OkHttp。

OkHttp 针对的不只是 Android，它对 Java 程序也有封装，其实从根本上来说，它是一个基于传输层（还记得网络的7层架构吗？）实现应用层协议的网络框架，而不仅是一个 Http 请求的库。

目前来说，很多公司都已经在使用 OkHttp 当做默认的网络请求库，一些著名的三方库也已经默认使用 OkHttp 来做网络请求，你在集成它的同时，它也会默默地帮你把 OkHttp 集成进项目中。

总的来说，OkHttp 解决的，就是网络开发中的各种 API 使用繁琐、可定制可拓展性不高、高级功能要自己造轮子的问题。

按照官方文档来看，它还提供了连接池，实现 GZIP 格式传输，重复请求的缓存等。并且新版的 OkHttp 已经支持 TLS1.3 协议([神马是TLS？](https://zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E6%80%A7%E5%8D%94%E5%AE%9A))，以及多 IP 地址轮询功能。OkHttp 提供丰富又简易的 API，支付同步/异步的调用，并且有着完善的回调机制供开发者使用。

### 二、代码结构

本文以 OkHttp 5.0.0-SNAPSHOT 代码做为示例。

Clone 一下 OkHttp 的源码，可以看到 OkHttp 的 module 有很多，如下图所示：

![](/img/okhttp-1.png)

我们依次简单介绍一下各个 module。

1. `buildSrc`: 项目打包时的一些脚本，可以略过。
2. `logging-interceptor`: 日志拦截器。负责记录网络请求和返回内容并打印日志，帮助记录与排查问题。使用方法如下：

```groovy
implementation("com.squareup.okhttp3:logging-interceptor:4.10.0")
```

```kotlin
val logging = HttpLoggingInterceptor()
logging.setLevel(HttpLoggingInterceptor.Level.BASIC)
val client = OkHttpClient.Builder()
    .addInterceptor(logging)
    .build()
```

3. `mockwebserver(3/3-juni4/3-junit5)`: 模拟服务器，一般用于本地的模拟 API 请求测试，针对不同的 junit 版本做出了适配。使用方法如下：

```groovy
testImplementation("com.squareup.okhttp3:mockwebserver:4.10.0")
```

```kotlin
@Throws(Exception::class)
fun test() {
    // 创建 MokWebServer 的实例。每个单元测试都可以创建不同的实例。
    val server = MockWebServer()

    // 计划一些回应消息
    server.enqueue(MockResponse(body = "hello, world!"))
    server.enqueue(MockResponse(body = "sup, bra?"))
    server.enqueue(MockResponse(body = "yo dog"))

    // 启动服务
    server.start()

    val baseUrl: okhttp3.HttpUrl = server.url("/v1/chat/")

    // 模拟请求
    val chat = Chat(baseUrl)
    chat.loadMore()
    assertEquals("hello, world!", chat.messages())
    chat.loadMore()
    chat.loadMore()
    assertEquals(
    """
        hello, world!
        sup, bra?
        yo dog
        """.trimIndent(), chat.messages()
    )

    // 可选：确定你的应用请求了正确路径
    val request1 = server.takeRequest()
    assert("/v1/chat/messages/" == request1.path)
    assert(request1.getHeader("Authorization") != null)
    val request2 = server.takeRequest()
    assert("/v1/chat/messages/2" == request2.path)
    val request3 = server.takeRequest()
    assert("/v1/chat/messages/3" == request3.path)

    // 关闭服务
    server.shutdown()
}
```
这部分代码我们在之后会再进行介绍。

4. `okcurl`: 一些测试指令。
5. `okhttp`: 核心代码，同时包含一些测试，是我们主要要阅读的部分。我们会在后面详细阅读。
6. `okhttp-android`: 针对 Android 系统增加的一些文件，主要是一些 Extension 方法，以及 Android 上的 DNS 解析器。
7. `okhttp-bom`: 针对 Gradle 的[BOM(Bill-of-Materials)](https://docs.gradle.org/6.2/userguide/platforms.html#sub:bom_import)特性的编译脚本，能够让 gradle 支持如下写法：

```groovy
dependencies {
    // define a BOM and its version
    implementation(platform("com.squareup.okhttp3:okhttp-bom:4.10.0"))

    // define any required OkHttp artifacts without version
    implementation("com.squareup.okhttp3:okhttp")
    implementation("com.squareup.okhttp3:logging-interceptor")
}
```

8. `okhttp-brotli`: 使用 [Brotli 压缩算法](https://github.com/google/brotli) 制作的拦截器。主要用来实现网络传输中的 GZip 压缩。它依赖了 brotli 库。使用方法如下：

```groovy
implementation("com.squareup.okhttp3:okhttp-brotli:4.10.0")
```

```kotlin
val client = OkHttpClient.Builder()
    .addInterceptor(BrotliInterceptor.INSTANCE)
    .build() 
```

9. `okhttp-coroutines`: 该 module 用来支持 Kotlin 的协程。
10. `okhttp-dnsoverhttps`: 对 [DNS over HTTPS](https://zh.wikipedia.org/wiki/DNS_over_HTTPS) 的实现。
11. `okhttp-hpacktests`: 一些测试，用来验证 OkHttp 对于 HPACK 功能的实现。
12. `okhttp-sse`: 实验性功能，可以接受服务端发送的事件。本质上是添加了 `Accept:text/event-stream` 的 Header 来实现的。但目前也只是实验性功能，实现方式随时可能变化。
13. `okhttp-testing-support`: 供 OkHttp 工程内部使用的测试用例。
14. `okhttp-tls`: 提供了一些让开发者可以使用 TLS 功能的 API。如 [HeldCertificate](https://square.github.io/okhttp/4.x/okhttp-tls/okhttp3.tls/-held-certificate/) 和 [HandshakeCertificate](https://square.github.io/okhttp/4.x/okhttp-tls/okhttp3.tls/-handshake-certificates/)。
15. `okhttp-urlconnection`: 从 `java.net` 包中集成了 `Authenticator` 和 `CookieHandler`。主要用于测试。
16. `samples`: 一些使用例子。初次使用建议先从 Samples 看起，可以了解一些初级用法和高级用法。但大多数时候，官网的教程足以涵盖绝大多数需求。

### 三、OkHttp 是如何完成它的主要目标 —— 完成一次网络请求的

带着问题，我们来看代码。

第一步，我们先来整体看核心部分的代码。如下图所示：

![](/img/okhttp-2.png)

在 `api` 目录中有个 `okhttp.api` 文件，其内容如下：

```
public final class okhttp3/Address {
	public final fun -deprecated_certificatePinner ()Lokhttp3/CertificatePinner;
	public final fun -deprecated_connectionSpecs ()Ljava/util/List;
	public final fun -deprecated_dns ()Lokhttp3/Dns;
	public final fun -deprecated_hostnameVerifier ()Ljavax/net/ssl/HostnameVerifier;
	public final fun -deprecated_protocols ()Ljava/util/List;
	public final fun -deprecated_proxy ()Ljava/net/Proxy;
	public final fun -deprecated_proxyAuthenticator ()Lokhttp3/Authenticator;
	public final fun -deprecated_proxySelector ()Ljava/net/ProxySelector;
	public final fun -deprecated_socketFactory ()Ljavax/net/SocketFactory;
	public final fun -deprecated_sslSocketFactory ()Ljavax/net/ssl/SSLSocketFactory;
	public final fun -deprecated_url ()Lokhttp3/HttpUrl;
	public fun <init> (Ljava/lang/String;ILokhttp3/Dns;Ljavax/net/SocketFactory;Ljavax/net/ssl/SSLSocketFactory;Ljavax/net/ssl/HostnameVerifier;Lokhttp3/CertificatePinner;Lokhttp3/Authenticator;Ljava/net/Proxy;Ljava/util/List;Ljava/util/List;Ljava/net/ProxySelector;)V
	public final fun certificatePinner ()Lokhttp3/CertificatePinner;
	public final fun connectionSpecs ()Ljava/util/List;
	public final fun dns ()Lokhttp3/Dns;
	public fun equals (Ljava/lang/Object;)Z
	public fun hashCode ()I
	public final fun hostnameVerifier ()Ljavax/net/ssl/HostnameVerifier;
	public final fun protocols ()Ljava/util/List;
	public final fun proxy ()Ljava/net/Proxy;
	public final fun proxyAuthenticator ()Lokhttp3/Authenticator;
	public final fun proxySelector ()Ljava/net/ProxySelector;
	public final fun socketFactory ()Ljavax/net/SocketFactory;
	public final fun sslSocketFactory ()Ljavax/net/ssl/SSLSocketFactory;
	public fun toString ()Ljava/lang/String;
	public final fun url ()Lokhttp3/HttpUrl;
}
...
```

我只截取了其中一部分，这个文件是为了使用 [goctl](https://go-zero.dev/cn/docs/goctl/goctl/)，以自动生成代码。类似的工具还有 Google 发明的 protobuf。有兴趣的可以去 goctl 的官网看一下。

第二步，我们从代码的入口开始看。

我们先来看看，平时在使用 OkHttp 时，是如何使用的：

```kotlin
val client = OkHttpClient()
val request = Request.Builder()
    .url("https://www.google.com")
    .build()

// 同步方法
try {
    val response = client.newCall(request).execute() 
    print(response.body.string())
} catch (e: IOException) {
    e.printStackTrace()
}

// 异步回调方法
try {
    client.newCall(request).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
        }

        override fun onResponse(call: Call, response: Response) {
        }
    })
} catch (e: Exception) {
    e.printStackTrace()
}
```

先创建 `OkHttpClient` 实例，接着使用 Builder 模式构建一个 `Request` 实例，然后请求，得到结果。

我们一个个看，先看 `OkHttpClient`。

```kotlin
// okhttp3.OkHttpClient
/**
 * Factory for [calls][Call], which can be used to send HTTP requests and read their responses.
 *
 * ## OkHttpClients Should Be Shared
 *
 * OkHttp performs best when you create a single `OkHttpClient` instance and reuse it for all of
 * your HTTP calls. This is because each client holds its own connection pool and thread pools.
 * Reusing connections and threads reduces latency and saves memory. Conversely, creating a client
 * for each request wastes resources on idle pools.
 ..
**/
open class OkHttpClient internal constructor(
  builder: Builder
) : Call.Factory, WebSocket.Factory {
    @get:JvmName("dispatcher")
    val dispatcher: Dispatcher = builder.dispatcher  // 请求调度器

    ...
    constructor() : this(Builder())
    ...


    init {
        if (connectionSpecs.none { it.isTls }) {
            this.sslSocketFactoryOrNull = null
            this.certificateChainCleaner = null
            this.x509TrustManager = null
            this.certificatePinner = CertificatePinner.DEFAULT
        } else if (builder.sslSocketFactoryOrNull != null) {
            this.sslSocketFactoryOrNull = builder.sslSocketFactoryOrNull
            this.certificateChainCleaner = builder.certificateChainCleaner!!
            this.x509TrustManager = builder.x509TrustManagerOrNull!!
            this.certificatePinner = builder.certificatePinner.withCertificateChainCleaner(certificateChainCleaner!!)
        } else {
            this.x509TrustManager = Platform.get().platformTrustManager()
            this.sslSocketFactoryOrNull = Platform.get().newSslSocketFactory(x509TrustManager!!)
            this.certificateChainCleaner = CertificateChainCleaner.get(x509TrustManager!!)
            this.certificatePinner = builder.certificatePinner.withCertificateChainCleaner(certificateChainCleaner!!)
        }

        verifyClientState()
    }

    private fun verifyClientState() {
        check(null !in (interceptors as List<Interceptor?>)) {
            "Null interceptor: $interceptors"
        }
        check(null !in (networkInterceptors as List<Interceptor?>)) {
            "Null network interceptor: $networkInterceptors"
        }

        if (connectionSpecs.none { it.isTls }) {
            check(sslSocketFactoryOrNull == null)
            check(certificateChainCleaner == null)
            check(x509TrustManager == null)
            check(certificatePinner == CertificatePinner.DEFAULT)
        } else {
            checkNotNull(sslSocketFactoryOrNull) { "sslSocketFactory == null" }
            checkNotNull(certificateChainCleaner) { "certificateChainCleaner == null" }
            checkNotNull(x509TrustManager) { "x509TrustManager == null" }
        }
    }
}
```

我们看到，`OkHttpClient` 的构造函数中，调用了内部的一个 `Builder` 类。然后在 `init` 方法中进行了一些初始化与检查。

```kotlin
class Builder() {
    internal var dispatcher: Dispatcher = Dispatcher()
    internal var connectionPool: ConnectionPool = ConnectionPool()
    internal val interceptors: MutableList<Interceptor> = mutableListOf()
    internal val networkInterceptors: MutableList<Interceptor> = mutableListOf()
    internal var eventListenerFactory: EventListener.Factory = EventListener.NONE.asFactory()
    internal var retryOnConnectionFailure = true
    internal var fastFallback = true
    internal var authenticator: Authenticator = Authenticator.NONE
    internal var followRedirects = true
    internal var followSslRedirects = true
    internal var cookieJar: CookieJar = CookieJar.NO_COOKIES
    internal var cache: Cache? = null
    internal var dns: Dns = Dns.SYSTEM
    internal var proxy: Proxy? = null
    internal var proxySelector: ProxySelector? = null
    internal var proxyAuthenticator: Authenticator = Authenticator.NONE
    internal var socketFactory: SocketFactory = SocketFactory.getDefault()
    internal var sslSocketFactoryOrNull: SSLSocketFactory? = null
    internal var x509TrustManagerOrNull: X509TrustManager? = null
    internal var connectionSpecs: List<ConnectionSpec> = DEFAULT_CONNECTION_SPECS
    internal var protocols: List<Protocol> = DEFAULT_PROTOCOLS
    internal var hostnameVerifier: HostnameVerifier = OkHostnameVerifier
    internal var certificatePinner: CertificatePinner = CertificatePinner.DEFAULT
    internal var certificateChainCleaner: CertificateChainCleaner? = null
    internal var callTimeout = 0
    internal var connectTimeout = 10_000
    internal var readTimeout = 10_000
    internal var writeTimeout = 10_000
    internal var pingInterval = 0
    internal var minWebSocketMessageToCompress = RealWebSocket.DEFAULT_MINIMUM_DEFLATE_SIZE
    internal var routeDatabase: RouteDatabase? = null
    internal var taskRunner: TaskRunner? = null
    ...
}
```

在 `Builder` 的初始化中，赋值了各种内部成员变量的初始值。

接下来，Client 就会等待提交新的 Request。

```kotlin
/** Prepares the [request] to be executed at some point in the future. */
override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

```kotlin
// okhttp3.internal.connection.RealCall

/**
 * Bridge between OkHttp's application and network layers. This class exposes high-level application
 * layer primitives: connections, requests, responses, and streams.
 *
 * This class supports [asynchronous canceling][cancel]. This is intended to have the smallest
 * blast radius possible. If an HTTP/2 stream is active, canceling will cancel that stream but not
 * the other streams sharing its connection. But if the TLS handshake is still in progress then
 * canceling may break the entire connection.
 */
class RealCall(
    val client: OkHttpClient,
    /** The application's original request unadulterated by redirects or auth headers. */
    val originalRequest: Request,
    val forWebSocket: Boolean
) : Call, Cloneable {
    ...
}
```

注释机翻：这个类是 OkHttp 的应用层和网络层之间的桥梁。它暴露了高级应用层的基本操作：连接、请求、响应和流。此类支持异步取消操作，旨在尽可能地减少影响范围。如果存在活跃的 HTTP/2 流，取消操作将取消该流，但不会影响共享同一连接的其他流。但是，如果 TLS 握手仍在进行中，则取消操作可能会中断整个连接。

说人话版：这个类的功能就是前面说的『它是一个基于传输层（还记得网络的7层架构吗？）实现应用层协议的网络框架』。它起到了一个承上启下的作用，为底层**网络层**与上层**应用层**进行连接，同时也有一些自己的小九九，比如支持异步取消等。

初始化完成后，如果是执行 `execute()` 方法，我们来看看是如何执行的：

```kotlin
override fun execute(): Response {
    // AtomicBoolean 保证原子操作的唯一性
    check(executed.compareAndSet(false, true)) { "Already Executed" }

    timeout.enter()
    callStart()
    try {
        // 将此次请求的信息交给 OkHttpClient 中的请求调度器
        client.dispatcher.executed(this)
        return getResponseWithInterceptorChain()
    } finally {
        client.dispatcher.finished(this)
    }
}

private fun callStart() {
    this.callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()")
    eventListener.callStart(this)
}
```

`callStart()` 方法用来设置事件回调，如果你设置了监听事件（比如使用了 logging-interceptor），在触发相应事件时就会得到回调。