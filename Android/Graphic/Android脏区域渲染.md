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
//core\java\android\view\View.java
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

先看 renderNode.beginRecording实际是获取了RecordingCanvas对象, RecordingCanvas对应native层的SkiaRecordingCanvas。

View.updateDisplayListIfDirty()具体做的事情可以总结为

* 获取RecordingCanvas，对应native层为SkiaRecordingCanvas
* 调用View的draw方法，实际是王SkiaRecordingCanvas中去更新DisplayListData
* 调用renderNode.endRecording通知view的绘制结束

```java
//core\java\android\view\View.java
@CallSuper
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    //当前View新增PFLAG_DRAWN标志位
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;
	//绘制背景
    drawBackground(canvas);

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    //横向边缘渐隐效果，对应接口为setVerticalFadingEdgeEnabled
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        //大部分的case都是走这个分支，处分设置setVerticalFadingEdgeEnabled(true)
        //或者setHorizontalFadingEdgeEnabled(true)
        // Step 3, draw the content
        onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);

        if (debugDraw()) {
            debugDrawFocus(canvas);
        }
        // we're done...
        return;
    }
    //忽略绘制边缘渐隐效果的case
}
```

我们关注 step3 draw the content, 其实就是调用View.onDraw(canvas) 将绘制命令记录在DisPlayList里面，我们以ImageView为例：

```java
//core\java\android\widget\ImageView.java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    if (mDrawMatrix == null && mPaddingTop == 0 && mPaddingLeft == 0) {
        mDrawable.draw(canvas);
    } else {
        //...
    }
}
```

mDrawable假设为BitmapDrawable

```java
//graphics\java\android\graphics\drawable\BitmapDrawable.java
@Override
public void draw(Canvas canvas) {
    final Bitmap bitmap = mBitmapState.mBitmap;
    //...
    final Shader shader = paint.getShader();
    final boolean needMirroring = needMirroring();
    if (shader == null) {
        //核心方法
        canvas.drawBitmap(bitmap, null, mDstRect, paint);
    } else {
        //...
    }

}
```

实际这里调用了canvas.drawBitmap(bitmap, null, mDstRect, paint);

canvas是从renderNode.beginRecording(width, height)获取的，实际为RecordingCanavas对象，对应的Native对象为SkiaRecordingCanavas

```java
//graphics\java\android\graphics\BaseCanvas.java
public void drawBitmap(@NonNull Bitmap bitmap, @NonNull Matrix matrix, @Nullable Paint paint) {
    throwIfHasHwBitmapInSwMode(paint);
    nDrawBitmapMatrix(mNativeCanvasWrapper, bitmap.getNativeInstance(), matrix.ni(),
            paint != null ? paint.getNativeInstance() : 0);
}
```

```c++
//core\jni\android_graphics_Canvas.cpp
static void drawBitmapMatrix(JNIEnv* env, jobject, jlong canvasHandle, jlong bitmapHandle,
                             jlong matrixHandle, jlong paintHandle) {
    const SkMatrix* matrix = reinterpret_cast<SkMatrix*>(matrixHandle);
    const Paint* paint = reinterpret_cast<Paint*>(paintHandle);
    Bitmap& bitmap = android::bitmap::toBitmap(bitmapHandle);
    get_canvas(canvasHandle)->drawBitmap(bitmap, *matrix, paint);
}
```

可以看到从这里开始，就从java层转到了Native层

```c++
//libs\hwui\pipeline\skia\SkiaRecordingCanvas.cpp
void SkiaRecordingCanvas::drawBitmap(Bitmap& bitmap, const SkMatrix& matrix, const SkPaint* paint) {
    SkAutoCanvasRestore acr(&mRecorder, true);
    concat(matrix);
    //bitmap转为SkImage对象
    sk_sp<SkImage> image = bitmap.makeImage();
    //mRecorder为RecordingCanvas对象
    mRecorder.drawImage(image, 0, 0, filterBitmap(paint), bitmap.palette());
    if (!bitmap.isImmutable() && image.get() && !image->unique()) {
        mDisplayList->mMutableImages.push_back(image.get());
    }
}

//libs\hwui\RecordingCanvas.cpp
void RecordingCanvas::drawImage(const sk_sp<SkImage>& image, SkScalar x, SkScalar y,
                                const SkPaint* paint, BitmapPalette palette) {
    fDL->drawImage(image, x, y, paint, palette);
}

//libs\hwui\RecordingCanvas.cpp
void DisplayListData::drawImage(sk_sp<const SkImage> image, SkScalar x, SkScalar y,
                                const SkPaint* paint, BitmapPalette palette) {
    this->push<DrawImage>(0, std::move(image), x, y, paint, palette);
}
```

canvas.drawBitmap最终将一个drawImage操作存储到了DisplayListData当中。



再看下
renderNode.endRecording();
setDisplayListProperties(renderNode);

```java
//graphics\java\android\graphics\RenderNode.java
/**
 * `
 * Ends the recording for this display list. Calling this method marks
 * the display list valid and {@link #hasDisplayList()} will return true.
 *
 * @see #beginRecording(int, int)
 * @see #hasDisplayList()
 */
public void endRecording() {
    //...
    RecordingCanvas canvas = mCurrentRecordingCanvas;
    mCurrentRecordingCanvas = null;
    //获取到SkiaRecodingCanvas中的displayList指针地址
    long displayList = canvas.finishRecording();
    nSetDisplayList(mNativeRenderNode, displayList);
    canvas.recycle();
}
```

```c++
//libs\hwui\pipeline\skia\SkiaRecordingCanvas.cpp
uirenderer::DisplayList* SkiaRecordingCanvas::finishRecording() {
    // close any existing chunks if necessary
    insertReorderBarrier(false);
    //减少一个计数状态
    mRecorder.restoreToCount(1);
    //从原有的持有者中释放该引用，并将指针返回
    return mDisplayList.release();
}
```

nSetDisplayList

```c++
//core\jni\android_view_RenderNode.cpp
static void android_view_RenderNode_setDisplayList(JNIEnv* env,
        jobject clazz, jlong renderNodePtr, jlong displayListPtr) {
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    DisplayList* newData = reinterpret_cast<DisplayList*>(displayListPtr);
    renderNode->setStagingDisplayList(newData);
}

//libs\hwui\RenderNode.cpp
void RenderNode::setStagingDisplayList(DisplayList* displayList) {
    mValid = (displayList != nullptr);
    mNeedsDisplayListSync = true;
    delete mStagingDisplayList;
    mStagingDisplayList = displayList;
}
```

最终将displayList存到了RenderNode.mStagingDisplayList当中， 每个RenderNode都会有自己的DisplayList

canvas.recycle()

```java
//graphics\java\android\graphics\RecordingCanvas.java
/** @hide */
void recycle() {
    //将RecordingCanvas当前操作的RenderNode引用置空
    mNode = null;
    sPool.release(this);
}
```



**至此updateRootDisplayList的流程结束了，总结一下：**

* **首先从跟view开始，调用updateDisplayListIfDirty，如果当前View的PFLAG_INVALIDATED标志位不为true，那么就会对其子view进行分发**
* **分发到子view，如果子view调用了invalidate, 那么子view的PFLAG_INVALIDATED标志位为true，这会触发该子view对应的renderNode的displayList重建**
* **重建过程是begingRecording获取到RecordingCanvas后，调用view的onDraw方法对RecordingCanvas进行操作，最后只能够的绘制操作就被记录到RecordingCanvas的displayList当中**
* **重建结束后endRecording将displayList存储到renderNode.mStagingDisplayList当中, 从以上的分析我们也可以得知，只有调用invalidate的那个View及其 子view的RenderNode.mStagingDisplayList是非空的，其他的无需更新的View RenderNode.mStagingDisplayList会是空的**



##### 2.2.2.2、syncAndDrawFrame

```java
//graphics\java\android\graphics\HardwareRenderer.java
@SyncAndDrawResult
public int syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
    return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
}
```

```c++
//core\jni\android_view_ThreadedRenderer.cpp
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    return proxy->syncAndDrawFrame();
}

//libs\hwui\renderthread\RenderProxy.cpp
int RenderProxy::syncAndDrawFrame() {
    return mDrawFrameTask.drawFrame();
}

//libs\hwui\renderthread\DrawFrameTask.cpp
int DrawFrameTask::drawFrame() {
    mSyncResult = SyncResult::OK;
    mSyncQueued = systemTime(CLOCK_MONOTONIC);
    postAndWait();
    return mSyncResult;
}

//libs\hwui\renderthread\DrawFrameTask.cpp
void DrawFrameTask::postAndWait() {
    AutoMutex _lock(mLock);
    mRenderThread->queue().post([this]() { run(); });
    //主线程阻塞等待
    mSignal.wait(mLock);
}

//libs\hwui\renderthread\DrawFrameTask.cpp
void DrawFrameTask::run() {
    ATRACE_NAME("DrawFrame");

    bool canUnblockUiThread;
    bool canDrawThisFrame;
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        //脏区域相关的逻辑核心看这里
        canUnblockUiThread = syncFrameState(info);
        canDrawThisFrame = info.out.canDrawThisFrame;
        //真渲染完成回调注册相关代码
    }

    // Grab a copy of everything we need
    CanvasContext* context = mContext;
    std::function<void(int64_t)> callback = std::move(mFrameCallback);
    mFrameCallback = nullptr;
    // From this point on anything in "this" is *UNSAFE TO ACCESS*
    if (canUnblockUiThread) {
        //将阻塞的主线程唤醒
        unblockUiThread();
    }

    // Even if we aren't drawing this vsync pulse the next frame number will still be accurate
    if (CC_UNLIKELY(callback)) {
        context->enqueueFrameWork(
                [callback, frameNr = context->getFrameNumber()]() { callback(frameNr); });
    }
    if (CC_LIKELY(canDrawThisFrame)) {
        //调用CanvasContext.draw，将DisplayList转为gpu指令
        context->draw();
    } else {
        // wait on fences so tasks don't overlap next frame
        context->waitOnFences();
    }

    if (!canUnblockUiThread) {
        unblockUiThread();
    }
}
```

DrawFrameTask::run()主要包含两个核心操作，一个是syncFrameState(info)， 另一个是context->draw()

syncFrameState主要是脏区域计算的逻辑相关，context->draw() 会调用各个RenderNodeDrawable将绘制指令转换为gpu指令。

我们先看syncFrameState

```c++
//libs\hwui\renderthread\DrawFrameTask.cpp
bool DrawFrameTask::syncFrameState(TreeInfo& info) {
    ATRACE_CALL();
    int64_t vsync = mFrameInfo[static_cast<int>(FrameInfoIndex::Vsync)];
    mRenderThread->timeLord().vsyncReceived(vsync);
    bool canDraw = mContext->makeCurrent();
    mContext->unpinImages();
    for (size_t i = 0; i < mLayers.size(); i++) {
        //TextureView注册的DefferLayerUpdater的apply调用，这里从SurfaceTexture的BufferQueue中acquireBuffer获取到消息队列中的图形缓冲
        mLayers[i]->apply();
    }
    //使用后清除
    mLayers.clear();
    mContext->setContentDrawBounds(mContentDrawBounds);
    //关键操作
    mContext->prepareTree(info, mFrameInfo, mSyncQueued, mTargetNode);
    //...
    // If prepareTextures is false, we ran out of texture cache space
    return info.prepareTextures;
}
```

这里的 mContext对象为CanvasContext.cpp对象

```c++
//libs\hwui\renderthread\CanvasContext.cpp
void CanvasContext::prepareTree(TreeInfo& info, int64_t* uiFrameInfo, int64_t syncQueued,
                                RenderNode* target) {
    mRenderThread.removeFrameCallback(this);
    if (!wasSkipped(mCurrentFrameInfo)) {
        mCurrentFrameInfo = mJankTracker.startFrame();
    }
    mCurrentFrameInfo->importUiThreadInfo(uiFrameInfo);
    mCurrentFrameInfo->set(FrameInfoIndex::SyncQueued) = syncQueued;
    mCurrentFrameInfo->markSyncStart();
	//获取到脏区域累加器，mDamageAccumulator是CanvasContext的成员变量
    info.damageAccumulator = &mDamageAccumulator;
    info.layerUpdateQueue = &mLayerUpdateQueue;
    info.out.canDrawThisFrame = true;
    //将mVectorDrawables清空
    mRenderPipeline->onPrepareTree();
    for (const sp<RenderNode>& node : mRenderNodes) {
        // Only the primary target node will be drawn full - all other nodes would get drawn in real time mode. In case of a window, the primary node is the window content and the other node(s) are non client / filler nodes.
        info.mode = (node.get() == target ? TreeInfo::MODE_FULL : TreeInfo::MODE_RT_ONLY);
        //遍历每个node，执行prepareTree， 一般情况在CanvasContext的mRenderNodes中只有rootRenderNode，其他的renderNode是rootRenderNode继续来管理的，所以info.mode为TreeInfo::MODE_FUL
        node->prepareTree(info);
        GL_CHECKPOINT(MODERATE);
    }
    //...
    mIsDirty = true;
	//...
}
```

核心 为遍历每个renderNode执行prepareTree

```c++
//libs\hwui\RenderNode.cpp
void RenderNode::prepareTree(TreeInfo& info) {
    ATRACE_CALL();
    //...
    prepareTreeImpl(observer, info, false);
}

//libs\hwui\RenderNode.cpp
/**
 * Traverse down the the draw tree to prepare for a frame.
 * MODE_FULL = UI Thread-driven (thus properties must be synced), otherwise RT driven
 * While traversing down the tree, functorsNeedLayer flag is set to true if anything that uses the stencil buffer may be needed. Views that use a functor to draw will be forced onto a layer.
 */
//rootRenderNode触发的流程
void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {
    if (mDamageGenerationId == info.damageGenerationId) {
        // We hit the same node a second time in the same tree. We don't know the minimal damage rect anymore, so just push the biggest we can onto our parent's transform, We push directly onto parent in case we are clipped to bounds but have moved position.
        info.damageAccumulator->dirty(DIRTY_MIN, DIRTY_MIN, DIRTY_MAX, DIRTY_MAX);
    }
    info.damageAccumulator->pushTransform(this);

    if (info.mode == TreeInfo::MODE_FULL) {
        //rootRenderNode更新
        pushStagingPropertiesChanges(info);
    }

    bool willHaveFunctor = false;
    if (info.mode == TreeInfo::MODE_FULL && mStagingDisplayList) {
        willHaveFunctor = mStagingDisplayList->hasFunctor();
    } else if (mDisplayList) {
        willHaveFunctor = mDisplayList->hasFunctor();
    }
    prepareLayer(info, animatorDirtyMask);
    if (info.mode == TreeInfo::MODE_FULL) {
        //rootRenderNode更新
        pushStagingDisplayListChanges(observer, info);
    }

    if (mDisplayList) {
        //后续的绘制中mDisplayList不会被情况，如果有更新 mStagingDisplayList会把mDisplayList更新掉
        info.out.hasFunctors |= mDisplayList->hasFunctor();
        //遍历子View返回是否dirty
        bool isDirty = mDisplayList->prepareListAndChildren(
                observer, info, childFunctorsNeedLayer,
                [](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                   bool functorsNeedLayer) {
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                });
        if (isDirty) {
            damageSelf(info);
        }
    }
    pushLayerUpdate(info);

    if (!mProperties.getAllowForceDark()) {
        info.disableForceDark--;
    }
    info.damageAccumulator->popTransform();
}
```

可以看到RootRenderNode触发各个子View的prepareTreeImpl，是一个递归的写法

prepareListAndChildren

```c++
//libs\hwui\pipeline\skia\SkiaDisplayList.cpp
bool SkiaDisplayList::prepareListAndChildren(
        TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer,
        std::function<void(RenderNode*, TreeObserver&, TreeInfo&, bool)> childFn) {

    bool hasBackwardProjectedNodesHere = false;
    bool hasBackwardProjectedNodesSubtree = false;
	
    //假设没有子view，这里可以忽略
    for (auto& child : mChildNodes) {
        hasBackwardProjectedNodesHere |= child.getNodeProperties().getProjectBackwards();
        RenderNode* childNode = child.getRenderNode();
        Matrix4 mat4(child.getRecordedMatrix());
        info.damageAccumulator->pushTransform(&mat4);
        info.hasBackwardProjectedNodes = false;
        //执行每个子View的prepareTreeImpl
        childFn(childNode, observer, info, functorsNeedLayer);
        hasBackwardProjectedNodesSubtree |= info.hasBackwardProjectedNodes;
        info.damageAccumulator->popTransform();
    }

    info.hasBackwardProjectedNodes =
                hasBackwardProjectedNodesSubtree || hasBackwardProjectedNodesHere;

    bool isDirty = false;
    for (auto& animatedImage : mAnimatedImages) {
        nsecs_t timeTilNextFrame = TreeInfo::Out::kNoAnimatedImageDelay;
        // If any animated image in the display list needs updated, then damage the node. 如果有animatedImage，则直接标记为true
        if (animatedImage->isDirty(&timeTilNextFrame)) {
            isDirty = true;
        }
        //...
    }

    for (auto& vectorDrawablePair : mVectorDrawables) {
        // If any vector drawable in the display list needs update, damage the node.
        auto& vectorDrawable = vectorDrawablePair.first;
        if (vectorDrawable->isDirty()) {
            Matrix4 totalMatrix;
            info.damageAccumulator->computeCurrentTransform(&totalMatrix);
            Matrix4 canvasMatrix(vectorDrawablePair.second);
            totalMatrix.multiply(canvasMatrix);
            const SkRect& bounds = vectorDrawable->properties().getBounds();
            if (intersects(info.screenSize, totalMatrix, bounds)) {
                //如果相交则标记为dirty
                isDirty = true;
                static_cast<SkiaPipeline*>(info.canvasContext.getRenderPipeline())
                        ->getVectorDrawables()
                        ->push_back(vectorDrawable);
                vectorDrawable->setPropertyChangeWillBeConsumed(true);
            }
        }
    }
    return isDirty;
}
```

