---
title: MVC-MVP-MVVM 进化之路
date: 2020-08-17
link: mvc-mvp-mvvm
cover: https://miro.medium.com/max/1280/0*1ZrS8t3HvPzRAuqg.png
---

众所周知，日常开发中，比较受欢迎的设计模式，除了单例、工厂、装饰器等之外，被大家讨论最多的，就是 MVC、MVP 和 MVVM 了。

这三种设计模式各有千秋，耦合度都比较低，并且易于测试与维护。我们今天就来讨论一下这三种设计模式在开发中的应用。

<!-- more -->


> 这篇文章中关于设计模式的讨论，都以 Android 平台做为示例环境，以 Kotlin 做为示例语言。

## MVC

MVC（Model-View-Controller），它有三个部分：**模型（Model）**、**视图（View）**和**控制器（Controller）**。

- Model
  这部分功能比较明显，就是我们的应用需要显示的数据，它是一个类，其中承载了业务模型和数据模型；同时，它又提供数据的操作，并且直观地展示了数据的变化，当然，这种变化要遵循一定的业务逻辑。

- View
  View 代表了 UI 组件，如 XML、HTML 等，在 MVC 模式中，View 负责展现 Controller 交过来的数据，它会监听 Model 的状态变化，并展示数据更新后的 Model。Model 和 View 的交互是基于观察者模式。在 Android 中，各种 XML 布局，就是我们的 View 层。

- Controller
  Controller 负责处理各种请求。它会通过 Model 处理用户数据，并将处理结果交给 View 去展示，它通常扮演着 View 和 Model 之间的调度者的角色。显而易见地，在 Android 中，Activity 或 Fragment 担当了这个角色。

其实 Activity 并非标准的 Controller，它一方面用来控制了布局，另一方面还要在 Activity 中写业务代码，造成了 Activity 既像 View 又像 Controller。

因此，**这种开发方式不太适合 Android 开发**。

MVC 的结构如下图所示：

![MVC 结构](http://blog.cigis-cloud.com/design-pattern-1597991354.png)

我们用代码来展示一下 Android 中如何使用 MVC 模式，我们的例子是用户填写用户名密码，点击登录按钮之后，请求 UserInfo，并最终展示在界面上。

首先是 Model：

```kotlin
class UserInfo {
    var uid: Int = 0 
    var name: String? = null
    var phone: String? = null
    var password: String? = null
}
```

接下来是我们的 View 层，也就是 XML 布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:id="@+id/llPhone"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_margin="16dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <TextView
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="PHONE" />

        <EditText
            android:id="@+id/etPhone"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="4"
            android:layout_marginStart="8dp"
            android:inputType="phone"
            android:hint="Input phone number"
             />
    </LinearLayout>

    <LinearLayout
        android:id="@+id/llPassword"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:orientation="horizontal"
        app:layout_constraintTop_toBottomOf="@+id/llPhone"
        tools:layout_editor_absoluteX="16dp">

        <TextView
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="PASSWORD" />

        <EditText
            android:id="@+id/etPassword"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="8dp"
            android:layout_weight="4"
            android:hint="Input password"
            android:inputType="textVisiblePassword" />
    </LinearLayout>

    <Button
        android:id="@+id/btnLogin"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        app:layout_constraintTop_toBottomOf="@+id/llPassword"
        tools:layout_editor_absoluteX="16dp"
        android:text="LOGIN"/>
    
    <TextView
        android:id="@+id/tvUserInfo"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintTop_toBottomOf="@id/btnLogin"
        android:layout_marginTop="16dp" />
    

</androidx.constraintlayout.widget.ConstraintLayout>
```

接着就是 Controller 层，也就是我们的 Activity：

```kotlin
package me.codingrabbit.jetpacklearn

import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import com.google.android.material.snackbar.Snackbar
import kotlinx.android.synthetic.main.activity_login.*
import java.io.BufferedInputStream
import java.io.InputStream
import java.net.HttpURLConnection
import java.net.URL
import javax.net.ssl.HttpsURLConnection

class LoginActivity : AppCompatActivity(), Callback {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        btnLogin.setOnClickListener {
            val userInfo = UserInfo()
            userInfo.phone = etPhone.editableText.toString()
            userInfo.password = etPassword.editableText.toString()
            requestUserInfo(userInfo, this@LoginActivity)
        }
    }

    override fun onSuccess(userInfo: UserInfo) {
        super.onSuccess(userInfo)

        Snackbar.make(tvUserInfo, "UserInfo: uid=${userInfo.uid}, " +
                "name=${userInfo.name}, " +
                "phone=${userInfo.phone}", Snackbar.LENGTH_SHORT)
    }

    override fun onError(msg: String) {
        super.onError(msg)

        Snackbar.make(tvUserInfo, "Error: ${msg}", Snackbar.LENGTH_SHORT)
    }

    private fun requestUserInfo(userInfo: UserInfo, callback: Callback) {
        // 请求数据，省略
        callback.onSuccess(userInfo)
    }
}
```

可以看到，Activity 如上面所说，也的确是同时扮演了 View 和 Controller 的角色。

上面的流程可以用下图总结：

![](http://blog.cigis-cloud.com/design-pattern-1597984167.png)

盘点一下 MVC 模式的优缺点：

### 优点

1. Model 与 View 耦合度较低，方便进行单独的单元测试；
2. 结构简单；
3. 代码量少。

### 缺点

1. Model 与 View 并没有完全解耦，后期业务复杂度较高时，维护起来稍显复杂；
2. Controller 与 Android API 耦合度很高，测试起来比较麻烦；
3. Controller 与 View 耦合度很高，如果我们修改了 View，那么回头来还要修改 Controller，这也意味着，Controller 将会变得越来越臃肿和难以维护；
4. MVC 模型不适合小型 APP 的开发。

## MVP

MVP（Model-View-Presenter）是 MVC 模式的演化版本，它也有三个部分：**模型（Model）**、**视图（View）**和**展示（Presenter）**。

- Model
  
  与 MVC 中的 Model 不太相同。这里的 Model 指存取数据，也就是**用来从指定的数据源中获取数据**，不要将其理解成 MVC 中的 Model。在MVC 中 Model 是**数据模型**，而在MVP中，我们用Bean来表示数据模型。

- View
  
  与 MVC 中的 View 相同，一般指 XML 布局。

- Presenter
  
  这就是 MVC 与 MVP 不同的一个部分了。它负责连接 Model 和 View，让 Model 和 View **彻底解耦**。Model 和 View 不会直接发生关系，它们需要通过 Presenter 来进行交互。在实际的开发中，我们可以用接口来定义一些规范，然后让我们的 Model 和 View 实现它们，并借助 Presenter 进行交互即可。

它的结构如下图所示：

![MVP 结构](http://blog.cigis-cloud.com/design-pattern-1597991166.png)

接下来我们使用 MVP 模式改进一下上面的例子。

我们先看一下，使用 MVP 模式之后，项目下包的结构：

![MVP 包结构](http://blog.cigis-cloud.com/design-pattern-1597972649.png)

可以看到，我抽出了一个单独的包`base`，用来做一些通用型的功能，所有的 View 都应该继承自这个父类。

那么，随着 View 和 Presenter 越来越多，维护的难度系数也会越来越高，这时我们需要引入新的一层 —— Contract。

这一层的意义在于，可以将某一个业务相关的指定的 View 和 Presenter 放在一个接口中，更加集中，每一个业务所需要的 View 和 Presenter 可以一目了然地展现在我们面前。上面的例子就可以修改成：

![](http://blog.cigis-cloud.com/design-pattern-1597973055.png)

LoginContract 代码如下：

```kotlin
interface LoginContract {
    interface IView : BaseView {
        fun onSuccess(userInfo: UserInfo?)
        fun onError(msg: String?)
    }

    interface IPresenter : BasePresenter {
        fun requestUserInfo()
    }
}
```

而 View 和 Presenter 类实现的接口，同样地要转变为实现该 Contract 中的对应接口：

```kotlin
interface LoginView : LoginContract.IView {
    override fun onSuccess(userInfo: UserInfo?) {
    }

    override fun onError(msg: String?) {
    }
}

class LoginPresenter(val view: LoginContract.IView) : LoginContract.IPresenter {
    override fun requestUserInfo() {
    }
}
```

从上面我们可以看出，我们需要在 Presenter 初始化的时候传入 View，Presenter 通过 Model 获取数据，并在拿到数据的时候，通过 View 的方法通知给 View 层。

这时，我们的 Activity 可以从之前的 Controller 的角色中解放出来，成为下面的样子：

```kotlin
class LoginActivity : AppCompatActivity(), LoginView {
    var presenter: LoginContract.IPresenter = LoginPresenter(this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        btnLogin.setOnClickListener {
            presenter.requestUserInfo(etPhone.editableText.toString(),
                    etPassword.editableText.toString())
        }
    }

    override fun onSuccess(userInfo: UserInfo?) {
        Snackbar.make(tvUserInfo, "UserInfo: uid=${userInfo?.uid}, " +
                "name=${userInfo?.name}, " +
                "phone=${userInfo?.phone}", Snackbar.LENGTH_SHORT)
    }

    override fun onError(msg: String?) {
        Snackbar.make(tvUserInfo, "Error: ${msg}", Snackbar.LENGTH_SHORT)
    }
}
```

实际在 View 中也要维护一个 Presenter 的实例。 当需要请求数据的时候会使用该实例的方法来请求数据，所以，在开发的时候，我们需要根据请求数据的情况，在 Presenter 中定义接口方法。也就是说，MVP 的原理就是 **View 通过 Presenter 获取数据，获取到数据之后再回调View的方法来展示数据**。

同样地，我们盘点一下 MVP 的优缺点。

### 优点

1. 降低耦合度，实现了 Model 和 View 真正的完全分离，可以修改 View 而不影响 Model；
2. 模块职责划分明显，层次清晰；
3. 隐藏数据；
4. Presenter 可以复用，一个 Presenter 可以用于多个 View，而不需要更改 Presenter 的逻辑；
5. 利于测试驱动开发，以前的 Android 开发是难以进行单元测试的；
6. View 可以进行组件化，在 MVP 当中，View 不依赖 Model。

### 缺点

1. Presenter 中除了应用逻辑以外，还有大量的 View → Model，Model → View 的手动同步逻辑，造成 Presenter 比较笨重，维护起来会比较困难；
2. 由于对视图的渲染放在了 Presenter 中，所以视图和 Presenter 的交互会过于频繁；
3. 如果 Presenter 过多地渲染了视图，往往会使得它与特定的视图的联系过于紧密，一旦视图需要变更，那么 Presenter 也需要变更了。

## MVC 与 MVP 的对比

1. MVC 中是允许 Model 和 View 进行交互的，而 MVP 中，Model 与 View 之间的交互由 Presenter 完成；
2. MVP 模式就是将 P 定义成一个接口，然后在每个触发的事件中调用接口的方法来处理，也就是将逻辑放进了 P 中，需要执行某些操作的时候调用 P 的方法就行了。

## MVVM

MVVM（Model-View-ViewModel）模式本质上也是 MVC 的改进版，它会将 View 的状态以及行为进行**抽象化**，从而让我们把业务与视图分开。

- Model
  负责从各种数据源中获取数据；

- View
  在 Android 中对应于 Activity 和 Fragment，用于展示给用户和处理用户交互，会驱动 ViewModel 从 Model 中获取数据；

- ViewModel
  用于将 Model 和 View 进行关联，我们可以在 View 中通过 ViewModel 从 Model 中获取数据；当获取到了数据之后，会通过自动绑定，比如 DataBinding，来将结果自动刷新到界面上。

乍看之下，MVVM 模式与 MVP 模式很相似，因为它们都很好地完成了对 View 层状态和行为的抽象化。在 MVP 中，Presenter 抽象了一个独立于特定 UI 的 View，而在 MVVM 模式中，它简化了 UI 事件驱动的编程方式。

如果 MVP 模式意味着 Presenter 直接告诉 View 要显示的内容，则在 MVVM 中，ViewModel 公开 View **可以绑定的事件流**（Stream of Events）。由此一来，ViewModel **不再需要像 Presenter 一样持有对 View 的引用**。这也意味着 MVP 中所需要的那些接口，都可以扔掉了。

View 还会通知 ViewModel 有关不同的操作。因此，MVVM 模式支持 View 和 ViewModel 之间的**双向数据绑定**，并且 View 和 ViewModel 之间存在多对一关系。View 引用了 ViewModel，但是 ViewModel 没有有关 View 的信息。数据的使用者应该了解生产者，但是生产者——也就是ViewModel——不知道，也不在乎谁使用数据。

它的模型如下图所示：

![MVVM 结构](http://blog.cigis-cloud.com/design-pattern-1597991055.png)

使用 Google 官方的 Android Architecture Components ，可以很轻松地将 MVVM 模式应用到我们的代码中。下面，我们就使用它来展示一下 MVVM 的实际的应用。