---
title: ConcurrentHashMap
link: concurrent-hashmap
date: 2020-05-12
---

![cover](/img/covers/cover-java-2020-05-12.png)
ConcurrentHashMap 底层是基于**数组 + 链表**组成的，不过在 jdk1.7 和 1.8 中具体实现稍有不同。

<!-- more -->

## JDK 1.7 

### 数据结构

在 JDK1.7 中是这样的：

![](/img/concurrent-hashmap-1589261515.png)

如图所示，是由 Segment 数组、HashEntry 组成，和 HashMap 一样，仍然是**数组加链表**。

Segment 是 ConcurrentHashMap 的一个内部类，主要的组成如下：

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {    

    private static final long serialVersionUID = 2249069246763182397L;    

    // 和 HashMap 中的 HashEntry 作用一样，真正存放数据的地方   
    transient volatile HashEntry<K,V>[] table;    

    transient int count;

    // 用于 fail-fast，快速失败
    transient int modCount;

    // 大小
    transient int threshold;

    // 负载因子
    final float loadFactor;
}
```

HashEntry 跟 HashMap 中的 Entry 基本相同，但是不同点是，他**使用`volatile`去修饰了他的数据`value`还有下一个节点`next`**。

> **`volatile`的功能**
>
> 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。（实现可见性）
>
> 禁止进行指令重排序。（实现有序性）
>
> 注意：`volatile`只能保证对**单次读/写的原子性**。i++ 这种操作不能保证原子性。

原理上来说，ConcurrentHashMap 采用了分段锁技术，其中**Segment 继承于 ReentrantLock**。

它不会像 HashTable 那样不管是`put()`还是`get()`操作都需要做同步处理，理论上 ConcurrentHashMap 支持 CurrencyLevel (值为 Segment 数组数量）的线程并发。

每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。

就是说如果 Segment 当前大小是16他的并发度就是16，可以同时允许16个线程操作16个 Segment 而且还是线程安全的。

### `put()`

我们看一下 ConcurrentHashMap 的`put()`方法：

```java
// jdk1.7 ConcurrentHashMap.java
public V put(K key, V value) {
    Segment<K,V> s;

    if (value == null)
        throw new NullPointerException(); //这就是为啥他不可以put null值的原因
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    // if 中拿到的 s 不是 volatile 的，将会在 ensureSegment 中重新进行检查
    if ((s = (Segment<K,V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

他先定位到当前 key 属于哪个 Segment，然后再对该 Segment 进行`put()`操作。

然后我们看看 Segment 的`put()`源代码，你就知道他是怎么做到线程安全的了，关键句子我注释了。

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 将当前 Segment 中的 table 通过 key 的 hashcode 定位到 HashEntry
    // 因为 Segment 继承自 ReentrantLock，所以此处直接调用了父类的 tryLock() 方法，持有当前锁
    HashEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 计算 index
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            // 如果拿到的当前 index 的值不为空，表示该 index 上有数据，需要进行覆盖
            if (e != null) {
                K k;
                // 遍历该 HashEntry，
                // 如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                // 不为空则需要新建一个 HashEntry 并加入到 Segment 中，同时会先判断是否需要扩容
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 判断是否需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        //释放锁
        unlock();
    }
    return oldValue;
}

private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1;
    while (!tryLock()) {  // 自旋获取锁
        HashEntry<K,V> f;
        if (retries < 0) {
            if (e == null) {
                if (node == null) // 尝试创建新的 HashEntry
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) { 
            // 直接使用阻塞锁的方式，暴力而有效
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                    (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

首先第一步的时候会尝试**获取锁**，如果获取失败肯定就有其他线程存在竞争，则利用`scanAndLockForPut()`**自旋获取锁**。如果重试的次数达到了`MAX_SCAN_RETRIES`（CPU数量大于1时，值为64，否则为1）则改为阻塞锁获取，保证能获取成功。

> 自旋就是执行一段无意义的循环。

### `get()`

`get()`逻辑比较简单，定位 Segment → 定位 HashEntry → 定位链表节点：

```java
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key.hashCode());
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        // 定位 Segment
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            // 定位 HashEntry
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                // 定位链表上的节点
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
```

由于 HashEntry 中的`value`属性是用`volatile`关键词修饰的，保证了内存可见性，所以每次获取时都是最新值。所以 ConcurrentHashMap 的`get()`方法是非常高效的，因为整个过程都不需要加锁。

但是，虽然 JDK1.7 中可以支持每个 Segment 并发访问，但是还是存在一些问题：**我们去查询的时候，还得遍历链表**，会导致效率很低，这个跟 JDK1.7 的 HashMap 是存在的一样问题，所以他在 JDK1.8 完全优化了。

### 扩容机制

然后讲讲在 JDK1.7 中的扩容机制。

我们向上翻一下，在`put()`方法中，当判断需要扩容时，调用的是`rehash()`方法，我们来看看它是如何做的：

```java
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1; // 与 HashMap 相同，采用2次幂扩容
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable = (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   // 链表中只有一个节点
                newTable[idx] = e;
            else { // 将原来的顺序原封不动地移到新的 table 中
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next; last != null; last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
               
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // 将新的节点加入
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

可以看出，与 HashMap 扩容机制类似，采取了**2次幂的方式**来决定新数组的大小。但是，Segment 没有扩容机制，在初始化时，就指定好了 Segment 数组的大小，也既该 ConcurrentHashMap 的并发数。

### `size()`

既然要考虑线程安全，在获取 ConcurrentHashMap 的大小时，也要特殊处理一下。这里采用的是**多次对比机制**，我们来看看它的代码：

```java
public int size() {
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            if (retries++ == RETRIES_BEFORE_LOCK) {  // 重试次数达到2
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // 强制让所有 Segment 获得阻塞锁
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            // 如果此次获取的大小与上次一致，则认为这就是当前大小
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();  // 比对完成后让所有 Segment 解锁
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

可以看到，至少要进行两次遍历，如果**相邻的两次遍历计算出的 key-value 的数量是相等**的，那就认为这个数量是当前的正确数量。如果一直对比不上，这说明数据在不断地更改，此时要让**所有 Segment 获取阻塞锁**，然后再进行对比，对比成功后，解锁，返回数量。

## JDK 1.8

### 数据结构

在 JDK1.8 中，ConcurrentHashMap 抛弃了原有的 Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性。

虽然它的源码中还是会包含 Segment 类，但是也只用在 Serializable 的**序列化和反序列化**中，而且多亏了 Serializable 的`serialVersionUID`特性，只要版本号保持一致，Segment 可以在 JDK1.7 和 JDK1.8 甚至 JDKX.X 之间自由转换，JVM 会认为是同一个类。

跟 HashMap 的1.7 → 1.8很像，ConcurrentHashMap 也把之前 JDK1.7 中的 HashEntry 改成了 Node，但是作用不变，把`value`和`next`采用了`volatile`去修饰，保证了可见性，并且也引入了**红黑树**，在链表大于一定值的时候会转换（默认是8）。

它的数据结构如下图所示：

![](/img/concurrent-hashmap-1589262515.png)

对于每一个链表，ConcurrentHashMap 将其称之 **『桶』（bin）**。

### `put()`

先看一下它的 put 方法：

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 计算 hashCode
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {  // 死循环确保可以插入数据
        Node<K,V> f; 
        int n, i, fh;
        // 如果 Node 数组为 null，则需要初始化，只有在第一次进行 put 时才会进行这个操作
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 利用 (n - 1) & hash 计算 index，得出要插入的 Node 数组的 index
            // 如果节点为空，则利用 CAS 机制写入数据
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null))) // 向空桶中插入数据
                break;  // 向空的桶中添加数据时，不需要加锁
        }
        // 要扩容了
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            // 尝试使用 synchronized 写入数据
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                break;
                            }
                        }
                    }
                    // 已经被转换为红黑树节点了，需要用 putTreeVal 来进行插入
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                // 桶中节点的数量大于 TREEIFY_THRESHOLD，值为8，转换红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

ConcurrentHashMap 在进行`put()`操作的还是比较复杂的，大致可以分为以下步骤：

1. 如果是第一次插入数据，则需要调用`initTable()`初始化 Node 数组；
2. 利用`(n - 1) & hash`得出要插入的数组 index；
3. 如果数组的 index 位置为 null，则直接利用 CAS 机制插入，不需要加锁；
4. 如果数组的 index 位置不为 null，则要使用 synchronized 方式插入到桶（链表）中；
5. 如果插入完成后，桶的节点数量大于等于8了，就要转换为红黑树（红黑桶🛢）；
6. 如果节点的 hash 值为 -1，则需要扩容。

> **CAS（Compare And Swap）**
> 
> CAS 是乐观锁的一种实现方式，是一种轻量级锁，JUC 中很多工具类的实现就是基于 CAS 的。
> CAS 的中心思想是线程**在读取数据时不进行加锁**，在准备写回数据时，**比较原值是否修改**，若未被其他线程修改则写回，若已被修改，则> 重新执行读取流程。
> 这是一种**乐观策略**，认为并发操作并不总会发生。
> 
> 但是经典的 ABA 问题，它就无法判断了。就是数据从 A 变为 B 又变成 A，看似没修改过，但实际上是修改过的。
> 想要解决 ABA 问题，要么使用版本号，要么使用时间戳标识，都可以。

### `get()`

再来看它的`get()`：

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    // 根据计算出来的 hashcode 寻址，如果就在链表上那么直接返回值。
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 如果是红黑树那就按照树的方式获取值。
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 都不满足那就按照链表的方式遍历获取值。
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

比较简单，也不需要加锁，因为节点的值都是 volatile 的。

### 扩容机制

我们直接来看它的代码：

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

ConcurrentHashMap 的 JDK1.8 与 JDK1.7 版本的并发实现相比，最大的区别在于 JDK1.8 的**锁的粒度更细**。

理想情况下 table 数组元素的大小就是其支持并发的最大个数，在 JDK1.7 里面最大并发个数就是 Segment 的个数，默认值是16，可以通过构造函数改变一经创建不可更改，这个值就是并发的粒度，每一个 Segment 下面管理一个 table 数组，加锁的时候其实锁住的是整个 Segment，这样设计的好处在于数组的扩容是不会影响其他的 Segment，简化了并发设计，不足之处在于并发的粒度稍粗，所以在 JDK1.8 里面，去掉了分段锁，将锁的级别控制在了更细粒度的 table 元素级别，也就是说只需要**锁住这个链表的 head 节点，并不会影响其他的 table 元素的读写**，好处在于**并发的粒度更细，影响更小**，从而并发效率更好，但不足之处在于并发扩容的时候，由于操作的 table 都是同一个，不像 JDK1.7 中分段控制，所以这里**需要等扩容完之后，所有的读写操作才能进行**，所以扩容的效率就成为了整个并发的一个瓶颈点。好在 Doug lea 大神对扩容做了优化，本来在一个线程扩容的时候，如果影响了其他线程的数据，那么其他的线程的读写操作都**应该阻塞**，但是他们闲着也是闲着，不如来一起参与扩容任务，这样人多力量大，办完事你们该干啥干啥，别浪费时间，于是在 JDK1.8 的源码里面就引入了一个 ForwardingNode 类，在一个线程发起扩容的时候，就会改变 `sizeCtl` 这个值，其含义如下：

> **sizeCtl** ：默认为0，用来控制table的初始化和扩容操作，具体应用在后续会体现出来。
> 值为 -1 时代表 table 正在初始化
> 值为 -(1 + n) 时，表示正在有 n 个线程正在扩容
> 如果 table 未初始化，值表示 table 需要初始化的大小
> 如果 table 初始化完成，表示 table 的容量，值是 `table 的大小 x 负载因子`

扩容时候会判断这个值，如果超过阈值就要扩容，首先根据运算得到需要遍历的次数i，然后利用`tabAt()`方法获得`i`位置的元素`f`，初始化一个ForwardingNode 实例`fwd`，如果`f == null`，则在table中的`i`位置放入`fwd`，否则采用头插法的方式把当前旧 table 数组的指定任务范围的数据给迁移到新的数组中，然后 给旧 table 原位置赋值`fwd`。直到遍历过所有的节点以后就完成了复制工作，把 table 指向 nextTable，并更新`sizeCtl`为新数组大小的0.75倍 ，扩容完成。在此期间如果其他线程的有读写操作都会判断`head`节点是否为 ForwardingNode 节点 ，如果是就帮助扩容。

我们来看看 ForwadingNode 的代码：

```java

static final int MOVED     = -1; // hash for forwarding nodes

/**
 * A node inserted at head of bins during transfer operations.
 */
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // 使用循环，避免多次碰到 ForwardingNode 导致递归过深
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))  // 第一个节点就是要找的节点，直接返回
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);  // 特殊节点，调用其find方法进行查找
                }
                if ((e = e.next) == null)  // 普通节点直接循环遍历链表
                    return null;
            }
        }
    }
}
```

ForwardingNode 是一种临时节点，在**扩容进行中**才会出现，hash 值固定为 -1，并且它不存储实际的数据数据。如果旧数组的一个 hash 桶中全部的节点都迁移到新数组中，旧数组就在这个 hash 桶中放置一个 ForwardingNode。读操作或者迭代读时碰到 ForwardingNode 时，将操作转发到扩容后的新的 table 数组上去执行，写操作碰见它时，则尝试帮助扩容。

### `size()`

与 jdk1.7 中的计数方式不同，jdk1.8 中的 ConcurrentHashMap 引入了一个 CounterCell 类，用于记录每一个桶中的节点数量，在`put()`值时，最后调用的`addCount()`中，就会对这个值做出修改，所以在`size()`时，直接使用循环方式累加所有的 CounterCell 的值即可得出当前的 key-value 的数量。

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

> 红黑树（Red Black Tree） 是一种自平衡二叉查找树。它是一种特化的平衡二叉树，都是在进行插入和删除操作时通过特定操作保持二叉查找> 树的平衡，从而获得较高的查找性能。它的查找时间复杂度是O(log n)。
> 
> 红黑树是每个节点都带有颜色属性的二叉查找树，颜色或红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的> 额外要求:
> 
> 1. 节点是红色或黑色
> 2. 根节点是黑色 
> 3. 所有叶子都是黑色。（叶子是NUIL节点）
> 4. 每个红色节点的两个子节点都是黑色。（从每个叶子到根的所有路径上不能有两个连续的红色节点）
> 5. 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点
