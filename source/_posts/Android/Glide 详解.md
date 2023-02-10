---
title: 关于 Glide 的一切
link: glide
cover: https://github.com/bumptech/glide/blob/master/static/glide_logo.png?raw=true
toc: true
tag: 
    - Android
    - Glide
    - 三方库
category: 
    - Android
    - Development
date: 2020-03-28
---

Glide 是比较著名的图片加载库之一，类似的库还有 Picasso。这篇文章来讲讲 Glide 是如何加载图片到显示的，并且讲一讲它有哪些设计精妙的地方。

该文章使用的是 Glide 目前的最新版本[4.11.0](https://search.maven.org/artifact/com.github.bumptech.glide/glide/4.11.0/jar)。

<!-- more -->

我们先看一下 Glide 在4.0+版本的基本用法。

```java
// 单一 imageview
ImageView imageView = (ImageView) findViewById(R.id.my_image_view);
Glide.with(this).load("http://image.url").into(imageView);

// list 中的 imageview
@Override 
public View getView(int position, View recycled, ViewGroup container) {
  final ImageView myImageView;
  if (recycled == null) {
    myImageView = (ImageView) inflater.inflate(R.layout.my_image_view, container, false);
  } else {
    myImageView = (ImageView) recycled;
  }

  String url = myUrls.get(position);

  Glide
    .with(myFragment)
    .load(url)
    .centerCrop()
    .placeholder(R.drawable.loading_spinner)
    .into(myImageView);

  return myImageView;
}
```

通过两种调用方式，我们可以看出，Glide 用了非常经典的 Builder 设计模式，将各种参数收集、处理，最后调用`into()`方法，将图片交给 ImageView。接下来我们来一步步分析。

## `with()`

首先看`Glide.with()`方法的源码，该方法有5个重载，大体相同，但也有不同的地方，我们在下面会解释：

```java
@NonNull
public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
}

@NonNull
public static RequestManager with(@NonNull Activity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull Fragment fragment) {
    return getRetriever(fragment.getContext()).get(fragment);
}

@SuppressWarnings("deprecation")
@Deprecated
@NonNull
public static RequestManager with(@NonNull android.app.Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

@NonNull
public static RequestManager with(@NonNull View view) {
    return getRetriever(view.getContext()).get(view);
}
```

可以看到，`with(android.app.Fragment)`方法已经被弃用了，取而代之的是`with(androidx.fragment.app.Fragment)`，我们接下来不再讨论被弃用的这个方法。

那么，这5个方法，有什么不同呢？

首先，从它们的传入参数可以看出，它们分别用在 Application 级别、Activity 级别、FragmentActivity 级别、Fragment 级别和 View 级别。

Application 级别意味着它可以用在 Service 里，甚至 Notification 的 Thumbnail 里。而其他的几个，只能用在主线程的 Activity 的 View 树中。

需要注意的是，**`with(View)`方法在 View 没有被`attach`之前无效**。并且该方法效率较低，不推荐使用。

接着看看这五个重载方法都调用的`getRetriever()`方法的代码：

```java
@NonNull
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // context 可能为 null（比如用户就传了个 null），
    // 但实际上它只会出现在 Fragment 的生命周期出现错误时
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
}

@NonNull
public RequestManagerRetriever getRequestManagerRetriever() {
    return requestManagerRetriever;
}
```

可见`getRetriever()`方法实际上是在获取一个 **RequestManger 的生成器**。我们先把生成器放一边，先看看 Glide 是如何实例化和初始化的：

```java
// 获取 Glide 的单例
@NonNull
public static Glide get(@NonNull Context context) {
    if (glide == null) {
        // 解析注解配置生成的类 com.bumptech.glide.GeneratedAppGlideModuleImpl
        GeneratedAppGlideModule annotationGeneratedModule =
            getAnnotationGeneratedGlideModules(context.getApplicationContext());
        synchronized (Glide.class) {
            if (glide == null) {
                checkAndInitializeGlide(context, annotationGeneratedModule);
            }
        }
    }

    return glide;
}

// 只有拿到了 Glide.class 这个引用的锁才可以调用该方法
@GuardedBy("Glide.class")
private static void checkAndInitializeGlide(
    @NonNull Context context, @Nullable GeneratedAppGlideModule generatedAppGlideModule) {
    if (isInitializing) {
        throw new IllegalStateException(
            "You cannot call Glide.get() in registerComponents(),"
                + " use the provided Glide instance instead");
    }
    isInitializing = true;
    initializeGlide(context, generatedAppGlideModule);
    isInitializing = false;
}

@GuardedBy("Glide.class")
private static void initializeGlide(
    @NonNull Context context, @Nullable GeneratedAppGlideModule generatedAppGlideModule) {
    initializeGlide(context, new GlideBuilder(), generatedAppGlideModule);
}

@GuardedBy("Glide.class")
@SuppressWarnings("deprecation")
private static void initializeGlide(
    @NonNull Context context,
    @NonNull GlideBuilder builder,
    @Nullable GeneratedAppGlideModule annotationGeneratedModule) {
    // 解析 GlideModule 配置
    Context applicationContext = context.getApplicationContext();
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
        manifestModules = new ManifestParser(applicationContext).parse();
    }

    if (annotationGeneratedModule != null
        && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
        Set<Class<?>> excludedModuleClasses = annotationGeneratedModule.getExcludedModuleClasses();
        Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
        while (iterator.hasNext()) {
            com.bumptech.glide.module.GlideModule current = iterator.next();
            if (!excludedModuleClasses.contains(current.getClass())) {
                continue;
            }
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
            }
            iterator.remove();
        }
    }

    ...

    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory()
            : null;
    builder.setRequestManagerFactory(factory);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
        module.applyOptions(applicationContext, builder);
    }
    if (annotationGeneratedModule != null) {
        annotationGeneratedModule.applyOptions(applicationContext, builder);
    }

    // 创建 Glide 实例
    Glide glide = builder.build(applicationContext);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
        try {
        module.registerComponents(applicationContext, glide, glide.registry);
        } catch (AbstractMethodError e) {
        throw new IllegalStateException(
            "Attempting to register a Glide v3 module. If you see this, you or one of your"
                + " dependencies may be including Glide v3 even though you're using Glide v4."
                + " You'll need to find and remove (or update) the offending dependency."
                + " The v3 module name is: "
                + module.getClass().getName(),
            e);
        }
    }
    if (annotationGeneratedModule != null) {
        annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    applicationContext.registerComponentCallbacks(glide);
    
    // glide 变量的声明方式是 private static volatile Glide glide; 使用了 volatile 关键字保证了可见性
    Glide.glide = glide;
}
```

在初始化工作中，主要是解析了 GlideModule，也即配置。在 Glide 4.0 中，有一个与 3.0 不一样的地方就在这里了：**Glide 的 Module 配置不再是在 Manifest 中注册，而是通过在配置类上注解（@GlideModule）的方式来声明一个配置类**。

它的使用方法如下：

```java
@GlideModule
public class GlideModuleExample extends AppGlideModule {
}
```

当我们编译时，因为如果要使用 Glide 4.0，我们必须在 app/build.gradle 中这样配置：

```groovy
implementation 'com.github.bumptech.glide:glide:4.11.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.11.0'
```

也即 glide 有自己的注解处理系统，当我们编译完成后，会生成下面几个类：

![](/img/68.png)

其中就有我们上面提到的`com.bumptech.glide.GeneratedAppGlideModuleImpl`类。

配置解析完成后，就通过`GlideBuilder.build()`方法来创建实例了。创建完成后，将该实例赋给一个静态的 volatile 的变量。来看看`build()`方法做了些什么：

```java
@NonNull
Glide build(@NonNull Context context) {
    // 新建一个源线程的线程池
    if (sourceExecutor == null) {
        sourceExecutor = GlideExecutor.newSourceExecutor();
    }
    // 新建一个缓存用的线程池
    if (diskCacheExecutor == null) {
        diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }
    // 新建一个动画用的线程池
    if (animationExecutor == null) {
        animationExecutor = GlideExecutor.newAnimationExecutor();
    }
    // 基于设备的一些信息设定缓存区域的大小
    if (memorySizeCalculator == null) {
        memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }
    // 监听网络状态，如果没有 android.permission.ACCESS_NETWORK_STATE 权限的话，就不监测
    if (connectivityMonitorFactory == null) {
        connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }
    // 根据之前获取的缓存区域的大小，决定要用哪种 bitmapPool
    if (bitmapPool == null) {
        int size = memorySizeCalculator.getBitmapPoolSize();
        if (size > 0) {
            // 使用 LRU 算法的 Bitmap 池
            bitmapPool = new LruBitmapPool(size);
        } else {
            // 使用空 Bitmap 池
            bitmapPool = new BitmapPoolAdapter();
        }
    }

    if (arrayPool == null) {
        arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }

    if (memoryCache == null) {
        memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {
        diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
    // Engine 用来管理上方初始化的各种线程池和缓存 
    if (engine == null) {
        engine =
            new Engine(
                memoryCache,
                diskCacheFactory,
                diskCacheExecutor,
                sourceExecutor,
                GlideExecutor.newUnlimitedSourceExecutor(),
                animationExecutor,
                isActiveResourceRetentionAllowed);
    }

    if (defaultRequestListeners == null) {
        defaultRequestListeners = Collections.emptyList();
    } else {
        defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }

    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptionsFactory,
        defaultTransitionOptions,
        defaultRequestListeners,
        isLoggingRequestOriginsEnabled,
        isImageDecoderEnabledForBitmaps);
}
```

可见 Glide 在初始化中做了很多的工作。

1. 首先初始化了3个线程池，分别用来处理线程、LRU缓存、动画
2. 根据设备的具体信息，初始化缓存区域
3. 初始化了一个 Engine 的实例用来管理上面实例化的东西
4. 生成一个 RequestManagerRetriver 的实例，后面会用来获取 RequestManager
5. 生成 Glide 实例

## `load()`

在调用with方法获取到RequestManager对象的前提下，调用load方法，并传递我们的url参数，来看下它的源码：

[继续看这里](https://www.cnblogs.com/guanmanman/p/7008259.html)

## `into()`

[继续看这里](https://www.cnblogs.com/guanmanman/p/7040942.html)