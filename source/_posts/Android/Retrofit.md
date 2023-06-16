---
title: Retrofit 实现原理解析
link: retrofit
toc: true
date: 2023-03-23
tag: 
    - Android
    - Retrofit
    - 三方库
category: 
    - Android
    - Development
---

[Retrofit](https://square.github.io/retrofit/) 是现在比较常用的网络请求库了，它的可扩展性强，底层网络请求集成了 Okhttp，异步处理可集成 RxJava，内容解析可集成 Gson，Jackson 等。而且全面支持 Restful 请求，并且通过注解的方式，支持链式调用，使用简洁方便。
精妙的源码设计模式，内部层次分工明确，解耦性强。

它的使用核心是实现一个 Java 接口，用注解的方式来指定各种网络请求中的字段、方法等。

<!-- more -->

拿官网的例子来说明：

## 活生生的例子

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

上面的例子，用了两个注解，分别是`@GET`和`@Path`，由字面意思也可以看出，`@GET`是指该方法是HTTP请求中的GET方法，`@Path("user")`是指将请求URL中的`{user}`部分替换为传入的`user`参数。

我们来看看这两个注解的源码：

```java
// retrofit2.http.GET.java

/** Make a GET request. */
@Documented
@Target(METHOD)
@Retention(RUNTIME)
public @interface GET {
  /**
   * A relative or absolute path, or full URL of the endpoint. This value is optional if the first
   * parameter of the method is annotated with {@link Url @Url}.
   * <p>
   * See {@linkplain retrofit2.Retrofit.Builder#baseUrl(HttpUrl) base URL} for details of how
   * this is resolved against a base URL to create the full endpoint URL.
   */
  String value() default "";
}
```

它使用了三个元注解，不再多解释。它的`value`指的是请求中的绝对/相对路径。

```java
// retrofit2.http.Path.java

@Documented
@Retention(RUNTIME)
@Target(PARAMETER)
public @interface Path {
  String value();

  /**
   * Specifies whether the argument value to the annotated method parameter is already URL encoded.
   */
  boolean encoded() default false;
}
```

它有两个参数，一个是`value`，一个是是否已经经过`URL encoded`。

那么，retrofit是如何将注解使用起来的呢？我们来看看它的原理。

## Retrofit 实现原理

先上一张图，简单了解一下它的整体流程：

![](/img/27.png)

在创建Retrofit实例时，我们一般会按照下面的方式来创建实例：

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

`build()`方法其实就是Java中比较典型的Builder设计模式，不过多解释，我们直接来看`create()`的源代码：

```java
// retrofit2.Retrofit.java
@SuppressWarnings("unchecked")
public <T> T create(final Class<T> service) {
    // 检查接口可用性
    validateServiceInterface(service);
    // 使用动态代理的方式创建一个 service 的实例
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override public @Nullable Object invoke(Object proxy, Method method,
                @Nullable Object[] args) throws Throwable {
                // 如果 method 是 Object 的方法，那么就正常调用
                if (method.getDeclaringClass() == Object.class) {
                    return method.invoke(this, args);
                }
                // 一般情况下 Android 的 Platform.isDefaultMethod() 都会返回 false
                if (platform.isDefaultMethod(method)) {
                    return platform.invokeDefaultMethod(method, service, proxy, args);
                }
                // 所以真正起作用的，还是这一句
                return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
            }
        }
    );
}
```

我们从这几行代码可以看出，Retrofit 是使用了动态代理的方式创建了一个实现service接口的对象，当外部调用 service 接口方法，会调用`invoke`方法，然后加载当前 method 对应的 ServiceMethod，并调用该 ServiceMethod 的 `invoke`。ServiceMethod 是个抽象类，那么，这里实际上是调用了 ServiceMethod 某一个实现类的 `invoke` 方法。具体的我们会在后面分析。

先看看`validateServiceInterface()`方法：

```java
// retrofit2.Retrofit.java

private void validateServiceInterface(Class<?> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }

    // Deque是Java1.6之后加入的『双向队列』，也即能从两个方向向外拿出数据
    Deque<Class<?>> check = new ArrayDeque<>(1);
    check.add(service);
    while (!check.isEmpty()) {
        // 获取该service的类
        Class<?> candidate = check.removeFirst();
        // getTypeParameters() 方法用于获取该类的声明中的参数列表
        // 这里要保证这个接口类没有任何的参数
        if (candidate.getTypeParameters().length != 0) {
            StringBuilder message = new StringBuilder("Type parameters are unsupported on ")
                .append(candidate.getName());

            if (candidate != service) {
                message.append(" which is an interface of ")
                    .append(service.getName());
            
            throw new IllegalArgumentException(message.toString());
        }

        // 将该service下所有的接口添加到check中
        Collections.addAll(check, candidate.getInterfaces());
    }

    // 如果配置了validateEagerly，则提前加载 method 对应的 ServiceMethod。
    // 此处调用 loadServiceMethod() 方法，其实是在做一个预处理的操作，只是提前创建了一些 ServiceMethod 对象在缓存中。
    if (validateEagerly) {
        Platform platform = Platform.get();
        for (Method method : service.getDeclaredMethods()) {
            if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
                loadServiceMethod(method);
            }
        }
    }
}
```

接下来看看在`create()`和`validateServiceInterface()`都会调用到的`loadServiceMethod()`方法：

```java
// 所有的 Method 都被存在一个 ConcurrentHashMap 中，避免同步问题
private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();

// 该方法的主要目的有两个：
// 1. 解析该 method 上的所有注解，并以 method - ServiceMethod 的形式放入缓存中，等待调用
// 2. 根据该 method 找到对应的 ServiceMethod，并返回
ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        // 此处 result 已经是 ServiceMethod 实现类 HttpSercieMethod 的实例
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
}
```

ServiceMethod 是 Retrofit 的一个抽象类，我们来看看它的源代码：

```java
// retrofit2.ServiceMethod.java

abstract class ServiceMethod<T> {
    static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
        RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

        Type returnType = method.getGenericReturnType();
        if (Utils.hasUnresolvableType(returnType)) {
            throw methodError(method,
                "Method return type must not include a type variable or wildcard: %s", returnType);
        }
        if (returnType == void.class) {
            throw methodError(method, "Service methods cannot return void.");
        }

        return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
    }

    abstract @Nullable T invoke(Object[] args);
}
```

RequestFactory是比较重要的一个类，它负责解析注解、构造 request 等，在后面会直接与okhttp进行交互，我们来看看它的源码：

```java
final class RequestFactory {
    static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
        return new Builder(retrofit, method).build();
    }

    // 储存 request 的一些数据
    private final Method method;
    private final HttpUrl baseUrl;
    final String httpMethod;
    private final @Nullable String relativeUrl;
    private final @Nullable Headers headers;
    private final @Nullable MediaType contentType;
    private final boolean hasBody;
    private final boolean isFormEncoded;
    private final boolean isMultipart;
    private final ParameterHandler<?>[] parameterHandlers;
    final boolean isKotlinSuspendFunction;

    ...

    /**
     * 检查接口中方法上的注解，来构造一个可复用的 service 方法。
     * 这个操作需要调用反射机制，所以每个 service 方法最好只构造一次，然后复用之。
     * Builder 不能复用。
     */
    static final class Builder {
        private static final String PARAM = "[a-zA-Z][a-zA-Z0-9_-]*";
        private static final Pattern PARAM_URL_REGEX = Pattern.compile("\\{(" + PARAM + ")\\}");
        private static final Pattern PARAM_NAME_REGEX = Pattern.compile(PARAM);
        ...
        Builder(Retrofit retrofit, Method method) {
            this.retrofit = retrofit;
            this.method = method;
            // 获取方法上的注解
            this.methodAnnotations = method.getAnnotations();
            // 获取方法上的参数类型
            this.parameterTypes = method.getGenericParameterTypes();
            // 获取参数上的注解
            this.parameterAnnotationsArray = method.getParameterAnnotations();
        }

        RequestFactory build() {
            // 解析 method 上的所有注解
            for (Annotation annotation : methodAnnotations) {
                parseMethodAnnotation(annotation);
            }
            ...
            // 各种 method 注解的检查
            ...
            // 解析 paremater 上的所有注解
            int parameterCount = parameterAnnotationsArray.length;
            parameterHandlers = new ParameterHandler<?>[parameterCount];
            for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
                // 并添加到一个 ParamaterHandler 的 Array 中
                parameterHandlers[p] =
                    parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
            }
            ...
            // 各种 parameter 注解的检查
            ...
            return new RequestFactory(this);
        }
    }

    // 解析 method 上的注解
    private void parseMethodAnnotation(Annotation annotation) {
        ...
        // 这里的 DELETE GET 等，都是 retrofit 自定义的各种 Http 请求类型的注解
        if (annotation instanceof DELETE) {
            parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
        } else if (annotation instanceof GET) {
            parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
        } ...

    }

    // 解析 method 上注解的详细数据
    private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
        if (this.httpMethod != null) {
            throw methodError(method, "Only one HTTP method is allowed. Found: %s and %s.",
                this.httpMethod, httpMethod);
        }
        this.httpMethod = httpMethod;
        this.hasBody = hasBody;

        if (value.isEmpty()) {
            return;
        }

        // 获取 URL 的相对路径，如果有 ? 参数的话，就获取一下
        int question = value.indexOf('?');
        if (question != -1 && question < value.length() - 1) {
            // Ensure the query string does not have any named parameters.
            String queryParams = value.substring(question + 1);
            Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
            if (queryParamMatcher.find()) {
            throw methodError(method, "URL query string \"%s\" must not have replace block. "
                + "For dynamic query parameters use @Query.", queryParams);
            }
        }

        // 拿到注解中的 value，此为 relativeUrl
        this.relativeUrl = value;
        // 解析参数们
        this.relativeUrlParamNames = parsePathParameters(value);
    }

    // 解析参数
    private @Nullable ParameterHandler<?> parseParameter(
        int p, Type parameterType, @Nullable Annotation[] annotations, boolean allowContinuation) {
        ParameterHandler<?> result = null;
        if (annotations != null) {
            for (Annotation annotation : annotations) {
                // 解析参数的注解
                ParameterHandler<?> annotationAction =
                    parseParameterAnnotation(p, parameterType, annotations, annotation);

                if (annotationAction == null) {
                    continue;
                }

                if (result != null) {
                    throw parameterError(method, p,
                        "Multiple Retrofit annotations found, only one allowed.");
                }

                result = annotationAction;
            }
        }

        if (result == null) {
            if (allowContinuation) {
                try {
                    if (Utils.getRawType(parameterType) == Continuation.class) {
                        isKotlinSuspendFunction = true;
                        return null;
                    }
                } catch (NoClassDefFoundError ignored) {
                }
            }
            throw parameterError(method, p, "No Retrofit annotation found.");
        }

        return result;
    }

    @Nullable
    private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {
        // 这里的 Url Path Query 等，是 retrofit 自定义的注解，通过字符串解析等各种方式，来组成正确的 ParameterHandler
        if (annotation instanceof Url) {
            ...
            return new ParameterHandler.RelativeUrl(method, p);
            ...
        } else if (annotation instanceof Path) {
            ...
            return new ParameterHandler.Path<>(method, p, name, converter, path.encoded());
            ...
        } else if (annotation instanceof Query) {
            // 注意三种不同的返回值
            Class<?> rawParameterType = Utils.getRawType(type);
            gotQuery = true;
            if (Iterable.class.isAssignableFrom(rawParameterType)) {
                if (!(type instanceof ParameterizedType)) {
                    throw parameterError(method, p, rawParameterType.getSimpleName()
                        + " must include generic type (e.g., "
                        + rawParameterType.getSimpleName()
                        + "<String>)");
                }
                ParameterizedType parameterizedType = (ParameterizedType) type;
                Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
                Converter<?, String> converter =
                    retrofit.stringConverter(iterableType, annotations);
                return new ParameterHandler.Query<>(name, converter, encoded).iterable();
            } else if (rawParameterType.isArray()) {
                Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
                Converter<?, String> converter =
                    retrofit.stringConverter(arrayComponentType, annotations);
                return new ParameterHandler.Query<>(name, converter, encoded).array();
            } else {
                Converter<?, String> converter =
                    retrofit.stringConverter(type, annotations);
                return new ParameterHandler.Query<>(name, converter, encoded);
            }
        } else if (annotation instanceof Header) {
            ...
            return new ParameterHandler.Header<>(name, converter);
            ...
        } else if (annotation instanceof Field) {
            ...
            return new ParameterHandler.Field<>(name, converter, encoded);
            ...
        } else if (annotation instanceof Body) {
            ...
            return new ParameterHandler.Body<>(method, p, converter);
        }

        ...

        return null; // Not a Retrofit annotation.
    }
}
```

从这几部分代码可以看出，最后RequestFactory的实例中，包含的是：

- HTTP 请求的 BaseUrl
- HTTP 请求的 RelativeUrl
- HTTP 请求的方法
- HTTP 请求的参数列表（可选）
- HTTP 请求的 Body（可选）
- HTTP 请求的 Headers（可选）
- ParameterHandler 的 Array

这ParameterHandler是什么呢？我们来看看它的代码：

```java
// retrofit2.ParameterHandler.java

abstract class ParameterHandler<T> {
    abstract void apply(RequestBuilder builder, @Nullable T value) throws IOException;

    final ParameterHandler<Iterable<T>> iterable() {
        return new ParameterHandler<Iterable<T>>() {
            @Override void apply(RequestBuilder builder, @Nullable Iterable<T> values)
                throws IOException {
                if (values == null) return; // Skip null values.

                for (T value : values) {
                    ParameterHandler.this.apply(builder, value);
                }
            }
        };
    }

    final ParameterHandler<Object> array() {
        return new ParameterHandler<Object>() {
            @Override void apply(RequestBuilder builder, @Nullable Object values) throws IOException {
                if (values == null) return; // Skip null values.

                for (int i = 0, size = Array.getLength(values); i < size; i++) {
                    //noinspection unchecked
                    ParameterHandler.this.apply(builder, (T) Array.get(values, i));
                }
            }
        };
    }

    // 一个 ParameterHandler的具体实现类
    // 包含 Method 和 index
    static final class RelativeUrl extends ParameterHandler<Object> {
        private final Method method;
        private final int p;

        RelativeUrl(Method method, int p) {
            this.method = method;
            this.p = p;
        }

        @Override void apply(RequestBuilder builder, @Nullable Object value) {
            if (value == null) {
                throw Utils.parameterError(method, p, "@Url parameter is null.");
            }
            builder.setRelativeUrl(value);
        }
    }
}
```

可以看出，每一种 retrofit 自定义的注解（GET、POST、FIELD、Path）等，最终都会转化成一个 ParameterHandler 的实例，并实现 `apply()` 方法。`apply()`方法中会传入 RequestBuild 的实例，这是产生 Request 的倒数第二步。

好，说了这么一大圈，我们还是得回到 ServiceMethod 类中。

生成了 RequestFactory 实例后，利用这个实例，又调用了 `HttpServiceMethod.parseAnnotations()`方法，我们来看看这个类，以及其包含的方法：

```java
// retrofit2.HttpServiceMethod.java

/** Adapts an invocation of an interface method into an HTTP call. */
// 这是 ServiceMethod 的唯一实现类
abstract class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT> {
      
      static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
        Retrofit retrofit, Method method, RequestFactory requestFactory) {
        boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
        boolean continuationWantsResponse = false;
        boolean continuationBodyNullable = false;

        Annotation[] annotations = method.getAnnotations();
        Type adapterType;
        ...
        // ❶ 一个比较核心的方法，也是 retrofit 的灵魂所在，我们在后面要详细解释
        CallAdapter<ResponseT, ReturnT> callAdapter =
            createCallAdapter(retrofit, method, adapterType, annotations);
        Type responseType = callAdapter.responseType();
        
        ...
        // 各种检查
        ...

        // ❷ retrofit 的另一个重要特性『Converters』，可以自动反序列化返回的数据
        Converter<ResponseBody, ResponseT> responseConverter =
            createResponseConverter(retrofit, method, responseType);

        okhttp3.Call.Factory callFactory = retrofit.callFactory;
        
        if (!isKotlinSuspendFunction) {
            return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
        } else if (continuationWantsResponse) {
            // 与 kotlin 协程有关的一些东西，后面再讲
            // noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
            return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForResponse<>(requestFactory,
                callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
        } else {
            // 与 kotlin 协程有关的一些东西，后面再讲
            // noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
            return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForBody<>(requestFactory,
                callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
                continuationBodyNullable);
        }
    }

    private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
        Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
        try {
            // noinspection unchecked
            return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
        } catch (RuntimeException e) { // Wide exception range because factories are user code.
            throw methodError(method, e, "Unable to create call adapter for %s", returnType);
        }
    }

    private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
        Retrofit retrofit, Method method, Type responseType) {
        Annotation[] annotations = method.getAnnotations();
        try {
            return retrofit.responseBodyConverter(responseType, annotations);
        } catch (RuntimeException e) { // Wide exception range because factories are user code.
            throw methodError(method, e, "Unable to create converter for %s", responseType);
        }
    }

    // 在 Retrofit.create() 方法的最后一句，调用的 invoke()，其实就是调用的这里
    @Override final @Nullable ReturnT invoke(Object[] args) {
        Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
        return adapt(call, args);
    }

    // 抽象方法，由 CallAdapted、SuspendForResponse、SuspendForBody 来实现
    protected abstract @Nullable ReturnT adapt(Call<ResponseT> call, Object[] args);

    static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
        private final CallAdapter<ResponseT, ReturnT> callAdapter;

        CallAdapted(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
            Converter<ResponseBody, ResponseT> responseConverter,
            CallAdapter<ResponseT, ReturnT> callAdapter) {
        super(requestFactory, callFactory, responseConverter);
            this.callAdapter = callAdapter;
        }

        @Override protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
            return callAdapter.adapt(call);
        }
    }

    static final class SuspendForResponse<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
        private final CallAdapter<ResponseT, Call<ResponseT>> callAdapter;

        SuspendForResponse(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
            Converter<ResponseBody, ResponseT> responseConverter,
            CallAdapter<ResponseT, Call<ResponseT>> callAdapter) {
        super(requestFactory, callFactory, responseConverter);
            this.callAdapter = callAdapter;
        }

        @Override protected Object adapt(Call<ResponseT> call, Object[] args) {
            call = callAdapter.adapt(call);

            //noinspection unchecked Checked by reflection inside RequestFactory.
            Continuation<Response<ResponseT>> continuation =
                (Continuation<Response<ResponseT>>) args[args.length - 1];

            // See SuspendForBody for explanation about this try/catch.
            try {
                return KotlinExtensions.awaitResponse(call, continuation);
            } catch (Exception e) {
                return KotlinExtensions.suspendAndThrow(e, continuation);
            }
        }
    }
}
```

HttpServiceMethod类的主要功能，是创建了一个 CallAdapter 对象，这个对象是用来把 OkHttpCall 适配为我们定义的方法的返回值类型的；创建 Converter 对象，用于把 ResponseBody 转换为我们最终需要的类型；生成 HttpServiceMethod 的三种派生类其中一种的实例，返回并放到 `serviceMethodCashe` 中。

上面的代码中有两处比较重要的地方，用序号标注了出来，我们来单独讲讲：

## 灵魂 ❶ CallAdapter

跟着`retrofit.callAdapter()`向下走，我们看到了它的代码：

```java
// retrofit2.Retrofit.java

public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    ...
    // callAdapterFactories 在 Builder.build() 中添加，见下
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
        CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
        if (adapter != null) {
            return adapter;
        }
    }

    // 如果在上面还没有 return，则处理一些错误
    ...
}

public static final class Builder {
    Builder(Retrofit retrofit) {
        // 不要添加默认的、build() 方法里添加的的平台相关的 CallAdapter
        for (int i = 0,
            size = retrofit.callAdapterFactories.size() - platform.defaultCallAdapterFactoriesSize();
            i < size; i++) {
            callAdapterFactories.add(retrofit.callAdapterFactories.get(i));
        }
    }

    public Retrofit build() {
        ...
        // 初始化 okhttpclient 等等
        ...
        // 执行器
        Executor callbackExecutor = this.callbackExecutor;
        if (callbackExecutor == null) {
            callbackExecutor = platform.defaultCallbackExecutor();
        }
        // 添加平台相关的 CallAdapter
        List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
        callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
        ...
        // 返回 Retrofit 实例
        return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
            unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
}
```

再看看Platform.defaultCallAdapterFactories()做了些什么：

```java
// retrofit.Platform.java

List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
    @Nullable Executor callbackExecutor) {
    DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);

    // Android 中 Build.VERSION.SDK_INT >= 24 时，hasJava8Types会被置为true
    return hasJava8Types
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
}
```

然后我们看看 DefautCallAdapterFactory 类，顾名思义，它是创建 CallAdapter 的一个工厂类：

```java
// retrofit2.DefaultCallAdapterFactory.java

final class DefaultCallAdapterFactory extends CallAdapter.Factory {
    ...
    @Override public @Nullable CallAdapter<?, ?> get(
        Type returnType, Annotation[] annotations, Retrofit retrofit) {
        if (getRawType(returnType) != Call.class) {
            return null;
        }
        if (!(returnType instanceof ParameterizedType)) {
            throw new IllegalArgumentException(
                "Call return type must be parameterized as Call<Foo> or Call<? extends Foo>");
        }
        final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

        final Executor executor = Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;

        return new CallAdapter<Object, Call<?>>() {
            @Override public Type responseType() {
                return responseType;
            }

            @Override public Call<Object> adapt(Call<Object> call) {
                return executor == null
                    ? call
                    : new ExecutorCallbackCall<>(executor, call);
            }
        };
    }
    ...
}
```

CallAdapter 的实现类终于处是找到了，但是它的 `adapt()` 方法才是真正起作用的方法。从上到下，调用到 `adapt()` 的时候，返回的是一个ExcutorCallbackCall的实例。

## 特性 ❷ Converter

跟着`retrofit.responseBodyConverter()`方法往下走，我们看到了它的代码：

```java
// retrofit2.Retrofit.java

public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
}


public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
    @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    ...
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        Converter<ResponseBody, ?> converter =
            converterFactories.get(i).responseBodyConverter(type, annotations, this);
        if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
        }
    }
}

public static final class Builder {
    Builder(Retrofit retrofit) {
        // 不要添加默认的、build() 方法里添加的的平台相关的 BuiltIntConverters
        for (int i = 1,
            size = retrofit.converterFactories.size() - platform.defaultConverterFactoriesSize();
            i < size; i++) {
            converterFactories.add(retrofit.converterFactories.get(i));
        }
    }

    public Retrofit build() {
        ...
        // 初始化 okhttoclient 等等
        ...
        List<Converter.Factory> converterFactories = new ArrayList<>(
            1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

        // 先添加内置的 converter，里面实现了多个转换器包括把字节流转换成 ResponseBody，
        // ResponseBody 转换为 java 中的 Void 或者 Kotlin 中的 Unit 时关闭流的操作等
        converterFactories.add(new BuiltInConverters());
        // 有自定义的就添加自定义，
        // 比如我们常用的 GsonConverterFactory 
        converterFactories.addAll(this.converterFactories);
        // 平台默认的 converter
        converterFactories.addAll(platform.defaultConverterFactories());

        // 返回 Retrofit 实例
        return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
            unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
}
```

```java
// retrofit2.Platform.java

List<? extends Converter.Factory> defaultConverterFactories() {
    // SDK 版本小于24，就没有默认的 converter
    return hasJava8Types
        ? singletonList(OptionalConverterFactory.INSTANCE)
        : emptyList();
}
```

可以看到，与 CallAdapter 的生成方式几乎一模一样。

## invoke 阶段

OK，所有初始化完成，接下来进入调用阶段，我们接着文章开头的例子，来看一下调用的入口：

```java
...
GitHubService service = retrofit.create(GitHubService.class);
Call<List<Repo>> call = service.listRepos("notex");
call.enqueue(new Callback<List<Repo>>() {
    @Override 
    public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
    }

    @Override 
    public void onFailure(Call<List<Repo>> call, Throwable t) {
    }
});
```

那么，我们就以`enqueue()`方法为入口，看看调用的流程是怎样的。

刚才经过我们的分析，目前的`call`变量应该是一个 ExecutorCallbackCall 的实例，我们来看看它的代码：

```java
// retrofit2.DefaultCallAdapterFactory.java

static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
        this.callbackExecutor = callbackExecutor;
        this.delegate = delegate;
    }

    @Override 
    public void enqueue(final Callback<T> callback) {
        Objects.requireNonNull(callback, "callback == null");

        delegate.enqueue(new Callback<T>() {
        @Override 
        public void onResponse(Call<T> call, final Response<T> response) {
            callbackExecutor.execute(() -> {
            if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
            } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
            }
            });
        }

        @Override 
        public void onFailure(Call<T> call, final Throwable t) {
            callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
        }
        });
    }

    @Override 
    public boolean isExecuted() {
        return delegate.isExecuted();
    }

    @Override 
    public Response<T> execute() throws IOException {
        return delegate.execute();
    }

    @Override 
    public void cancel() {
        delegate.cancel();
    }

    @Override 
    public boolean isCanceled() {
        return delegate.isCanceled();
    }

    @SuppressWarnings("CloneDoesntCallSuperClone") // Performing deep clone.
    @Override 
    public Call<T> clone() {
        return new ExecutorCallbackCall<>(callbackExecutor, delegate.clone());
    }

    @Override 
    public Request request() {
        return delegate.request();
    }

    @Override 
    public Timeout timeout() {
        return delegate.timeout();
    }
}
```

`delegate` 是 OkHttpCall 对象，使用过 OkHttp 的应该知道它的 `enqueue` 是执行在子线程的，所以 ExecutorCallbackCall 为我们做的就是把在子线程的回调中，通过 MainThreadExecutor 在主线程中调用 `retrofit2.Callback` 的回调。当然这个 `enqueue` 还不是真正的 OkHttp 的 `enqueue`，它做了封装。

接着就看看`OkHttpCall.enqueue()`方法：

```java
// retrofit2.OkhttpCall.java
 @Override 
 public void enqueue(final Callback<T> callback) {
    ...
    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
        if (executed) throw new IllegalStateException("Already executed.");
        executed = true;

        call = rawCall;
        failure = creationFailure;
        if (call == null && failure == null) {
            try {
                // createRawCall() 方法见下
                call = rawCall = createRawCall();
            } catch (Throwable t) {
                throwIfFatal(t);
                failure = creationFailure = t;
            }
        }
    }

    if (failure != null) {
        callback.onFailure(this, failure);
        return;
    }

    if (canceled) {
        call.cancel();
    }

    // 这里才是真正调用到了 Okhttp.enqueue()
    call.enqueue(new okhttp3.Callback() {
        @Override 
        public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
            Response<T> response;
            try {
                response = parseResponse(rawResponse);
            } catch (Throwable e) {
                throwIfFatal(e);
                callFailure(e);
                return;
            }

            try {
                callback.onResponse(OkHttpCall.this, response);
            } catch (Throwable t) {
                throwIfFatal(t);
                t.printStackTrace(); // TODO this is not great
            }
        }

        @Override 
        public void onFailure(okhttp3.Call call, IOException e) {
            callFailure(e);
        }

        private void callFailure(Throwable e) {
            try {
                callback.onFailure(OkHttpCall.this, e);
            } catch (Throwable t) {
                throwIfFatal(t);
                t.printStackTrace(); // TODO this is not great
            }
        }
    });
}


private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
        throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
```

看到这里，不禁感叹，Retrofit 的工厂模式真是应用得出神入化，无论是 Call，还是 Request，全部用工厂模式来创建。

在上面我们讲过，RequestFactory 的一个重要功能是 `parseAnnotation()`，现在它的另一个重要功能来了——构建 Request。

```java
// retrofit2.RequestFactory.java

okhttp3.Request create(Object[] args) throws IOException {
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    // 还记得 parseAnnotations() 方法吗，解析出来的 parameterHandlers 此处派上用场了
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args.length;
    if (argumentCount != handlers.length) {
        throw new IllegalArgumentException("Argument count (" + argumentCount
            + ") doesn't match expected count (" + handlers.length + ")");
    }

    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl,
        headers, contentType, hasBody, isFormEncoded, isMultipart);

    if (isKotlinSuspendFunction) {
        // The Continuation is the last parameter and the handlers array contains null at that index.
        argumentCount--;
    }

    List<Object> argumentList = new ArrayList<>(argumentCount);
    for (int p = 0; p < argumentCount; p++) {
        argumentList.add(args[p]);
        handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.get()
        .tag(Invocation.class, new Invocation(method, argumentList))
        .build();
    }
}
```

此时，有了 Request，就可以真正地调用`Okhttp.enqueue()`方法来进行请求了。

在有了正确的回调结果后，还需要有一步`response = parseResponse(rawResponse)`来解析数据，我们看看它是如何做的：

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
        try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
        } finally {
        rawBody.close();
        }
    }

    if (code == 204 || code == 205) {
        rawBody.close();
        return Response.success(null, rawResponse);
    }

    ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
    try {
        T body = responseConverter.convert(catchingBody);
        return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
        // If the underlying source threw an exception, propagate that rather than indicating it was
        // a runtime exception.
        catchingBody.throwIfCaught();
        throw e;
    }
}
```

回调操作，这里还是在子线程，通过 ExecutorCallbackCall 里面的流程，调用到主线程，到这里我们就真正的走过了一次 Retrofit 从 create 创建 Call 到发送请求，然后到回调主线程的操作了。

至此，整个 Retrofit 的一次完整的流程便完成了，我们使用下面一张图来总结一下：

![](/img/28.png)

### 创建Retrofit实例阶段
1. 调用`new Retrofit.Builder().build()`
2. Retrofit 类在`build()`方法中有如下步骤：
   1) 创建 OkHttpClient 实例
   2) MainThreadExecutor 实例
   3) 调用`Platform.defaultCallAdapterFactories()`方法创建 CallAdapter
   4) 调用`Platform.defaultConverterFactoriesSize()`方法创建 Converter
   5) 新建并返回 Retrofit 实例

### 创建接口代理阶段
1. 调用`Retrofit.create()`方法创建接口实例
2. 在`Retrofit.create()`方法中有如下步骤：
   1) 检查接口
   2) 调用`loadServiceMethod()`方法
   3) 创建一个方法代理并返回
3. 在`loadServiceMethod()`方法中有如下步骤：
   1) 解析注解
   2) 解析参数
   3) 生成 RequestFactory 实例
   4) 调用`HttpServiceMethod.parseAnnotations()`方法
4. 在`HttpServiceMethod.parseAnnotations()`方法中有如下步骤：
   1) 调用`createCallAdapter()`方法，从 Retrofit 实例的 callAdapterFactories 中拿取 CallAdapter
   2) 返回 HttpServiceMethod 的派生类 CallAdapted | SuspendForResponse | SuspendForBody 的实例
5. 将上一步返回的实例添加到缓存中等待读取和调用

### 触发代理机制阶段
1. 调用接口方法，触发代理机制
2. 在代理的`invoke()`方法被调用时，有如下步骤：
   1) 使用`loadServiceMethod()`获取该方法对应的ServiceMethod
   2) 调用`ServiceMethod.invoke()`方法
   3) 调用`HttpServiceMethod.invoke()`方法
   4) 调用 HttpServiceMethod 的`adapt()`方法，其派生类之一的`adapt()`方法被调用
   5) `CallAdapter.adapt()`被调用
   6) 返回 ExecutorCallbackCall 实例

### 请求阶段
1.  调用`ExecutorCallbackCall.enqueue()`方法发起请求
2.  在`ExecutorCallbackCall.enqueue()`方法中有如下步骤：
   1) 创建 Request 实例，并调用`callFactory.newCall(Request)`方法创建 RealCall 实例
   2) 调用`RealCall.enqueue()`方法，使用 okhttp 发起请求
   3)  回调到主线程
   4)  解析并利用 Converter 反序列化返回的结果
   5)  回调到调用者