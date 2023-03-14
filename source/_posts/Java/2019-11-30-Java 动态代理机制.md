---
title: Java动态代理机制
date: 2019-11-30
toc: true
link: dynamic-proxy
cover: http://blog.cigis-cloud.com/java-dynamic-proxy-1593752525.png
category: 
    - Development
tag:
    - Java
    - 动态代理
    - 设计模式
---

Java 动态代理是一种代理机制，它允许我们动态地、在不修改原代码的基础上，让代码完成它本来无法完成的工作。

<!-- more -->


## 静态代理是什么

哎哎？标题不是在讲动态代理吗？

要想知道动态代理，必须先知道静态代理。

静态代理模式可以在**不修改被代理对象**的基础上，通过扩展代理类，进行一些功能的附加与增强。值得注意的是，代理类和被代理类应该共同实现一个接口，或者是共同继承某个类。它的类型是事先预定好的，在编译时**会被编译进 .class 文件**中。

举个例子。

比如我电影院要放电影，放电影之前要放广告吧，但是电影本身是不能进行修改的，我们就可以用代理的方法，在电影的前、后都加入广告的播放功能。

首先我们设计一个接口，有了通用接口，才有了代理模式实现的基础。

```java
package com.debugLife.proxyTest;

public interface Movie {
    public void play();
}
```

接下来我们来一部电影（被代理类）：

```java
public com.debugLife.proxyTest;

public class AMovie implements Movie {
    @Override
    public void play() {
        System.out.println("正在播放电影 -《指环王》");
    }
}
```

它实现了 Movie 接口，当调用`play()`方法时，电影就开始播放了。

接下来是电影院（代理类）：

```java
public com.debugLife.proxyTest;

public class Cinema implements Movie {
    AMovie aMovie;

    public Cinema(AMovie aMovie) {
        super();
        this.aMovie = aMovie;
    }

    @Override
    public void play() {
        playAds();

        aMovie.play();

        playAds();
    }

    private void playAds() {
        System.out.println("得了痔疮怎么办？大铁棍子医院找捅主任！");
    }
}
```

此时的 Cinema 已经是代理对象了，它有一个`play()`方法，里面有它自己的一些小九九。

而影院开始运营时，就是这样了：

```java
package com.debugLife.proxyTest;

public class WandaCinema {
    public static void main(String[] args) {
        AMovie aMovie = new AMovie();
        Movie movie = new Cinema(aMovie);
        movie.play();
    }
}
```

结果不必多想，肯定如下：
```
得了痔疮怎么办？大铁棍子医院找捅主任！
正在播放电影 -《指环王》
得了痔疮怎么办？大铁棍子医院找捅主任！
```

静态代理的缺点是什么呢？

它通常**只能代理一个类**，这对于日益增长的需求来看，100个类，就几乎要写100个代理方法，这显然是不合实际的。

那么，能否不写代理类，而直接得到代理 Class 对象，然后根据它创建代理实例（反射）呢？为了解决这种问题，出现了动态代理。

## 动态代理是什么

Java 的动态代理机制可谓是『狸猫换太子』的典范了。为了**不暴露内部的接口**，采取了为其他对象**提供一个代理**以控制对某个对象的访问。代理类主要负责为委托类（真实对象）预处理消息、过滤消息、传递消息给委托类，代理类不现实具体服务，而是利用委托类来完成服务，并将执行结果封装处理。

JDK 提供了`java.lang.reflect.InvocationHandler`接口和`java.lang.reflect.Proxy`类，这两个类相互配合，入口是 Proxy，所以我们先聊它。

Proxy有个静态方法：`getProxyClass(ClassLoader, interfaces)`，只要你给它传入类加载器和一组接口，它就给你返回代理 Class 对象。

用通俗的话说，`getProxyClass()`这个方法，会从你传入的接口 Class 中，『拷贝』类结构信息到一个新的 Class 对象中，但新的 Class 对象带有构造器，是可以创建对象的。打个比方，一个大内太监（接口Class），空有一身武艺（类信息），但是无法传给后人。现在江湖上有个妙手神医（Proxy类），发明了克隆大法（getProxyClass），不仅能克隆太监的一身武艺，还保留了小DD（构造器）。

所以，一旦我们明确接口，完全可以通过接口的Class对象，创建一个代理Class，通过代理Class即可创建代理对象。

大体思路如下：

<div class="center-img">

![](/img/java-proxy-1588601750.png)

</div>

其中静态代理的机制如下：

<div class="center-img">

![](/img/java-proxy-1588602762.png)

</div>

动态代理的机制如下：

<div class="center-img">

![](/img/java-proxy-1588603762.png)

</div>

所以，`Proxy.getProxyClass()`这个方法的本质就是：**用 Class 来制造 Class**

但在实际开发中，一般都会使用`Proxy.newProxyInstance()`直接返回代理实例，中间的过程全部隐藏。

还是来看刚才电影院的例子。使用动态代理的话，应该是下面这种方式：

```java
Movie proxyMovie = (Movie)Proxy.newProxyInstance(
        Movie.class.getClassLoader(),
        new Class<?>[] { Movie.class } ,
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("得了痔疮怎么办？大铁棍子医院找捅主任！");
                Movie movie = new AMovie();
                method.invoke(movie, args);
                System.out.println("得了痔疮怎么办？大铁棍子医院找捅主任！");
                return null;
            }
        });

proxyMovie.play();
```

可以看出，我们已经不再需要静态代理类『Cinema』了，可以直接想怎么玩就怎么玩了，甚至我可以不调用`method.invoke()`方法，让`proxyMovie.play()`时，不再调用`AMovie.play()`方法，直接全程广告！

```java
Movie proxyMovie = (Movie)Proxy.newProxyInstance(
        Movie.class.getClassLoader(),
        new Class<?>[] { Movie.class } ,
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("得了痔疮怎么办？大铁棍子医院找捅主任！");
                // Movie movie = new AMovie();
                // method.invoke(movie, args);
                System.out.println("得了痔疮怎么办？大铁棍子医院找捅主任！");
                return null;
            }
        });

proxyMovie.play();
```

输入结果就如下：

```
得了痔疮怎么办？大铁棍子医院找捅主任！
得了痔疮怎么办？大铁棍子医院找捅主任！
```

真是干(sang)得(xin)漂(bing)亮(kuang)。有了动态代理机制，可以在原有接口和被代理类的基础上**做出任何拓展**，而**不会影响到原有接口和被代理类**。而且还省掉了一个代理类。

如果新添加了一部电影（被代理类）叫BMovie，那么我就得改`invoke()`方法了，这种方法忒差劲，我们可以改进一下：

```java
public static void main(String[] args) throws Exception {
    Movie aMovie = (Movie)getProxy(new AMovie());
    Movie bMovie = (Movie)getProxy(new BMovie());

    aMovie.play();
    bMovie.play();
}

private static Object getProxy(final Object target) throws Exception {
    return Proxy.newProxyInstance(target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    System.out.println("得了痔疮怎么办？大铁棍子医院找捅主任！");
                    method.invoke(target, args);
                    System.out.println("得了痔疮怎么办？大铁棍子医院找捅主任！");
                    return null;
                }
            });
}
```

就可以输出下面的结果：

```
得了痔疮怎么办？大铁棍子医院找捅主任！
正在播放电影 -《指环王》
得了痔疮怎么办？大铁棍子医院找捅主任！
得了痔疮怎么办？大铁棍子医院找捅主任！
正在播放电影 -《断背山》
得了痔疮怎么办？大铁棍子医院找捅主任！
```

## 为何要使用动态代理机制

1. 设计模式中有一个设计原则是**开闭原则**，是说对修改关闭，对扩展开放。我们在工作中有时会接手很多前人的代码，里面代码逻辑让人摸不着头脑，这时就很难去下手修改代码，那么这时我们就可以通过代理对类进行增强。
2. 实现无侵入式的代码扩展，也就是**方法的增强**。同时让你可以在不用修改源码的情况下，增强一些方法，甚至在方法的前后你可以做你任何想做的事情（甚至不去执行这个方法都可以）。
3. 采用动态代理的机制可以实现面向切面编程。

## 动态代理的实现原理

上面的例子中，我们是通过`Proxy.newInstance()`来创建的代理对象，我们来看看它是如何实现的：

```java
// java.lang.reflect.Proxy.java

public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
        throws IllegalArgumentException {
    Objects.requireNonNull(h);

    final Class<?>[] intfs = interfaces.clone();

    // Android 中会移除下面几行代码：SecurityManager 相关
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    // 查找或者生成指定的代理类
    Class<?> cl = getProxyClass0(loader, intfs);

    // 使用指定的调用者调用它的构造函数
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

上方代码中，`getProxyClass0()`方法如下：

```java
// java.lang.reflect.Proxy.java

// 生成代理类。在调用这里之前，必须要先调用 checkProxyAccess() 方法
private static Class<?> getProxyClass0(ClassLoader loader,
                                        Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // 如果指定接口的代理类已经存在于缓存 proxyClassCache 中，直接从缓存中取了返回即可；
    // 如果缓存中没有指定代理对象，则通过 ProxyClassFactory 来创建一个代理类。
    return proxyClassCache.get(loader, interfaces);
}
```

继续看 ProxyClassFactory 是怎样创建代理类的：

```java
// java.lang.reflect.Proxy.java

/**
 * 一个工厂方法，给定 ClassLoader 和 interface 的列表之后，可以创建、定义并返回代理类
 */
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    // 所有的代理类名前缀给它搞成 "$Proxy"
    private static final String proxyClassNamePrefix = "$Proxy";

    // 每个代理类名生成时再来个数字编号
    private static final AtomicLong nextUniqueNumber = new AtomicLong();

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            Class<?> interfaceClass = null;
            try {
                // 用Class.forName 查找一下给定的接口是否存在
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            // 然后对比 apply 中传入的 class 是否与 ClassLoader 中找到的 class 是否相同
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            // 验证这个 class 是不是接口
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            // 验证这个 class 的唯一性
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }

        String proxyPkg = null;     // 要生成的代理类的包名
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        // 记录下一个非 public 的 class 的包名，代理类也将会被定义在这个包里。
        // 同时验证所有的非 public 的 class 都在这个包里。
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        // 如果没有非 public 的 class，就使用 com.sun.proxy 包
        if (proxyPkg == null) {
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        // 搞个名字给这个代理类
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        // 生成这个代理类的字节码
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            // 调用 native 方法
            // private static native Class<?> defineClass0(ClassLoader loader, String name,
            //                                   byte[] b, int off, int len);
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            /*
                * A ClassFormatError here means that (barring bugs in the
                * proxy class generation code) there was some other
                * invalid aspect of the arguments supplied to the proxy
                * class creation (such as virtual machine limitations
                * exceeded).
                */
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

看来主要生成 class 的过程在`ProxyGenerator.generateProxyClass()`中，继续跟踪：

```java
// sun.misc.ProxyGenerator.java

// 此处是 Java 的一个系统变量，是否要保存生成的文件
private static final boolean saveGeneratedFiles = (Boolean)AccessController.doPrivileged(new GetBooleanAction("sun.misc.ProxyGenerator.saveGeneratedFiles"));

public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
    ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
    // 生成字节码文件
    final byte[] var4 = var3.generateClassFile();
    if (saveGeneratedFiles) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                try {
                    int var1 = var0.lastIndexOf(46);
                    Path var2;
                    if (var1 > 0) {
                        Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar));
                        Files.createDirectories(var3);
                        var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                    } else {
                        var2 = Paths.get(var0 + ".class");
                    }

                    Files.write(var2, var4, new OpenOption[0]);
                    return null;
                } catch (IOException var4x) {
                    throw new InternalError("I/O exception saving generated file: " + var4x);
                }
            }
        });
    }

    return var4;
}

// 生成 class 文件
private byte[] generateClassFile() {
    // 将 Object 中的一些方法添加到代理类中
    this.addProxyMethod(hashCodeMethod, Object.class);
    this.addProxyMethod(equalsMethod, Object.class);
    this.addProxyMethod(toStringMethod, Object.class);
    Class[] var1 = this.interfaces;
    int var2 = var1.length;

    int var3;
    Class var4;
    for(var3 = 0; var3 < var2; ++var3) {
        var4 = var1[var3];
        Method[] var5 = var4.getMethods();
        int var6 = var5.length;

        // 将 interface 中的方法添加到代理类中
        for(int var7 = 0; var7 < var6; ++var7) {
            Method var8 = var5[var7];
            this.addProxyMethod(var8, var4);
        }
    }

    Iterator var11 = this.proxyMethods.values().iterator();

    List var12;
    while(var11.hasNext()) {
        var12 = (List)var11.next();
        checkReturnTypes(var12);
    }

    Iterator var15;
    try {
        // 添加构造函数
        this.methods.add(this.generateConstructor());
        var11 = this.proxyMethods.values().iterator();

        while(var11.hasNext()) {
            var12 = (List)var11.next();
            var15 = var12.iterator();

            while(var15.hasNext()) {
                ProxyGenerator.ProxyMethod var16 = (ProxyGenerator.ProxyMethod)var15.next();
                this.fields.add(new ProxyGenerator.FieldInfo(var16.methodFieldName, "Ljava/lang/reflect/Method;", 10));
                this.methods.add(var16.generateMethod());
            }
        }

        // 添加静态方法
        this.methods.add(this.generateStaticInitializer());
    } catch (IOException var10) {
        throw new InternalError("unexpected I/O Exception", var10);
    }

    if (this.methods.size() > 65535) {
        throw new IllegalArgumentException("method limit exceeded");
    } else if (this.fields.size() > 65535) {
        throw new IllegalArgumentException("field limit exceeded");
    } else {
        this.cp.getClass(dotToSlash(this.className));
        this.cp.getClass("java/lang/reflect/Proxy");
        var1 = this.interfaces;
        var2 = var1.length;

        for(var3 = 0; var3 < var2; ++var3) {
            var4 = var1[var3];
            this.cp.getClass(dotToSlash(var4.getName()));
        }

        this.cp.setReadOnly();
        ByteArrayOutputStream var13 = new ByteArrayOutputStream();
        DataOutputStream var14 = new DataOutputStream(var13);

        try {
            var14.writeInt(-889275714);
            var14.writeShort(0);
            var14.writeShort(49);
            this.cp.write(var14);
            var14.writeShort(this.accessFlags);
            var14.writeShort(this.cp.getClass(dotToSlash(this.className)));
            var14.writeShort(this.cp.getClass("java/lang/reflect/Proxy"));
            var14.writeShort(this.interfaces.length);
            Class[] var17 = this.interfaces;
            int var18 = var17.length;

            for(int var19 = 0; var19 < var18; ++var19) {
                Class var22 = var17[var19];
                var14.writeShort(this.cp.getClass(dotToSlash(var22.getName())));
            }

            var14.writeShort(this.fields.size());
            var15 = this.fields.iterator();

            while(var15.hasNext()) {
                ProxyGenerator.FieldInfo var20 = (ProxyGenerator.FieldInfo)var15.next();
                var20.write(var14);
            }

            var14.writeShort(this.methods.size());
            var15 = this.methods.iterator();

            while(var15.hasNext()) {
                ProxyGenerator.MethodInfo var21 = (ProxyGenerator.MethodInfo)var15.next();
                var21.write(var14);
            }

            var14.writeShort(0);
            return var13.toByteArray();
        } catch (IOException var9) {
            throw new InternalError("unexpected I/O Exception", var9);
        }
    }
}

private void addProxyMethod(Method var1, Class<?> var2) {
    String var3 = var1.getName();
    Class[] var4 = var1.getParameterTypes();
    Class var5 = var1.getReturnType();
    Class[] var6 = var1.getExceptionTypes();
    String var7 = var3 + getParameterDescriptors(var4);
    Object var8 = (List)this.proxyMethods.get(var7);
    if (var8 != null) {
        Iterator var9 = ((List)var8).iterator();

        while(var9.hasNext()) {
            ProxyGenerator.ProxyMethod var10 = (ProxyGenerator.ProxyMethod)var9.next();
            if (var5 == var10.returnType) {
                ArrayList var11 = new ArrayList();
                collectCompatibleTypes(var6, var10.exceptionTypes, var11);
                collectCompatibleTypes(var10.exceptionTypes, var6, var11);
                var10.exceptionTypes = new Class[var11.size()];
                var10.exceptionTypes = (Class[])var11.toArray(var10.exceptionTypes);
                return;
            }
        }
    } else {
        var8 = new ArrayList(3);
        this.proxyMethods.put(var7, var8);
    }

    ((List)var8).add(new ProxyGenerator.ProxyMethod(var3, var4, var5, var6, var2));
}
```

字节码生成之后，调用 native 方法`defineClass0()`来解析字节码，生成了代理类。

native 层的具体实现[看这个链接](https://stackoverflow.com/questions/40217559/how-to-find-the-native-method-from-the-jvm-source-code)

用一个流程图来总结一下：

<div style="width: 100%; text-align:center;">

![](/img/29.png)

</div>

