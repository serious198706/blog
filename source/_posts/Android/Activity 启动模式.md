---
title: Activity四大启动模式解析
link: activity-launchmode
cover: http://blog.cigis-cloud.com/android-1593080864.jpg
toc: true
tag: 
    - Android
    - Activity
category: 
    - Android
    - Development
date: 2020-04-22
---

这篇文章我们来详细解释 Activity 的四种启动模式。

<!-- more -->

# Activity的四种启动模式

它们分别是：

- standard
- singleTop
- singleTask
- singleInstance

下面来分别做介绍。

这些模式可分为两大类：standard和singleTop为一类，singleTask和singleInstance为另一类。

使用standard或singleTop启动模式的 Activity 可多次进行实例化。实例可归属任何任务，并且可位于 Activity 堆栈中的任何位置。通常，它们会启动到名为 `startActivity()`的任务中（除非 Intent 对象包含 FLAG_ACTIVITY_NEW_TASK 指令，在此情况下会选择其他任务 — 请参阅 taskAffinity属性）。

相比之下，singleTask和singleInstance只能启动任务。它们始终位于 Activity 堆栈的根位置。此外，设备一次只能保留一个 Activity 实例，即一次只允许一个此类任务。

## standard

顾名思义，standard英文意思就是『标准的』。

也就是说这种启动模式是默认的，我们平时在开发中使用最多的就是standard模式的。

如果一个Activity的启动模式被设置成standard，那么它可以无限制的创建。你每一次通过Intent去启动这种模式的Activity都会重新创建一个。

大家可以想象一下邮箱里的收件箱（假设我们将打开邮件的Activity的启动模式设置为Standard，当然这也是默认的模式）里有10封邮件。我们给查看邮件的Activity起名为CheckEmailActivity,我点击第一封邮件将会打开一个CheckEmailActivity，当我看完之后点击下一封邮件，另一个CheckEmailActivity又会被创建，这样如果我们将10封邮件全部看完，那在Activity任务栈中将会有10个CheckEmailActivity，而且如果我想回到收件箱页面还必须点10次返回键！想想是不是很可怕？

所以说standard模式虽然很常用，但也不是适用于任何场合。

另外说一点，standard模式在Android 5.0（Lollipop）之前和之后是有区别的。

### Android Lollipop之前

standard模式的Activity总是会被创建在启动它的Activity同一个任务栈中顶端（任务栈是一个栈结构，先进后出 First In Last Out），就算他们来自不同的应用。

想象一个场景，如果你在A应用中要分享一个本地图片，这样会打开系统的图片查看应用中的图片选择器Activity，虽然这两个Activity来自不同的应用，但Android系统仍将会把他们放在同一个任务栈中，即A应用的任务栈中。如下图：

![webp](/img/16.png)

### Android Lollipop之后

如果将要启动的Activity和启动它的Activity来自同一个应用，那没话说，和Lollipop之前一样，新的Activity会被创建在当前任务栈中的顶端。

但是如果它们来自不同的应用，那就会创建一个新的任务栈，再把要启动的Activity放在新的任务栈中，这时这个新启动的Activity就是新创建的任务站点的根Activity。如下图所示：

![webp](/img/17.png)

## singleTop

顾名思义，singleTop的意思就是『在顶部只能有一个』，也即『栈顶复用』。

这种启动模式非常类似于standard，但是也有一些区别：

如果在启动这种模式的Activity的时候，当前任务栈的顶端已经存在了相同的Activity，那系统就不会再创建新的，而是回调任务栈中已经存在的该Activity的`onNewIntent()`方法。请看下面的示意图：

![webp](/img/18.png)

也正因为singleTop启动模式的特殊性，所以在开发时，如果指定了一个Activity的启动模式是singleTop的那就应该既要重写`onCreated()`方法用于应对第一次创建的情况，也要重写`onNewIntent()`方法来应对重复创建的情况。

其实大家可以想象一下，这种启动模式的应用场景。Android既然提供了这种启动模式，说明肯定有应有场景需要这样的方式。其实最常用的场景就是搜索，比方说我们在搜索框中输入想要搜索的内容点击搜索进入SearchResultActivty查看搜索的结果（一般我们也会在搜索结果页提供搜索框，这样用户无需点击返回键回到上一个页面再在搜索框中输入搜索内容点击搜索），如果此时用户还想搜点别的东西，就可以直接在当前的搜索结果页SearchResultActivty中的搜索框输入搜索内容继续搜索。

大家想象一下，如果我们把SearchResultActivty的启动模式设置为Standard的话会是什么样的景象。比如我们连着搜了10个内容，那就会启动10个不同的SearchResultActivty，然而这些SearchResultActivty功能完全一样，完全没有必要创建这么多，而且还有一个和上一节中的邮箱一样的问题，就是用户搜索结束想回到首页，那就还得按10次返回键才能回到首页，- -！

这时，singleTop启动模式就派上用场了，我们首先把SearchResultActivty的启动模式设置为singleTop，这样用户在SearchResultActivty页面中继续搜索的时候，我们只需把用户要搜索的内容放在Intent里面然后启动SearchResultActivty，这时系统并不会重新创建新的SearchResultActivty，而是回调当前任务栈栈顶的SearchResultActivty的`onNewIntent()`方法来接收带有用户搜索内容信息的Intent，然后我们拿到用户搜索内容后调搜索接口，并根据接口返回内容重新刷新布局即可，似不似很神奇？其实我们在上一节提到的邮箱的问题，也是用这种方式来解决的，原理和搜索一样的。

还有比较常见的应用场景就是IM应用的聊天界面，消息被推送过来之后，就会以singleTop的方式来启动该Activity，并在`onNewIntent()`中处理数据就可以了。

> “standard”和“singleTop”模式只有一处不同：每次“standard”Activity 有新的 Intent 时，系统都会创建新的类实例来响应该 Intent。每个实例处理单个 Intent。同样地，您也可以创建新的“singleTop”Activity 实例来处理新的 Intent。不过，如果目标任务的 Activity 堆栈顶部已有一个 Activity 实例，则该实例会（通过调用 onNewIntent()）接收新的 Intent；此时不会创建新实例。在其他情况下（例如，如果“singleTop”Activity 的某个现有实例虽在目标任务内，但未处于堆栈顶部，或者虽然位于堆栈顶部，但不在目标任务中），系统会创建新实例并将其送入堆栈。

## singleTask

这种启动模式的Activity在Android系统中只允许存在一个实例。

如果系统中已经存在了该种启动模式的目标Activity，则系统并不会重新创建一个目标Activity，而是首先将持有目标Activity的整个任务栈都会被置于前台（用户可见），并且通过`onNewIntent()`方法将启动目标Activity的Intent传递给目标Activity，置于目标Activity拿到这个Intent之后要做什么操作，系统就不管了，随便你拿来干什么。

但是这里有个问题，就是目标Activity和源Activity是不是来自同一应用。

- 源Activity和目标Activity来自**同一个应用**

    这种情况还要分两种情况说：

    1. 当前系统中还没有目标Activity的实例。这种情况最简单，直接在当前的任务栈中创建singleTask模式的Activity并置于栈顶即可。

    2. 当前系统中已经存在目标Activity的实例。这种情况比较特殊，因为系统会把任务栈中目标Activity之上的所有Activity销毁，以让目标Activity处在栈顶的位置。

    这里还要还要再提醒大家的是，因为目标Activity已经存在，系统不会重新创建，而是通过`onNewIntent()`的方式把Intent传递过来，这点和singleTop模式有些类似。注意了，这里让我们回想一下文章开头的我所说的场景，如何让用户在支付完成页直接跳转到首页，并把不需要的Activity销毁？singleTask启动模式是不是刚好和我们的需求一致？请看下面的示意图：

    ![singleTask示例](/img/19.png)

- 源Activity和目标Activity来自**不同应用**

    这种情况也要分两种情况说：

    1. 当前系统中还没有目标Activity的实例。这时系统首先会看任务管理器中是否有目标Actvity所在应用的任务栈？如果有的话，那就直接在目标Activity所在应用的任务栈的栈顶创建即可。如果任务管理器中没有目标Activity所在应用的任务栈，系统就会创建其所在应用的任务栈和目标Activity，并且把目标Activity作为新建任务栈的根Activity。如下图所示：

    ![singleTask示例](/img/20.png)

    2. 当前系统中已经存在目标Activity的实例。目标Activity所在任务栈会被置于前台（即用户可见），而且也会把目标Activity之上的所有Actvity全部销毁。

上方的几种情况，可以总结为下面的流程图：

![](/img/21.png)

singleTask比较常见的应用场景是当应用的主页面，从层级比较深的页面（如订单完成页面）直接返回主页的时候，就可以清除其他的Activity，只保留主页面Activity。

## singleInstance

这种启动模式和singleTask几乎一样，它也只允许系统中存在一个目标Activity的实例，包括上面我们所说的singleTask的一些特性singleInstance都有。唯一不同的是，持有目标Activity的任务栈中只能有目标Activity一个Actvitiy，不能再有别的Activity。

对！就是承包了这片鱼塘！![](/img/doge.png)

其实从这种启动模式的名字也可以看出来它表示的意思，singleInstance直译过来就是『单一实例』，什么意思呢？有两层意思，我们来分析一下：

1. 跟系统说，『我是独一无二的，不许和我一样的人存在！』，这就是说系统中存在一个目标Activity；
2. 跟任务栈说，『我是独一无二的，不许你心里再装别的人！』，这就是说持有目标Activity的任务栈中只能有目标Activity一个Activity。

所以，如果要启动singleInstance模式的Activity，那只能新创建一个任务栈用来放它。同样的，如果从这种启动模式的Activity中启动别的Activity，那不好意思，我不管你是不是和我处在同一个应用，我所在的任务栈只能拥有我一个人，您呐，另外让系统给你创建一个任务栈待着去吧。

比较常见的应用场景是用在比较频繁调用的Activity。比如系统浏览器，比如闹钟等。

> “singleTask”和“singleInstance”模式同样只有一处不同：“singleTask”Activity 允许其他 Activity 成为其任务的一部分。该 Activity 始终位于其任务的根位置，但其他 Activity（必然是“standard”和“singleTop”Activity）可以启动到该任务中。另一方面，“singleInstance”Activity 不允许其他 Activity 成为其任务的一部分。它是任务中唯一的 Activity。如果它启动另一个 Activity，则系统会将该 Activity 分配给其他任务，就如同 Intent 中包含 FLAG_ACTIVITY_NEW_TASK 一样。

好了，至此我们介绍了Activity的4种启动模式了，也大致了解了每种启动模式的特点了。

再引用一下官网对四种启动模式的解释：

<table>
    <tr>
	    <th>用例</th>
	    <th>启动模式</th>
	    <th>多个实例？</th>  
	    <th>注释</th>  
	</tr>
    <tr>
        <td rowspan="2">大多数 Activity 的正常启动</td>
	    <td>standard</td>
	    <td>是</td>
	    <td>默认。系统始终会在目标任务中创建新的 Activity 实例，并向其传送 Intent。</td>
    </tr>
    <tr>
	    <td>singleTop</td>
	    <td>视情况而定</td>
	    <td>如果目标任务的顶部已存在 Activity 实例，则系统会通过调用该实例的 <span style="color: #1E90FF">onNewIntent()</span> 方法向其传送 Intent，而非创建新的 Activity 实例。</td>
    </tr>
    <tr>
        <td rowspan="2">专用启动<br/>（不建议在一般情况下使用）</td>
	    <td>singleTask</td>
	    <td>否</td>
	    <td>系统会在新任务的根位置创建 Activity 并向其传送 Intent。不过，如果已存在 Activity 实例，则系统会调用该实例的 <span style="color: #1E90FF">onNewIntent()</span> 方法（而非创建新的 Activity 实例），向其传送 Intent。</td>
    </tr>
    <tr>
	    <td>singleInstance</td>
	    <td>否</td>
	    <td>与<b>『singleTask』</b>相同，只是系统不会将任何其他 Activity 启动到包含实例的任务中。该 Activity 始终是其任务中的唯一 Activity。</td>
    </tr>
</table>

同时，官方提出：

> 如上表所示，standard 是默认模式，并且适用于大多数类型的 Activity。对众多类型的 Activity 而言，SingleTop 也是常见且有用的启动模式。其他模式（singleTask 和 singleInstance）<span style="color:red">不适用于大多数应用</span>，因为它们所形成的交互模式可能让用户感到陌生，并且与大多数其他应用差别较大。

> 无论您选择哪种启动模式，在 Activity 启动期间以及使用返回按钮从其他 Activity 和任务返回该 Activity 时，请务必对其进行易用性测试。

## 关于taskAffinity

在官方文档中是这样介绍`android:taskAffinity`属性的：

> 与该 Activity 有着相似性的任务。从概念上讲，具有同一相似性的 Activity 归属同一任务（从用户的角度来看，则是归属同一『应用』）。任务的相似性由其根 Activity 的相似性确定。
> 
> 相似性确定两点内容 — Activity 更改父项后的任务（请参阅`allowTaskReparenting`属性），以及通过`FLAG_ACTIVITY_NEW_TASK`标记启动 Activity 时，用于容纳该 Activity 的任务。
> 
> 默认情况下，应用中的所有 Activity 都具有同一相似性。您可以设置该属性，以不同方式将其分组，甚至可以在同一任务内放置不同应用中定义的 Activity。如要指定 Activity 与任何任务均无相似性，请将其设置为空字符串。
> 
> 如果未设置该属性，则 Activity 会继承为应用设置的相似性（请参阅 `<application\>` 元素的`taskAffinity`属性）。应用默认相似性的名称为 `<manifest>` 元素所设置的软件包名称。

`taskAffinity`，可以翻译为**任务相关性**。这个参数标识了一个 Activity 所需要的任务栈的名字，默认情况下，所有 Activity 所需的任务栈的名字为应用的包名，当 Activity 设置了 `taskAffinity` 属性，那么这个 Activity 在被创建时就会运行在和 `taskAffinity` 名字相同的任务栈中，如果没有，则新建 `taskAffinity` 指定的任务栈，并将 Activity 放入该栈中。另外 `taskAffinity` 属性主要和 `singleTask` 或者  `allowTaskReparenting` 属性配对使用，在其他情况下没有意义。