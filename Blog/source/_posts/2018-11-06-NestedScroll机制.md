---
layout: post
title: NestedScroll机制
date: 2018-11-06 20:42:25
categories: 
- 高级UI

tags:
- Android
- 源码解析
- 高级UI
---

## 一、概述

有关 Android 滑动机制以及两个都可以滑动的 View 在滑动过程中可能产生的滑动冲突问题我们之前已经专门分析过了，一般遇到滑动冲突问题时我们都通过自己修改父 View 或者子 View 的事件分发方法来解决。但是在有些情况下这个方式并不能解决我们的需求。

例如下面 gif 图中的效果，Inner 部分是一个 ScrollView，Out 部分是一个 ScrollView，整个 Inner 部分是在 Out 的一部分

<!-- more -->

在开发时如果只时为 InnerScrollView 设置对应的高度，最终效果就会是 InnerScrollView 部分并不能滑动，并且如果高度不够时 InnerScrollView 内容会展示不全。为了解决这个问题我们可以通过自定义 ScrollView 修改事件分发方法来实现 InnerScrollView 支持滑动，但是其滑动到顶部或者底部时并不能平滑的将滑动事件交给外部的 ScrollView。

这时候我们可以进一步改写 ScrollView 的事件分发方法使其支持 InnerScrollView 滑动到底部或者顶部后的其他事件重新由 Out 来处理，但在同一个事件序列中，InnerScrollView 就再也不能处理滑动事件了。所以这时候为了实现图中的效果我们就必须通过其他的方式来实现。

这阵子在研究 CoordinatorLayout 来实现一些滑动效果，发现了两个神奇的接口 **NestedScrollingParent** 和 **NestedScrollingChild**。这两个接口都是 support 包中提供的，顾名思义，这就是 Google 提供的支持处理嵌套滑动的接口。

下面我们就来一点一点分析如何通过这两个接口实现上图中嵌套滑动的效果。

## 二、接口简介

### （一）NestedScrollingParent

这个接口应该被那些支持滑动操作可以被一个嵌套子 View 委托的类实现。简单来说，这个接口应该被那些支持跟子 View 同时处理嵌套滑动的父 View 来实现。其实从这个介绍我们就可以得出一个很关键的信息，那就是整个滑动都是委托给子 View 的，那到底如何实现呢 ？答案就在下面的分析中。

**boolean onStartNestedScroll()** 开始嵌套滑动时被调用，返回当前 ViewGroup 是否接受开始这个嵌套滑动操作，如果接受，那么父 View 和子 View 会相互绑定

**void onNestedScrollAccepted()** 滑动开始时，支持嵌套滑动的子 View 与当前 View 成功绑定时会调用

**void onStopNestedScroll()** 嵌套滑动结束

**void onNestedScroll()** 嵌套滑动时，如果子 View 有未消耗的距离，会分发到父 View 的 onNestedScroll 方法中，父 View 会通过滑动自身的内容消耗这段距离或不处理直接返回 false

**void onNestedPreScroll()** 在嵌套滑动开始前被调用父，父 View 可以在这个方法中自定义消耗一部分滑动，也可以不进行消耗

**boolean onNestedFling()** 快速滑动，一般这个方法在子 View 滑动到边缘时被调用，子 View 滑动到边缘时，当前 View 就要处理剩余的滑动操作。返回值是当前 ViewGroup 是否完全消耗了这段距离，如果需要全部或部分消耗滑动距离，就需要自己重写这个方法实现逻辑

**boolean onNestedPreFling()** 在快速滑动前被调用，返回值是当前 ViewGroup 是否消耗这个操作，如果不消耗，则子 View 会继续分发 Fling 事件以及执行 Fling 滑动，如果消耗则子 View 不会继续处理，当前 View 则需要重写这个方法实现自己的逻辑

**int getNestedScrollAxes()** 返回当前滑动的方向，SCROLL_AXIS_HORIZONTAL，SCROLL_AXIS_VERTICAL，SCROLL_AXIS_NONE 三个之中的一个


### （二）NestedScrollingChild

这个接口应该被那些支持跟父 View 同时处理滑动嵌套的 View 实现

**void setNestedScrollingEnabled(boolean enabled)** 启动或禁用这个 View 的嵌套滑动

**boolean isNestedScrollingEnabled()** 是否可以滑动

**boolean startNestedScroll()** 为当前 View 寻找合作嵌套滑动的父 View ，及判断是否启动了滑动状态，如果满足条件将父 View 和当前 View 相互绑定，返回值为是否相互绑定成功

**void stopNestedScroll()** 滑动停止时调用

**boolean hasNestedScrollingParent()** 是否有嵌套滑动的父 View

**boolean dispatchNestedPreScroll()** 将嵌套滑动事件即将执行之前将 preScroll 操作分发到父 View，返回值为是否有支持嵌套滑动的父 View 且该父 View 消耗这个操作，如果返回 true 那么子 View 滑动的距离为产生的滑动减去父 View 消耗的距离

**boolean dispatchNestedScroll()** 分发滑动过程中自己消耗后剩余的距离给父 View，返回值未父 View 是否消耗了这份距离，如果未消耗，当前 View 可选择性绘制边缘阴影等效果

**boolean dispatchNestedFling()** 将 Fling 操作分发到父 View，返回值为是否有支持嵌套滑动的父 View 且该父 View 消耗了这个操作，如果返回 true 那么子 View 将不会处理该操作

**boolean dispatchNestedPreFling()** 在 Fling 操作执行之前将事件分发到父 View，返回值为是否有支持嵌套滑动的父 View 且该父 View 消耗了这个操作，如果返回 true 那么子 View 将不会处理该操作


上面时对两个接口类定义的方法做了简单介绍，在复杂的类关系中，首先将继承关系顶部的类做一点了解，会在分析具体的类时有一点帮助。实现了 NestedScrollingParent 的 ViewGroup 和实现了 NestedScrollingChild 的 ViewGroup 相互嵌套，可以实现父 View 和子 View 同时支持滑动的情况。

接下来分析 Android 提供的一个同时实现了这两个接口的具体的实现类 **NestedScrollView**，由这个类的源码来剖析这两个接口的具体用法。文章开始是的效果也是由两个 NestedScrollView 嵌套实现的。

## 三、NestedScrollView 类源码分析

NestedScrollView 是一个类似于 ScrollView 的类，但是它支持对父 View 和子 View 都可以滑动时的情况的处理。这个类实现了 ScrollingView，所以支持对内容的滑动。

```
public class NestedScrollView extends FrameLayout implements NestedScrollingParent,
        NestedScrollingChild2, ScrollingView {
    // ...
}
```

### （一）NestedScrollView 嵌套滑动时的效果

1. 滑动内部部分时，内部部分会先处理滑动，直到滑倒底部或顶部时，滑动会自然的传递到外部
2. 滑动外部时，内部不受影响


### （二）NestedScrollView 的 onInterceptTouchEvent 方法

滑动事件的处理是从触摸事件中产生的，所以我们从事件的分发方法开始分析 NestedScrollView 类的工作过程。NestedScrollView 没有重写 dispatchTouchEvent 方法，重写了事件拦截和事件处理方法。那我们就从事件拦截开始分析。

```
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
   /*
    * 这个方法决定了是否拦截这个事件，如果需要拦截则返回 true 那么滑动事件将由当前 View 处理
    * 
    * 这里会拦截短距离的反复来回滑动的事件
    */
     
    // 处于滑动模式时直接判断为拦截
    final int action = ev.getAction();
    if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
        return true;
    }

    switch (action & MotionEvent.ACTION_MASK) {
    
        case MotionEvent.ACTION_MOVE: { // MOVE 事件
            // mActivePointerId 是活跃的指针 ID，用来保存在拖拽过程中多个指针的一致性
            final int activePointerId = mActivePointerId;
            if (activePointerId == INVALID_POINTER) {
                // 如果没有有效的 ID，说明事件不存在
                break;
            }

            final int pointerIndex = ev.findPointerIndex(activePointerId);
            if (pointerIndex == -1) {
                Log.e(TAG, "Invalid pointerId=" + activePointerId
                        + " in onInterceptTouchEvent");
                break;
            }

            final int y = (int) ev.getY(pointerIndex);
            // 计算 Y 方向移动的绝对值 
            final int yDiff = Math.abs(y - mLastMotionY);
            
            // 判断滑动距离是否大于最小距离，以及当前 View 滑动方向是否与事件的滑动方向一致
            if (yDiff > mTouchSlop
                    && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
                mIsBeingDragged = true;
                mLastMotionY = y;
                
                // 初始化滑动跟踪器
                initVelocityTrackerIfNotExists();
                mVelocityTracker.addMovement(ev);
                mNestedYOffset = 0;
                final ViewParent parent = getParent();
                
                // 设置父 View 不许拦截之后的事件
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }
            break;
        }

        case MotionEvent.ACTION_DOWN: {
            final int y = (int) ev.getY();
            
            // inChild 用来判断事件是否落在子 View 的区域，这里实现了滑动效果中的第二点
            // 事件在子 View 区域，则将事件传递到子 View
            if (!inChild((int) ev.getX(), y)) { // 如果在子 View 区域，则不拦截
                mIsBeingDragged = false;
                
                // 回收跟踪器
                recycleVelocityTracker();
                break;
            }

            // 事件不在子 View 区域则自己拦截
            // 记录 Y 坐标
            mLastMotionY = y;
            mActivePointerId = ev.getPointerId(0);

            // 初始化或充值跟踪器并将新的事件加入跟踪器
            initOrResetVelocityTracker();
            mVelocityTracker.addMovement(ev);
            
            // 如果用户触摸了屏幕，只有当 mScroller.isFinished() 为 false 时才需要将状态设置为拖动状态
            // OverScroller 用来实现内部内容滑出边界的操作，用来代替 Scroller
            mScroller.computeScrollOffset(); // 计算偏移量
            mIsBeingDragged = !mScroller.isFinished();
            
            // 开始滑动
            startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
            break;
        }

        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP:
            // 释放拖动事件
            mIsBeingDragged = false;
            mActivePointerId = INVALID_POINTER;
            recycleVelocityTracker();
            if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0, getScrollRange())) {
                ViewCompat.postInvalidateOnAnimation(this);
            }
            
            // 停止滚动
            stopNestedScroll(ViewCompat.TYPE_TOUCH);
            break;
        case MotionEvent.ACTION_POINTER_UP:
            // 次要事件的 UP
            onSecondaryPointerUp(ev);
            break;
    }

    /*
    * 唯一需要拦截的情况就是当前 View 正处于滑动模式
    */
    return mIsBeingDragged;
}
```

事件的拦截方法中，主要根据事件的类型及当前 View 的状态决定是否拦截当前事件，在 MOVE 事件时如果当前 View 是拖动状态则会拦截事件，在 DOWN 事件时如果是滑动未结束状态也会拦截，在 UP 时则会停止滑动。其他情况下都会直接不拦截，将事件传递到子 View 中。启动和停止时的逻辑比较简单，现在我们先具体来分析开始滑动和停止滑动时的具体代码。


#### 1. startNestedScroll 方法
NestedScrollView 的 startNestedScroll 方法中，调用了 mChildHelper 对象的 startNestedScroll 方法，这个 mChildHelper 是 NestedScrollingChildHelper 类的对象，这个类的主要指责是帮助绑定的 NestedScrollView 完成滑动的过程。在构造方法中将绑定的 NestedScrollView 保存。下面是 startNestedScroll 方法的代码。

```
// NestedScrollingChildHelper

// 为当前绑定的 View 启动一个嵌套滑动
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
    if (hasNestedScrollingParent(type)) {
        // 有正在滑动的并且支持嵌套滑动的父 View，父 View 会通过 set 系列方法传递到 NestedScrollingChildHelper
        // 如果父 View 存在，则直接返回 true
        return true;
    }
    
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        
        // 向上查找直到找到一个可以合作嵌套滑动的父 View，如果找到则将两个 View 相互绑定
        while (p != null) {
        
            // 调用父类的 onStartNestedScroll 方法通知父 View 开始滑动
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                setNestedScrollingParentForType(type, p);
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    
    // 条件不满足，返回 false
    return false;
}
```

这里同步看一下 NestedScrollView 实现的 NestedScrollingParent 接口的 onStartNestedScroll 中的判断过程,很简单，只是判断了一下滑动方向是否是垂直方向的。

```
@Override
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
}
```

**Tips：在子 View 的事件拦截方法中，ACTION_DOWN 事件到来时要通过 startNestedScroll 方法找到对应的父 View ，并且完成相互绑定的操作**

#### 2. stopNestedScroll 方法

同 startNestedScroll 方法一样，stopNestedScroll 方法也是由 NestedScrollingChildHelper 来实现。停止的方法比较简单，主要就是将滑动状态和滑动过程中的一些变量置空。

```
public void stopNestedScroll(@NestedScrollType int type) {
    ViewParent parent = getNestedScrollingParentForType(type);
    if (parent != null) {
        // 将 parent 置空
        ViewParentCompat.onStopNestedScroll(parent, mView, type);
        // 将滑动状态置空
        setNestedScrollingParentForType(type, null);
    }
}


@Override
public void onStopNestedScroll(View target) {
   // 通过 NestedScrollingParentHelper 将滑动方向置空
    mParentHelper.onStopNestedScroll(target);
    stopNestedScroll();
}

// mParentHelper
public void onStopNestedScroll(@NonNull View target, @NestedScrollType int type) {
    mNestedScrollAxes = 0;
}
```

**Tips：在子 View 的事件拦截方法中，ACTION_UP 事件到来时要通过调用 stopNestedScroll 方法将相互合作 View 解绑，并将一些状态和变量置空**

### （三）NestedScrollView 的 onTouchEvent 方法

分析了事件的拦截，就该分析事件的处理了，这才是重中之重，也是代码量最多的部分

```
@Override
public boolean onTouchEvent(MotionEvent ev) {
    // 检查跟踪器
    initVelocityTrackerIfNotExists();

    MotionEvent vtev = MotionEvent.obtain(ev);

    final int actionMasked = ev.getActionMasked();

    if (actionMasked == MotionEvent.ACTION_DOWN) {
        mNestedYOffset = 0;
    }
    
    // 将 x 轴方向偏移量设置为 0，并更新 y 轴偏移量
    vtev.offsetLocation(0, mNestedYOffset);

    switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: {
            if (getChildCount() == 0) {
                return false;
            }
            
            // 当前 View 的滚动未停止，则不允许父 View 拦截事件
            if ((mIsBeingDragged = !mScroller.isFinished())) {
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }

            // 如果滚动未结束，则需要主动停止滚动
            if (!mScroller.isFinished()) {
                mScroller.abortAnimation();
            }

            // Remember where the motion event started
            mLastMotionY = (int) ev.getY();
            mActivePointerId = ev.getPointerId(0);
            startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
            break;
        }
        case MotionEvent.ACTION_MOVE:
            // ... MOVE 事件
            break;
        case MotionEvent.ACTION_UP:
            final VelocityTracker velocityTracker = mVelocityTracker;
            velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
            int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);
            
            // 手指抬起时计算是否需要激活 fling 状态
            // mMinimumVelocity 是系统识别 fling 状态的最短移动距离
            if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                flingWithNestedDispatch(-initialVelocity);
            } else if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0,
                    getScrollRange())) {
                ViewCompat.postInvalidateOnAnimation(this);
            }
            mActivePointerId = INVALID_POINTER;
            
            // 结束拖动
            endDrag();
            break;
        case MotionEvent.ACTION_CANCEL: // 取消
            if (mIsBeingDragged && getChildCount() > 0) {
                if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0,
                        getScrollRange())) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
            }
            mActivePointerId = INVALID_POINTER;
            endDrag();
            break;
        case MotionEvent.ACTION_POINTER_DOWN: { // 次要手指按下，更新 Y 值和触摸 ID
            final int index = ev.getActionIndex();
            mLastMotionY = (int) ev.getY(index);
            mActivePointerId = ev.getPointerId(index);
            break;
        }
        case MotionEvent.ACTION_POINTER_UP: // 次要手指抬起
            onSecondaryPointerUp(ev);
            mLastMotionY = (int) ev.getY(ev.findPointerIndex(mActivePointerId));
            break;
    }

    if (mVelocityTracker != null) {
        mVelocityTracker.addMovement(vtev);
    }
    vtev.recycle();
    return true;
}
```

#### 1. 滑动过程中 ACTION_DOWN 和 ACTION_UP 事件的处理

按下和抬起事件比较简单，需要注意的就是抬起的时候，需要将正在执行的滚动动画停止，抬起的时候还需要判断是否需要启动 fling 滑动状态。fling 效果的实现代码如下：

```
// NestedScrollView
private void flingWithNestedDispatch(int velocityY) {
    final int scrollY = getScrollY();
    
    // 根据滑动状态判断是否可以 fling
    final boolean canFling = (scrollY > 0 || velocityY > 0)
            && (scrollY < getScrollRange() || velocityY < 0);
    
    // dispatchNestedPreFling 分发 preFling 操作
    if (!dispatchNestedPreFling(0, velocityY)) { // 分发 preFling
        // 如果在 PreFling 中没有全部消耗滑动，则需要当前 View 继续分发 Fling 事件
        
        // 分发 Fling 到父 View 
        dispatchNestedFling(0, velocityY, canFling);

        // 处理剩余的 Fling 距离
        fling(velocityY);
    }
}

// NestedScrollingChildHelper
public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
    if (isNestedScrollingEnabled()) {
        // 将 preFling 事件分发给支持嵌套滑动的父 View
        ViewParent parent = getNestedScrollingParentForType(TYPE_TOUCH);
        if (parent != null) {
            return ViewParentCompat.onNestedPreFling(parent, mView, velocityX,
                    velocityY);
        }
    }
    return false;
}

// NestedScrollView
@Override
public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
    // 再次分发事件到支持嵌套滑动的父 View
    return dispatchNestedPreFling(velocityX, velocityY);
}
```

由代码可以看出，如果当前状态支持 Fling 动画，则会先向上分发 preFling 到支持嵌套滑动的父 View，如果没有父 View 消耗这个操作，就继续将 fling 事件分发到支持嵌套滑动的父 View，如果 preFling 事件被父 View 接受，则当前 View 就不需要处理 fling 操作。

这两个事件分发之后，如果父 View 没有完全消耗 fling 操作，就会调用当前 View 的 fling 方法由当前 View 处理滑动。OverScroller 的工作类似 Scroller，之前分析过，这里就不多说了，如果不清楚的可以看一下之前 Scroll 滑动的文章。

**Tips：在子 View 的 onTouchEvent 中，ACTION_UP 事件到来时，需要考虑 Fling 效果的处理**

```
// NestedScrollView
public void fling(int velocityY) {
    if (getChildCount() > 0) {
        startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_NON_TOUCH);
        
        // OverScroller 的 fling 方法会根据当前速度和方向，执行 View 内容的滑动
        mScroller.fling(getScrollX(), getScrollY(), // start
                0, velocityY, // velocities
                0, 0, // x
                Integer.MIN_VALUE, Integer.MAX_VALUE, // y
                0, 0); // overscroll
        mLastScrollerY = getScrollY();
        ViewCompat.postInvalidateOnAnimation(this);
    }
}
```

#### 2. 滑动过程中 ACTION_MOVE 事件的处理

```
// NestedScrollView 的 onTouchEvent 方法中 ACTION_MOVE 事件对应代码
{
    final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
    if (activePointerIndex == -1) { // 指针无效
        break;
    }

    final int y = (int) ev.getY(activePointerIndex);
    
    // deltaY 为需要滑动到位置的坐标
    int deltaY = mLastMotionY - y;
    
    // 分发滑动发生前的 preScroll 操作
    if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset,
            ViewCompat.TYPE_TOUCH)) {
        // 如果父 View 消耗该操作，则计算除父 View 消耗的剩余的偏移量
        // NestedScrollView 作为父 View 时不消耗
        deltaY -= mScrollConsumed[1];
        vtev.offsetLocation(0, mScrollOffset[1]);
        mNestedYOffset += mScrollOffset[1];
    }
    
    // 如果满足条件则将状态置为拖动，并且设置之后的事件父 View 不可以拦截
    if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
        final ViewParent parent = getParent();
        if (parent != null) {
            parent.requestDisallowInterceptTouchEvent(true);
        }
        mIsBeingDragged = true;
        if (deltaY > 0) {
            deltaY -= mTouchSlop;
        } else {
            deltaY += mTouchSlop;
        }
    }
    if (mIsBeingDragged) {
        // 滑动到指定位置
        
        mLastMotionY = y - mScrollOffset[1];

        final int oldY = getScrollY();
        final int range = getScrollRange();
        final int overscrollMode = getOverScrollMode();
        boolean canOverscroll = overscrollMode == View.OVER_SCROLL_ALWAYS
                || (overscrollMode == View.OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

        // 关键点，overScrollByCompat 方法中当前 NestedScrollView 会执行 onOverScrolled 方法，其中执行 scrollTo 方法，完成内部内容的滑动
        if (overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0,
                0, true) && !hasNestedScrollingParent(ViewCompat.TYPE_TOUCH)) {
            // Break our velocity if we hit a scroll barrier.
            mVelocityTracker.clear();
        }

        final int scrolledDeltaY = getScrollY() - oldY;
        
        // 计算剩余需要需要滑动的距离，也就是当前 NestedScrollView 没有消耗的距离，会分发给父 View
        final int unconsumedY = deltaY - scrolledDeltaY;
        
        // 关键点，将自己为消耗的滑动分发到父 View
        // 一般 unconsumedY 不为 0 的情况为，当前 NestedScrollView 已经滑动到了边缘，不能再继续处理滑动事件，这时候需要将为消耗的距离分发到父 View
        if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset,
                ViewCompat.TYPE_TOUCH)) {
            mLastMotionY -= mScrollOffset[1];
            vtev.offsetLocation(0, mScrollOffset[1]);
            mNestedYOffset += mScrollOffset[1];
        } else if (canOverscroll) { // 子 View 滑到边缘且父 View 未消耗剩余的滑动距离时绘制 OverScroll 效果
            ensureGlows();
            final int pulledToY = oldY + deltaY;
            if (pulledToY < 0) {
                EdgeEffectCompat.onPull(mEdgeGlowTop, (float) deltaY / getHeight(),
                        ev.getX(activePointerIndex) / getWidth());
                if (!mEdgeGlowBottom.isFinished()) {
                    mEdgeGlowBottom.onRelease();
                }
            } else if (pulledToY > range) {
                EdgeEffectCompat.onPull(mEdgeGlowBottom, (float) deltaY / getHeight(),
                        1.f - ev.getX(activePointerIndex)
                                / getWidth());
                if (!mEdgeGlowTop.isFinished()) {
                    mEdgeGlowTop.onRelease();
                }
            }
            if (mEdgeGlowTop != null
                    && (!mEdgeGlowTop.isFinished() || !mEdgeGlowBottom.isFinished())) {
                ViewCompat.postInvalidateOnAnimation(this);
            }
        }
    }
}
```

如果由当前的 NestedScrollView 处理滑动，在 MOVE 事件发生时，会调用 dispatchNestedPreScroll 方法，其中会调用 NestedScrollingChildHelper 的 dispatchNestedPreScroll 方法。其作用为，将滑动产生前的 preScroll 操作分发到当前 View 的父 View。

```
// NestedScrollingChildHelper
public boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed,
        @Nullable int[] offsetInWindow, @NestedScrollType int type) {
    if (isNestedScrollingEnabled()) {
        final ViewParent parent = getNestedScrollingParentForType(type);
        if (parent == null) {
            return false;
        }

        if (dx != 0 || dy != 0) {
            int startX = 0;
            int startY = 0;
            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }

            if (consumed == null) {
                if (mTempNestedScrollConsumed == null) {
                    mTempNestedScrollConsumed = new int[2];
                }
                consumed = mTempNestedScrollConsumed;
            }
            consumed[0] = 0;
            consumed[1] = 0;
            
            // 关键点，将 preScroll 操作分发到父 View
            ViewParentCompat.onNestedPreScroll(parent, mView, dx, dy, consumed, type);

            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            
            // 根据 consumed 的值判断父 View 是否消耗
            return consumed[0] != 0 || consumed[1] != 0;
        } else if (offsetInWindow != null) {
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    
    // 默认父 View 不消耗该事件
    return false;
}


// ViewParentCompat.onNestedPreScroll 方法最终会调用到实现了 NestedScrollingParent 接口的父 View 的 onNestedPreScroll 方法
// 这里也就是 NestedScrollView 的 onNestedPreScroll 方法
@Override
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    // 继续将事件向上分发
    dispatchNestedPreScroll(dx, dy, consumed, null);
}
```
父 View 的 onNestedPreScroll 方法很关键，其主要作用就是由父 View 计算是否需要消耗该事件，如果需要消耗，则将需要消耗的值写入 consumed 数组，子 View 会根据 consumed 数组的值是否改变来判断父 View 是否消耗，并将判断返回。**这里 NestedScrollView 作为父 View 是不消耗滑动事件的。所以会返回 false。如果自定义支持嵌套滑动的父 View 则需要根据业务来自己实现 onNestedPreScroll 方法**


处理完 preScroll 操作后，如果当前 NestedScrollView 进入了拖拽状态，就会先计算可以滑动的范围大小等数据，然后调用 overScrollByCompat 方法，这个方法中调用了 onOverScrolled 这个方法。onOverScrolled 非常重要，因为在这个方法中调用了真正执行内容滑动的代码。调用 overScrollByCompat 方法时会将需要滑动的距离通过参数的形式传递。

```
// NestedScrollView 

// 重要方法，其中会调用 onOverScrolled
boolean overScrollByCompat(int deltaX, int deltaY,
        int scrollX, int scrollY,
        int scrollRangeX, int scrollRangeY,
        int maxOverScrollX, int maxOverScrollY,
        boolean isTouchEvent) {
    final int overScrollMode = getOverScrollMode();
    final boolean canScrollHorizontal =
            computeHorizontalScrollRange() > computeHorizontalScrollExtent();
    final boolean canScrollVertical =
            computeVerticalScrollRange() > computeVerticalScrollExtent();
    final boolean overScrollHorizontal = overScrollMode == View.OVER_SCROLL_ALWAYS
            || (overScrollMode == View.OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollHorizontal);
    final boolean overScrollVertical = overScrollMode == View.OVER_SCROLL_ALWAYS
            || (overScrollMode == View.OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollVertical);

    int newScrollX = scrollX + deltaX;
    if (!overScrollHorizontal) {
        maxOverScrollX = 0;
    }

    int newScrollY = scrollY + deltaY;
    if (!overScrollVertical) {
        maxOverScrollY = 0;
    }

    // Clamp values if at the limits and record
    final int left = -maxOverScrollX;
    final int right = maxOverScrollX + scrollRangeX;
    final int top = -maxOverScrollY;
    final int bottom = maxOverScrollY + scrollRangeY;

    boolean clampedX = false;
    if (newScrollX > right) {
        newScrollX = right;
        clampedX = true;
    } else if (newScrollX < left) {
        newScrollX = left;
        clampedX = true;
    }

    boolean clampedY = false;
    if (newScrollY > bottom) {
        newScrollY = bottom;
        clampedY = true;
    } else if (newScrollY < top) {
        newScrollY = top;
        clampedY = true;
    }

    if (clampedY && !hasNestedScrollingParent(ViewCompat.TYPE_NON_TOUCH)) {
        mScroller.springBack(newScrollX, newScrollY, 0, 0, 0, getScrollRange());
    }

    // 关键方法 
    onOverScrolled(newScrollX, newScrollY, clampedX, clampedY);

    return clampedX || clampedY;
}

// NestedScrollView 作为子 View 时对内部内容进行滑动
@Override
protected void onOverScrolled(int scrollX, int scrollY,
        boolean clampedX, boolean clampedY) {
        
    // 滑动操作
    super.scrollTo(scrollX, scrollY);
}
```

当前 NestedScrollView 完成对其内容的滑动后，接着计算除自己消耗 和已经产生的滑动之外还需要滑动的距离，然后调用 dispatchNestedScroll 方法，该方法中会调用 NestedScrollingChildHelper 的 dispatchNestedScroll 方法。

```
// NestedScrollingChildHelper

// 分发嵌套滑动进度中的一个到父 View
// 参数只需要重点关注 dyUnconsumed 为需要滑动的距离
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
        int dxUnconsumed, int dyUnconsumed, @Nullable int[] offsetInWindow,
        @NestedScrollType int type) {
    if (isNestedScrollingEnabled()) {
        final ViewParent parent = getNestedScrollingParentForType(type);
        if (parent == null) {
            return false;
        }

        // 
        if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
            int startX = 0;
            int startY = 0;
            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }
            
            // 向父 View 分发需要滑动事件，传递需要滑动的距离
            ViewParentCompat.onNestedScroll(parent, mView, dxConsumed,
                    dyConsumed, dxUnconsumed, dyUnconsumed, type);

            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            
            // 如果父 View 消耗了滑动，则返回 true
            return true;
        } else if (offsetInWindow != null) {
            // No motion, no dispatch. Keep offsetInWindow up to date.
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}

// NestedScrollView 作为 父 View 时 onNestedScroll 被调用
@Override
public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
        int dyUnconsumed) {
    final int oldScrollY = getScrollY();
    // 终于找到啦，这么大半天，终于找到了滑动的方法
    // scrollBy 方法会根据当前已经滑动的距离滑动指定的长度
    scrollBy(0, dyUnconsumed);
    
    // 计算自己已完成的滑动，计算未完成的滑动，在内部的内容已经滑倒边缘但还未消耗完产生的滑动时，会出现这种情况，还需要向父 View 分发未消耗的距离
    final int myConsumed = getScrollY() - oldScrollY;
    final int myUnconsumed = dyUnconsumed - myConsumed;
    dispatchNestedScroll(0, myConsumed, 0, myUnconsumed, null);
}
```

NestedScrollingChildHelper 的 dispatchNestedScroll 方法中，会判断是否有未消耗的滑动，如果有，则会将滑动分发到父 View，并且返回父 View 消耗了该距离。如果父 View 消耗，则当前 View 不会处理该滑动。如果父 View 不消耗，则子 View 会绘制相应的 OverScroll 阴影等内容。

**一般情况下如果有相互合作的父 View，父 View
会消耗这个距离，也就是上面的代码中表现的，父 View 的 onNestedScroll 方法中同样会调用自己本身的 scrollBy 方法，进行自己内容的滑动。这样就实现了当子 View 中的内容滑动到自身边缘时，父 View 可以无缝的接管未消耗的滑动，实现了我们开始时提到的嵌套滑动的第一点效果。**

到这里，NestedScrollView 这个类以及相关联的 NestedScrollingParent、NestedScrollingChild 这两个接口的工作过程就分析完了。

## 三、延伸

通过对 NestedScrollView 类的分析，我们发现，大多数的事件都是会先分发到子 View，再由子 View 通过接口方法分发事件到父 View 或者自己处理，并且通过这些接口方法的返回值子 View 也能直到父 View 是否完全消耗了滑动的距离。

这样，在实现了 NestedScrollingParent 接口的父 View 中就可以通过重写对应的方法拦截或消耗一部分滑动，而实现了 NestedScrollingChild 接口的子 View 也可以通过重写对应的方法来处理滑动事件。两个接口的实现类相互合作，最终就可以实现任意的滑动效果。

其实通过看源码，我们会发现 SwipeRefreshLayout、RecyclerView、CoordinatorLayout 等 support 包的很多类都是实现了上面的一个或者两个接口，并且在实对应接口方法的过程中定义自己的逻辑。

这里我们可以再次梳理一下这些接口方法的调用逻辑，再遇到类似需求时就可以直接套用。

1. 当事件发生时，只要 Parent 不在滑动状态，那么所有事件都交给 Child 来处理

2. 在 Child 的 onInterceptTouchEvent 方法中，DOWN 事件来临时要通过调用 startNestedScroll 方法找到对应的 Parent ，并且通过调用 Parent 的 onStartNestedScroll 和 onNestedScrollAccepted 开启整个滑动，并且完成相互绑定的操作

3. 在 Child 的 onInterceptTouchEvent 方法中，ACTION_UP 事件到来时要通过调用 stopNestedScroll 方法将相互合作 Parent 解绑，并将一些状态和变量置空，然后调用 Parent 的 onStopNestedScroll 方法，Parent 的 onStopNestedScroll 方法中也是执行清理一些数据的操作

4. 在 Child 的 onInterceptTouchEvent 方法中，ACTION_UP 事件时还需要判断是否需要启动 Fling 效果。如果满足条件，可以先通过调用 Child 的 dispatchNestedPreFling 方法来分发事件，一般这个方法中会调用 Parent 的 onNestedPreFling 方法，让 Parent 先进行处理。如果未将滑动完全消耗，就调用 Child 的 dispatchNestedFling 方法。该方法中会通过 Parent 的 onNestedFling 方法和 Child 的代码来实现想要的效果

5. 在 Child 的 onInterceptTouchEvent 方法中，ACTION_MOVE 事件到来时，通过 Child 的 dispatchNestedPreScroll 方法，现将事件分发到 Parent 的 onNestedPreFling 方法中，Parent 可以按需求消耗一部分

6. 如果 Parent 的 onNestedPreFling 中没有将事件完全消耗，那么 Child 就可以按需求进行滑动了，在 Child 滑动结束后如果为将距离完全消耗，还可以通过在 Child 的 dispatchNestedScroll 方法中调用 Parent 的 onNestedScroll 方法将剩余的距离传入 Parent 由 Parent 进行消耗

7. 通过这几个接口方法的配合使用完成想要的效果

## 四、总结

延伸部分其实就是抽象了 NestedScrollView 类的实现过程。在开发中，我们可以根据需求，灵活的搭配这两个接口的所有方法。没有固定的前后顺序。

到这里 support 包中的 NestedScroll 机制就说完了～


