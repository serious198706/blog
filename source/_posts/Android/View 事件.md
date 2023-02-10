---
title: View 事件分发机制
link: view-event-dispatch
toc: true
categor: 
    - Development
    - Android
tag:
    - Android
    - View
    - Android Framework
---

这篇文章来梳理一下 Android 最令人头疼的基本元素——事件机制。

<!-- more -->

## 事件序列
当用户点击屏幕里View或者ViewGroup的时候，将会产生一个事件对象，这个事件对象就是MotionEvent对象，这个对象记录了事件的类型，触摸的位置，以及触摸的时间等。MotionEvent里面定义了事件的类型，其实很容易理解，因为用户可以在屏幕触摸，滑动，离开屏幕动作，分别对应：

- MotionEvent.ACTION_DOWN：用户触摸View & ViewGroup
- MotionEvent.ACTION_MOVE：用户手指移动View & ViewGroup
- MotionEvent.ACTION_UP：用户手指离开屏幕
- MotionEvent.ACTION_CANCEL：事件退出了，不是用户导致的

因此用户在触摸屏幕到离开屏幕会产生一系列事件：
```
ACTION_DOWN -> ACTION_MOVE(0个或者多个) -> ACTION_UP
```

如下图：
![事件序列](https://s3.ax1x.com/2020/12/09/rCptjP.png)

## 事件传递的顺序
事件传递的顺序如下：

```
InputManageSystem -> WindowInputEventReceiver(ViewRootImpl) -> DecorView -> Activity -> DecorView -> 子View
```

如果所有的 View 都没有消耗事件，那最后事件会传回到 Activity，由 Activity 处理（也即 Activity 的`onTouchEvent()`方法被调用）。

```
子View --> DecorView --> Activity
```

由 DecorView 向下，事件的传递还算清晰，没有太多的疑问。那最关键的是，触摸事件是怎么产生并交给 DecorView 的呢？

我们知道，触摸屏幕，首先肯定是硬件产生的一个电信号，但是我们能接触到的触摸事件直接就到了 MotionEvent，那么这个 MotionEvent 在哪里产生？

其实这个过程是在 Framework 层做的处理。

屏幕对应 Android 来说，担任了**键盘**的作用，就是我们计算机组成的输入设备。我们知道 Android 是基于 Linux 系统的，当我们的输入设备可用时（对当前的例子来说——触摸屏），我们对触摸屏进行操作，Linux 就会收到相应的**硬件中断**，然后将中断加工成原始的**输入事件**并写入相应的**设备节点**中。而 Android 的输入系统所做的事情概括起来说就是监控这些设备节点，当某个设备节点有数据可读时，将数据读出并进行一系列的翻译加工，然后在所有的窗口中找到合适的事件接收者，并派发给它。这里所说的 Android 输入系统，就是 InputManagerService（IMS），它和我们熟知的 ActivityManagerService（AMS）一样，作为系统服务，都是在 ServiceServer 中被创建。

根据 [Activity、Window 和 View 之间的关系](/activity/)，我们知道，Activity 在创建时，会创建对应的 PhoneWindow，创建完成之后，会同时创建 ViewRootImpl 实例，在这个实例的 `setView()` 方法中，会将该 Window 与 IMS 进行关联，关联的方式是创建 InputChannel 实例并通过 Binder 通信的方式把 PhoneWindow 和 InputChannel 一起扔给 IMS，IMS 去进行注册。待有事件发生，就会有所回调，而回调的一个关键类是下面即将会提到的 WindowInputEventReceiver。

## 代码分析

WindowInputEventReceiver 是 ViewRootImpl 中的一个子类。它的其中一个功能是搭建 DecorView 与 IMS 之间的桥梁。 在 `ViewRootImpl.setView()` 中，有如下代码：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView, int userId) {
    ...
    // Schedule the first layout -before- adding to the window
    // manager, to make sure we do the relayout before receiving
    // any other events from the system.
    requestLayout();                // 1
    InputChannel inputChannel = null;
    if ((mWindowAttributes.inputFeatures
            & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
        inputChannel = new InputChannel();          // 2
    }
    ...
    try {
        ...
        res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mDisplayCutout, inputChannel,
                            mTempInsets, mTempControls);        // 3
        ...
    }
    ...
    if (inputChannel != null) {
        ...
        mInputEventReceiver = new WindowInputEventReceiver(inputChannel, Looper.myLooper());  // 4
    }
}
```

1. 准备开始进行 layout 阶段；
2. 创建 InputChannel 实例，注意，此时并未初始化该实例；
3. 将 PhoneWindow 实例和 InputChannel 实例都通过 Binder 扔给 Session（mWindowSession 是 IWindowSession 类型的，其对应的实现为 Session 类，位于 com/android/server/wm/Session.java），Session 通过一系列操作将 PhoneWindow 与 InputChannel 绑定。值得一提的是，在绑定之前，才会利用 `InputChannel.openChannelPair(name)` 方法创建两个 InputChannel，一个是 Server 端，用于发布事件；一个是 Client 端，用于消耗事件，然后将新创建的 Client 端直接 `transferTo()` 到步骤 2 中创建的实例，从而完成其初始化的操作。
4. 注册 WindowInputEventReceiver，当 InputChannel 有消息过来的时候，立刻回调它的 `onInputEvent()` 方法。

需要注意的是，在创建 WindowInputEventReceiver 实例的时候，也给它了一个主线程的 Looper，这意味着，事件的接收和后续分发，都是在主线程上进行的，这也是为什么触摸事件不能做太复杂的操作的原因——会卡住主线程。

由 WindowInputEventReceiver 的`onInputEvent()`方法层层向下走，我们最终可以找到 ViewRootImpl.java 中，有这么一个方法：

```java
// ViewRootImpl.java

View mView;

private int processPointerEvent(QueuedInputEvent q) {
    final MotionEvent event = (MotionEvent)q.mEvent;
    ... 
    boolean handled = mView.dispatchPointerEvent(event);
    ...
    return handled ? FINISH_HANDLED : FORWARD;
}
```

此处的`mView`，是在`ViewRootImpl.setView()`中赋值的，这个方法做了很多的重要工作，我会在[关于 ViewRootImpl](/viewrootimpl/)一文中进行详解。这个`setView()`方法，是在`WindowManagerGlobal.addView()`中被调用的。而`WindowManagerGlobal.addView()`又是在`WindowManagerImpl.addView()`中被调用的，这个方法，又是在`Activity.makeVisible()`方法中调用的，于是我们找到了上面代码中`mView`的出处：

```java
// android.app.Activity.java

View mDecor = null;

void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

原来就是 DecorView 啊。到此，事件终于稳稳地交给了我们的 DecorView。

接下来在 DecorView.java 中找找`dispatchPointerEvent()`。WTF？没找到？表着急，那肯定在 ViewGroup.java 里。WTF？没找到？表着急，那肯定在 View.java 里。

```java
// View.java
@UnsupportedAppUsage
public final boolean dispatchPointerEvent(MotionEvent event) {
    if (event.isTouchEvent()) {
        return dispatchTouchEvent(event);
    } else {
        return dispatchGenericMotionEvent(event);
    }
}
```

由此引出下面的部分。

## 三个重要的方法

事件分发中有3个比较重要的入口方法，分别是：

- `public boolean dispatchTouchEvent(MotionEvent event)`
- `public boolean onTouchEvent(MotionEvent event)`
- `public boolean onInterceptTouchEvent(MotionEvent ev)`。

先来看看这几个方法的调用关系：
*![事件分发机制图](/img/15.png)*

上图中绿色线条表示默认的事件处理流程，即我们没有做任何处理，事件会按照绿色线条所示的方向由Activity -> ...ViewGroup... -> View -> ...ViewGroup... -> Activity 这个U型图进行传递。即一直默认调用`super.xxx`方法。

黑色线条表示默认Activity ->...ViewGroup... -> View ->...ViewGroup... -> Activity 这个U型图的任一节点中（不包括`onInterceptTouchEvent`）返回了true，事件即结束，不再向下一节点传递。

红色线条表示一些特殊情况，尤其是ViewGroup，`ViewGroup.onInterceptTouchEvent()`表示询问当前ViewGroup是否需要拦截此事件即要不要处理，为什么要『多此一举』呢，因为`ViewGroup.dispatchTouchEvent()`这个函数的特殊，从上图可知，该函数返回true，是消费事件，返回false是交由上一级的ViewGroup或者Activity的`onTouchEvent()`。那么它怎么向下传递事件或者想把事件交给自己的`onTouchEvent`处理呢，所以ViewGroup多了个`onInterceptTouchEvent`（View是没有该函数的），`onInterceptTouchEvent`起到作用的是分流。`onInterceptTouchEvent`返回`false`或者返回`super.xxx`是向下级View或者ViewGroup传递，返回`true`呢是把事件交给自己的onTouchEvent处理。

接上节，刚才事件交给了 DecorView，但是无奈最后还是在 View 中找到 `dispatchPointerEvent()` 方法，我们看看这个事件是怎么个走向：

### `dispatchTouchEvent`

`dispatchTouchEvent`方法用来进行事件的分发。事件传递到当前 View/ViewGroup 时，这个方法就会被调用。`dispatchTouchEvent`方法里面包含了具体的事件分发逻辑，返回结果受当前 View 的`onTouchEvent`方法和下级 View 的`dispatchTouchEvent`方法的影响。

因为 DecorView 继承自 ViewGroup，所以我们先看ViewGroup是如何通过`dispatchTouchEvent`分发事件的：

```java
// ViewGroup.java
 @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev); // 判断是否要拦截事件
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;

            // 在这里寻找合适的子view并派发事件
            if (!canceled && !intercepted) {

                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```

简单总结，就是：

1. 先判断是否要拦截（通过`onInterceptTouchEvent()`方法），如果不拦截，那就寻找合适的子view进行事件的派发，也既调用子view的`dispatchTouchEvent()`。如果有任何一个子view消费了此次事件，即`dispatchTouchEvent()`返回了`true`，那派发中止，并返回true，表示事件已经被消费。如果没有任何一个子view消费此次事件，即`dispatchTouchEvent()`返回了`false`，那ViewGroup就自己处理此次事件，调用父类View的`onTouchEvent()`。如果最后还未消耗，那就会一直向上返回，直到Activity的`onTouchEvent()`。
2. 如果要拦截，那就会直接调用自己父类View的`onTouchEvent()`。

上一部分的代码里我们也也看到，`dispatchPointerEvent`直接调用了`dispatchTouchEvent(event)`，而DecorView又重写了这个方法。

所以我们看看`DecorView.dispatchTouchEvent()`做了些啥：

```java
// DecorView.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

Callback来自于Window.java：
```java
// Window.java
public final Callback getCallback() {
    return mCallback;
}
```


```java
// Callback.java:
public interface Callback {
    ...
    public boolean dispatchTouchEvent(MotionEvent event);
    ...
}
```

那既然有`getCallback()`就肯定有`setCallback()`，我们找一下。
![Callback调用者](/img/14.png)

在`Activity.attach()`方法中找到了`window.setCallback(this)`：

```java
// Activity.java
@UnsupportedAppUsage
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    ...
    mWindow.setCallback(this);  
    ...
}
```

所以说，MotionEvent最终又是传递到了Activity，`cb.dispatchTouchEvent(ev)`也就是相当于`Activity.dispatchTouchEvent()`。

那么来看看Activity是怎么处理的：

```java
    // Activty.java
    /**
     * 被调用来处理触摸屏幕的事件。
     * 你可以覆写此方法以在事件分发到window之前拦截所有的事件。
     * 不过要确保调用此处的实现保证触摸事件都被正常处理。 
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

Activity又调用了`getWindow().superDispatchTouchEvent(ev)`也就是PhoneWindow。

```java
// PhoneWindow.java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {    
    // 兜兜转转一大圈，还是把事件交给我们的DecorView，    
    // DecorView继承自FrameLayout，FrameLayout呢又继承自ViewGroup，    
    // 所以作为一个ViewGroup，DecorView继续向其子View派发事件，其流程在文章的开头就已经给了    
    return mDecor.superDispatchTouchEvent(event);
}
```

这里又调用了DecorView的superDispatchTouchEvent(event)：

```java
// DecorView.java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

这里就是直接继承自ViewGroup的`dispatchTouchEvent(MotionEvent ev)`方法，也就是说，事件从 DecorView 传递到Activity，最终又回到 DecorView，最后按照分发机制分发到ViewGroup再到所有的子View。

兜兜转转一大圈我们神经都被绕弯了:joy: 。在这里总结一下，当我们触摸（点击）屏幕时，Android输入系统InputManagerService通过对事件的加工处理再找到合适的 Window 接收者并通过 InputChannel 向 Window 派发加工后的事件，并触发 WindowInputEventReceiver 的`onInputEvent`的调用，由此产生后面一系列的调用，把事件派发给整个控件树的根 DecorView 。而 DecorView 又上演了一出偷梁换柱的把戏，先把事件交给 Activity 处理，在 Activity 中又把事件交还给了我们的 DecorView 。自此沿着控件树自上向下依次派发事件。如果最后没有任何 View 处理此次点击事件，则 Activity 再来做最后的处理。

那就继续看看View的事件分发：

```java
// View.java
/**
* 将触摸屏幕的手势事件向下传递给目标 view，如果当前 view 是目标 view的话，就传递给当前 view。
*
* @param event The motion event to be dispatched.
* @return 如果事件被处理（消耗），就返回 true，否则返回 false。
*/
public boolean dispatchTouchEvent(MotionEvent event) {
    // If the event should be handled by accessibility focus first.
    if (event.isTargetAccessibilityFocus()) {
        // We don't have focus or no virtual descendant has it, do not handle the event.
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
    }

    boolean result = false;

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Defensive cleanup for new gesture
        stopNestedScroll();
    }

    // 这里决定了该 view 是否要消耗此次事件。
    //
    // 1. 如果该 view 有`TouchListener`的话，交给它的派生类的 `onTouch` 实现去决定是否要消耗事件。
    // 2. 如果没有 `TouchListener`，那么就看 view 自己处理 touchEvent 事件之后，是否要消耗掉该事件。 
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

### `onTouchEvent`

该方法在`View.dispatchTouchEvent()`方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。例如一个`Button`设置为 `android:enabled="false"`，那从 ACTION_DOWN 开始，就不消耗该事件，那么后面的 ACTION_MOVE、ACTION_UP等事件也就不会再接收到了。这是由一个叫`mIgnoreNextUpEvent`的变量控制的。

还是来看看它的代码：

```java
// View.java
/**
    * 实现这个方法来处理屏幕触摸手势事件。
    * 
    * 如果这个方法被用来检测点击动作，那你最好使用 performClick() 方法来触发此次点击事件。
    * 这样做能保证它是一个高度一致性的系统行为，包括：
    *
    * · 遵守点击音效的设置
    * · 分发 OnClickListener
    * · 当启用 accessibity 时，正确处理 AccessibilityNodeInfo 中的 ACTION_CLICK 事件
    *
    * @param event The motion event.
    * @return 如果事件被处理（消耗），就返回 true，否则返回 false。
    */
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return clickable;
    }
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                    handleTooltipUp();
                }
                if (!clickable) {
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
                }
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true, x, y);
                    }

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClickInternal();
                            }
                        }
                    }

                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }

                    removeTapCallback();
                }
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                }
                mHasPerformedLongPress = false;

                if (!clickable) {
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    break;
                }

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true, x, y);
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                if (clickable) {
                    setPressed(false);
                }
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                break;

            case MotionEvent.ACTION_MOVE:
                if (clickable) {
                    drawableHotspotChanged(x, y);
                }

                final int motionClassification = event.getClassification();
                final boolean ambiguousGesture =
                        motionClassification == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE;
                int touchSlop = mTouchSlop;
                if (ambiguousGesture && hasPendingLongPressCallback()) {
                    final float ambiguousMultiplier =
                            ViewConfiguration.getAmbiguousGestureMultiplier();
                    if (!pointInView(x, y, touchSlop)) {
                        // The default action here is to cancel long press. But instead, we
                        // just extend the timeout here, in case the classification
                        // stays ambiguous.
                        removeLongPressCallback();
                        long delay = (long) (ViewConfiguration.getLongPressTimeout()
                                * ambiguousMultiplier);
                        // Subtract the time already spent
                        delay -= event.getEventTime() - event.getDownTime();
                        checkForLongClick(
                                delay,
                                x,
                                y,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    }
                    touchSlop *= ambiguousMultiplier;
                }

                // Be lenient about moving outside of buttons
                if (!pointInView(x, y, touchSlop)) {
                    // Outside button
                    // Remove any future long press/tap checks
                    removeTapCallback();
                    removeLongPressCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        setPressed(false);
                    }
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                }

                final boolean deepPress =
                        motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
                if (deepPress && hasPendingLongPressCallback()) {
                    // process the long click action immediately
                    removeLongPressCallback();
                    checkForLongClick(
                            0 /* send immediately */,
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
                }

                break;
        }

        return true;
    }

    return false;
}
```

### `onInterceptTouchEvent`

`onInterceptTouchEvent()`方法在`ViewGroup.dispatchTouchEvent()`方法内部被调用，用来判断是否拦截某个事件。如果当前ViewGroup拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。这个方法只有ViewGroup中有，View中没有。

```java
/**
    * 实现该方法来拦截所有的触摸事件。这能让你监测所有发送到子view的事件，并且得到该事件的所有信息。
    *
    * 使用该方法的时候要小心，因为它与View.onTouchEvent()有着相当复杂的交互。
    * 事件会按照下面的顺序被接收：
    *
    * · 按下事件(ACTION_DOWN)
    * · 按下事件要么被该ViewGroup的某个子view处理，要么就进到你自己的onTouchEvent()方法去处理；
    *   这意味着你在覆写onTouchEvent()时要返回 true，才能接收到接下来其他的事件（而不是由父view来接收处理)。
    *   这个方法返回 true时，你不会接收到来自 onInterceptTouchEvent() 的任何事件，
    *   并且接下来的触摸事件就会正常发生。
    * · 只要这个方法返回了 false，接下来的事件（直到ACTION_UP并包括ACTION_UP事件）都会先经过
    *   onInterceptTouchEvent，再被发送到目标view的 onTouchEvent。
    * · 如果这儿返回了 true，后面就不会接收到相同的事件了（除了ACTION_CANCEL），后续的事件将被传递到你自己的 onTouchEvent()
    *   并且不会再经过这儿。子view将不会再接收到任何事件。
    *
    * @param ev The motion event being dispatched down the hierarchy.
    * @return Return true to steal motion events from the children and have
    * them dispatched to this ViewGroup through onTouchEvent().
    * The current target will receive an ACTION_CANCEL event, and no further
    * messages will be delivered here.
    */
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
            && ev.getAction() == MotionEvent.ACTION_DOWN
            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
            && isOnScrollbarThumb(ev.getX(), ev.getY())) {
        return true;
    }
    return false;
}

```

`onInterceptTouchEvent`起到作用的是分流。`onInterceptTouchEvent`返回`false`或者返回`super.xxx`是向下级View或者ViewGroup传递，返回`true`呢是把事件交给自己的`onTouchEvent`处理。

ViewGroup默认不拦截任何事件。

到此，关于事件分发的机制，就差不多了。

## 其他几个需要注意的点

### requestDisallowInterceptTouchEvent方法
`requestDisallowInterceptTouchEvent`方法用于影响父元素的事件拦截策略，`requestDisallowInterceptTouchEvent(true)`，表示不允许父元素拦截事件，这样事件就会传递给子View。一般这个方法子View用的多，可以用来处理滑动冲突问题。

### onTouchListener、onTouchEvent、onClickListener的优先级
1. `onTouchListener`和`onTouchEvent`都在`dispatchTouchEvent`方法中被调用，`onClickListener`在`onTouchEvent`方法中被调用；

2. `onTouchListener`的优先级高于`onTouchEvent`方法，如果`onTouchListener`的`onTouch`方法返回`true`，则`onTouchEvent`方法不会被调用，当然`onClickListener`就更不会被调用了；

3. 在`onTouchEvent`方法中，如果当前View设置了`onClickListener`，那么`onClickListener`的`onClick`方法会被调用；

4. 只要View的`clickable`和`long_clickable`有一个为`true`，View就会消耗当前事件，也就是说`onTouchEvent`方法最后会返回`true`。

5. View的`long_clickable`属性默认为`false`，而`clickable`属性和具体的View有关，可点击的View的`clickable`属性为`true`，不可点击的View的`clickable`属性为`false`。

