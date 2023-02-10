---
title: 关于 ListView 的渲染、缓存及优化
link: listview-optimization
cover: https://www.dev2qa.com/wp-content/uploads/2017/12/listview-execution-diagram.png
toc: true
tag: 
    - Android
    - ListView
    - 性能优化
category: 
    - Android
    - Development
---

ListView 优化是老生常谈的事情，虽然现在有 RycyclerView 可以完美替代 ListView，但是了解 ListView 的渲染、缓存及优化，也并不是什么坏事。

<!-- more -->

## ListView 与它的朋友们

ListView 继承自 AbsListView，AbsListView 继承自 AdapterView，AdapterView 继承自 ViewGroup，它们也有各自的派生类。它们的关系如下图所示：

![](/img/listview-1589441712.png)

AdapterView 是一个抽象类，它一个可以由内部元素来决定如何展现的 ViewGroup。它内部包含了对内部 View 的管理和基本事件处理，比如`addView()`、`removeView()`、`setOnItemClickListener()`等等。

AbsListView 也是一个抽象类，它是被用来展示大量数据的一个基类，而且它并没有指定展示的样式，完全由派生类来实现，比如网格、列表、瀑布流等，完全由子类来决定展示的样式。它负责处理内部 View 的渲染、缓存、回收等。

## AbsListView 

AbsListView 中有一个类，叫做 RecycleBin，意为回收站。我们看看它的注释：

> The RecycleBin facilitates reuse of views across layouts. The RecycleBin has two levels of storage: ActiveViews and ScrapViews. ActiveViews are those views which were onscreen at the start of a layout. By construction, they are displaying current information. At the end of layout, all views in ActiveViews are demoted to ScrapViews. ScrapViews are old views that could potentially be used by the adapter to avoid allocating views unnecessarily.

机翻一下：回收站机制可以实现 View 的重用。回收站有两个级别的存储：活动的 View （ActiveViews）和废弃的 View（ScrapViews）。ActiveViews 是指在 layout 过程的开始时显示在屏幕上的 View。刚开始构建时，它们会显示当前的信息，在 layout 结束时，所有的 ActiveViews 会被降级为 ScrapViews。**ScrapViews 是将来有可能会被 adater 重复利用的**，所以为了避免被不必要的 View 重建，将它们缓存起来。

我们来简单看看它的代码：

```java
class RecycleBin {
    @UnsupportedAppUsage
    private RecyclerListener mRecyclerListener;
    private int mFirstActivePosition;
    private View[] mActiveViews = new View[0];

    // 未排序的 View，可以被 adapter 当作 converter view 来使用
    // 这里是一个包含 ArrayList 的数组，不同的 ViewType 会存储到不同的 ArrayList 中
    private ArrayList<View>[] mScrapViews;
    private int mViewTypeCount;
    private ArrayList<View> mCurrentScrap;
    private ArrayList<View> mSkippedScrap;

    private SparseArray<View> mTransientStateViews;
    private LongSparseArray<View> mTransientStateViewsById;

    public void markChildrenDirty() {
        if (mViewTypeCount == 1) {
            final ArrayList<View> scrap = mCurrentScrap;
            final int scrapCount = scrap.size();
            for (int i = 0; i < scrapCount; i++) {
                scrap.get(i).forceLayout();
            }
        } else {
            final int typeCount = mViewTypeCount;
            for (int i = 0; i < typeCount; i++) {
                final ArrayList<View> scrap = mScrapViews[i];
                final int scrapCount = scrap.size();
                for (int j = 0; j < scrapCount; j++) {
                    scrap.get(j).forceLayout();
                }
            }
        }
        if (mTransientStateViews != null) {
            final int count = mTransientStateViews.size();
            for (int i = 0; i < count; i++) {
                mTransientStateViews.valueAt(i).forceLayout();
            }
        }
        if (mTransientStateViewsById != null) {
            final int count = mTransientStateViewsById.size();
            for (int i = 0; i < count; i++) {
                mTransientStateViewsById.valueAt(i).forceLayout();
            }
        }
    }

    ...

    // 清除 ScrapViews
    @UnsupportedAppUsage
    void clear() {
        ...
    }

    // 将 AbsListView 中所有的 children 填充到 ActiveViews
    // firstActivePosition 指的是第一个可见 view 被添加时在 mActiveView 数组中的 position
    void fillActiveViews(int childCount, int firstActivePosition) {
        // 如果当前数组过小，则要新建一个数组
        if (mActiveViews.length < childCount) {
            mActiveViews = new View[childCount];
        }
        // 记录外部（由 AbsListView 的派生类，如 ListView）传入的 firstActivePosition
        mFirstActivePosition = firstActivePosition;

        // 我没弄明白这里为什么要再新建一个数组变量指向 mActiveViews 内存
        // 有明白的同学可以留言告诉我
        final View[] activeViews = mActiveViews;
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
            // 莫要将 footer 和 header view 添加到 ScrapViews 中
            if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                activeViews[i] = child;
                // 记录位置，setupChild() 方法就不会将它的状态重置了，具体看下面的 setupChild() 方法
                lp.scrappedFromPosition = firstActivePosition + i;
            }
        }
    }

    // 获取 ActiveView，某个 ActiveView 被获取后，就会从 mActiveViews 中被移除
    View getActiveView(int position) {
        int index = position - mFirstActivePosition;
        final View[] activeViews = mActiveViews;
        if (index >=0 && index < activeViews.length) {
            final View match = activeViews[index];
            activeViews[index] = null;
            return match;
        }
        return null;
    }

    // 获取正在转变状态的 View
    View getTransientStateView(int position) {
        if (mAdapter != null && mAdapterHasStableIds && mTransientStateViewsById != null) {
            long id = mAdapter.getItemId(position);
            View result = mTransientStateViewsById.get(id);
            mTransientStateViewsById.remove(id);
            return result;
        }
        if (mTransientStateViews != null) {
            final int index = mTransientStateViews.indexOfKey(position);
            if (index >= 0) {
                View result = mTransientStateViews.valueAt(index);
                mTransientStateViews.removeAt(index);
                return result;
            }
        }
        return null;
    }

    ...

    // 获取 ScrapView，这些 View 们是未排序的
    View getScrapView(int position) {
        // 根据位置获取 ViewType
        final int whichScrap = mAdapter.getItemViewType(position);
        if (whichScrap < 0) {
            return null;
        }
        // 如果只有一种 ViewType，那就直接获取
        if (mViewTypeCount == 1) {
            return retrieveFromScrap(mCurrentScrap, position);
        // 如果有多种 ViewType，那要从对应的 ScrapViews 列表中获取
        } else if (whichScrap < mScrapViews.length) {
            return retrieveFromScrap(mScrapViews[whichScrap], position);
        }
        return null;
    }

    // 如果列表中的数据没有变化或者 adapter 有稳定的 ID，变化状态的 view 将会保留它们的状态以便后面获取
    void addScrapView(View scrap, int position) {
        final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
        // 无法回收，连个 LayoutParams 都没有
        if (lp == null) {
            return;
        }

        lp.scrappedFromPosition = position;

        // 移除但不要将 header 或者 footer 废弃掉
        final int viewType = lp.viewType;
        if (!shouldRecycleViewType(viewType)) {
            // 无法回收。如果它不是 header 或者 footer 的话（它们有特殊的处理方式，应该被忽略），
            // 就给它跳过，等回头把它 detach 掉。
            if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                // 加入 detach 列表
                getSkippedScrap().add(scrap);
            }
            return;
        }

        // View 中的方法，用于将 View 中的 callback 等移除
        scrap.dispatchStartTemporaryDetach();


        notifyViewAccessibilityStateChangedIfNeeded(
                AccessibilityEvent.CONTENT_CHANGE_TYPE_SUBTREE);

        // 不要废弃掉瞬态的 View
        final boolean scrapHasTransientState = scrap.hasTransientState();
        if (scrapHasTransientState) {
            if (mAdapter != null && mAdapterHasStableIds) {
                // If the adapter has stable IDs, we can reuse the view for
                // the same data.
                if (mTransientStateViewsById == null) {
                    mTransientStateViewsById = new LongSparseArray<>();
                }
                mTransientStateViewsById.put(lp.itemId, scrap);
            } else if (!mDataChanged) {
                // If the data hasn't changed, we can reuse the views at
                // their old positions.
                if (mTransientStateViews == null) {
                    mTransientStateViews = new SparseArray<>();
                }
                mTransientStateViews.put(position, scrap);
            } else {
                // Otherwise, we'll have to remove the view and start over.
                clearScrapForRebind(scrap);
                getSkippedScrap().add(scrap);
            }
        } else {
            clearScrapForRebind(scrap);
            if (mViewTypeCount == 1) {
                mCurrentScrap.add(scrap);
            } else {
                mScrapViews[viewType].add(scrap);
            }

            if (mRecyclerListener != null) {
                mRecyclerListener.onMovedToScrapHeap(scrap);
            }
        }
    }

    private ArrayList<View> getSkippedScrap() {
        if (mSkippedScrap == null) {
            mSkippedScrap = new ArrayList<>();
        }
        return mSkippedScrap;
    }

    void removeSkippedScrap() {
        if (mSkippedScrap == null) {
            return;
        }
        final int count = mSkippedScrap.size();
        for (int i = 0; i < count; i++) {
            removeDetachedView(mSkippedScrap.get(i), false);
        }
        mSkippedScrap.clear();
    }

    // 将 mActiveViews 中剩余的 View 移到 mScrapViews 中
    void scrapActiveViews() {
        final View[] activeViews = mActiveViews;
        final boolean hasListener = mRecyclerListener != null;
        final boolean multipleScraps = mViewTypeCount > 1;

        ArrayList<View> scrapViews = mCurrentScrap;
        final int count = activeViews.length;
        for (int i = count - 1; i >= 0; i--) {
            final View victim = activeViews[i];
            if (victim != null) {
                final AbsListView.LayoutParams lp
                        = (AbsListView.LayoutParams) victim.getLayoutParams();
                final int whichScrap = lp.viewType;

                activeViews[i] = null;

                if (victim.hasTransientState()) {
                    // Store views with transient state for later use.
                    victim.dispatchStartTemporaryDetach();

                    if (mAdapter != null && mAdapterHasStableIds) {
                        if (mTransientStateViewsById == null) {
                            mTransientStateViewsById = new LongSparseArray<View>();
                        }
                        long id = mAdapter.getItemId(mFirstActivePosition + i);
                        mTransientStateViewsById.put(id, victim);
                    } else if (!mDataChanged) {
                        if (mTransientStateViews == null) {
                            mTransientStateViews = new SparseArray<View>();
                        }
                        mTransientStateViews.put(mFirstActivePosition + i, victim);
                    } else if (whichScrap != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                        // The data has changed, we can't keep this view.
                        removeDetachedView(victim, false);
                    }
                } else if (!shouldRecycleViewType(whichScrap)) {
                    // Discard non-recyclable views except headers/footers.
                    if (whichScrap != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                        removeDetachedView(victim, false);
                    }
                } else {
                    // Store everything else on the appropriate scrap heap.
                    if (multipleScraps) {
                        scrapViews = mScrapViews[whichScrap];
                    }

                    lp.scrappedFromPosition = mFirstActivePosition + i;
                    removeDetachedView(victim, false);
                    scrapViews.add(victim);

                    if (hasListener) {
                        mRecyclerListener.onMovedToScrapHeap(victim);
                    }
                }
            }
        }
        pruneScrapViews();
    }

    // layout 过程的最后，所有暂时被 detach 掉的 View 要么应该被重新 attach，或者完全被 detach。这个方法能保证在
    // scrap 列表中的所有 View 都被 detach 掉
    void fullyDetachScrapViews() {
        final int viewTypeCount = mViewTypeCount;
        final ArrayList<View>[] scrapViews = mScrapViews;
        for (int i = 0; i < viewTypeCount; ++i) {
            final ArrayList<View> scrapPile = scrapViews[i];
            for (int j = scrapPile.size() - 1; j >= 0; j--) {
                final View view = scrapPile.get(j);
                if (view.isTemporarilyDetached()) {
                    removeDetachedView(view, false);
                }
            }
        }
    }

    // 确保 scrap 列表的大小不要超过 active 列表的大小，因为有时 adapter 可能会不回收 view。
    // 同时移除所有缓存的已经不再拥有瞬态的瞬态 view
    private void pruneScrapViews() {
        final int maxViews = mActiveViews.length;
        final int viewTypeCount = mViewTypeCount;
        final ArrayList<View>[] scrapViews = mScrapViews;
        for (int i = 0; i < viewTypeCount; ++i) {
            final ArrayList<View> scrapPile = scrapViews[i];
            int size = scrapPile.size();
            while (size > maxViews) {
                scrapPile.remove(--size);
            }
        }

        final SparseArray<View> transViewsByPos = mTransientStateViews;
        if (transViewsByPos != null) {
            for (int i = 0; i < transViewsByPos.size(); i++) {
                final View v = transViewsByPos.valueAt(i);
                if (!v.hasTransientState()) {
                    removeDetachedView(v, false);
                    transViewsByPos.removeAt(i);
                    i--;
                }
            }
        }

        final LongSparseArray<View> transViewsById = mTransientStateViewsById;
        if (transViewsById != null) {
            for (int i = 0; i < transViewsById.size(); i++) {
                final View v = transViewsById.valueAt(i);
                if (!v.hasTransientState()) {
                    removeDetachedView(v, false);
                    transViewsById.removeAt(i);
                    i--;
                }
            }
        }
    }

    void reclaimScrapViews(List<View> views) {
        if (mViewTypeCount == 1) {
            views.addAll(mCurrentScrap);
        } else {
            final int viewTypeCount = mViewTypeCount;
            final ArrayList<View>[] scrapViews = mScrapViews;
            for (int i = 0; i < viewTypeCount; ++i) {
                final ArrayList<View> scrapPile = scrapViews[i];
                views.addAll(scrapPile);
            }
        }
    }

    /**
        * Updates the cache color hint of all known views.
        *
        * @param color The new cache color hint.
        */
    void setCacheColorHint(int color) {
        if (mViewTypeCount == 1) {
            final ArrayList<View> scrap = mCurrentScrap;
            final int scrapCount = scrap.size();
            for (int i = 0; i < scrapCount; i++) {
                scrap.get(i).setDrawingCacheBackgroundColor(color);
            }
        } else {
            final int typeCount = mViewTypeCount;
            for (int i = 0; i < typeCount; i++) {
                final ArrayList<View> scrap = mScrapViews[i];
                final int scrapCount = scrap.size();
                for (int j = 0; j < scrapCount; j++) {
                    scrap.get(j).setDrawingCacheBackgroundColor(color);
                }
            }
        }
        // Just in case this is called during a layout pass
        final View[] activeViews = mActiveViews;
        final int count = activeViews.length;
        for (int i = 0; i < count; ++i) {
            final View victim = activeViews[i];
            if (victim != null) {
                victim.setDrawingCacheBackgroundColor(color);
            }
        }
    }

    private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {
        final int size = scrapViews.size();
        if (size > 0) {
            // See if we still have a view for this position or ID.
            // Traverse backwards to find the most recently used scrap view
            for (int i = size - 1; i >= 0; i--) {
                final View view = scrapViews.get(i);
                final AbsListView.LayoutParams params =
                        (AbsListView.LayoutParams) view.getLayoutParams();

                if (mAdapterHasStableIds) {
                    final long id = mAdapter.getItemId(position);
                    if (id == params.itemId) {
                        return scrapViews.remove(i);
                    }
                } else if (params.scrappedFromPosition == position) {
                    final View scrap = scrapViews.remove(i);
                    clearScrapForRebind(scrap);
                    return scrap;
                }
            }
            final View scrap = scrapViews.remove(size - 1);
            clearScrapForRebind(scrap);
            return scrap;
        } else {
            return null;
        }
    }

    private void clearScrap(final ArrayList<View> scrap) {
        final int scrapCount = scrap.size();
        for (int j = 0; j < scrapCount; j++) {
            removeDetachedView(scrap.remove(scrapCount - 1 - j), false);
        }
    }

    private void clearScrapForRebind(View view) {
        view.clearAccessibilityFocus();
        view.setAccessibilityDelegate(null);
    }

    private void removeDetachedView(View child, boolean animate) {
        child.setAccessibilityDelegate(null);
        AbsListView.this.removeDetachedView(child, animate);
    }
} 
```

这个类中比较重要的方法就是对于 View 当前状态的管理方法，有下面几个：

- `fillActiveViews(int childCount, int firstActivePosition)`：这个方法会将 AbsListView 中所有的子 View 添加到 mActiveViews 数组中。`childCount`指子 View 的个数，`firstActivePosition`指第一个可见元素的 position。
- `getActiveView(int position)`：从`mActiveViews`中获取 View。当 View 被获取后，就会从`mActiveViews`中移除，再次访问就会返回`null`，这意味着 ActiveView 无法复用。
- `addScrapView(View view, int position)`：将 ScrapView 缓存到`mScrapViews`中。mScrapViews 是一个 ArrayList 数组，这个数组中按 ViewType 分类并存储，如果有3种 ViewType，数组中就会有3个 ArrayList。这些缓存都是无序的。
- `getScrapView(int position)`：从缓存中取出一个 View。

AbsListView 也会有自己的 measure、layout、draw 过程，但是 draw 过程在 AbsListView 中没有实现，在 ListView 中也没有实现，也即它的绘制全部交给子 View 来做，我们只看它的`onMeasure()`和`onLayout`。

### AbsListView 的`onMesaure()`

话不多说，我们直接看看它的代码：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // mSelector 是一个 Drawable，用来绘制 selector
    if (mSelector == null) {
        useDefaultSelector();
    }
    final Rect listPadding = mListPadding;
    listPadding.left = mSelectionLeftPadding + mPaddingLeft;
    listPadding.top = mSelectionTopPadding + mPaddingTop;
    listPadding.right = mSelectionRightPadding + mPaddingRight;
    listPadding.bottom = mSelectionBottomPadding + mPaddingBottom;

    // Check if our previous measured size was at a point where we should scroll later.
    if (mTranscriptMode == TRANSCRIPT_MODE_NORMAL) {
        final int childCount = getChildCount();
        final int listBottom = getHeight() - getPaddingBottom();
        final View lastChild = getChildAt(childCount - 1);
        final int lastBottom = lastChild != null ? lastChild.getBottom() : listBottom;
        mForceTranscriptScroll = mFirstPosition + childCount >= mLastHandledItemCount &&
                lastBottom <= listBottom;
    }
}
```

可见 measure 过程没什么特殊的，只是确定了一些 padding 值。

### AbsListView 的`onLayout()`

这个是 AbsListView 的重头戏。下面我们即将会讲到的 **ListView 中是没有`onLayout`方法的**，也就是说，它的 layout 过程，完全由父类 AbsListView 来实现。

```java
/**
    * Subclasses should NOT override this method but
    *  {@link #layoutChildren()} instead.
    */
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    super.onLayout(changed, l, t, r, b);

    mInLayout = true;

    final int childCount = getChildCount();

    // 如果 AbsListView 的大小或 position 发生了变化，则要强制所有的子 View 重新 layout
    if (changed) {
        for (int i = 0; i < childCount; i++) {
            getChildAt(i).forceLayout();
        }
        // 回收站将 ScrapViews 也全部执行 forceLayout
        mRecycler.markChildrenDirty();
    }

    // 让所有子 View 进行 layout 过程
    layoutChildren();

    mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

    // TODO: Move somewhere sane. This doesn't belong in onLayout().
    if (mFastScroll != null) {
        mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
    }
    mInLayout = false;
}
```

注释中我们看到：**子类不要覆写这个方法，应该覆写`layoutChildren()`方法**。待会我们会详解 ListView 对这个方法的覆写。

## ListView

ListView 继承了 AbsListView，并实现了自己的 measure、layout 过程

### ListView 的`onMesaure()`

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // Sets up mListPadding
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    int childWidth = 0;
    int childHeight = 0;
    int childState = 0;

    mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
    if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED
            || heightMode == MeasureSpec.UNSPECIFIED)) {
        final View child = obtainView(0, mIsScrap);

        // Lay out child directly against the parent measure spec so that
        // we can obtain exected minimum width and height.
        measureScrapChild(child, 0, widthMeasureSpec, heightSize);

        childWidth = child.getMeasuredWidth();
        childHeight = child.getMeasuredHeight();
        childState = combineMeasuredStates(childState, child.getMeasuredState());

        if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
                ((LayoutParams) child.getLayoutParams()).viewType)) {
            mRecycler.addScrapView(child, 0);
        }
    }

    if (widthMode == MeasureSpec.UNSPECIFIED) {
        widthSize = mListPadding.left + mListPadding.right + childWidth +
                getVerticalScrollbarWidth();
    } else {
        widthSize |= (childState & MEASURED_STATE_MASK);
    }

    if (heightMode == MeasureSpec.UNSPECIFIED) {
        heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                getVerticalFadingEdgeLength() * 2;
    }

    if (heightMode == MeasureSpec.AT_MOST) {
        // TODO: after first layout we should maybe start at the first visible position, not 0
        heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
    }

    // 调用 View 的 setMeasuredDimension 来保存测量结果
    setMeasuredDimension(widthSize, heightSize);

    mWidthMeasureSpec = widthMeasureSpec;
}
```

由上面的代码可以看到，measure 过程比较简单，只是判断了一下目前自身设置的 MeasureSpec，来决定自身的大小。

### ListView 的`layoutChildren()`

上面说过了，ListView 不能覆写`onLayout()`方法，而是覆写`layoutChildren()`方法来对自身进行 layout 过程，我们来看看它的代码：

```java
@Override
protected void layoutChildren() {
    // mBlockLayoutRequests 是 AdapterView 中的成员变量，用来标识当前是否正在 layout 过程中
    final boolean blockLayoutRequests = mBlockLayoutRequests;
    if (blockLayoutRequests) {
        return;
    }

    mBlockLayoutRequests = true;

    try {
        // 先调用 AbsListView 的 layoutChildren() 方法（虽然是未实现）
        super.layoutChildren();

        // 刷新，在 layout 完成之后调用 draw 过程
        invalidate();

        if (mAdapter == null) {
            resetList();
            invokeOnItemScrollListener();
            return;
        }

        final int childrenTop = mListPadding.top;
        final int childrenBottom = mBottom - mTop - mListPadding.bottom;
        final int childCount = getChildCount();

        int index = 0;
        int delta = 0;

        View sel;
        View oldSel = null;
        View oldFirst = null;
        View newSel = null;

        // 记录处于 select mode 的 item
        switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                index = mNextSelectedPosition - mFirstPosition;
                if (index >= 0 && index < childCount) {
                    newSel = getChildAt(index);
                }
                break;
            case LAYOUT_FORCE_TOP:
            case LAYOUT_FORCE_BOTTOM:
            case LAYOUT_SPECIFIC:
            case LAYOUT_SYNC:
                break;
            case LAYOUT_MOVE_SELECTION:
            default:
                // Remember the previously selected view
                index = mSelectedPosition - mFirstPosition;
                if (index >= 0 && index < childCount) {
                    oldSel = getChildAt(index);
                }

                // Remember the previous first child
                oldFirst = getChildAt(0);

                if (mNextSelectedPosition >= 0) {
                    delta = mNextSelectedPosition - mSelectedPosition;
                }

                // Caution: newSel might be null
                newSel = getChildAt(index + delta);
        }

        // 如果在 layout 过程中 adapter 中的数据发生了变化
        boolean dataChanged = mDataChanged;
        if (dataChanged) {
            // AbsListView 中的方法，用来确定是哪条数据发生了改变
            handleDataChanged();
        }

        // 如果是空数据，就清除所有可见的 view，将各种数据全部重置，然后返回
        if (mItemCount == 0) {
            resetList();
            invokeOnItemScrollListener();
            return;
        } else if (mItemCount != mAdapter.getCount()) {
            throw new IllegalStateException("The content of the adapter has changed but "
                    + "ListView did not receive a notification. Make sure the content of "
                    + "your adapter is not modified from a background thread, but only from "
                    + "the UI thread. Make sure your adapter calls notifyDataSetChanged() "
                    + "when its content changes. [in ListView(" + getId() + ", " + getClass()
                    + ") with Adapter(" + mAdapter.getClass() + ")]");
        }

        setSelectedPositionInt(mNextSelectedPosition);

        // 处理一些 accessibility 相关的
        AccessibilityNodeInfo accessibilityFocusLayoutRestoreNode = null;
        View accessibilityFocusLayoutRestoreView = null;
        int accessibilityFocusPosition = INVALID_POSITION;

        // Remember which child, if any, had accessibility focus. This must
        // occur before recycling any views, since that will clear
        // accessibility focus.
        final ViewRootImpl viewRootImpl = getViewRootImpl();
        if (viewRootImpl != null) {
            final View focusHost = viewRootImpl.getAccessibilityFocusedHost();
            if (focusHost != null) {
                final View focusChild = getAccessibilityFocusedChild(focusHost);
                if (focusChild != null) {
                    if (!dataChanged || isDirectChildHeaderOrFooter(focusChild)
                            || (focusChild.hasTransientState() && mAdapterHasStableIds)) {
                        // The views won't be changing, so try to maintain
                        // focus on the current host and virtual view.
                        accessibilityFocusLayoutRestoreView = focusHost;
                        accessibilityFocusLayoutRestoreNode = viewRootImpl
                                .getAccessibilityFocusedVirtualView();
                    }

                    // If all else fails, maintain focus at the same
                    // position.
                    accessibilityFocusPosition = getPositionForView(focusChild);
                }
            }
        }

        // 处理焦点相关
        View focusLayoutRestoreDirectChild = null;
        View focusLayoutRestoreView = null;

        // Take focus back to us temporarily to avoid the eventual call to
        // clear focus when removing the focused child below from messing
        // things up when ViewAncestor assigns focus back to someone else.
        final View focusedChild = getFocusedChild();
        if (focusedChild != null) {
            // TODO: in some cases focusedChild.getParent() == null

            // We can remember the focused view to restore after re-layout
            // if the data hasn't changed, or if the focused position is a
            // header or footer.
            if (!dataChanged || isDirectChildHeaderOrFooter(focusedChild)
                    || focusedChild.hasTransientState() || mAdapterHasStableIds) {
                focusLayoutRestoreDirectChild = focusedChild;
                // Remember the specific view that had focus.
                focusLayoutRestoreView = findFocus();
                if (focusLayoutRestoreView != null) {
                    // Tell it we are going to mess with it.
                    focusLayoutRestoreView.dispatchStartTemporaryDetach();
                }
            }
            requestFocus();
        }

        // 将所有 View 扔到回收站，这些 View 有可能会被重用
        final int firstPosition = mFirstPosition;
        final RecycleBin recycleBin = mRecycler;
        if (dataChanged) {
            for (int i = 0; i < childCount; i++) {
                recycleBin.addScrapView(getChildAt(i), firstPosition+i);
            }
        } else {
            recycleBin.fillActiveViews(childCount, firstPosition);
        }

        // 将所有的子 View 的 parent 置为 null，并将 View 置为 null
        detachAllViewsFromParent();
        recycleBin.removeSkippedScrap();

        switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                if (newSel != null) {
                    sel = fillFromSelection(newSel.getTop(), childrenTop, childrenBottom);
                } else {
                    sel = fillFromMiddle(childrenTop, childrenBottom);
                }
                break;
            case LAYOUT_SYNC:
                sel = fillSpecific(mSyncPosition, mSpecificTop);
                break;
            case LAYOUT_FORCE_BOTTOM:
                sel = fillUp(mItemCount - 1, childrenBottom);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_SPECIFIC:
                final int selectedPosition = reconcileSelectedPosition();
                sel = fillSpecific(selectedPosition, mSpecificTop);
                /**
                    * When ListView is resized, FocusSelector requests an async selection for the
                    * previously focused item to make sure it is still visible. If the item is not
                    * selectable, it won't regain focus so instead we call FocusSelector
                    * to directly request focus on the view after it is visible.
                    */
                if (sel == null && mFocusSelector != null) {
                    final Runnable focusRunnable = mFocusSelector
                            .setupFocusIfValid(selectedPosition);
                    if (focusRunnable != null) {
                        post(focusRunnable);
                    }
                }
                break;
            case LAYOUT_MOVE_SELECTION:
                sel = moveSelection(oldSel, newSel, delta, childrenTop, childrenBottom);
                break;
            // 在 ListView 的第一次 layout 过程中，childCount = 0
            default:
                if (childCount == 0) {
                    // mStackFromBottom 为 true 时，表示整个 ListView 是倒着填充的
                    if (!mStackFromBottom) {
                        final int position = lookForSelectablePosition(0, true);
                        setSelectedPositionInt(position);
                        // 主要的填充方法
                        sel = fillFromTop(childrenTop);
                    } else {
                        final int position = lookForSelectablePosition(mItemCount - 1, false);
                        setSelectedPositionInt(position);
                        sel = fillUp(mItemCount - 1, childrenBottom);
                    }
                } else {
                    // 如果已经有 View 了，那就只填充特定位置的 View
                    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                        sel = fillSpecific(mSelectedPosition,
                                oldSel == null ? childrenTop : oldSel.getTop());
                    } else if (mFirstPosition < mItemCount) {
                        sel = fillSpecific(mFirstPosition,
                                oldFirst == null ? childrenTop : oldFirst.getTop());
                    } else {
                        sel = fillSpecific(0, childrenTop);
                    }
                }
                break;
        }

        ...
    } finally {
        ...
        // 允许进行下一次 layout
        if (!blockLayoutRequests) {
            mBlockLayoutRequests = false;
        }
    }
}
```

可见，在 layout 过程中，会处理一些 Accessibility 和焦点方面的事情，最后开始判断 ListView 的类型和子 View 数量。在初次 layout 时，ListView 中必然不会添加进子 View，所以此时 childCount 为 0，在填充时会调用`fillFromTop()`方法：

```java
private View fillFromTop(int nextTop) {
    mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
    mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
    if (mFirstPosition < 0) {
        mFirstPosition = 0;
    }
    return fillDown(mFirstPosition, nextTop);
}

private View fillDown(int pos, int nextTop) {
    View selectedView = null;

    int end = (mBottom - mTop);
    if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
        end -= mListPadding.bottom;
    }

    while (nextTop < end && pos < mItemCount) {
        // is this the selected item?
        boolean selected = pos == mSelectedPosition;
        View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

        nextTop = child.getBottom() + mDividerHeight;
        if (selected) {
            selectedView = child;
        }
        pos++;
    }

    setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
    return selectedView;
}
```

可见在`fillDown()`方法中，使用了循环的方式将子 View 都创建出来，具体的操作还是交给了`makeAndAddView()`方法：

```java
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
        boolean selected) {
    if (!mDataChanged) {
        // 这里尝试从回收站中拿出已经渲染过的 View，初次获取时，必为 null
        final View activeView = mRecycler.getActiveView(position);
        if (activeView != null) {
            // 如果在回收站中找到了对应的 View，就直接给它放到对应的位置上
            setupChild(activeView, position, y, flow, childrenLeft, selected, true);
            return activeView;
        }
    }

    // 新建一个 View，或者是将一个未使用的 View convert 一下（如果可能的话）
    final View child = obtainView(position, mIsScrap);

    // 放到对应的位置上
    setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

    return child;
}
```

这里会尝试先获取回收站中对应的 View，在首次 layout 时肯定获取到的是`null`，就会调用`obtainView()`来创建一个 View，这个方法由 AbsListView 来实现：

```java
// AbsListView.java

View obtainView(int position, boolean[] outMetadata) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "obtainView");

    outMetadata[0] = false;

    final View transientView = mRecycler.getTransientStateView(position);  // 1
    if (transientView != null) {
        final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

        // If the view type hasn't changed, attempt to re-bind the data.
        if (params.viewType == mAdapter.getItemViewType(position)) {
            final View updatedView = mAdapter.getView(position, transientView, this); //2

            // If we failed to re-bind the data, scrap the obtained view.
            if (updatedView != transientView) {
                setItemViewLayoutParams(updatedView, position);
                mRecycler.addScrapView(updatedView, position);
            }
        }

        outMetadata[0] = true;

        // Finish the temporary detach started in addScrapView().
        transientView.dispatchFinishTemporaryDetach();
        return transientView;
    }

    final View scrapView = mRecycler.getScrapView(position);  // 3
    final View child = mAdapter.getView(position, scrapView, this);  // 4
    if (scrapView != null) {
        if (child != scrapView) {
            // Failed to re-bind the data, return scrap to the heap.
            mRecycler.addScrapView(scrapView, position);  // 5
        } else if (child.isTemporarilyDetached()) {
            outMetadata[0] = true;

            // Finish the temporary detach started in addScrapView().
            child.dispatchFinishTemporaryDetach();
        }
    }

    if (mCacheColorHint != 0) {
        child.setDrawingCacheBackgroundColor(mCacheColorHint);
    }

    if (child.getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
        child.setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
    }

    setItemViewLayoutParams(child, position);

    if (AccessibilityManager.getInstance(mContext).isEnabled()) {
        if (mAccessibilityDelegate == null) {
            mAccessibilityDelegate = new ListItemAccessibilityDelegate();
        }
        if (child.getAccessibilityDelegate() == null) {
            child.setAccessibilityDelegate(mAccessibilityDelegate);
        }
    }

    Trace.traceEnd(Trace.TRACE_TAG_VIEW);

    return child;
}
```

这个方法是各种 ListView 及其派生类中最重要的方法了，我们来详细解释一下。

1. 检查回收站中是否有瞬态的 View，如果有的话，尝试重新绑定数据；如果没有的话，就直接扔掉，重新创建。
2. 如果在上一步中获取到了瞬态 View，并且之前存储的 ViewType 与 Adapter 中存储的 ViewType 相同的话，就直接把瞬态 View 交给 Adapter，重新绑定数据。
3. 如果在第1步中没有获取到瞬态 View，则尝试从 ScrapViews 中获取，看有没有被废弃的 View。
4. 拿着上一步获取的 ScrapView，交给 Adapter 绑定数据。
5. 如果获取的 ScrapView 和 Adapter 绑定数据后的 View 是同一个实例（复用），则将这个 View 添加到 ScrapViews 中。
6. 返回 Adapter 绑定数据后的 View。

这里还有个比较重要的方法：`mAdapter.getView()`，眼熟吗？这不就是我们平时在开发时要覆写的方法之一吗，我们平时会这样写：

```java
@Override
public View getView(int position, View convertView, ViewGroup parent) {
	Person person = getItem(position);
	View view;
	if (convertView == null) {
		view = LayoutInflater.from(getContext()).inflate(resourceId, null);
	} else {
		view = convertView;
	}
	ImageView avatar = (ImageView) view.findViewById(R.id.iv_avatar);
	TextView name = (TextView) view.findViewById(R.id.tv_name);
	avatar.setImageResource(person.getImageId());
	name.setText(person.getName());
	return view;
}
```

这下能明白复用的`convertView`是哪儿来的了吗？它如果是`null`的话，说明是初次 layout 过程，如果不是`null`的话，说明它是瞬态 View 或者是 ScrapView。这也是 ListView 优化处之一——复用`convertView`，节省掉用 LayoutInflater 重新渲染的过程。

当然，在初次 layout 过程中，所有的`convertView`必然为`null`，也即全部为 LayoutInflater 渲染，会比较耗时。

我们继续回到`makeAndAddView()`方法中，下面还有个比较重要的方法是`setupChild()`：

```java
  private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
            boolean selected, boolean isAttachedToWindow) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "setupListItem");

    final boolean isSelected = selected && shouldShowSelector();
    final boolean updateChildSelected = isSelected != child.isSelected();
    final int mode = mTouchMode;
    final boolean isPressed = mode > TOUCH_MODE_DOWN && mode < TOUCH_MODE_SCROLL
            && mMotionPosition == position;
    final boolean updateChildPressed = isPressed != child.isPressed();
    final boolean needToMeasure = !isAttachedToWindow || updateChildSelected
            || child.isLayoutRequested();

    // Respect layout params that are already in the view. Otherwise make
    // some up...
    AbsListView.LayoutParams p = (AbsListView.LayoutParams) child.getLayoutParams();
    if (p == null) {
        p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
    }
    p.viewType = mAdapter.getItemViewType(position);
    p.isEnabled = mAdapter.isEnabled(position);

    // Set up view state before attaching the view, since we may need to
    // rely on the jumpDrawablesToCurrentState() call that occurs as part
    // of view attachment.
    if (updateChildSelected) {
        child.setSelected(isSelected);
    }

    if (updateChildPressed) {
        child.setPressed(isPressed);
    }

    if (mChoiceMode != CHOICE_MODE_NONE && mCheckStates != null) {
        if (child instanceof Checkable) {
            ((Checkable) child).setChecked(mCheckStates.get(position));
        } else if (getContext().getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.HONEYCOMB) {
            child.setActivated(mCheckStates.get(position));
        }
    }

    if ((isAttachedToWindow && !p.forceAdd) || (p.recycledHeaderFooter
            && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
        attachViewToParent(child, flowDown ? -1 : 0, p);

        // If the view was previously attached for a different position,
        // then manually jump the drawables.
        if (isAttachedToWindow
                && (((AbsListView.LayoutParams) child.getLayoutParams()).scrappedFromPosition)
                        != position) {
            child.jumpDrawablesToCurrentState();
        }
    } else {
        p.forceAdd = false;
        if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
            p.recycledHeaderFooter = true;
        }
        addViewInLayout(child, flowDown ? -1 : 0, p, true);
        // add view in layout will reset the RTL properties. We have to re-resolve them
        child.resolveRtlPropertiesIfNeeded();
    }

    if (needToMeasure) {
        final int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
                mListPadding.left + mListPadding.right, p.width);
        final int lpHeight = p.height;
        final int childHeightSpec;
        if (lpHeight > 0) {
            childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
        } else {
            childHeightSpec = MeasureSpec.makeSafeMeasureSpec(getMeasuredHeight(),
                    MeasureSpec.UNSPECIFIED);
        }
        child.measure(childWidthSpec, childHeightSpec);
    } else {
        cleanupLayoutState(child);
    }

    final int w = child.getMeasuredWidth();
    final int h = child.getMeasuredHeight();
    final int childTop = flowDown ? y : y - h;

    if (needToMeasure) {
        final int childRight = childrenLeft + w;
        final int childBottom = childTop + h;
        child.layout(childrenLeft, childTop, childRight, childBottom);
    } else {
        child.offsetLeftAndRight(childrenLeft - child.getLeft());
        child.offsetTopAndBottom(childTop - child.getTop());
    }

    if (mCachingStarted && !child.isDrawingCacheEnabled()) {
        child.setDrawingCacheEnabled(true);
    }

    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}
```

这部分代码的核心是`attachViewToParent()`和`addViewInLayout()`方法，代表着将 View 正式加入 ListView 中。

[看看这个，日后继续写吧](https://blog.csdn.net/guolin_blog/article/details/44996879)
[看看这个](https://www.jianshu.com/p/01f161cb498c)