---
title: 关于Binder的一切
description: 介绍关于Binder的一切
link: binder
cover: http://blog.cigis-cloud.com/android-1593080864.jpg
toc: true
category: 
    - Android
    - Development
tag: 
    - Android
    - Binder
    - Android Framework
date: 2020-03-18
---

# 关于Binder的一切

毫不夸张地说，Binder 是 Android 系统中最重要的特性之一。

正如其名『粘合剂』所喻，它是系统间各个组件的桥梁，Android 系统的开放式设计也很大程度上得益于这种及其方便的跨进程通信机制。

理解 Binder 对于理解整个 Android 系统有着非常重要的作用，Android 系统的四大组件，ActivityManagerServer，PackageManagerService 等系统服务无一不与 Binder 挂钩；如果对 Binder 不甚了解，那么就很难了解这些系统机制，从而仅仅浮游与表面，不懂 Binder 你都不好意思说自己会 Android 开发；要深入 Android，Binder 是必须迈出的一步。

这篇文章将由浅入深搞懂 Binder ：

<!-- more -->

1. [多进程通信的原理是什么？](#多进程通信的原理)
2. [何为 Binder？](#binder介绍)
3. [为何用 Binder 而不用其他的方法？](#binder对比其他进程间通信的方法的优势)
4. [Binder 的通信流程是什么样的？](#binder通信机制)

另外，推荐两篇文章。

[Binder设计与实现](https://blog.csdn.net/universus/article/details/6211589)

[Android进程间通信（IPC）机制Binder简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/6618363)

先看这里，再看这两篇文章，应该就差不多了。

## 多进程通信的原理

因为 Android 基于 Linux ，所以要谈多进行通信，就必须要谈到 Linux 下的多进程。

### 进程隔离
> 进程隔离是为**保护操作系统中进程互不干扰**而设计的一组不同硬件和软件的技术。这个技术是为了避免进程A写入进程B的情况发生。 进程的隔离实现，使用了**虚拟地址空间**。进程A的虚拟地址和进程B的虚拟地址不同，这样就防止进程A将数据信息写入进程B。

以上来自维基百科；操作系统的不同进程之间，数据不共享；对于每个进程来说，它都天真地以为自己独享了整个系统，完全不知道其他进程的存在。因此一个进程需要与另外一个进程通信，需要某种系统机制才能完成。

### 用户空间/内核空间
详细解释可以参考[Kernel Space Definition](http://www.linfo.org/kernel_space.html)；简单理解如下：

Linux Kernel 是操作系统的核心，独立于普通的应用程序，**可以访问受保护的内存空间**，也有访问底层硬件设备的所有权限。

对于 Kernel 这么一个高安全级别的东西，显然是不容许其它的应用程序随便调用或访问的，所以需要对 Kernel 提供一定的保护机制，这个保护机制用来告诉那些应用程序，你只可以访问某些许可的资源，不许可的资源是拒绝被访问的，于是就把Kernel和上层的应用程序抽象地隔离开，分别称之为**Kernel Space（内核空间）** 和 **User Space（用户空间）**。

### 系统调用/内核态/用户态
虽然从逻辑上抽离出用户空间和内核空间；但是不可避免的的是，总有那么一些用户空间需要访问内核的资源；比如应用程序访问文件，网络是很常见的事情，怎么办呢？

> Kernel space can be accessed by user processes only through the use of system calls.

用户空间访问内核空间的唯一方式就是**系统调用**；通过这个统一入口接口，所有的资源访问都是在内核的控制下执行，以免导致对用户程序对系统资源的越权访问，从而保障了系统的安全和稳定。用户软件良莠不齐，要是它们乱搞把系统玩坏了怎么办？因此对于某些特权操作必须交给安全可靠的内核来执行。

当一个任务（进程）执行系统调用而陷入内核代码中执行时，我们就称进程处于**内核运行态**（或简称为**内核态**）此时处理器处于特权级最高的（0级）内核代码中执行。当进程在执行用户自己的代码时，则称其处于**用户运行态**（用户态）。即此时处理器在特权级最低的（3级）用户代码中运行。处理器在特权等级高的时候才能执行那些特权CPU指令。

### 内核模块/驱动
通过系统调用，用户空间可以访问内核空间，那么如果一个用户空间想与另外一个用户空间进行通信怎么办呢？很自然想到的是让操作系统内核添加支持；传统的 Linux 通信机制，比如 Socket ，管道等都是内核支持的；但是 Binder 并不是 Linux 内核的一部分，它是怎么做到访问内核空间的呢？Linux 的动态可加载内核模块（Loadable Kernel Module，LKM）机制解决了这个问题。

**模块**是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间运行。这样，Android系统可以通过添加一个内核模块运行在内核空间，用户进程之间的通过这个模块作为桥梁，就可以完成通信了。

在 Android 系统中，这个运行在内核空间的，负责各个用户进程通过 Binder 通信的内核模块叫做 **Binder驱动**;

驱动程序一般指的是设备驱动程序（Device Driver），是一种可以使计算机和设备通信的特殊程序。相当于硬件的接口，操作系统只有通过这个接口，才能控制硬件设备的工作。

驱动就是操作硬件的接口，为了支持 Binder 通信过程，Binder 使用了一种『硬件』，因此这个模块被称之为 Binder 驱动。

## Binder介绍
Binder 的设计采用了面向对象的思想。

Binder 的通信模型有 4 个角色：**Client**、**Server**、**Binder Driver（Binder 驱动）**、**ServiceManager**。

想象一个情景：现在我要给我的一个同学寄一封信，信上要标明收信地址。那我怎么拿到这个地址呢？去翻一下我的毕业相册。而这个记录着同学们通信地址的毕业相册，就相当于一个通讯录。它在 Binder 的通信模型中扮演的是 ServiceManager 的角色。

当有了收信地址之后，找到邮局寄出去就好了。过几天同学就收到了明信片。那么这个邮局在 Binder 通信模型中扮演的是 Binder Driver 的角色，而作为寄信人的我就是 Client，收信人同学就是 Server。

上面的例子可以变成下面这个模型图：

<div class="center-img">

![Binder通信模型图](/img/7.png)

</div>

在 Binder 通信模型的四个角色里面，他们的代表都是『Binder』，这样，对于 Binder 通信的使用者而言，Server 里面的 Binder 和 Client 里面的 Binder 没有什么不同，一个 Binder 对象就代表了所有，它不用关心实现的细节，甚至不用关心驱动以及 ServiceManager 的存在：这就是抽象。

- 通常意义下， Binder 指的是一种通信机制；我们说 AIDL 使用 Binder 进行通信，指的就是 Binder 这种进程间通信机制。
- 对于 Server 进程来说， Binder 指的是 Binder 本地对象。
- 对于 Client 来说， Binder 指的是 Binder 代理对象，它只是 Binder 本地对象的一个远程代理；对这个 Binder 代理对象的操作，会通过驱动最终转发到 Binder 本地对象上去完成；对于一个拥有 Binder 对象的使用者而言，它无须关心这是一个 Binder 代理对象还是 Binder 本地对象；对于代理对象的操作和对本地对象的操作对它来说没有区别。
- 对于传输过程而言， Binder 是可以进行跨进程传递的对象； Binder 驱动会对具有跨进程传递能力的对象做特殊处理：自动完成代理对象和本地对象的转换。

> 面向对象思想的引入将进程间通信转化为通过对某个 Binder 对象的引用调用该对象的方法，而其独特之处在于 Binder 对象是一个可以跨进程引用的对象，它的实体（本地对象）位于一个进程中，而它的引用（代理对象）却遍布于系统的各个进程之中。最诱人的是，这个引用和 java 里引用一样既可以是**强类型**，也可以是**弱类型**，而且可以从一个进程传给其它进程，让大家都能访问同一 Server ，就象将一个对象或引用赋值给另一个引用一样。 Binder 模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。形形色色的 Binder 对象以及星罗棋布的引用仿佛粘接各个应用程序的胶水，这也是 Binder 在英文里的原意。

## Binder对比其他进程间通信的方法的优势
让我们带着问题去看这一段。

> Linux下有那么多进程间通信方式，为什么 Android 偏偏选择了用 Binder？

我们先来看看Linux下都有哪些进程间通信方式。

### Linux

#### 1. Pipeline（管道）

管道作为进程间通信方式，由来已久。也基本上是Linux中使用得最多的进程间通信方式。

```bash
$history | grep 'gradlew'
```

上面的代码就完成了一次最简单的管道通信方式。`history`进程将自己的输出结果放入管道，`grep`进程从中取出结果，并进行筛选，得出我们最后想要的结果。

这个是**匿名管道**最简单的例子，管道又分为[**有名管道**](https://en.wikipedia.org/wiki/Named_pipe)和[**匿名管道**](https://en.wikipedia.org/wiki/Anonymous_pipe)，就不在这里详细展开了。

管道的通信模型如下图：

<div class="center-img">

![管理通信模型](/img/8.png)

</div>

管道可用于进程间的通信，管道是由内核管理的一个缓存区， 相当于我们放入内存中的一个**纸条**。管道的一端连接一个进程的输出，这个进程会向管道中放入信息。另一端连接一个进程的输入，这个进程取出被放入管道的信息。

这个缓存区不需要很大，在 Linux 中，默认的管道空间大小是 [64KB](https://en.wikipedia.org/w/index.php?title=Pipeline_(Unix)#Implementation)。当管道中没有信息的话，从管道中读取的进程会等待，直到另一端的进程放入信息。当管道被放满信息的时候，尝试放入信息的进程会等待，直到另一端的进程取出信息。当两个进程都终结的时候，管道也自动消失。

**缺点**: 在创建时分配一个管道时，缓存区大小比较有限；并不适合 Android 大量的进程间通信。

#### 2. Message Queue（消息队列）

消息队列提供了一种从一个进程向另一个进程发送一个数据块的方法。每个数据块都被认为含有一个类型，接收进程可以独立地接收含有不同类型的数据结构。我们可以通过发送消息来避免命名管道的同步和阻塞问题。但是消息队列与命名管道一样，每个数据块都有一个最大长度的限制。

<span style="color: green">#TODO 日后详细展开</span>

**缺点**: 信息复制两次，额外的 CPU 消耗；不合适频繁或信息量大的通信。

#### 3. Shared Memory（共享内存）

顾名思义，共享内存就是允许两个不相关的进程访问同一个逻辑内存。共享内存是在两个正在运行的进程之间共享和传递数据的一种非常有效的方式。不同进程之间共享的内存通常安排为同一段物理内存。进程可以将同一段共享内存连接到它们自己的地址空间中，所有进程都可以访问共享内存中的地址。

<span style="color: green">#TODO 日后详细展开</span>

**优点**：**无须复制**，共享缓冲区直接付附加到进程虚拟地址空间，速度快；
**缺点**：进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决。同时，安全问题也比较突出，如果 Android 采用共享内存无异于将每个 App 放在一个内存中，这样是非常不安全的。

#### 4. Socket（套接字）

通常作为更通用的接口，传输效率低，主要用于不同机器或跨网络的通信。

<span style="color: green">#TODO 日后详细展开</span>

#### 5. Semaphore（信号量）

常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段，并不太适合 Android 的进程间通信。

#### 6. Signal（信号）

适用于**进程中断控制**，比如非法内存访问，杀死某个进程等；

**缺点**：不适用于信息交换。

### Binder

- 从性能的角度

    数据拷贝次数：Binder数据拷贝**只需要一次**，而管道、消息队列、Socket 都需要2次，但共享内存方式一次内存拷贝都不需要；从性能角度看，Binder 性能仅次于共享内存。

- 从稳定性的角度

    Binder 是**基于 C/S 架构**的。简单解释下 C/S 架构，是指客户端（Client）和服务端（Server）组成的架构，Client 端有什么需求，直接发送给 Server 端去完成，架构清晰明朗，Server 端与 Client 端相对独立，稳定性较好；而共享内存实现方式复杂，没有客户与服务端之别，需要充分考虑到访问临界资源的并发同步问题，否则可能会出现死锁等问题；从这稳定性角度看，Binder 架构优越于共享内存。
    

仅仅从以上两点，各有优劣，还不足以支撑 Google 去采用 Binder 作为 IPC 机制，那么更重要的原因是——

- 从安全的角度

    传统 Linux IPC 的接收方无法获得对方进程可靠的 UID/PID，从而无法鉴别对方身份；而 Android 作为一个开放的开源体系，拥有非常多的开发平台，App 来源甚广，因此手机的安全显得额外重要；对于普通用户，绝不希望从 App 商店下载偷窥隐私数据、后台造成手机耗电等等问题，传统 Linux IPC 无任何保护措施，完全由上层协议来确保。Android **为每个安装好的应用程序分配了自己的 UID**，故进程的 UID 是鉴别进程身份的重要标志，前面提到 C/S 架构，Android 系统中对外只暴露 Client 端，Client 端将任务发送给 Server 端，Server 端会根据权限控制策略，判断 UID/PID 是否满足访问权限，目前权限控制很多时候是通过弹出权限询问对话框，让用户选择是否运行。

- 从语言层面的角度

    大家多知道 Linux 是基于 C 语言（面向过程的语言），而 Android 是基于 Java 语言（面向对象的语言），而对于 Binder 恰恰也符合面向对象的思想，将进程间通信转化为通过对某个 Binder 对象的引用调用该对象的方法，而其独特之处在于 Binder 对象是一个可以**跨进程引用**的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。可以从一个进程传给其它进程，让大家都能访问同一 Server，就像将一个对象或引用赋值给另一个引用一样。
    
    Binder 模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。从语言层面，Binder 更适合基于面向对象语言的 Android 系统，对于Linux系统可能会有点『水土不服』。另外，Binder 是为 Android 这类系统而生，而并非 Linux 社区没有想到 Binder IPC 机制的存在，也并非 Linux 现有的 IPC 机制不够好，相反地，经过这么多优秀工程师的不断打磨，依然非常优秀，每种 Linux 的 IPC 机制都有其存在的价值，同时在 Android 系统中也依然采用了大量 Linux 现有的 IPC 机制，根据每类 IPC 的原理特性，因地制宜，不同场景特性往往会采用其下最适宜的。比如在 Android 中的 Zygote 进程的 IPC 采用的是 Socket 机制，Android 中的 Kill Process 采用的 Signal 机制等等。而 Binder 更多则用在 system_server 进程与上层 App 层的 IPC 交互。
    
- 从公司战略的角度

    众所周知，Linux 内核是开源的系统，所开放源代码许可协议由 GPL 保护，该协议具有『病毒式感染』的能力。
    
    怎么理解这句话呢？受 GPL 保护的 Linux Kernel 是运行在**内核空间**，对于上层的任何类库、服务、应用等运行在**用户空间**，一旦进行 System Call（系统调用），调用到底层 Kernel，那么也必须遵循 GPL 协议。 而 Android 之父 Andy Rubin 对于 GPL 显然是不能接受的，为此，Google 巧妙地将 GPL 协议控制在内核空间，将用户空间的协议采用 Apache-2.0 协议（允许基于 Android 的开发商不向社区反馈源码），同时在 GPL 协议与 Apache-2.0 之间的 Lib 库中采用 BSD 证授权方法，有效隔断了 GPL 的传染性。此举虽有较大争议，但至少目前能够解决 Android 『被传染』，让 GPL 止步于内核空间，这是 Google 在 GPL Linux 下开源与商业化共存的一个成功典范。

综合上述 5 点，可知 Binder 是 Android 系统上层进程间通信的不二选择。

## Binder通信机制
先上一张Binder的工作流程图:
![Binder工作流程图](/img/6.png)

然后我们还是用刚才那个例子来解释上面的流程图。

首先是有一个 ServiceManager，刚开始这个通讯录是空白的，然后 Server 进程向 ServiceManager 注册一个映射关系表，比如同学把自己的地址`北京市 xx 区 xx 街道`写进通讯录，那么就形成了一张表（或者是 Map）：

```
徐同学 —> 北京市 xx 区 xx 街道
```

之后 Client 进程想要和 Server 进程通信，首先向 ServiceManager 查询地址，ServiceManager 收到查询的请求之后，返回查询结果给 Client。注意到这里不管是 Server 进程注册，还是 Client 查询，都是经过 Binder 驱动的，这也正是 Binder 驱动的作用所在，先不急，下面的原理会分析到。

这时候我就拿着地址就开始寄信。当我把信件放扔进邮筒，之后的工作就是由邮局去完成了。邮递员从邮筒取出明信片，然后跨越千山万水将明信片送
达，也就是 Binder 驱动去完成通信的转发。

接下来我们来详细解释一下它的通信原理。

### Binder 跨进程通信原理

上文给出了 Binder 的通信模型，指出了通信过程的四个角色: **Client**, **Server**, **ServiceManager**, **Binder Driver**。但是我们仍然不清楚 Client 到底是如何与 Server 完成通信的。

跨进程通信是需要内核空间做支持的。传统的 IPC 机制如管道、Socket 都是内核的一部分，因此通过内核支持来实现进程间通信自然是没问题的。但是 Binder 并不是 Linux 系统内核的一部分，那怎么办呢？这就得益于Linux 的**动态内核可加载模块**（Loadable Kernel Module，LKM）的机制。

> 模块是具有独立功能的程序，它可以被单独编译，但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行。

这样，Android 系统就可以通过**动态添加一个内核模块运行在内核空间**，用户进程之间通过这个内核模块作为桥梁来实现通信。

于是，在 Android 系统中，这个运行在内核空间的可加载模块就叫 Binder Driver。

传统的进程间通信，采用的是上面说过的6种方案，它们的机制，无非都是先将数据从发送方进程拷贝到内核缓存区，然后再将数据从内核缓存区拷贝到接收方进程，通过两次拷贝来实现的数据交换。如下图：

![传统IPC机制](/img/9.jpg)

而 Binder 别出心裁的地方就在于，它只需要拷贝一次数据，就可以完成数据交换，这得益于一种叫**内存映射**（`mmap()`）的概念。

> `mmap()`是操作系统中一种内存映射的方法。
> 
> 简单地讲就是将用户空间的一块内存区域映射到内核空间。
> 
> 映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。内存映射能减少数据拷贝次数，实现用户空间和内核空间的高效互动。两个空间各自的修改能直接反映在映射的内存区域，从而被对方空间及时感知。也正因为如此，内存映射能够提供对进程间通信的支持。

Binder正是基于`mmap()`来实现的。但是`mmap()`通常是用在**有物理介质的文件系统上**的。举例来说，进程中的用户区域是不能直接和物理设备打交道的，如果想要把磁盘上的数据读取到进程的用户区域，需要两次拷贝（磁盘 --> 内核空间 --> 用户空间）；通常在这种场景下`mmap()`就能发挥作用，通过在物理介质和用户空间之间建立映射，减少数据的拷贝次数，用内存读写取代 I/O 读写，提高文件读取效率。

而 Binder 并不存在物理介质，因此 Binder 驱动使用 `mmap()` 并不是为了在物理介质和用户空间之间建立映射，而是用来在内核空间**创建用于接收数据的缓存空间**。

一次完整的 Binder IPC 通信过程通常是这样：

  1. 首先 Binder 驱动在内核空间创建一个**数据接收缓存区**； 
  2. 接着在内核空间开辟一块**内核缓存区**，建立内核缓存区和内核中数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系；
  3. 发送方进程通过系统调用 `copy_from_user()` 将数据 copy 到内核中的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。

如下图所示：

<div class="center-img">

![Binder内存映射机制](/img/10.png)

</div>

[Android Binder 设计与实现](https://blog.csdn.net/universus/article/details/6211589)一文中对 Client、Server、ServiceManager、Binder 驱动有很详细的描述，以下是部分摘录：

> #### **Binder 驱动**
> Binder 驱动就如同路由器一样，是整个通信的核心；驱动负责进程之间 Binder 通信的建立，Binder 在进程之间的传递，Binder 引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。
> 
> #### **ServiceManager 与实名 Binder**
> ServiceManager 和 DNS 类似，作用是将字符形式的 Binder 名字转化成 Client 中对该 Binder 的引用，使得 Client 能够通过 Binder 的名字获得对 Binder 实体的引用。注册了名字的 Binder 叫实名 Binder，就像网站一样除了除了有 IP 地址意外还有自己的网址。Server 创建了 Binder，并为它起一个字符形式，可读易记得名字，将这个 Binder 实体连同名字一起以数据包的形式通过 Binder 驱动发送给 ServiceManager ，通知 ServiceManager 注册一个名为“张三”的 Binder，它位于某个 Server 中。驱动为这个穿越进程边界的 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager。ServiceManger 收到数据后从中取出名字和引用填入查找表。
> 
> 细心的读者可能会发现，ServierManager 是一个进程，Server 是另一个进程，Server 向 ServiceManager 中注册 Binder 必然涉及到进程间通信。当前实现进程间通信又要用到进程间通信，这就好像蛋可以孵出鸡的前提却是要先找只鸡下蛋！Binder 的实现比较巧妙，就是预先创造一只鸡来下蛋。ServiceManager 和其他进程同样采用 Binder 通信，ServiceManager 是 Server 端，有自己的 Binder 实体，其他进程都是 Client，需要通过这个 Binder 的引用来实现 Binder 的注册，查询和获取。ServiceManager 提供的 Binder 比较特殊，它没有名字也不需要注册。当一个进程使用 BINDER SETCONTEXT_MGR 命令将自己注册成 ServiceManager 时 Binder 驱动会自动为它创建 Binder 实体（这就是那只预先造好的那只鸡）。其次这个 Binder 实体的引用在所有 Client 中都固定为 0 而无需通过其它手段获得。也就是说，一个 Server 想要向 ServiceManager 注册自己的 Binder 就必须通过这个 0 号引用和 ServiceManager 的 Binder 通信。类比互联网，0 号引用就好比是域名服务器的地址，你必须预先动态或者手工配置好。要注意的是，这里说的 Client 是相对于 ServiceManager 而言的，一个进程或者应用程序可能是提供服务的 Server，但对于 ServiceManager 来说它仍然是个 Client。
> 
> #### **Client 获得实名 Binder 的引用**
> Server 向 ServiceManager 中注册了 Binder 以后， Client 就能通过名字获得 Binder 的引用了。Client 也利用保留的 0 号引用向 ServiceManager 请求访问某个 Binder: 我申请访问名字叫张三的 Binder 引用。ServiceManager 收到这个请求后从请求数据包中取出 Binder 名称，在查找表里找到对应的条目，取出对应的 Binder 引用作为回复发送给发起请求的 Client。从面向对象的角度看，Server 中的 Binder 实体现在有两个引用：一个位于 ServiceManager 中，一个位于发起请求的 Client 中。如果接下来有更多的 Client 请求该 Binder，系统中就会有更多的引用指向该 Binder ，就像 Java 中一个对象有多个引用一样。

至此，我们大致能总结出 Binder 通信过程：

1. 首先，一个进程使用一个命令通过 Binder 驱动将自己**注册**成为 ServiceManager；
2. 接着，Server 进程通过驱动向 ServiceManager 中注册 Binder（Server 中的 Binder 实体），表明可以对外提供服务。驱动为这个 Binder 创建位于内核中的**实体节点**以及 ServiceManager 对实体的**引用**，将**名字以及新建的引用打包传给 ServiceManager**，ServiceManger 将其填入查找表。
3. 最后，Client 通过名字，在 Binder 驱动的帮助下从 ServiceManager 中获取到对 Binder 实体的引用，通过这个引用就能实现和 Server 进程的通信。

我们看到整个通信过程都需要 Binder 驱动的接入。下图能更加直观的展现整个通信过程（为了进一步抽象通信过程以及呈现上的方便，下图我们忽略了 Binder实体及其引用的概念）：

<div class="center-img">

![Binder通信过程](/img/11.jpg)

</div>

### Binder 通信中的代理模式
我们已经解释清楚 Client、Server 借助 Binder 驱动完成跨进程通信的实现机制了，但是还有个问题会让我们困惑。A 进程想要 B 进程中某个对象（object）是如何实现的呢？毕竟它们分属不同的进程，A 进程没法直接使用 B 进程中的对象实例。

前面我们介绍过跨进程通信的过程都有 Binder 驱动的参与，因此在数据流经 Binder 驱动的时候驱动会对数据做一层转换。当 A 进程想要获取 B 进程中的 object 时，驱动并不会真的把 object 返回给 A，而是返回了一个跟 object 看起来一模一样的代理对象 objectProxy，这个 objectProxy 具有和 object 一模一样的方法，但是这些方法并没有 B 进程中 object 对象那些方法的能力，这些方法只需要把把请求参数交给驱动即可。对于 A 进程来说和直接调用 object 中的方法是一样的。

当 Binder 驱动接收到 A 进程的消息后，发现这是个 objectProxy 就去查询自己维护的表单，一查发现这是 B 进程 object 的代理对象。于是就会去通知 B 进程调用 object 的方法，并要求 B 进程把返回结果发给自己。当驱动拿到 B 进程的返回结果后就会转发给 A 进程，一次通信就完成了。

上面的代理模式可以归结为下图：
![Binder的代理模式](/img/12.png)

现在我们可以对 Binder 做个更加全面的定义了：

- 从进程间通信的角度看，Binder 是一种**进程间通信的机制**；
- 从 Server 进程的角度看， Binder 指的是 **Server 中的 Binder 实体对象**； 
- 从 Client 进程的角度看，Binder 指的是对 **Binder 代理对象**，是 Binder 实体对象的一个远程代理；
- 从传输过程的角度看，Binder 是一个**可以跨进程传输的对象**；Binder 驱动会对这个跨越进程边界的对象对一点点特殊处理，自动完成代理对象和本地对象之间的转换。

## 实操

在写例子之前，我们先介绍一下接下来要出场的几位角色：

**1. IInterface**

IInterface 是 Binder 接口的基类，每当有一个 Binder 类的接口，都要继承于该接口。它的定义很简单：

```JAVA
// android.os.IInterface

package android.os;

public interface IInterface
{
    // 在返回实例时不要进行显式强转，而是必须使用这个方法来转换成 IBinder。
    public IBinder asBinder();
}
```

只有一个方法`asBinder()`，用来返回对应的 Binder 对象。

**2. IBinder**

IBinder 是一个接口，约定了一些常量，比如 transaction code 的范围值、最大的 IPC 数量等等。

<div class="center-img">

![](/img/binder-1588152047.png)

</div>

IBinder 代表了一种**跨进程传输的能力**，实现这个接口就意味着这个对象能进行跨进程传递。

它最重要的一个方法就是`transact()`。而 Binder 实现了 IBinder 接口，我们一会看看它做了什么。

**3. Binder**

Binder 实现了 IBinder 定义的操作，它是 Android IPC 的基础，平常接触到的各种 Manager（ActivityManager, ServiceManager 等），以及绑定 Service 时都在使用它进行跨进程操作。

Android 官方建议的是：日常开发中一般不需要我们再实现 IBinder，直接使用系统提供的 Binder 即可。

它的存在不会影响一个应用的生命周期，只要创建它的进程在运行它就一直可用。

通常我们需要在顶级的组件（Service, Activity, ContentProvider）中使用它，这样系统才知道你的进程应该一直被保留。

它的两个关键方法，第一个是实现了 IBinder 的`transact()`：

```java
public final boolean transact(int code, @NonNull Parcel data, @Nullable Parcel reply,
        int flags) throws RemoteException {
    if (false) Log.v("Binder", "Transact: " + code + " to " + this);

    if (data != null) {
        data.setDataPosition(0);
    }
    boolean r = onTransact(code, data, reply, flags);
    if (reply != null) {
        reply.setDataPosition(0);
    }
    return r;
}
```

它的过程，实际上就是调用了它的`onTransact`方法，然后将执行成功与否返回。继续看看`onTransact`：

```java
protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply,
        int flags) throws RemoteException {
    if (code == INTERFACE_TRANSACTION) {  // 获取描述符
        ...
        return true;
    } else if (code == DUMP_TRANSACTION) {  // 转存 Binder 信息
        ...
        return true;
    } else if (code == SHELL_COMMAND_TRANSACTION) {   // 执行 SHELL 命令
        ...
        return true;
    }
    return false;
}
```

没有什么特别的，只是处理了三种情况。而我们自定义 Binder，或者使用系统生成的 Binder 时，往往都会自己实现`onTransact`方法，所以这里不用特别关注。

好，介绍完角色们，接下来要讲讲剧本了。

平时在开发中，实现 IPC 机制时，最多使用的是 [AIDL（Android Interface Definition Language）](https://developer.android.com/guide/components/aidl.html)。当我们定义好 AIDL 文件，在编译时编译器会帮我们生成代码实现 IPC 通信。

Android 在这一点上，代码的可读性并不高。比如一个 MyClass.aidl 的文件在经过编译时生成的 MyClass.java 中，会包含一个 MyClass 接口、一个 Stub 静态抽象类和一个 Proxy 静态类。Proxy 是 Stub 的静态内部类，Stub 又是 BookManager 的静态内部类，这就造成了可读性和可理解性的问题。

但是 Android 这样设计是有它的道理的。如果有多个 AIDL 文件时，这些类在同一个文件里，可以有效避免 Stub 和 Proxy 的重名问题。

我们先来写一个简单的 AIDL 文件：

```java
// IPerson.aidl
package com.debuglife.aidl.test;

// Declare any non-default types here with import statements

interface IPerson {
    void setName(String aString);
    String getName();
}
```

可以见到，我们只需要声明一个接口，并在其中定义方法即可。编译一次后，就会在`app/build/generated/aidl_source_output_dir/debug/out`中找到对应的生成类（与 AIDL 文件同名，但扩展名会换为 .java），我们来看看生成后的代码，注释中会解释每一部分的用途：

```java
// IPerson.java
package com.debuglife.aidl.test;

public interface IPerson extends android.os.IInterface {

    // 本地 IPC 实现的『存根』类
    public static abstract class Stub extends android.os.Binder implements com.debuglife.aidl.test.IPerson {
    	// 一个唯一标识该 Binder 类的字符串
        private static final java.lang.String DESCRIPTOR = "com.debuglife.aidl.test.IPerson";

        // 构造函数里调用 attachInterface 方法，然后将自身的静态实例和标识符传入父类 Binder 类中
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        // 将 IBinder 对象转换成 com.debuglife.aidl.test.IPerson 接口
        // 如果需要的话，会生成一个 Proxy
        // 如果需要通信的 Client 和 Server 在同一个进程中，那意味着可以直接使用该对象的实例，就不用 
        public static com.debuglife.aidl.test.IPerson asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            // 通过刚才设定的 DESCRIPTOR 来确定对应的接口是否正确
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            // 判断 Client 和 Server 是否在同一个进程内
            if (((iin != null) && (iin instanceof com.debuglife.aidl.test.IPerson))) {
            	// 如果是，则直接返回实例
                return ((com.debuglife.aidl.test.IPerson) iin);
            }
            // 如果不是，则要新建 Proxy 并返回 Proxy 的实例
            return new com.debuglife.aidl.test.IPerson.Stub.Proxy(obj);
        }

		// 复写 IInterface 的 asBinder 方法，用来返回与 IPerson 相对应的 Binder 对象
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

		// 该方法是每次通信时会被调用
		// 参数：int code 标识码，类似 Handler 中的 msg.what
		// Parcel data 发送的数据
		// Parcel reply 返回的数据
		// int flags 附加的标识位，0表示正常通信，FLAG_ONEWAY 表示单向通信
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
            	// 询问接收方（被调用方）接口描述符
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_setName: {
                    data.enforceInterface(descriptor);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    // 此处的 setName 调用的是 IPerson 中的 setName 方法。
                    // 如果 Client 和 Server 在同一个进程中，那么实现 setName 方法的，是该 Binder 对象
                    // 如果 Client 和 Server 不在同一个进程中，那么实现 setName 方法的，是 Proxy 对象
                    this.setName(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_getName: {
                    data.enforceInterface(descriptor);
                    // 同上
                    java.lang.String _result = this.getName();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        // Proxy 是当 Client 和 Server 不是同一个进程时使用的
        private static class Proxy implements com.debuglife.aidl.test.IPerson {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public void setName(java.lang.String aString) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(aString);
                    // 被代理的 Binder.Stub 的 transact 方法，然后这里面会接着调用 onTransact 方法
                    mRemote.transact(Stub.TRANSACTION_setName, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public java.lang.String getName() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    // 同上
                    mRemote.transact(Stub.TRANSACTION_getName, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        // 这两个静态常量是用来在 Binder 通信过程中，唯一标识一次传输的。类似`startActivityForResult()`中的 code。
        // Binder 通信时，Client 向 Server 发起请求，Server 向 Client 返回请求，必须要有一个『鸡毛信』来确定这次沟通找对了人。// 在这个例子中，它们的值分别是 IBinder.FIRST_CALL_TRANSACTION + 0 和 IBinder.FIRST_CALL_TRANSACTION + 1，也就是 1 和 2。
        // 如果有多个方法，那就会有多个常量，值也会继续递增。
        static final int TRANSACTION_setName = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getName = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    // AIDL 文件中定义的两个方法，被照搬过来，同时抛出异常，方便捕获
    public void setName(java.lang.String aString) throws android.os.RemoteException;
    public java.lang.String getName() throws android.os.RemoteException;
}
```

Android Studio 会提醒我们**不要编辑这个文件**。其实编辑了也会被重新覆盖掉。因为 AIDL 生成的文件是按照严格的通信格式来生成的，不允许修改，否则在 Binder 通信过程中就会出现错误。

我们来总结一下 IPersion.java 中都有些什么：

- IPerson 接口，继承了 IInterface 接口；
- 在 AIDL 文件中声明的两个方法`setName()`和`getName()`。不同的是，它抛出了`android.os.RemoteException`异常；
- 静态抽象类`Stub`；
- Stub 类中的私有静态类`Proxy`。

好，现在我们应该写一个 Service，作为 Server 端。Server 端当然要想办法实现刚才定义的`setName`和`getName`：

```java
package com.debuglife.aidl.test;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.os.Parcel;
import android.os.RemoteException;
import android.util.Log;

import androidx.annotation.Nullable;

public class AIDLService extends Service {

    private String name;

    private Binder binder = new IPerson.Stub() {
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            return super.onTransact(code, data, reply, flags);
        }

        @Override
        public void setName(String s) throws RemoteException {
            name = s;
        }

        @Override
        public String getName() throws RemoteException {
            return name;
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        // 在此处返回了上面声明的 Binder 实例
        return binder;
    }
}
```

接下来，便需要将这个 Service 加入 AndroidManifest 中：

```xml
<service
    android:name=".AIDLService"
    android:enabled="true"
    android:exported="true" >
    <intent-filter>
        <action android:name="com.debuglife.aidl.test.IPersonService" />
    </intent-filter>
</service>
```

此处的 action，是给跨进程通信的 App 使用的，现在暂时用不到。

至此，Server 端的工作完成，接下来就要看看如何向这个 Server 端发送消息了。

我们先来个『C/S同进程』的例子，在当前工程下，再来个 MainActivity：

```java
IPerson iPerson;

ServiceConnection conn = new ServiceConnection({
    @Override
    void onServiceConnected(ComponentName name, IBinder service) {
        // 将 service 转换为 IPerson 
        iPerson = IPerson.Stub.asInterface(service);
    }

    @Override
    void onServiceDisconnected(ComponentName name) {
    }
});

@Override
void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Intent intent = new Intent(this, AIDLService.class);
    bindService(intent, conn, Context.BIND_AUTO_CREATE);

    iPerson.setName("DebugLife");
    System.out.println(iPerson.getName());
}

@Override
void onDestroy() {
    unbindService(conn);
    super.onDestroy();
}
```

思路很清晰，使用`bindService`方法，将目标 Service 绑定，然后在 ServiceConnection 的回调中，将 iPerson 赋值为 Binder 实例，这样，在接下来就可以调用 IPerson 定义的方法了。

那么『C/S 不同进程』的，是什么步骤呢？

1. 我们需要将 Server 端的 AIDL 文件，按原路径拷贝到 Client 端工程中，并且编译一遍，生成对应的 java 文件。
2. 在`bindService`时，要使用下面的方法：

```java
Intent intent = new Intent();
intent.action = "com.debuglife.aidl.test.IPersonService";  // 这是刚才我们在 Server 端的 manifest 中定义的 action
intent.setPackage("com.cy198706.review");  // 在 Android 5.0 之后，IPC 通信需要使用显式 Intent，也即指定接收者

bindService(intent, conn, Context.BIND_AUTO_CREATE);
```

这样，就能达到 IPC 的目的了。

有的同学要问了，你这光传个 String 啊，我要传对象怎么办？

在[Serializable和Parcelable](2020-04-14-serializable-and-parcelable.md)一文中，我们提到了，IPC 中，是可以传递 Parcelable 的，那么，在此处，只需要将对象转换为 Parcelable，就可以做到了。