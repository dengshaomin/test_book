---
description: 转：https://blog.csdn.net/gdutxiaoxu/article/details/71616547
---

# CoordinatorLayout

CoordinatorLayout 实现了 NestedScrollingParent 接口，是一个容器，我们可以通过指定 behavior 来实现各种各样炫酷的效果。需要注意的是本篇博客分析的版本是 25.2.0 的。

分析之前，先放一下效果图，给大家看一看，增加一下博客访问量，下一篇博客会教大家如何实现。

Behavior 的初始化\
Behavior 是 CoordinatorLayout 里面的一个静态类。重写里面的若干方法，我们可以实现各种炫酷的效果，比如仿 UC 主页，仿新浪微博，仿 QQ 浏览器主页，仿知乎首页等效果。

Behavior 的主要几个方法

```
public static abstract class Behavior<V extends View> {

    // 中间省略了若干方法
    public Behavior() {
    }


    public Behavior(Context context, AttributeSet attrs) {
    }


    public void onAttachedToLayoutParams(@NonNull CoordinatorLayout.LayoutParams params) {
    }


    public void onDetachedFromLayoutParams() {
    }


    public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
        return false;
    }


    public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
        return false;
    }



    public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) {
        return false;
    }


    public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {
        return false;
    }

    public void onDependentViewRemoved(CoordinatorLayout parent, V child, View dependency) {
    }



    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout,
            V child, View directTargetChild, View target, int nestedScrollAxes) {
        return false;
    }


    public void onNestedScrollAccepted(CoordinatorLayout coordinatorLayout, V child,
            View directTargetChild, View target, int nestedScrollAxes) {
        // Do nothing
    }


    public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target) {
        // Do nothing
    }


    public void onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target,
            int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
        // Do nothing
    }


    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target,
            int dx, int dy, int[] consumed) {
        // Do nothing
    }


    public boolean onNestedFling(CoordinatorLayout coordinatorLayout, V child, View target,
            float velocityX, float velocityY, boolean consumed) {
        return false;
    }


    public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target,
            float velocityX, float velocityY) {
        return false;
    }


}
```

Behavior 的初始化时机\
第一种方法，我们可以通过 app:layout\_behavior=”@string/behavior\_weibo\_header” 指定 behavior。\
这种方式指定的 Behavior 是在生成 layoutParams 里面进行转化的。

```
LayoutParams(Context context, AttributeSet attrs) {
    super(context, attrs);
    if (mBehaviorResolved) {
        mBehavior = parseBehavior(context, attrs, a.getString(
                R.styleable.CoordinatorLayout_Layout_layout_behavior));
    }
    a.recycle();

    if (mBehavior != null) {
        // If we have a Behavior, dispatch that it has been attached
        mBehavior.onAttachedToLayoutParams(this);
    }
}
```

第二种方法:我们可以调用 CoordinatorLayout.LayoutParams 的 setBehavior(@Nullable Behavior behavior) 来设置 behavior。它最终会在 onMeasure 的时候设置给相应的 View。

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    prepareChildren();

}

private void prepareChildren() {
    mDependencySortedChildren.clear();
    mChildDag.clear();

    for (int i = 0, count = getChildCount(); i < count; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = getResolvedLayoutParams(view);
        lp.findAnchorView(this, view);
        mChildDag.addNode(view);

        // Now iterate again over the other children, adding any dependencies to the graph
        for (int j = 0; j < count; j++) {
            if (j == i) {
                continue;
            }
            final View other = getChildAt(j);
            final LayoutParams otherLp = getResolvedLayoutParams(other);
            if (otherLp.dependsOn(this, other, view)) {
                if (!mChildDag.contains(other)) {
                    // Make sure that the other node is added
                    mChildDag.addNode(other);
                }
                // Now add the dependency to the graph
                mChildDag.addEdge(view, other);
            }
        }
    }

    // Finally add the sorted graph list to our list
    mDependencySortedChildren.addAll(mChildDag.getSortedList());
    // We also need to reverse the result since we want the start of the list to contain
    // Views which have no dependencies, then dependent views after that
    Collections.reverse(mDependencySortedChildren);
}

LayoutParams getResolvedLayoutParams(View child) {
    final LayoutParams result = (LayoutParams) child.getLayoutParams();
    if (!result.mBehaviorResolved) {
        Class<?> childClass = child.getClass();
        DefaultBehavior defaultBehavior = null;
        while (childClass != null &&
                (defaultBehavior = childClass.getAnnotation(DefaultBehavior.class)) == null) {
            childClass = childClass.getSuperclass();
        }
        if (defaultBehavior != null) {
            try {
                result.setBehavior(defaultBehavior.value().newInstance());
            } catch (Exception e) {
                Log.e(TAG, "Default behavior class " + defaultBehavior.value().getName() +
                        " could not be instantiated. Did you forget a default constructor?", e);
            }
        }
        result.mBehaviorResolved = true;
    }
    return result;
}
```

NestedScrollingParent 与 NestedScrollingChild 接口方法是怎样回调的\
执行流程

在 Action\_Down 的时候，Scrolling child 会调用 startNestedScroll 方法，通过 childHelper 回调 Scrolling Parent 的 startNestedScroll 方法\
在 Action\_move 的时候，Scrolling Child 要开始滑动的时候，会调用dispatchNestedPreScroll 方法，通过 ChildHelper 询问 Scrolling Parent 是否要先于 Child 进行 滑动，若需要的话，会调用 Parent 的 onNestedPreScroll 方法，协同 Child 一起进行滑动\
当 ScrollingChild 滑动完成的时候，会调用 dispatchNestedScroll 方法，通过 ChildHelper 询问 Scrolling Parent 是否需要进行滑动，需要的话，会 调用 Parent 的 onNestedScroll 方法\
在 Action\_down,Action\_move 的时候，会调用 Scrolling Child 的stopNestedScroll ，通过 ChildHelper 询问 Scrolling parent 的 stopNestedScroll 方法。\
如果需要处理 Fling 动作，我们可以通过 VelocityTrackerCompat 获得相应的速度，并在 Action\_up 的时候，调用 dispatchNestedPreFling 方法，通过 ChildHelper 询问 Parent 是否需要先于 child 进行 Fling 动作\
在 Child 处理完 Fling 动作时候，如果 Scrolling Parent 还需要处理 Fling 动作，我们可以调用 dispatchNestedFling 方法，通过 ChildHelper ，调用 Parent 的 onNestedFling 方法\
在上一篇博客NestedScrolling 机制深入解析 的 时候其实已经说到：是通过 NestedScrollingChildHelper 来完成的，具体 的请看里面分分析，这里就不再一一阐述。

Behavior 方法与 NestedScrollingParent 方法之间的关系\
我们知道 NestedScrollingParent 主要有这些方法，

onStartNestedScroll、onNestedScrollAccepted\
onNestedPreScroll\
onNestedScroll\
onNestedPreFling\
onNestedFling\
onStopNestedScroll\
在 Behavior 方法里面也有这些方法，与 NestedScrollingParent 方法 几乎也是一一对应的。在 CoordinatorLayout 里面。NestedScrollingParent 接口的方法的具体 实现逻辑 都会交给 Behavior 对应的方法去处理。下面我们一起来看一下是怎样处理的。

onStartNestedScroll 方法\
思想大概是这样的：

遍历所有的孩子 ，如果可见性是 GONE，跳过。如果可见性不是 GONE，通过 layoutParams 拿到 Behavior，判断 behavior 是否为空，不为空，调用 behavior 的对应方法 onStartNestedScroll 和 acceptNestedScroll 方法。

```
// 开始滑动的时候
@Override
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    boolean handled = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        if (view.getVisibility() == View.GONE) {
            // If it's GONE, don't dispatch
            continue;
        }
        // 通过  LayoutParams 拿到对应的  Behavior 
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
             // 交给 Behavior  去处理
            final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child, target,
                    nestedScrollAxes);
            handled |= accepted;

            lp.acceptNestedScroll(accepted);
        } else {
            lp.acceptNestedScroll(false);
        }
    }
    return handled;
}
```

onNestedScrollAccepted 方法\
思想大概是这样的

调用 mNestedScrollingParentHelper 的相关方法\
遍历孩子，通过 layoutParams 判断是否要处理滑动事件，处理的 话，回调 Behavior 的相关方法，不处理的话，跳过当前 View。

```
@Override
public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes) {
    mNestedScrollingParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes);
    mNestedScrollingDirectChild = child;
    mNestedScrollingTarget = target;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            // 调用 behavior 的相应方法
            viewBehavior.onNestedScrollAccepted(this, view, child, target, nestedScrollAxes);
        }
    }
}
```

onNestedPreScroll 方法\
我们知道 onNestedPreScroll 是在 Scrolling child 滑动之前回调的，提供机会给 Scrooling Parent 先于 child 进行滑动的。

在 CoordinatorLayout 里面，它的处理流程是这样的。 遍历所有的孩子，判断可见性是否为 GONE，如果是 ，跳过当前 子 View,通过 LayoutParams 判断是否处理滑动事件，不处理滑动 事件，跳过，拿到 Behavior，判断 Behavior 是否为空，不过空，回调 Behavior 的 onNestedPreScroll 方法。

```
@Override
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    int xConsumed = 0;
    int yConsumed = 0;
    boolean accepted = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        if (view.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            mTempIntPair[0] = mTempIntPair[1] = 0;
            viewBehavior.onNestedPreScroll(this, view, target, dx, dy, mTempIntPair);

            xConsumed = dx > 0 ? Math.max(xConsumed, mTempIntPair[0])
                    : Math.min(xConsumed, mTempIntPair[0]);
            yConsumed = dy > 0 ? Math.max(yConsumed, mTempIntPair[1])
                    : Math.min(yConsumed, mTempIntPair[1]);

            accepted = true;
        }
    }

    consumed[0] = xConsumed;
    consumed[1] = yConsumed;

    if (accepted) {
        onChildViewsChanged(EVENT_NESTED_SCROLL);
    }
}
```

onNestedScroll 方法\
在 Scrolling Child 滑动之后，提供机会给 Scrolling Parent 滑动，事件的处理 逻辑就不一一阐述了 ，跟前面的差不多

```
@Override
public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
        int dxUnconsumed, int dyUnconsumed) {
    final int childCount = getChildCount();
    boolean accepted = false;

    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        if (view.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
          // 如果之前没有处理滑动事件，直接返回，不调用 onStopNestedScroll 方法
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
             // 调用 behavior 的相应方法
            viewBehavior.onNestedScroll(this, view, target, dxConsumed, dyConsumed,
                    dxUnconsumed, dyUnconsumed);
            accepted = true;
        }
    }

    if (accepted) {
        onChildViewsChanged(EVENT_NESTED_SCROLL);
    }
}
```

onNestedPreFling 和 onNestedFling 方法

```
@Override
public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
    boolean handled = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        if (view.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            handled |= viewBehavior.onNestedPreFling(this, view, target, velocityX, velocityY);
        }
    }
    return handled;
}
@Override
public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed) {
    boolean handled = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        if (view.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            handled |= viewBehavior.onNestedFling(this, view, target, velocityX, velocityY,
                    consumed);
        }
    }
    if (handled) {
        onChildViewsChanged(EVENT_NESTED_SCROLL);
    }
    return handled;
}
```

onStopNestedScroll 方法

```
@Override
public void onStopNestedScroll(View target) {
    mNestedScrollingParentHelper.onStopNestedScroll(target);

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
         // 如果之前没有处理滑动事件，直接返回，不调用 onStopNestedScroll 方法
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            viewBehavior.onStopNestedScroll(this, view, target);
        }
        lp.resetNestedScroll();
        lp.resetChangedAfterNestedScroll();
    }

    mNestedScrollingDirectChild = null;
    mNestedScrollingTarget = null;
}
```

Behavior 相比 NestedScrollingParent 独有的方法\
public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency)；\
返回 true，当 dependency 改变的 时候，将会回调 onDependentViewChanged 方法.比如，当我们依赖于 AppBarLayout 的时候，我们可以这样写 。

```
@Override
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
    // We depend on any AppBarLayouts
    return dependency instanceof AppBarLayout;
}
```

public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency)\
与 layoutDependsOn 息息相关，当 layoutDependsOn 返回TRUE的时候，才会回调这个方法。

onDependentViewRemove\
当 dependency 移除的时候，会回调这个方法。\
CoordinatorLayout 是如何监听 View 的状态的？\
那 onDependentViewChanged 和 onDependentViewRemove 这两个方法是如何监听得到 View 变化和移除的？其实是在 onAttachedToWindow 方法里面，他会为 ViewTreeObserver 视图树添加 OnPreDrawListener 监听。

```
@Override
public void onAttachedToWindow() {
    super.onAttachedToWindow();
    resetTouchBehaviors();
    if (mNeedsPreDrawListener) {
        if (mOnPreDrawListener == null) {
            mOnPreDrawListener = new OnPreDrawListener();
        }
        final ViewTreeObserver vto = getViewTreeObserver();
        vto.addOnPreDrawListener(mOnPreDrawListener);
    }
    if (mLastInsets == null && ViewCompat.getFitsSystemWindows(this)) {
        // We're set to fitSystemWindows but we haven't had any insets yet...
        // We should request a new dispatch of window insets
        ViewCompat.requestApplyInsets(this);
    }
    mIsAttachedToWindow = true;
}

class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
        @Override
        public boolean onPreDraw() {
            onChildViewsChanged(EVENT_PRE_DRAW);
            return true;
        }
    }

```

在 OnPreDrawListener 监听里面会调用 onChildViewsChanged 方法，在该方法里面会根据 View的状态回调 onDependentViewRemoved 或者 onDependentViewChanged 方法。

```
final void onChildViewsChanged(@DispatchChangeEvent final int type) {
       ---
       / / 省略若干方法

    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        if (type == EVENT_PRE_DRAW && child.getVisibility() == View.GONE) {
            // Do not try to update GONE child views in pre draw updates.
            continue;
        }

     ---

        // Get the current draw rect of the view
     ----
       / / 省略若干方法

        if (type == EVENT_PRE_DRAW) {
            // Did it change? if not continue
            getLastChildRect(child, lastDrawRect);
            if (lastDrawRect.equals(drawRect)) {
                continue;
            }
            recordLastChildRect(child, drawRect);
        }

        // Update any behavior-dependent views for the change
        for (int j = i + 1; j < childCount; j++) {
            final View checkChild = mDependencySortedChildren.get(j);
            final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
            final Behavior b = checkLp.getBehavior();

            if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                if (type == EVENT_PRE_DRAW && checkLp.getChangedAfterNestedScroll()) {
                    // If this is from a pre-draw and we have already been changed
                    // from a nested scroll, skip the dispatch and reset the flag
                    checkLp.resetChangedAfterNestedScroll();
                    continue;
                }

                final boolean handled;
                switch (type) {
                    case EVENT_VIEW_REMOVED:
                        // EVENT_VIEW_REMOVED means that we need to dispatch
                        // onDependentViewRemoved() instead
                        // 回调 Behavior 的 onDependentViewRemoved 方法
                        b.onDependentViewRemoved(this, checkChild, child);
                        handled = true;
                        break;
                    default:
                        // Otherwise we dispatch onDependentViewChanged() 
                            // 回调 Behavior 的 onDependentViewChanged 方法
                        handled = b.onDependentViewChanged(this, checkChild, child);
                        break;
                }

                if (type == EVENT_NESTED_SCROLL) {
                    // If this is from a nested scroll, set the flag so that we may skip
                    // any resulting onPreDraw dispatch (if needed)
                    checkLp.setChangedAfterNestedScroll(handled);
                }
            }
        }
    }

  ---
}
```

CoordinatorLayout 如何移除 View 的监听状态的\
我们知道当 View 被销毁的时候，会回调 onDetachedFromWindow 这个方法，因此适合在这个方法里面移除 View 视图树的 PreDrawListener 监听。

```
@Override
public void onDetachedFromWindow() {
    super.onDetachedFromWindow();
    resetTouchBehaviors();
    if (mNeedsPreDrawListener && mOnPreDrawListener != null) {
        final ViewTreeObserver vto = getViewTreeObserver();
        vto.removeOnPreDrawListener(mOnPreDrawListener);
    }
    if (mNestedScrollingTarget != null) {
        onStopNestedScroll(mNestedScrollingTarget);
    }
    mIsAttachedToWindow = false;
}
```

CoordinatorLayout 的 measure 和 layout\
measure

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 处理 child 的一些 相关属性 ，比如 Behavior等
    prepareChildren();
    // 如果有依赖的话，添加 OnPreDrawListener 监听，没有的话，移除 OnPreDrawListener 监听
    ensurePreDrawListener();

    ---
    // 省略了 染过逻辑，主要是处理  padding的 
    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        if (child.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) child.getLayoutParams();

          ----


        final Behavior b = lp.getBehavior();
          // 回调 Behavior 的 onMeasureChild 方法
        if (b == null || !b.onMeasureChild(this, child, childWidthMeasureSpec, keylineWidthUsed,
                childHeightMeasureSpec, 0)) {
            onMeasureChild(child, childWidthMeasureSpec, keylineWidthUsed,
                    childHeightMeasureSpec, 0);
        }

        widthUsed = Math.max(widthUsed, widthPadding + child.getMeasuredWidth() +
                lp.leftMargin + lp.rightMargin);

        heightUsed = Math.max(heightUsed, heightPadding + child.getMeasuredHeight() +
                lp.topMargin + lp.bottomMargin);
        childState = ViewCompat.combineMeasuredStates(childState,
                ViewCompat.getMeasuredState(child));
    }

    final int width = ViewCompat.resolveSizeAndState(widthUsed, widthMeasureSpec,
            childState & ViewCompat.MEASURED_STATE_MASK);
    final int height = ViewCompat.resolveSizeAndState(heightUsed, heightMeasureSpec,
            childState << ViewCompat.MEASURED_HEIGHT_STATE_SHIFT);
    setMeasuredDimension(width, height);
}
```

我们进入 prepareChildren 方法里面，可以发现它对 CoordinatorLayout 里面的子 View 进行了排序，排序的结果是 最后被依赖的 View 会排在最前面。举个例子 A 依赖于 B，那么 B会排在前面，A 会排在 B 的 后面。这样的排序结果是合理的，因为 A 既然依赖于 B，那么 B 肯定要有限 measure。

```
private void prepareChildren() {
    mDependencySortedChildren.clear();
    mChildDag.clear();

    for (int i = 0, count = getChildCount(); i < count; i++) {
        final View view = getChildAt(i);

        final LayoutParams lp = getResolvedLayoutParams(view);
        lp.findAnchorView(this, view);

        mChildDag.addNode(view);

        // Now iterate again over the other children, adding any dependencies to the graph
        for (int j = 0; j < count; j++) {
            if (j == i) {
                continue;
            }
            final View other = getChildAt(j);
            final LayoutParams otherLp = getResolvedLayoutParams(other);
            if (otherLp.dependsOn(this, other, view)) {
                if (!mChildDag.contains(other)) {
                    // Make sure that the other node is added
                    mChildDag.addNode(other);
                }
                // Now add the dependency to the graph
                mChildDag.addEdge(view, other);
            }
        }
    }

    // Finally add the sorted graph list to our list
    mDependencySortedChildren.addAll(mChildDag.getSortedList());
    // We also need to reverse the result since we want the start of the list to contain
    // Views which have no dependencies, then dependent views after that
    Collections.reverse(mDependencySortedChildren);
}
```

接下来 我们进入 ensurePreDrawListener 方法里面，看看里面到底做了什么？

```
void ensurePreDrawListener() {
    boolean hasDependencies = false;
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View child = getChildAt(i);
        if (hasDependencies(child)) {
            hasDependencies = true;
            break;
        }
    }

    if (hasDependencies != mNeedsPreDrawListener) {
        if (hasDependencies) {
            addPreDrawListener();
        } else {
            removePreDrawListener();
        }
    }
}
```

其实它所做的工作就是 判断 子View ，如果有依赖的话，添加 OnPreDrawListener 监听，没有的话，移除 OnPreDrawListener 监听。

那这个 OnPreDrawListener 监听是干什么用的呢?

```
void addPreDrawListener() {
    if (mIsAttachedToWindow) {
        // Add the listener
        if (mOnPreDrawListener == null) {
            mOnPreDrawListener = new OnPreDrawListener();
        }
        final ViewTreeObserver vto = getViewTreeObserver();
        vto.addOnPreDrawListener(mOnPreDrawListener);
    }

    // Record that we need the listener regardless of whether or not we're attached.
    // We'll add the real listener when we become attached.
    mNeedsPreDrawListener = true;
}

    class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
        @Override
        public boolean onPreDraw() {
            onChildViewsChanged(EVENT_PRE_DRAW);
            return true;
        }
    }
```

其实就是当 Child 改变的 时候，回调依赖它的 Behavior 的 onDependentViewChanged 或者 onDependentViewRemoved 方法，从而来调整 View 在界面的显示位置 。

```
final void onChildViewsChanged(@DispatchChangeEvent final int type) {
    ---

    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        if (type == EVENT_PRE_DRAW && child.getVisibility() == View.GONE) {
            // Do not try to update GONE child views in pre draw updates.
            continue;
        }

         ---
         // 省略若干方法
        // Dodge inset edges if necessary
        if (lp.dodgeInsetEdges != Gravity.NO_GRAVITY && child.getVisibility() == View.VISIBLE) {
            offsetChildByInset(child, inset, layoutDirection);
        }

        if (type == EVENT_PRE_DRAW) {
            // Did it change? if not continue
            getLastChildRect(child, lastDrawRect);
            if (lastDrawRect.equals(drawRect)) {
                continue;
            }
            recordLastChildRect(child, drawRect);
        }

        // Update any behavior-dependent views for the change
        for (int j = i + 1; j < childCount; j++) {
            final View checkChild = mDependencySortedChildren.get(j);
            final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
            final Behavior b = checkLp.getBehavior();

            if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                if (type == EVENT_PRE_DRAW && checkLp.getChangedAfterNestedScroll()) {
                    // If this is from a pre-draw and we have already been changed
                    // from a nested scroll, skip the dispatch and reset the flag
                    checkLp.resetChangedAfterNestedScroll();
                    continue;
                }

                final boolean handled;
                switch (type) {
                    case EVENT_VIEW_REMOVED:
                        // EVENT_VIEW_REMOVED means that we need to dispatch
                        // onDependentViewRemoved() instead
                        b.onDependentViewRemoved(this, checkChild, child);
                        handled = true;
                        break;
                    default:
                        // Otherwise we dispatch onDependentViewChanged()
                        handled = b.onDependentViewChanged(this, checkChild, child);
                        break;
                }

                if (type == EVENT_NESTED_SCROLL) {
                    // If this is from a nested scroll, set the flag so that we may skip
                    // any resulting onPreDraw dispatch (if needed)
                    checkLp.setChangedAfterNestedScroll(handled);
                }
            }
        }
    }

    releaseTempRect(inset);
    releaseTempRect(drawRect);
    releaseTempRect(lastDrawRect);
}
```

layout 过程\
layout 过程相对比较简单，遍历所有孩子，如果可见性为 GONE ，跳过该孩子的 Layout\
接着通过 layoutParams 拿到 Behavior，如果 Behavior 为空或者 Behavior 没有处理自己的 layout 过程，调用 onLayoutChild 方法 去处理哈子的摆放\
如果 Behavior 有处理自己的 layout 过程，交给 Behavior 去处理 。

```
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    final int layoutDirection = ViewCompat.getLayoutDirection(this);
    final int childCount = mDependencySortedChildren.size();
    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        if (child.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        final Behavior behavior = lp.getBehavior();

        if (behavior == null || !behavior.onLayoutChild(this, child, layoutDirection)) {
            onLayoutChild(child, layoutDirection);
        }
    }
}
```

CoordinatorLayout 的事件传递\
CoordinatorLayout 并不会直接处理事件，而是会尽可能地交给子 View 的Behavior 进行处理。onInterceptTouchEvent 和 onToucheEvent 这两个方法都会调用 performIntercept 来处理事件。

```
private boolean performIntercept(MotionEvent ev, final int type) {
    boolean intercepted = false;
    boolean newBlock = false;

    MotionEvent cancelEvent = null;

    final int action = MotionEventCompat.getActionMasked(ev);

    final List<View> topmostChildList = mTempList1;
    //在5.0以上，按照z属性来排序，以下，则是按照添加顺序或者自定义的绘制顺序来排列
    getTopSortedChildren(topmostChildList);

    // Let topmost child views inspect first
    final int childCount = topmostChildList.size();
    for (int i = 0; i < childCount; i++) {
        final View child = topmostChildList.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        final Behavior b = lp.getBehavior();

        if ((intercepted || newBlock) && action != MotionEvent.ACTION_DOWN) {
            // Cancel all behaviors beneath the one that intercepted.
            // If the event is "down" then we don't have anything to cancel yet.
                 // 如果有一个behavior对事件进行了拦截，就发送Cancel事件给后续的所有Behavior。
                 //假设之前还没有Intercept发生，那么所有的事件都平等地对所有含有behavior的view进行分发，
                 //现在intercept忽然出现，那么相应的我们就要对除了Intercept的view发出Cancel
            if (b != null) {
                if (cancelEvent == null) {
                    final long now = SystemClock.uptimeMillis();
                    cancelEvent = MotionEvent.obtain(now, now,
                            MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                }
                switch (type) {
                    case TYPE_ON_INTERCEPT:
                        b.onInterceptTouchEvent(this, child, cancelEvent);
                        break;
                    case TYPE_ON_TOUCH:
                        b.onTouchEvent(this, child, cancelEvent);
                        break;
                }
            }
            continue;
        }

        if (!intercepted && b != null) {
            switch (type) {
                case TYPE_ON_INTERCEPT:
                    intercepted = b.onInterceptTouchEvent(this, child, ev);
                    break;
                case TYPE_ON_TOUCH:
                    intercepted = b.onTouchEvent(this, child, ev);
                    break;
            }
            // 记录拦截事件 的child
            if (intercepted) {
                mBehaviorTouchView = child;
            }
        }

        // Don't keep going if we're not allowing interaction below this.
        // Setting newBlock will make sure we cancel the rest of the behaviors.
        final boolean wasBlocking = lp.didBlockInteraction();
        final boolean isBlocking = lp.isBlockingInteractionBelow(this, child);
        newBlock = isBlocking && !wasBlocking;
        if (isBlocking && !newBlock) {
            // Stop here since we don't have anything more to cancel - we already did
            // when the behavior first started blocking things below this point.
            break;
        }
    }

    topmostChildList.clear();

    return intercepted;
}
```

处理流程大概是这样的

对 Child 进行排序，在5.0以上，按照z属性来排序，以下，则是按照添加顺序或者自定义的绘制顺序来排列\
遍历排序好的所有 Child，分别调用 onInterceptTouchEvent 或者 onTouchEvent 方法，如果有 Child 进行处理 （Intercept），记录下该 child；\
某个 Child intercept 了事件，那么相应的我们就要对除了 Intercept 的view发出 Cancel\
总结\
CoordinatorLayout 是一个容器，实现了 NestedScrollingParent 接口。它并不会直接处理事件，而是会尽可能地交给子 View 的Behavior 进行处理。\
CoordinatorLayout 的内部类 Behavior 有点类似于代理，在 CoordinatorLayout 里面会调用 Behavior 的相应方法。\
CoordinatorLayout 在 measure 和 layout 的过程中，会优先 measure 和 layout 需要先 measure 好 layout 的View （比如 A 依赖于 B，那必须先 measure 和 layout A）。\
在 Scrolling Child 进行事件分发的时候，CoordinatorLayout 收到事件之后，会回调孩子的 Behavior 里面对应的方法，从而来调整视图。
