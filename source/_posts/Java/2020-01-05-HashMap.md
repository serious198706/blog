---
title: HashMap 解析
toc: true
date: 2020-01-05
link: hashmap
cover: http://blog.cigis-cloud.com/hashmap-1593753043.png
category: 
    - Development
tag: 
    - Java
    - HashMap
---

在 Java 和 Android 的开发中，我相信没有人没用过 HashMap 了吧。这几乎是 Java 中最重要的类之一，作为 Key-Value 存储类型的典范，我们在学会使用的同时，也必须要明白 HashMap 内部的实现原理，同时也必须知道它的一些衍生类，以及它们的用法。

<!-- more -->

## 概述

话不多说，先来看一下 HashMap 在官方文档里的定义：

> HashMap 是 Map 接口基于哈希表的实现。这个实现提供了所有可用的对 map 的操作，而且还允许插入 null 值和 null 键。
> 
> HashMap 与 Hashtable 类差不多，但 HashMap 是异步的，并且允许 null 值/键。这个类无法保证 map 内元素的顺序；更特别的是，它也无法保证能一直按照某个顺序排列。

Map 接口有三个比较重要的具体实现类：**HashMap**、**WeakHashMap** 和 **TreeMap**。HashMap 还有一个重要的子类 **LinkedHashMap**，它们几个都是非线程安全的类。

我们可以通过一张图简单了解一下它们之间的关系：

<div align="center">

![](/img/56.png)

</div>

可以看到，刚才官方文档中提到的 Hashtable，也是 Map 的一个实现类，它的官方描述中有这么一段：

> 不像 collection 的实现那样，Hashtable 是同步的。如果不要求线程安全，那你可以使用 HashMap 来替代 Hashtable；如果你对线程安全要求非常高，那推荐你使用 java.util.concurrent.ConcurrentHashMap 来替代 Hashtable。

后面我们会讨论 [Hashtable](#hashtable) 和 [ConcurrentHashMap](#concurrenthashmap)。

## HashMap 的数据结构

HashMap 的数据结构是我们非常常用的数据结构，就是由**数组和链表组合构成**的数据结构。如下图所示：

<div align="center">

![](/img/57.png)

</div>

数组的每个位置都存储了一个key-value这样的实例，在 Java 7 中叫Entry，在 Java 8 中叫Node。这个数组每个元素原本都是 null，在 put 插入数据的时候会根据 key 值的 hash 去计算一个 index 值。

比如我调用`put("debugLife", 42)`，这个时候会通过哈希函数计算出插入的位置，比如说 index 为2吧，那结果就如下图所示：

<div align="center">

![](/img/58.png)

</div>

那么，只用一个数组就好了呀，为啥还要用到链表？

原因很简单，数组长度有限，在有限的长度里使用哈希，会有很大概率出现不同的 key 的计算出的 index 结果相同。比如我调用`put("lifeDebug, 24)`计算出的 index 也为2，就会出现下面的情况：

<div align="center">

![](/img/59.png)

</div>

上面 index 为2的两对key-value形成了一个链表。这点我们可以通过查看 Node 的代码来佐证：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    ...
}
```

那么问题来了，如果两次计算的 index 相同，那新的 Node 在插入的时候，是怎么插入的？

在 Java 8 之前是**头插法**，也即新来的 Node 会取代原来的 Node，原有值就被推到链表中去，像上面的图一样。

但是从 Java 8 之后，就采用**尾部插入**了。为什么呢？这个要先从 HashMap 的**扩容机制**说起。

### 扩容机制 

刚才上面提到了，数组的长度是有限的，当数据多次插入，达到一定的数量之后，就会进行**扩容（resize）**。扩容的因素有两个：

- Capacity：HashMap 当前的容量
- LoadFactor：负载因子，默认值是`0.75f`

```java
/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

负载因子可以理解为，如果当前容量是100，存75个还没事，但存第76个的时候，就要进行扩容。我们来看看代码：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    ...
    if (++size > threshold)
        resize();
}

// 扩容的阈值（容量 * 负载因子)
int threshold;

// 默认的容量，必须是2的幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 如果构造函数未指定的话，负载因子就是0.75f
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

而扩容也并不是简单地扩大点容量就完事了，我们来简单看一下`resize()`的代码：

```java
// 初始化或者扩容为原来的2倍。
// 如果内容为 null，分配默认容量大小的区域。
// 如果不是，因为我们使用了『2次幂扩展』，所有的元素要么待在原来的 index，要么就在新 table 里偏移『2次幂』个位置。
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 数据量已经到最大值 1 << 30了，就将阈值设置为 Integer.MAX_VALUE 吧，也不用扩容了，还扩啥扩
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新的容量是旧容量 * 2，新的阈值也是旧的阈值 * 2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    // 数据量为0，但是旧阈值大于0，新的容量就是旧的阈值
    else if (oldThr > 0)
        newCap = oldThr;
    // 数据量为0，旧阈值等于0，新的容量就是默认容量，新的阈值就是默认容量 * 默认负载因子
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 经过上面的处理，新的阈值还是0的话，就进行对比：
    // 如果：新的容量小于最大容量 且 用户设置的阈值小于最大容量的话，新的阈值就是用户设置的阈值
    // 否则：新的阈值就是 Integer.MAX_VALUE
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                (int)ft : Integer.MAX_VALUE);
    }
    // 设置新的阈值
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 先申请一个长度为之前2倍的新数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 清空原有数组
    table = newTab;
    if (oldTab != null) {
        // 开始循环处理旧数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 如果旧数组[j]元素不为 null
            if ((e = oldTab[j]) != null) {
                // 把旧数组[j]置为 null
                oldTab[j] = null;
                // 如果旧数组[j]的next无Node，也即该index上只有它自己
                if (e.next == null)
                    // e.hash & (newCap - 1) 是在计算该元素的 index
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果旧数组[j]是红黑树节点，就调用它的split方法，将其子节点复制到新数组中
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 保留之前数据的顺序
                else {
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        // 获取该节点的 next
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

可以看得出来，扩容基本上分为两步：

1. 将数组的容量扩展为原来的2倍
2. 重新计算所有 Node 的 hash 值，也即 rehash，并插入到新数组中

那么，为什么要 rehash？直接扩容不就完事了？

这样做是不行的，因为数组的容量扩大后，**计算 index 的规则也发生了变化**。

index 的计算规则是`index = key.hash & (capacity - 1)`，之前 capacity 16，变成了 32 的话，值明显就不一样了哇。

好，回到刚才提到的头插尾插。

我们来举个例子，如果现在我们采用头插法，HashMap 的容量是2，负载因子是0.75，那么在插入第二个元素的时候，就要准备扩容了。

现在我们往这个容量是2的数组里用**不同线程**插入 C、B、A 三个 Entry（头插法是 Java8 之前的哦，要叫 Entry 的哦），那么，在数据插入之后，扩容之前，数据结构可能是下面这样：

<div align="center">

![](/img/60.png)

</div>

我们可以看到链表的指向是 A->B->C，因为使用的是头插法，同一 index 时，**新元素总是会被放在链表头的位置**。

resize()之后，旧数组中同一个 index 上的链表，其 Entry 可能会被分配到其他的 index 上，也即出现下面的情况：

<div align="center">

![](/img/61.png)

</div>

那么待3个线程都调整完毕后，就有可能出现下面的情况：

<div align="center">

![](/img/62.png)

</div>

出现了！无限循环！ 因为 HashMap 是线程不安全的啊！

<div align="center">

![](/img/loop.jpg)

</div>

而尾插法就不会出现这样的问题。使用头插时会改变链表上的顺序，但是如果使用尾插，在扩容时会保持链表元素原本的顺序，就不会出现链表成环的问题了。也就是说原本是A->B，在扩容后那个链表还是A->B：

<div align="center">

![](/img/63.png)

</div>

总结起来就是：

1. Java 7 在多线程操作 HashMap 时可能引起死循环，原因是扩容转移后前后链表顺序倒置，在转移过程中修改了原来链表中节点的引用关系。
2. Java 8 在同样的前提下并不会引起死循环，原因是扩容转移后前后链表顺序不变，保持之前节点的引用关系。

但这并不意味着在 Java 8 中可以把 HashMap 用于多线程。从源码的`put/get`方法中，没有任何同步机制，也就是说，无法保证你某个线程`put`进去的值，再`get`的时候还是原值，所以，还是无法保证线程安全。如果要线程安全，请使用 [Hashtable](#hashtable) 或者 [ConcurrentHashMap](#concurrenthashmap)。 

## 均匀分布

从上面的代码里，我们看到了，HashMap 默认初始化长度是`1 << 4`，也就是16。那么为什么是16呢？我们来看一下。

HashMap 使用了位运算来得到『16』这个数字，我们都知道，位运算比算数运算的效率要高很多，之所以选择16这个数字，是为了服务将 key 映射到index 的这个算法（前面提到过）：

```java
index = key.hash & (capacity - 1)
```

比如我们有个 key 为`wtf`，它的 hashCode 值是`118057`，转换为二进制是`11100110100101001`，使用capacity的默认值16，capacity - 1 是15，转换为二进制是`1111`，那么套入 index 计算公式的话，就是：

```java
index = 0b11100110100101001 & 0b1111
index = 9
```

至于为什么要使用16，而不是14、19、25，是因为使用2次幂数字的时候，减去1，转换为二进制全为1，这种情况下，`&`出来的结果等同于 hashCode 后几位的值。这种位运算的速度可比取余的方法性能高多了。同时，只要 hashcode 本身分布均匀，index 的分布就是均匀的。

那么，为什么8和32也是2次幂，偏偏选择了16？其实8和32也都可以，可能只是作者认为这个数字比较符合常用吧。

另外，还有个小知识点：Hashmap中的链表大小超过8个时会**自动转化为红黑树**，当删除小于6时**重新变为链表**。这是因为根据**[泊松分布](https://zh.wikipedia.org/zh-hans/%E6%B3%8A%E6%9D%BE%E5%88%86%E4%BD%88)**，在负载因子默认为0.75的时候，单个 hash 槽内元素个数为8的**概率小于百万分之一**，所以将7作为一个分水岭，等于7的时候不转换，大于等于8的时候才进行转换，小于等于6的时候就化为链表。

## equals 方法和 hashCode 方法

我们有时为了对比两个对象是否相等，会重写`equals()`方法，那么，我们是不是也需要重写`hashCode()`方法呢？

答案是，如果你的对象要放在 HashMap 中的话，需要。

在 Java 中，所有的对象都继承自 Object 类，Obejct 类中有两个方法`equals()`和`hashCode()`，用于比较两个对象是否相等。当我们**未重写`equals()`**时，比较的是两个对象的内存地址，如果是`new`了两个对象的话，那肯定不同。

刚才我们提到过，HashMap 是通过 hashCode 来计算 index 的，当我们去`get`的时候，很可能会拿到一个在链表上的值，那么如何确定是链表上的哪个值呢？是了，通过`equals()`，我们来看看它的代码部分：

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                // 就是这里啦，对比 hash 和 equals
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

所以，如果我们对`equals()`方法进行了重写，那最好也对`hashCode()`方法进行重写，以保证相同的对象返回相同的 hash 值，不同的对象返回不同的 hash 值。

## Hashtable

这个 Map 的派生类是为了在`put/get`达到一定的线程安全，才出现的。它的解决方案比较粗暴，直接在`put/get`方法上加 synchronized 关键字，锁住方法。由此可见，并发度会比较低：

```java
public synchronized V get(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

所以，这个类必定会渐渐地退出舞台，在茫茫历史长河中被慢慢淹没。

## ConcurrentHashMap

这个必须要单独来一篇[ConcurrentHashMap的文章](/concurrent-hashmap/)了。因为 ConcurrentHashMap 内部的原理比 HashMap 要复杂得多，如果赋在这下面，这篇就太长了。
