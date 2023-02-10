---
title: ViewRootImpl 解析
link: viewrootimpl
cover: https://www.cherokeemfg.com/wp-content/uploads/2017/04/tree-root-questions.jpg
toc: true
category: 
    - Development
    - Android
tag:
    - Android
    - View
    - Android Framework
---

之前反复提到过的DecorView不是整个View树的根吗？怎么又出来一个看起来像是『根』的东西？

我们看看ViewRootImpl的代码，就能明白Android为什么要添加两个『根』在树上。

<!-- more -->

我们先看看`requestLayout()`方法究竟在做什么：

```java
// ViewRoomImpl.java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();  // 检查操作线程是否为创建线程
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void scheduleTraversals() {

    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

好，看到一个Android中一个比较重要的方法`performTraversals()`，实现了View树的『从根到叶』的遍历。ViewRootImpl中接收的各种变化，如来自WMS的窗口属性变化、来自控件树的尺寸变化以及重绘请求等都引发`performTraversals()`的调用，并在其中完成处理。View类及其子类的`onMeasure()`、`onLayout()`、`onDraw()`等回调也都是在该方法执行的过程中直接或间接的引发。该函数可谓是是ViewRootImpl的『心跳』。我们就来看一下这个方法。

```java
private void performTraversals() {
    ...
    boolean windowSizeMayChange = false;
    boolean surfaceChanged = false;
    WindowManager.LayoutParams lp = mWindowAttributes;

    // I. pre-measure阶段

    // 这两个变量就是DecorViewSPEC_SIZE的候选
    int desiredWindowWidth;
    int desiredWindowHeight;
    ...
    Rect frame = mWinFrame;
    // 是否是『第一次遍历』
    // 在ViewRootImpl的生命周期里，会有很多次的『遍历』，第一次的 desiredWindowWidth / desiredWindowHeight
    // 与之后的 desiredWindowWidth / desiredWindowHeight 的测量方式是不一样的
    if (mFirst) {
        mFullRedrawNeeded = true;
        mLayoutRequested = true;

        final Configuration config = mContext.getResources().getConfiguration();
        if (shouldUseDisplaySize(lp)) {
            // NOTE -- system code, won't try to do compat mode.
            Point size = new Point();
            mDisplay.getRealSize(size);
            desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else {
            // 第1次“遍历”的测量，采用了应用可以使用的最大尺寸作为SPEC_SIZE的候选
            desiredWindowWidth = mWinFrame.width();
            desiredWindowHeight = mWinFrame.height();
        }
        ...
    } else {
        // 在非第1次遍历的情况下，会采用窗口的最新尺寸作为SPEC_SIZE的候选
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        //如果窗口的最新尺寸与ViewRootImpl中的现有尺寸不同，说明WMS单方面改变了窗口的尺寸，将导致下面三个结果
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
        //需要完整的重绘以适应新的窗口尺寸
        mFullRedrawNeeded = true;
        //需要对控件树重新布局
        mLayoutRequested = true;
        //控件树可能拒绝接受新的窗口尺寸，可能需要窗口在布局阶段尝试设置新的窗口尺寸。只是尝试嘛，尝试而已
        windowSizeMayChange = true;
        }
    }
    ...
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {

        final Resources res = mView.getContext().getResources();

        if (mFirst) {
            ...
        } else {
            // 检查WMS是否单方面改变了一些参数，标记下来，然后作为之后是否进行控件布局的条件之一
            // 如果窗口的width或height被指定为WRAP_CONTENT时。表示该窗口为悬浮窗口
            if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                    || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                // 标记一下，WMS单方面改变了一些参数
                windowSizeMayChange = true;

                if (shouldUseDisplaySize(lp)) {
                    // NOTE -- system code, won't try to do compat mode.
                    Point size = new Point();
                    mDisplay.getRealSize(size);
                    desiredWindowWidth = size.x;
                    desiredWindowHeight = size.y;
                } else {
                    Configuration config = res.getConfiguration();
                    desiredWindowWidth = dipToPx(config.screenWidthDp);
                    desiredWindowHeight = dipToPx(config.screenHeightDp);
                }
            }
        }

        // 测量
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }
    ...
    if (layoutRequested) {
        // 如果后面还有layout的请求的话，用这个FLAG来标记，然后再重新来一次layout
        // 所以现在先置为false
        mLayoutRequested = false;
    }

    boolean windowShouldResize = layoutRequested && windowSizeMayChange
        && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
            || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.width() < desiredWindowWidth && frame.width() != mWidth)
            || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.height() < desiredWindowHeight && frame.height() != mHeight));
    windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;
    ...
    // I. pre-measure阶段结束

    // II. 窗口layout阶段
    if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        ...
        boolean hadSurface = mSurface.isValid();

        try {
            ...
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
            ...
        } catch (RemoteException e) {
        }

        ...
        // II. 窗口layout阶段结束

        // III. measure阶段

        if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                    (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                    updatedConfiguration) {
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!  mWidth="
                        + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                        + " mHeight=" + mHeight
                        + " measuredHeight=" + host.getMeasuredHeight()
                        + " coveredInsetsChanged=" + contentInsetsChanged);

                // 这里其实就是调用了View.measure()方法
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                // Implementation of weights from WindowManager.LayoutParams
                // We just grow the dimensions as needed and re-measure if
                // needs be
                int width = host.getMeasuredWidth();
                int height = host.getMeasuredHeight();
                boolean measureAgain = false;

                if (lp.horizontalWeight > 0.0f) {
                    width += (int) ((mWidth - width) * lp.horizontalWeight);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                if (lp.verticalWeight > 0.0f) {
                    height += (int) ((mHeight - height) * lp.verticalWeight);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }

                if (measureAgain) {
                    if (DEBUG_LAYOUT) Log.v(mTag,
                            "And hey let's measure once more: width=" + width
                            + " height=" + height);
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }

                layoutRequested = true;
            }
        }
    } else {
        // Not the first pass and no window/insets/visibility change but the window
        // may have moved and we need check that and if so to update the left and right
        // in the attach info. We translate only the window frame since on window move
        // the window manager tells us only for the new frame but the insets are the
        // same and we do not want to translate them more than once.
        maybeHandleWindowMove(frame);
    }

    if (surfaceSizeChanged) {
        updateBoundsSurface();
    }

    // III. measure阶段结束

    // IV. layout阶段开始

    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    ...
    if (didLayout) {
        // 进行layout过程
        performLayout(lp, mWidth, mHeight);

        // By this point all views have been sized and positioned
        // We can compute the transparent area

        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            // start out transparent
            // TODO: AVOID THAT CALL BY CACHING THE RESULT?
            host.getLocationInWindow(mTmpLocation);
            mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                    mTmpLocation[0] + host.mRight - host.mLeft,
                    mTmpLocation[1] + host.mBottom - host.mTop);

            host.gatherTransparentRegion(mTransparentRegion);
            if (mTranslator != null) {
                mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
            }

            if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                mPreviousTransparentRegion.set(mTransparentRegion);
                mFullRedrawNeeded = true;
                // reconfigure window manager
                try {
                    mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
                } catch (RemoteException e) {
                }
            }
        }

        if (DBG) {
            System.out.println("======================================");
            System.out.println("performTraversals -- after setFrame");
            host.debug();
        }
    }

    // IV. layout阶段结束

    // V. draw过程开始

    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

    if (!cancelDraw) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }

        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }

    mIsInTraversal = false;

    // V. draw过程结束
}
```

好吧，我承认，这是我见过的最长的方法了。只能给它分为几个重要阶段，并提炼一下，无法进行太详细的分析了。

I. pre-measure阶段

这是第一个阶段，它会对控件树进行第一次测量。在此阶段中将会计算出控件树为显示其内容所需的尺寸，即期望的窗口尺寸。在这个阶段中View及其子类的`onMeasure()`方法将会沿着控件树依次得到回调。

预测量也是一次完整的测量过程，它与最终测量的区别仅在于参数不同而已。实际的测量工作是在View或其子类的`onMeasure()`方法中完成，并且其测量结果需要受限于来自其父控件的指示。这个指示由`onMeasure()`方法中的两个参数进行传达：`widthSpec`和`heightSpec`。它们是被称为MeasureSpec的复合整型变量，用于指导控件对自身进行测量。它又两个分量，结构如下表：

<table class="table is-bordered is-narrow">
    <thead>
        <tr>
            <th><abbr>32, 31</abbr></th>
            <th><abbr>30 ... 1</abbr></th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>SPEC_MODE</td>
            <td>SPEC_SIZE</td>
        </tr>
    </tbody>
</table>

其实就是个32位的整型，高2位用于储存它的类型，后30位用于储存它的内容。

I、II、III三个阶段可知预测量时的SPEC_SIZE按照如下原则进行取值：

- 第一次“遍历”时，使用可用的最大尺寸作为SPEC_SIZE的候选
- 此窗口是一个悬浮窗口时，即LayoutParams.width/height其中之一被指定为WRAP_CONTENT时，使用可用的最大尺寸作为SPEC_SIZE的候选
- 其他情况下，使用窗口最新尺寸作为SPEC_SIZE的候选