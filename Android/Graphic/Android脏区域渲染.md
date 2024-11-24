# 脏区域渲染

## 1、背景

Android的渲染流程中，具体的渲染区域的解析，涉及脏区域计算 与 对应的GPU绘制指令的转化，从一个view调用invalidate开始



## 2、脏区域渲染流程

### 2.1、Before Vsync

从用户的invalidate开始：

```java
//core\java\android\view\View.java
public void invalidate() {
    invalidate(true);
}

//core\java\android\view\View.java
@UnsupportedAppUsage
public void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}

//core\java\android\view\View.java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    //...
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        //标记该view为dirty
        mPrivateFlags |= PFLAG_DIRTY;
        if (invalidateCache) {
            //为true
            mPrivateFlags |= PFLAG_INVALIDATED;
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }
        //将自己的脏区域传递给父view
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            //该view当前在屏幕上，并且该view所在区域是有效的(宽高不为0)
            final Rect damage = ai.mTmpInvalRect;
           	//设置当前的脏区域
            damage.set(l, t, r, b);
            //传递给父view
            p.invalidateChild(this, damage);
        }
        // 后面为投影机制相关的代码，先忽略 ...
    }
}
```

**阶段1：View调用invalidate后，自己的标志位会加上PFLAG_DIRTY、PFLAG_INVALIDATED，并去除标志位PFLAG_DRAWING_CACHE_VALID，然后将自己的脏区域传递给父view，** 

父view即以后的脏区域传递

```java
//core\java\android\view\ViewGroup.java
public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null && attachInfo.mHardwareAccelerated) {
        // HW accelerated fast path
        onDescendantInvalidated(child, child);
        return;
    }
    //非硬件加速相关的，软件层绘制基本已经不适用，先忽略，主要关注硬件渲染相关的操作
    //...
}

//core\java\android\view\ViewGroup.java
public void onDescendantInvalidated(@NonNull View child, @NonNull View target) {
    /*
     * HW-only(只有硬件加速才调用)
     * 在主线程不会处理脏区域，因为RenderThread会在native层计算每一帧的脏区域
     * We don't deal with rectangles here, since RenderThread native code computes damage foreverything drawn by HWUI (and SW layer / drawing cache doesn't keep track of damage area)
     */
    // 属性动画相关
    mPrivateFlags |= (target.mPrivateFlags & PFLAG_DRAW_ANIMATION);

    if ((target.mPrivateFlags & ~PFLAG_DIRTY_MASK) != 0) {
        //如果这个view设置的flag 包含了其他属性的flag，则重新设置下PFLAG_DIRTY标志位并把PFLAG_DRAWING_CACHE_VALID标志位给去掉，这个标志位注释的意思是确保安全这里可以先忽略
        // We lazily use PFLAG_DIRTY, since computing opaque isn't worth the potential optimization in provides in a DisplayList world.
        mPrivateFlags = (mPrivateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DIRTY;
        // simplified invalidateChildInParent behavior: clear cache validity to be safe...
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
    }
    //非硬件渲染则直接忽略
    if (mParent != null) {
        mParent.onDescendantInvalidated(this, target);
    }
}
```

**阶段2：脏区域传递给父view后，父view也会设置为PFLAG_DIRTY，并去除PFLAG_DRAWING_CACHE_VALID标志位**

继续向上会通过onDescendantInvalidated一路的传递，最终会到达ViewRootImpl,  而且onDescendantInvalidated的第二个参数，target一直会为最开始发起invalidate的view。



```java
//core\java\android\view\ViewRootImpl.java
@Override
public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
    // TODO: Re-enable after camera is fixed or consider targetSdk checking this
    // checkThread();
    if ((descendant.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
        //属性动画相关
        mIsAnimating = true;
    }
    invalidate();
}

@UnsupportedAppUsage
void invalidate() {
    //标记脏区域
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        //注册Vsync通知，等待绘制
        scheduleTraversals();
    }
}
```

**阶段3：invalidate后，对于主线程，java层的脏区域渲染已经标记完成了，脏区域会被记录到ViewRootImpl的mDirty变量中，并且等待Vsync信号到来开始真正的渲染。**



**已完成的阶段总结：**

* **阶段1：View调用invalidate后，自己的标志位会加上PFLAG_DIRTY、PFLAG_INVALIDATED，并去除标志位PFLAG_DRAWING_CACHE_VALID，然后将自己的脏区域传递给父view，** 
* **阶段2：脏区域传递给父view后，父view也会设置为PFLAG_DIRTY，并去除PFLAG_DRAWING_CACHE_VALID标志位**
* **阶段3：invalidate后，对于主线程，java层的脏区域渲染已经标记完成了，脏区域会被记录到ViewRootImpl的mDirty变量中，并且等待Vsync信号到来开始真正的渲染。**



### 2.2、Vsync Come

#### 2.2.1、ViewRootImpl.performDraw

vsync来临，一切的渲染都要从ViewRootImpl的performDraw开始：

```java
//core\java\android\view\ViewRootImpl.java
private void performDraw() {
    //...
    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
	//部分注册帧渲染完成的监听器代码先忽略
    try {
        //fullRedrawNeeded一般情况下位false，我们只关注fullRedrawNeeded=false的case
        boolean canUseAsync = draw(fullRedrawNeeded);
        if (usingAsyncReport && !canUseAsync) {
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
            usingAsyncReport = false;
        }
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    //帧渲染的绘制监听相关的代码，可能与开发者设置的某些渲染监听有关，先忽略..
}


//core\java\android\view\ViewRootImpl.java
private boolean draw(boolean fullRedrawNeeded) {
    //忽略surface有效性检测、fps监听、viewrootimpl首次绘制监听、mTreeObserver.dispatchOnScrollChanged滑动监听等代码
    final Rect dirty = mDirty;
	//...
    if (fullRedrawNeeded) {
        //一般情况下fullRedrawNeeded位false，可以看到如果为true，脏区域会被标记为整个窗口的大小
        dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
    }
    //分发OnDraw监听
    mAttachInfo.mTreeObserver.dispatchOnDraw();
 	//...
    boolean accessibilityFocusDirty = false;
    //mAccessibility相关代码忽略
    //记录开始绘制时间戳
    mAttachInfo.mDrawingTime =
            mChoreographer.getFrameTimeNanos() / TimeUtils.NANOS_PER_MS;
    boolean useAsyncReport = false;
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            //忽略一些全局更新的代码...
            //这列可以看到dirty脏区域这里被置空了，说明ViewRootImpl的mDirty的脏区域记录并没有在真正绘制时生效
            dirty.setEmpty();
            useAsyncReport = true;
            //mView为DecorView
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
            //异常场景与软件层渲染代码忽略 ...
        }
    }
    //动画情况下注册下次的vsync回调，并标记为全屏重新渲染，忽略相关代码
    return useAsyncReport;
}
```

**阶段1：在接收sync信号以后，ViewRootImpl会触发真正的绘制流程，而ViewRootImpl记录的mDirty脏区域在调用mAttachInfo.mThreadedRenderer.draw之前就被清除掉了，只是做了是否有脏区域的判断，看后脏区域的计算还是跟 mAttachInfo.mThreadedRenderer.draw后的流程有关**



```java
//core\java\android\view\ThreadedRenderer.java
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
    choreographer.mFrameInfo.markDrawStart();
    //更新SkiaRecordingCanavas的mStagingDisplayList的过程
    updateRootDisplayList(view, callbacks);
    //RenderThread prepare View的过程
    int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
    //syncResult=false， 与RenderThread同步失败的case忽略
}
```

到这里有两个核心的操作，一个是updateRootDisplayList， 更新SkiaRecordingCanavas的mStagingDisplayList的过程，是由主线程来完成的，另一个是syncAndDrawFrame， renderThread更新DisplayList的过程，实际工作是renderThread来完成的，主线程在这个同步过程中是需要阻塞等待的。



#### 2.2.2、ThreadedRenderer.draw

##### 2.2.2.1、updateRootDisplayList

先关注第一个过程，由主线程完成

```java
//core\java\android\view\ThreadedRenderer.java
private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
    updateViewTreeDisplayList(view);
    //涉及需要全局更新 以及 mRootNode对应的DisplayList为空需要创建的代码先忽略
    //主要是mRootNode.beginRecording跟mRootNode.endRecording()的操作，属于硬件加速相关的流程解析
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}

//core\java\android\view\ThreadedRenderer.java
private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    //decorview只设置了PFLAG_DIRTY标志位，没有PFLAG_INVALIDATED标志位，所以mRecreateDisplayList=false
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}
```

给DecorView标记的标志位有View.PFLAG_DRAWN， PFLAG_INVALIDATED标志位可以先忽略

```java
//core\java\android\view\View.java
@NonNull
@UnsupportedAppUsage
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    //...
    //PFLAG_DRAWING_CACHE_VALID标志位是没有设置的
    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0 /*符合条件*/
            || !renderNode.hasDisplayList() /*非第一次更新，不符合条件*/
            || (mRecreateDisplayList) /*不符合条件*/) {
        // Don't need to recreate the display list, just need to tell our
        // children to restore/recreate theirs
        if (renderNode.hasDisplayList()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            dispatchGetDisplayList();

            return renderNode; // no work needed
        }
        //重建DisplayList相关逻辑忽略...
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}

//core\java\android\view\View.java
/**
 * This method is used by ViewGroup to cause its children to restore or recreate their
 * display lists. It is called by getDisplayList() when the parent ViewGroup does not need
 * to recreate its own display list, which would happen if it went through the normal
 * draw/dispatchDraw mechanisms.
 *
 * @hide
 */
protected void dispatchGetDisplayList() {}
```

ViewGroup获取dispatchGetDisplayList

```java
//core\java\android\view\ViewGroup.java
/**
 * This method is used to cause children of this ViewGroup to restore or recreate their
 * display lists. It is called by getDisplayList() when the parent ViewGroup does not need
 * to recreate its own display list, which would happen if it went through the normal
 * draw/dispatchDraw mechanisms.
 *
 * @hide
 */
@Override
@UnsupportedAppUsage
protected void dispatchGetDisplayList() {
    final int count = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < count; i++) {
        final View child = children[i];
        if (((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null)) {
            //可见或者做动画的子view
            recreateChildDisplayList(child);
        }
    }
    //...
    if (mDisappearingChildren != null) {
        //消失的DisplayList
        final ArrayList<View> disappearingChildren = mDisappearingChildren;
        final int disappearingCount = disappearingChildren.size();
        for (int i = 0; i < disappearingCount; ++i) {
            final View child = disappearingChildren.get(i);
            recreateChildDisplayList(child);
        }
    }
}
```

ViewGroup.dispatchGetDisplayList核心就是分发获取所有可见的字view以及变化的子view的DisplayList

```java
//core\java\android\view\ViewGroup.java
private void recreateChildDisplayList(View child) {
    //这里只有调用了invalidate的View的PFLAG_INVALIDATED才会被置1
    child.mRecreateDisplayList = (child.mPrivateFlags & PFLAG_INVALIDATED) != 0;
    child.mPrivateFlags &= ~PFLAG_INVALIDATED;
    child.updateDisplayListIfDirty();
    child.mRecreateDisplayList = false;
}
```

**这里又回到了View.updateDisplayListIfDirty()方法，注意前面我们将Vysnc到来前的准备阶段，只有调用invaliate的View PFLAG_INVALIDATED标志位才会为true，其他的为false，一旦PFLAG_INVALIDATED为true，就会导致mRecreateDisplayList为true。**

回到View.updateDisplayListIfDirty()

```java
public RenderNode updateDisplayListIfDirty() {
    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.hasDisplayList()
            || (mRecreateDisplayList) /*invalidate的view 这个为true*/) {
        // Don't need to recreate the display list, just need to tell our
        // children to restore/recreate theirs
        if (renderNode.hasDisplayList()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            dispatchGetDisplayList();
            //对于invalidate的View不会走到这段逻辑需要重建DisplayList
            return renderNode; // no work needed
        }
        mRecreateDisplayList = true;
        int width = mRight - mLeft;
        int height = mBottom - mTop;
        int layerType = getLayerType();

        final RecordingCanvas canvas = renderNode.beginRecording(width, height);
        try {
            if (layerType == LAYER_TYPE_SOFTWARE) {
                //软件渲染的case先忽略
            } else {
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                //去除PFLAG_DIRTYdirty标志位
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                // Fast path for layouts with no backgrounds
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    //忽略跳过绘制的case
                } else {
                    draw(canvas);
                }
            }
        } finally {
            renderNode.endRecording();
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```

先看 renderNode.beginRecording实际是获取了RecordingCanvas对象





##### 2.2.2.2、syncAndDrawFrame



