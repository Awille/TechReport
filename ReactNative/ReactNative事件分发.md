# ReactNative事件分发

## 1、Androd Java层事件传递

ReactAndroid\src\main\java\com\facebook\react\ReactRootView.java

```java
  @Override
  public boolean onInterceptTouchEvent(MotionEvent ev) {
    dispatchJSTouchEvent(ev);
    return super.onInterceptTouchEvent(ev);
  }

  @Override
  public boolean onTouchEvent(MotionEvent ev) {
    dispatchJSTouchEvent(ev);
    super.onTouchEvent(ev);
    // In case when there is no children interested in handling touch event, we return true from
    // the root view in order to receive subsequent events related to that gesture
    return true;
  }

  private void dispatchJSTouchEvent(MotionEvent event) {
    ReactContext reactContext = mReactInstanceManager.getCurrentReactContext();
    EventDispatcher eventDispatcher =
        reactContext.getNativeModule(UIManagerModule.class).getEventDispatcher();
    mJSTouchDispatcher.handleTouchEvent(event, eventDispatcher);
  }
```

ReactRootView中，会在OnInterceptTouchEvent、onTouchEvent事件传递的时候，调用dispatchJsTouchEvent把事件传递到Js层。 另外在onTouchEvent中， 会直接返回true， 这确保了如果ReactRootView的所有子View不消费事件时， 一个事件序列的后续事件能继续传递到ReactRootView， ReactRootView能将事件序列后续所有的事件传递到JS层。



ReactAndroid\src\main\java\com\facebook\react\uimanager\JSTouchDispatcher.java

```java
 public void handleTouchEvent(MotionEvent ev, EventDispatcher eventDispatcher) {
    int action = ev.getAction() & MotionEvent.ACTION_MASK;
    if (action == MotionEvent.ACTION_DOWN) {
      // this gesture
      mChildIsHandlingNativeGesture = false;
      mGestureStartTime = ev.getEventTime();
      //ACTION_DOWN事件时寻找要消费该事件的ReactView并获取到ReactTag, 具体实现参照TouchTargetHelper.findTargetTagAndCoordinatesForTouch
      mTargetTag = findTargetTagAndSetCoordinates(ev);
      eventDispatcher.dispatchEvent(
          TouchEvent.obtain(
              mTargetTag,
              TouchEventType.START,
              ev,
              mGestureStartTime,
              mTargetCoordinates[0],
              mTargetCoordinates[1],
              mTouchEventCoalescingKeyHelper));
    } else if (mChildIsHandlingNativeGesture) {
        //如果有子ReactRootView 需要消费事件，则暂停JS事件的分发
      // If the touch was intercepted by a child, we've already sent a cancel event to JS for this
      // gesture, so we shouldn't send any more touches related to it.
      return;
    } else if (mTargetTag == -1) {
      // Action DOWN事件中没有找到消费的ReactView，这是一种异常情况，则 忽略往JS发送事件
      FLog.e(
          ReactConstants.TAG,
          "Unexpected state: received touch event but didn't get starting ACTION_DOWN for this "
              + "gesture before");
    } else if (action == MotionEvent.ACTION_UP) {
      // End of the gesture. We reset target tag to -1 and expect no further event associated with
      // this gesture.
      findTargetTagAndSetCoordinates(ev);
      eventDispatcher.dispatchEvent(
          TouchEvent.obtain(
              mTargetTag,
              TouchEventType.END,
              ev,
              mGestureStartTime,
              mTargetCoordinates[0],
              mTargetCoordinates[1],
              mTouchEventCoalescingKeyHelper));
      mTargetTag = -1;
      mGestureStartTime = TouchEvent.UNSET;
    } else if (action == MotionEvent.ACTION_MOVE) {
      // Update pointer position for current gesture
      findTargetTagAndSetCoordinates(ev);
      eventDispatcher.dispatchEvent(
          TouchEvent.obtain(
              mTargetTag,
              TouchEventType.MOVE,
              ev,
              mGestureStartTime,
              mTargetCoordinates[0],
              mTargetCoordinates[1],
              mTouchEventCoalescingKeyHelper));
    } else if (action == MotionEvent.ACTION_POINTER_DOWN) {
      // New pointer goes down, this can only happen after ACTION_DOWN is sent for the first pointer
      eventDispatcher.dispatchEvent(
          TouchEvent.obtain(
              mTargetTag,
              TouchEventType.START,
              ev,
              mGestureStartTime,
              mTargetCoordinates[0],
              mTargetCoordinates[1],
              mTouchEventCoalescingKeyHelper));
    } else if (action == MotionEvent.ACTION_POINTER_UP) {
      // Exactly onw of the pointers goes up
      eventDispatcher.dispatchEvent(
          TouchEvent.obtain(
              mTargetTag,
              TouchEventType.END,
              ev,
              mGestureStartTime,
              mTargetCoordinates[0],
              mTargetCoordinates[1],
              mTouchEventCoalescingKeyHelper));
    } else if (action == MotionEvent.ACTION_CANCEL) {
      if (mTouchEventCoalescingKeyHelper.hasCoalescingKey(ev.getDownTime())) {
        dispatchCancelEvent(ev, eventDispatcher);
      } else {
        FLog.e(
            ReactConstants.TAG,
            "Received an ACTION_CANCEL touch event for which we have no corresponding ACTION_DOWN");
      }
      mTargetTag = -1;
      mGestureStartTime = TouchEvent.UNSET;
    } else {
      FLog.w(
          ReactConstants.TAG,
          "Warning : touch event was ignored. Action=" + action + " Target=" + mTargetTag);
    }
  }

  private int findTargetTagAndSetCoordinates(MotionEvent ev) {
    // This method updates `mTargetCoordinates` with coordinates for the motion event.
    return TouchTargetHelper.findTargetTagAndCoordinatesForTouch(
        ev.getX(), ev.getY(), mRootViewGroup, mTargetCoordinates, null);
  }
```

ReactAndroid\src\main\java\com\facebook\react\uimanager\TouchTargetHelper.java

```java
  public static int findTargetTagAndCoordinatesForTouch(
      float eventX,
      float eventY,
      ViewGroup viewGroup,
      float[] viewCoords,
      @Nullable int[] nativeViewTag) {
    UiThreadUtil.assertOnUiThread();
    int targetTag = viewGroup.getId();
    // Store eventCoords in array so that they are modified to be relative to the targetView found.
    viewCoords[0] = eventX;
    viewCoords[1] = eventY;
    // 根据坐标寻找到目标的Anroid View
    View nativeTargetView = findTouchTargetView(viewCoords, viewGroup);
    if (nativeTargetView != null) {
        //根据Android View 找到最近的React父节点，如果该AndroidView本身就是一个ReactView，那这个方法就会返回该View本身，具体参照下面的findClosestReactAncestor实现
      View reactTargetView = findClosestReactAncestor(nativeTargetView);
      if (reactTargetView != null) {
        if (nativeViewTag != null) {
          nativeViewTag[0] = reactTargetView.getId();
        }
        //根据React View跟坐标获取目标tag
        targetTag = getTouchTargetForView(reactTargetView, viewCoords[0], viewCoords[1]);
      }
    }
    return targetTag;
  }

  private static View findClosestReactAncestor(View view) {
    while (view != null && view.getId() <= 0) {
      view = (View) view.getParent();
    }
    return view;
  }

  /*DFS算法寻找TargetView。  该函数会返回入参ViewGroup或者ViewGroup的子view作为目标。这是一个回溯DFS算法，会遍历整个view树直到目标被找到。  如果不进行回溯的话，本次DFS的分支很有可能不会寻找到目标。比如 C 和E都能成为事件的target， 假设有两个分支：第一个分支为：A(pointerEvents: auto) - B (pointerEvents: box-none) - C (pointerEvents: none) ， 第二个分支为D(pointerEvents: auto) - E (pointerEvents: auto)。 如果走到第一个分支不进行回溯，那么会找到C为target，这是不合法的，因为C被设置的属性为NONE,代表它以及它的子View是不会响应时事件的。
  */
  private static View findTouchTargetView(float[] eventCoords, ViewGroup viewGroup) {
    int childrenCount = viewGroup.getChildCount();
    // Consider z-index when determining the touch target.
    //有关ReactZIndexedViewGroup的介绍，可以看下附录中的 ReactZIndexedViewGroup章节
    ReactZIndexedViewGroup zIndexedViewGroup =
        viewGroup instanceof ReactZIndexedViewGroup ? (ReactZIndexedViewGroup) viewGroup : null;
    for (int i = childrenCount - 1; i >= 0; i--) {
        //for 循环，让每个view为DFS算法的起点，防止出现注释上的case
      int childIndex =
          zIndexedViewGroup != null ? zIndexedViewGroup.getZIndexMappedChildIndex(i) : i;
      View child = viewGroup.getChildAt(childIndex);
      PointF childPoint = mTempPoint;
      if (isTransformedTouchPointInView(
          eventCoords[0], eventCoords[1], viewGroup, child, childPoint)) {
          /*该函数用于判断触摸事件坐标是否在入参 child以内*/
          //触摸事件在子view以内
        // If it is contained within the child View, the childPoint value will contain the view
        // coordinates relative to the child
        // We need to store the existing X,Y for the viewGroup away as it is possible this child
        // will not actually be the target and so we restore them if not
        float restoreX = eventCoords[0];
        float restoreY = eventCoords[1];
        eventCoords[0] = childPoint.x;
        eventCoords[1] = childPoint.y;
        View targetView = findTouchTargetViewWithPointerEvents(eventCoords, child);
        if (targetView != null) {
          return targetView;
        }
        eventCoords[0] = restoreX;
        eventCoords[1] = restoreY;
      }
    }
    return viewGroup;
  }

  /*该函数用于判断触摸事件坐标是否在入参 child以内*/
  private static boolean isTransformedTouchPointInView(
      float x, float y, ViewGroup parent, View child, PointF outLocalPoint) {
    //计算触摸点，相对于子view x坐标的偏移量，即在子view x左上顶点坐标的右边或左边，考虑了父view水平盘偏移的量，正常情况下可以先把parent.getScrollX()=0的case来便于理解
    float localX = x + parent.getScrollX() - child.getLeft();
      //计算触摸点，相对于子view y坐标的偏移量，即在子view y左上顶点坐标坐标右边或左边，考虑了父view竖直偏移的量，正常情况下可以先把parent.getScrollY()=0的case来便于理解
    float localY = y + parent.getScrollY() - child.getTop();
    //获取子视图的变化矩阵
    Matrix matrix = child.getMatrix();
    if (!matrix.isIdentity()) {
      //非单位矩阵，代表发生了坐标变化，需要获取变换矩阵的逆矩阵来变化坐标，可以先不考虑该case
      float[] localXY = mMatrixTransformCoords;
      localXY[0] = localX;
      localXY[1] = localY;
      Matrix inverseMatrix = mInverseMatrix;
      matrix.invert(inverseMatrix);
      inverseMatrix.mapPoints(localXY);
      localX = localXY[0];
      localY = localXY[1];
    }
    if (child instanceof ReactHitSlopView && ((ReactHitSlopView) child).getHitSlopRect() != null) {
        //该分支可以先忽略
      Rect hitSlopRect = ((ReactHitSlopView) child).getHitSlopRect();
      if ((localX >= -hitSlopRect.left
              && localX < (child.getRight() - child.getLeft()) + hitSlopRect.right)
          && (localY >= -hitSlopRect.top
              && localY < (child.getBottom() - child.getTop()) + hitSlopRect.bottom)) {
        outLocalPoint.set(localX, localY);
        return true;
      }

      return false;
    } else {
      if ((localX >= 0 && localX < (child.getRight() - child.getLeft()))
          && (localY >= 0 && localY < (child.getBottom() - child.getTop()))) {
        //表示触摸点在 子view的返回内
        outLocalPoint.set(localX, localY);
        return true;
      }

      return false;
    }
  }


  /**
   * Returns the touch target View of the event given, or null if neither the given View nor any of
   * its descendants are the touch target.
   */
  private static @Nullable View findTouchTargetViewWithPointerEvents(
      float eventCoords[], View view) {
    PointerEvents pointerEvents =
        view instanceof ReactPointerEventsView
            ? ((ReactPointerEventsView) view).getPointerEvents()
            : PointerEvents.AUTO;

    // Views that are disabled should never be the target of pointer events. However, their children
    // can be because some views (SwipeRefreshLayout) use enabled but still have children that can
    // be valid targets.
    if (!view.isEnabled()) {
        //如果view被禁用了
      if (pointerEvents == PointerEvents.AUTO) {
          //当前为AUTO状态则转为BOX_NONE，允许子view接收事件
        pointerEvents = PointerEvents.BOX_NONE;
      } else if (pointerEvents == PointerEvents.BOX_ONLY) {
          //如果本身为BOX_ONLY, 只允许自己接收事件，则转为NONE，不允许自己和子view接收事件
        pointerEvents = PointerEvents.NONE;
      }
    }

    if (pointerEvents == PointerEvents.NONE) {
      // This view and its children can't be the target
      return null;

    } else if (pointerEvents == PointerEvents.BOX_ONLY) {
      // This view is the target, its children don't matter
      return view;

    } else if (pointerEvents == PointerEvents.BOX_NONE) {
        //自己不允许接收事件，但子view可以
      // This view can't be the target, but its children might.
      if (view instanceof ViewGroup) {
          //DFS递归寻找
        View targetView = findTouchTargetView(eventCoords, (ViewGroup) view);
        if (targetView != view) {
            //如果其有子view响应，则直接返回
          return targetView;
        }

        // PointerEvents.BOX_NONE means that this react element cannot receive pointer events.
        // However, there might be virtual children that can receive pointer events, in which case
        // we still want to return this View and dispatch a pointer event to the virtual element.
        // Note that this currently only applies to Nodes/FlatViewGroup as it's the only class that
        // is both a ViewGroup and ReactCompoundView (ReactTextView is a ReactCompoundView but not a
        // ViewGroup).
          //如果该view有虚拟子元素，则尝试查找是否有虚拟子元素需要响应事件，若果发现虚拟子元素响应了事件，也可以返回虚拟子元素
         ////有关ReactCompoundView的介绍，可以看下附录中的 ReactCompoundView章节
        if (view instanceof ReactCompoundView) {
          int reactTag =
              ((ReactCompoundView) view).reactTagForTouch(eventCoords[0], eventCoords[1]);
          if (reactTag != view.getId()) {
              //虚拟子元素需要响应事件
            // make sure we exclude the View itself because of the PointerEvents.BOX_NONE
            return view;
          }
        }
      }
      return null;

    } else if (pointerEvents == PointerEvents.AUTO) {
      // Either this view or one of its children is the target
      if (view instanceof ReactCompoundViewGroup) {
        if (((ReactCompoundViewGroup) view).interceptsTouchEvent(eventCoords[0], eventCoords[1])) {
          return view;
        }
      }
      if (view instanceof ViewGroup) {
          //递归查找
        return findTouchTargetView(eventCoords, (ViewGroup) view);
      }
      return view;

    } else {
      throw new JSApplicationIllegalArgumentException(
          "Unknown pointer event type: " + pointerEvents.toString());
    }
  }
```

在DFS寻找targetView的过程中，有一个zIndexedViewGroup的概念， 我们发现ReactViewGroup继承了

ReactZIndexedViewGroup，其中包含的两个方法：


```java
/** ViewGroup that supports z-index. */
public interface ReactZIndexedViewGroup {
  /**
   * Determine the index of a child view at {@param index} considering z-index.
   *
   * @param index The child view index
   * @return The child view index considering z-index
   */
  int getZIndexMappedChildIndex(int index);

  /**
   * Redraw the view based on updated child z-index. This should be called after updating one of its
   * child z-index.
   */
  void updateDrawingOrder();
}

```





### 小结

通过以上的分子，在ReactRootView的 onInterceptTouchEvent 与 onTouchEvent 都会往JS分发事件，其中在ACITON DOWN事件发生时，ReactRootView内部会根据DFS算法找要消费该事件的React View，并获取到该View的ReactTag。 下面开始ReactTag 跟坐标事件的分发。



## 2、Android C层的事件向JS传递

在C层其实做的事情很简单，核心就是把Android Java层处理好的触摸事件坐标，ReactTag分发到JS层

核心就是EventDispatcher.dispatchEvent

触发的代码可以参照JsTouchDispatcher:

```java
eventDispatcher.dispatchEvent(
  TouchEvent.obtain(
      mTargetTag,
      TouchEventType.START,
      ev,
      mGestureStartTime,
      mTargetCoordinates[0],
      mTargetCoordinates[1],
      mTouchEventCoalescingKeyHelper));
```



ReactAndroid\src\main\java\com\facebook\react\uimanager\events\TouchEvent.java

```java
public static TouchEvent obtain(
  int viewTag,
  TouchEventType touchEventType,
  MotionEvent motionEventToCopy,
  long gestureStartTime,
  float viewX,
  float viewY,
  TouchEventCoalescingKeyHelper touchEventCoalescingKeyHelper) {
TouchEvent event = EVENTS_POOL.acquire();
    if (event == null) {
      event = new TouchEvent();
    }
    event.init(
        viewTag,
        touchEventType,
        motionEventToCopy,
        gestureStartTime,
        viewX,
        viewY,
        touchEventCoalescingKeyHelper);
    return event;
}
```

ReactAndroid\src\main\java\com\facebook\react\uimanager\events\EventDispatcher.java

```java
  /** Sends the given Event to JS, coalescing eligible events if JS is backed up. */
  public void dispatchEvent(Event event) {
    Assertions.assertCondition(event.isInitialized(), "Dispatched event hasn't been initialized");

    for (EventDispatcherListener listener : mListeners) {
      listener.onEventDispatch(event);
    }

    synchronized (mEventsStagingLock) {
        //添加到事件队列当中
      mEventStaging.add(event);
      Systrace.startAsyncFlow(
          Systrace.TRACE_TAG_REACT_JAVA_BRIDGE, event.getEventName(), event.getUniqueID());
    }
    maybePostFrameCallbackFromNonUI();
  }

//注册一个JS事件分发的Choreographer.FraneCallbak， 在每一个Vsync回调中把Staging队列中事件转移到分发队列， 并进行分发
private class ScheduleDispatchFrameCallback extends ChoreographerCompat.FrameCallback {
    @Override
    public void doFrame(long frameTimeNanos) {
      UiThreadUtil.assertOnUiThread();

      if (mShouldStop) {
        mIsPosted = false;
      } else {
        post();
      }
      try {
        //staging队列中事件转移到分发队列当中
        moveStagedEventsToDispatchQueue();
        if (!mHasDispatchScheduled) {
          mHasDispatchScheduled = true;
          //在JS的一个消息线程中分发事件
          mReactContext.runOnJSQueueThread(mDispatchEventsRunnable);
        }
      }
    }
}

private void moveStagedEventsToDispatchQueue() {
    synchronized (mEventsStagingLock) {
      synchronized (mEventsToDispatchLock) {
        for (int i = 0; i < mEventStaging.size(); i++) {
          Event event = mEventStaging.get(i);
          //这里把代码精简了，据悉查看wrap的操作，请请自己查看源码
          Event eventToAdd = wrap(event);
          if (eventToAdd != null) {
              //转移事件到分发队列
            addEventToEventsToDispatch(eventToAdd);
          }
          if (eventToDispose != null) {
            eventToDispose.dispose();
          }
        }
      }
      mEventStaging.clear();
    }
}

private void addEventToEventsToDispatch(Event event) {
    if (mEventsToDispatchSize == mEventsToDispatch.length) {
      mEventsToDispatch = Arrays.copyOf(mEventsToDispatch, 2 * mEventsToDispatch.length);
    }
    mEventsToDispatch[mEventsToDispatchSize++] = event;
}

//在JS线程中分发事件的代码，在ScheduleDispatchFrameCallback.doFrame中执行，参照mReactContext.runOnJSQueueThread(mDispatchEventsRunnable);
  private class DispatchEventsRunnable implements Runnable {

    @Override
    public void run() {
      try {
        mHasDispatchScheduled = false;
        synchronized (mEventsToDispatchLock) {
          if (mEventsToDispatchSize > 0) {
            if (mEventsToDispatchSize > 1) {
               //事件优先级排序
              Arrays.sort(mEventsToDispatch, 0, mEventsToDispatchSize, EVENT_COMPARATOR);
            }
            for (int eventIdx = 0; eventIdx < mEventsToDispatchSize; eventIdx++) {
              Event event = mEventsToDispatch[eventIdx];
              //发送事件到JS
              event.dispatch(mReactEventEmitter);
              event.dispose();
            }
            clearEventsToDispatch();
            mEventCookieToLastEventIdx.clear();
          }
        }
        for (BatchEventDispatchedListener listener : mPostEventDispatchListeners) {
          listener.onBatchEventDispatched();
        }
      } 
    }
  }
```

其中最核心的是在DispatchEventsRunnable调用了Event.dispatch(mReactEventEmitter) 往JS发送了事件，其中Event是个抽象类，我们这种触摸事件的具体实现了为ReactAndroid\src\main\java\com\facebook\react\uimanager\events\TouchEvent.java

```java
@Override
public void dispatch(RCTEventEmitter rctEventEmitter) {
    TouchesHelper.sendTouchEvent(
        rctEventEmitter, Assertions.assertNotNull(mTouchEventType), getViewTag(), this);
}
```

ReactAndroid\src\main\java\com\facebook\react\uimanager\events\TouchesHelper.java

```java
public static void sendTouchEvent(
  RCTEventEmitter rctEventEmitter,
  TouchEventType type,
  int reactTarget,
  TouchEvent touchEvent) {

    //... 忽略代码
    rctEventEmitter.receiveTouches(TouchEventType.getJSEventName(type), pointers, changedIndices);
}
```

ReactAndroid\src\main\java\com\facebook\react\uimanager\events\RCTEventEmitter.java

```java
@DoNotStrip
public interface RCTEventEmitter extends JavaScriptModule {
  void receiveEvent(int targetTag, String eventName, @Nullable WritableMap event);

  void receiveTouches(String eventName, WritableArray touches, WritableArray changedIndices);
}
```

实现类为：ReactAndroid\src\main\java\com\facebook\react\uimanager\events\ReactEventEmitter.java

```java
@Override
public void receiveTouches(
  String eventName, WritableArray touches, WritableArray changedIndices) {
    int reactTag = touches.getMap(0).getInt(TARGET_KEY);
    RCTEventEmitter eventEmitter = getEventEmitter(reactTag);
    if (eventEmitter != null) {
      eventEmitter.receiveTouches(eventName, touches, changedIndices);
    }
}

@Nullable
private RCTEventEmitter getEventEmitter(int reactTag) {
    int type = ViewUtil.getUIManagerType(reactTag);
    RCTEventEmitter eventEmitter = mEventEmitters.get(type);
    if (eventEmitter == null) {
      if (mReactContext.hasActiveCatalystInstance()) {
        eventEmitter = mReactContext.getJSModule(RCTEventEmitter.class);
      } else {
        ReactSoftException.logSoftException(
            TAG,
            new ReactNoCrashSoftException(
                "Cannot get RCTEventEmitter from Context for reactTag: "
                    + reactTag
                    + " - uiManagerType: "
                    + type
                    + " - No active Catalyst instance!"));
      }
    }
    return eventEmitter;
}
```

ReactAndroid\src\main\java\com\facebook\react\fabric\events\FabricEventEmitter.java

```java
@Override
public void receiveEvent(int reactTag, @NonNull String eventName, @Nullable WritableMap params) {
    Systrace.beginSection(
        Systrace.TRACE_TAG_REACT_JAVA_BRIDGE,
        "FabricEventEmitter.receiveEvent('" + eventName + "')");
    mUIManager.receiveEvent(reactTag, eventName, params);
    Systrace.endSection(Systrace.TRACE_TAG_REACT_JAVA_BRIDGE);
}

@Override
public void receiveTouches(
  @NonNull String eventTopLevelType,
  @NonNull WritableArray touches,
  @NonNull WritableArray changedIndices) {
    Pair<WritableArray, WritableArray> result =
        TOP_TOUCH_END_KEY.equalsIgnoreCase(eventTopLevelType)
                || TOP_TOUCH_CANCEL_KEY.equalsIgnoreCase(eventTopLevelType)
            ? removeTouchesAtIndices(touches, changedIndices)
            : touchSubsequence(touches, changedIndices);

    WritableArray changedTouches = result.first;
    touches = result.second;

    for (int jj = 0; jj < changedTouches.size(); jj++) {
          WritableMap touch = getWritableMap(changedTouches.getMap(jj));
          // Touch objects can fulfill the role of `DOM` `Event` objects if we set
          // the `changedTouches`/`touches`. This saves allocations.

          touch.putArray(CHANGED_TOUCHES_KEY, copyWritableArray(changedTouches));
          touch.putArray(TOUCHES_KEY, copyWritableArray(touches));
          WritableMap nativeEvent = touch;
          int rootNodeID = 0;
          int target = nativeEvent.getInt(TARGET_KEY);
          if (target < 1) {
            FLog.e(TAG, "A view is reporting that a touch occurred on tag zero.");
          } else {
            rootNodeID = target;
          }

          receiveEvent(rootNodeID, eventTopLevelType, touch);
    }
}
```

具体请参照：
ReactCommon\cxxreact\NativeToJsBridge.cpp

https://zxfcumtcs.github.io/2017/10/12/ReactNativeCommunicationMechanism2/

https://www.jianshu.com/p/02be425d7b13

最终会分发到JS层：

Libraries\Core\setUpBatchedBridge.js

```javascript
'use strict';

let registerModule;

registerModule('RCTDeviceEventEmitter', () =>
  require('../EventEmitter/RCTDeviceEventEmitter'),
);
```



Libraries\EventEmitter\RCTDeviceEventEmitter.js

```javascript
class RCTDeviceEventEmitter extends EventEmitter {
  sharedSubscriber: EventSubscriptionVendor;

  constructor() {
    const sharedSubscriber = new EventSubscriptionVendor();
    super(sharedSubscriber);
    this.sharedSubscriber = sharedSubscriber;
  }

  addListener(
    eventType: string,
    listener: Function,
    context: ?Object,
  ): EmitterSubscription {
    if (__DEV__) {
      checkNativeEventModule(eventType);
    }
    return super.addListener(eventType, listener, context);
  }

  removeAllListeners(eventType: ?string) {
    if (__DEV__) {
      checkNativeEventModule(eventType);
    }
    super.removeAllListeners(eventType);
  }

  removeSubscription(subscription: EmitterSubscription) {
    if (subscription.emitter !== this) {
      subscription.emitter.removeSubscription(subscription);
    } else {
      super.removeSubscription(subscription);
    }
  }
}
```



## JavaScript中对点击事件的处理

具体参照

Libraries/Renderer/implementations/ReactNativeRenderer-dev.js

```javascript
var ResponderEventPlugin = {
  /* For unit testing only */
  _getResponder: function() {
    return responderInst;
  },
  eventTypes: eventTypes,

  /**
   * We must be resilient to `targetInst` being `null` on `touchMove` or
   * `touchEnd`. On certain platforms, this means that a native scroll has
   * assumed control and the original touch targets are destroyed.
   */
  extractEvents: function(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags
  ) {
    if (isStartish(topLevelType)) {
      //ACTION_DOWN事件时trackedTouchCount + 1
      trackedTouchCount += 1;
    } else if (isEndish(topLevelType)) {
        //ACTION_UP或者ACTION_CANCEL事件时trackedTouchCount - 1
      if (trackedTouchCount >= 0) {
        trackedTouchCount -= 1;
      } else {
        return null;
      }
    }
    //canTriggerTransfer Action_DOWN 跟 Action_Move事件会返回true，其他的返回false
    var extracted = canTriggerTransfer(topLevelType, targetInst, nativeEvent)
      ? setResponderAndExtractTransfer(
          topLevelType,
          targetInst,
          nativeEvent,
          nativeEventTarget
        )
      : null; // Responder may or may not have transferred on a new touch start/move.
    // Regardless, whoever is the responder after any potential transfer, we
    // direct all touch start/move/ends to them in the form of
    // `onResponderMove/Start/End`. These will be called for *every* additional
    // finger that move/start/end, dispatched directly to whoever is the
    // current responder at that moment, until the responder is "released".
    //
    // These multiple individual change touch events are are always bookended
    // by `onResponderGrant`, and one of
    // (`onResponderRelease/onResponderTerminate`).

    var isResponderTouchStart = responderInst && isStartish(topLevelType);
    var isResponderTouchMove = responderInst && isMoveish(topLevelType);
    var isResponderTouchEnd = responderInst && isEndish(topLevelType);
    var incrementalTouch = isResponderTouchStart
      ? eventTypes.responderStart
      : isResponderTouchMove
      ? eventTypes.responderMove
      : isResponderTouchEnd
      ? eventTypes.responderEnd
      : null;

    if (incrementalTouch) {
      var gesture = ResponderSyntheticEvent.getPooled(
        incrementalTouch,
        responderInst,
        nativeEvent,
        nativeEventTarget
      );
      gesture.touchHistory = ResponderTouchHistoryStore.touchHistory;
      accumulateDirectDispatches(gesture);
      extracted = accumulate(extracted, gesture);
    }

    var isResponderTerminate =
      responderInst && topLevelType === TOP_TOUCH_CANCEL;
    var isResponderRelease =
      responderInst &&
      !isResponderTerminate &&
      isEndish(topLevelType) &&
      noResponderTouches(nativeEvent);
    var finalTouch = isResponderTerminate
      ? eventTypes.responderTerminate
      : isResponderRelease
      ? eventTypes.responderRelease
      : null;

    if (finalTouch) {
      var finalEvent = ResponderSyntheticEvent.getPooled(
        finalTouch,
        responderInst,
        nativeEvent,
        nativeEventTarget
      );
      finalEvent.touchHistory = ResponderTouchHistoryStore.touchHistory;
      accumulateDirectDispatches(finalEvent);
      extracted = accumulate(extracted, finalEvent);
      changeResponder(null);
    }

    return extracted;
  },
  GlobalResponderHandler: null,
  injection: {
    /**
     * @param {{onChange: (ReactID, ReactID) => void} GlobalResponderHandler
     * Object that handles any change in responder. Use this to inject
     * integration with an existing touch handling system etc.
     */
    injectGlobalResponderHandler: function(GlobalResponderHandler) {
      ResponderEventPlugin.GlobalResponderHandler = GlobalResponderHandler;
    }
  }
};
```

RN中console.log的日志，输出在浏览器的console当中



在extractEvents: function(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags
  ) 

中打印日志，测试代码为：

```javascript
extractEvents: function(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags
) {
    console.log(`topLevelType: ${topLevelType}`);
}
```

一次点击，输出为：

```javascript
topLevelType: topTouchStart
topLevelType: topTouchMove1
topLevelType: topTouchMove1
topLevelType: topTouchEnd
```

可以看到输出了一个topTouchStart ，两个topTouchMove， 最后一个topTouchEnd， 跟我们Native的日志基本能保持一致

正常情况的日志：

```shell
[Sat Jun 15 2024 22:12:24.578]  LOG      new topLevelType: topTouchStart
[Sat Jun 15 2024 22:12:24.657]  LOG      isStartish true
[Sat Jun 15 2024 22:12:24.658]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sat Jun 15 2024 22:12:24.659]  LOG      skipOverBubbleShouldSetFrom: false
[Sat Jun 15 2024 22:12:24.659]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
[Sat Jun 15 2024 22:12:24.659]  LOG      new topLevelType: topTouchCancel
[Sat Jun 15 2024 22:12:24.659]  LOG      isEndish true trackedTouchCount: 1
[Sat Jun 15 2024 22:12:24.660]  LOG      isResponderTerminate： true，
        finalTouch： {"dependencies": [], "registrationName": "onResponderTerminate"}
        
        
[Sat Jun 15 2024 22:12:24.660]  LOG      new topLevelType: topTouchStart
[Sat Jun 15 2024 22:12:24.660]  LOG      isStartish true
[Sat Jun 15 2024 22:12:24.660]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sat Jun 15 2024 22:12:24.660]  LOG      skipOverBubbleShouldSetFrom: false
[Sat Jun 15 2024 22:12:24.661]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
[Sat Jun 15 2024 22:12:24.661]  LOG      new topLevelType: topTouchMove
[Sat Jun 15 2024 22:12:24.661]  LOG      setResponderAndExtractTransfer: topTouchMove
[Sat Jun 15 2024 22:12:24.661]  LOG      skipOverBubbleShouldSetFrom: true
[Sat Jun 15 2024 22:12:24.662]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
[Sat Jun 15 2024 22:12:24.662]  LOG      new topLevelType: topTouchMove
[Sat Jun 15 2024 22:12:24.662]  LOG      setResponderAndExtractTransfer: topTouchMove
[Sat Jun 15 2024 22:12:24.662]  LOG      skipOverBubbleShouldSetFrom: true
[Sat Jun 15 2024 22:12:24.663]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
[Sat Jun 15 2024 22:12:24.663]  LOG      new topLevelType: topTouchEnd
[Sat Jun 15 2024 22:12:24.664]  LOG      isEndish true trackedTouchCount: 1
[Sat Jun 15 2024 22:12:24.664]  LOG      isResponderTerminate： false，
        finalTouch： {"dependencies": ["topTouchCancel", "topTouchEnd"], "registrationName": "onResponderRelease"}
```





异常情况日志：



```shell
[Sat Jun 15 2024 22:33:18.307]  LOG      new topLevelType: topTouchStart, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 90.28571319580078, "locationY": 91.42857360839844, "pageX": 219.42857360839844, "pageY": 531.047607421875, "target": 707, "timestamp": 259739139, "touches": [[Circular]]}
[Sat Jun 15 2024 22:33:18.353]  LOG      isStartish true
[Sat Jun 15 2024 22:33:18.353]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sat Jun 15 2024 22:33:18.354]  LOG      skipOverBubbleShouldSetFrom: false
[Sat Jun 15 2024 22:33:18.354]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
        
[Sat Jun 15 2024 22:33:18.354]  LOG      new topLevelType: topTouchStart, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 90.28571319580078, "locationY": 91.42857360839844, "pageX": 198.85714721679688, "pageY": 216.38095092773438, "target": 707, "timestamp": 259739153, "touches": [[Circular]]}
[Sat Jun 15 2024 22:33:18.354]  LOG      isStartish true
[Sat Jun 15 2024 22:33:18.355]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sat Jun 15 2024 22:33:18.355]  LOG      skipOverBubbleShouldSetFrom: true
[Sat Jun 15 2024 22:33:18.355]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
        
[Sat Jun 15 2024 22:33:18.355]  LOG      new topLevelType: topTouchMove, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 90.28571319580078, "locationY": 91.42857360839844, "pageX": 198.85714721679688, "pageY": 216.38095092773438, "target": 707, "timestamp": 259739163, "touches": [[Circular]]}
[Sat Jun 15 2024 22:33:18.356]  LOG      setResponderAndExtractTransfer: topTouchMove
[Sat Jun 15 2024 22:33:18.356]  LOG      skipOverBubbleShouldSetFrom: true
[Sat Jun 15 2024 22:33:18.356]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
[Sat Jun 15 2024 22:33:18.393]  LOG      new topLevelType: topTouchMove, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 90.28571319580078, "locationY": 91.42857360839844, "pageX": 198.85714721679688, "pageY": 216.38095092773438, "target": 707, "timestamp": 259739200, "touches": [[Circular]]}
[Sat Jun 15 2024 22:33:18.394]  LOG      setResponderAndExtractTransfer: topTouchMove
[Sat Jun 15 2024 22:33:18.394]  LOG      skipOverBubbleShouldSetFrom: true
[Sat Jun 15 2024 22:33:18.394]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
[Sat Jun 15 2024 22:33:18.394]  LOG      new topLevelType: topTouchEnd, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 90.28571319580078, "locationY": 91.42857360839844, "pageX": 219.42857360839844, "pageY": 531.047607421875, "target": 707, "timestamp": 259739203, "touches": []}
[Sat Jun 15 2024 22:33:18.395]  LOG      isEndish true trackedTouchCount: 2
[Sat Jun 15 2024 22:33:18.395]  LOG      isResponderTerminate： false，
        finalTouch： {"dependencies": ["topTouchCancel", "topTouchEnd"], "registrationName": "onResponderRelease"}
        
        
[Sat Jun 15 2024 22:33:18.395]  LOG      new topLevelType: topTouchEnd, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 90.28571319580078, "locationY": 91.42857360839844, "pageX": 198.85714721679688, "pageY": 216.38095092773438, "target": 707, "timestamp": 259739207, "touches": []}
[Sat Jun 15 2024 22:33:18.395]  LOG      isEndish true trackedTouchCount: 1
[Sat Jun 15 2024 22:33:18.396]  LOG      isResponderTerminate： null，
        finalTouch： null
```



可以发现所有的事件都发了两遍，并且最后的事件有些不对

对比extracted， 发现elementType不一样

有问题的：

```shell
[
  {
    "_dispatchInstances": null,
    "_dispatchListeners": null,
    "_targetInst": {
      "_debugHookTypes": null,
      "_debugID": 2678,
      "_debugIsCurrentlyTiming": false,
      "_debugNeedsRemount": false,
      "_debugOwner": "[FiberNode]",
      "_debugSource": {},
      "actualDuration": 1,
      "actualStartTime": 733,
      "alternate": null,
      "child": "[FiberNode]",
      "childExpirationTime": 0,
      "dependencies": null,
      "effectTag": 128,
      "elementType": "RCTView",
      "expirationTime": 0,
      "firstEffect": null,
      "index": 0,
      "key": null,
      "lastEffect": null,
      "memoizedProps": {},
      "memoizedState": null,
      "mode": 8,
      "nextEffect": null,
      "pendingProps": {},
      "ref": "[Function forwardRef]",
      "return": "[FiberNode]",
      "selfBaseDuration": 0,
      "sibling": null,
      "stateNode": "[ReactNativeFiberHostComponent]",
      "tag": 5,
      "treeBaseDuration": 1,
      "type": "RCTView",
      "updateQueue": null
    },
    "bubbles": null,
    "cancelable": null,
    "currentTarget": null,
    "defaultPrevented": null,
    "dispatchConfig": {
      "dependencies": [],
      "registrationName": "onResponderEnd"
    },
    "eventPhase": null,
    "isDefaultPrevented": "[Function functionThatReturnsFalse]",
    "isPropagationStopped": "[Function functionThatReturnsFalse]",
    "isTrusted": null,
    "nativeEvent": {
      "changedTouches": [],
      "identifier": 0,
      "locationX": 93.71428680419922,
      "locationY": 98.66666412353516,
      "pageX": 222.85714721679688,
      "pageY": 403.047607421875,
      "target": 707,
      "timestamp": 260103852,
      "touches": []
    },
    "target": {
      "_children": [],
      "_internalFiberInstanceHandleDEV": "[FiberNode]",
      "_nativeTag": 707,
      "viewConfig": {}
    },
    "timeStamp": 1718462366859,
    "touchHistory": {
      "indexOfSingleActiveTouch": 0,
      "mostRecentTimeStamp": 260103852,
      "numberActiveTouches": 0,
      "touchBank": []
    },
    "type": null
  },
  {
    "_dispatchInstances": {
      "_debugHookTypes": null,
      "_debugID": 2678,
      "_debugIsCurrentlyTiming": false,
      "_debugNeedsRemount": false,
      "_debugOwner": "[FiberNode]",
      "_debugSource": {},
      "actualDuration": 1,
      "actualStartTime": 733,
      "alternate": null,
      "child": "[FiberNode]",
      "childExpirationTime": 0,
      "dependencies": null,
      "effectTag": 128,
      "elementType": "RCTView",
      "expirationTime": 0,
      "firstEffect": null,
      "index": 0,
      "key": null,
      "lastEffect": null,
      "memoizedProps": {},
      "memoizedState": null,
      "mode": 8,
      "nextEffect": null,
      "pendingProps": {},
      "ref": "[Function forwardRef]",
      "return": "[FiberNode]",
      "selfBaseDuration": 0,
      "sibling": null,
      "stateNode": "[ReactNativeFiberHostComponent]",
      "tag": 5,
      "treeBaseDuration": 1,
      "type": "RCTView",
      "updateQueue": null
    },
    "_dispatchListeners": "[Function onResponderRelease]",
    "_targetInst": {
      "_debugHookTypes": null,
      "_debugID": 2678,
      "_debugIsCurrentlyTiming": false,
      "_debugNeedsRemount": false,
      "_debugOwner": "[FiberNode]",
      "_debugSource": {},
      "actualDuration": 1,
      "actualStartTime": 733,
      "alternate": null,
      "child": "[FiberNode]",
      "childExpirationTime": 0,
      "dependencies": null,
      "effectTag": 128,
      "elementType": "RCTView",
      "expirationTime": 0,
      "firstEffect": null,
      "index": 0,
      "key": null,
      "lastEffect": null,
      "memoizedProps": {},
      "memoizedState": null,
      "mode": 8,
      "nextEffect": null,
      "pendingProps": {},
      "ref": "[Function forwardRef]",
      "return": "[FiberNode]",
      "selfBaseDuration": 0,
      "sibling": null,
      "stateNode": "[ReactNativeFiberHostComponent]",
      "tag": 5,
      "treeBaseDuration": 1,
      "type": "RCTView",
      "updateQueue": null
    },
    "bubbles": null,
    "cancelable": null,
    "currentTarget": null,
    "defaultPrevented": null,
    "dispatchConfig": {
      "dependencies": [],
      "registrationName": "onResponderRelease"
    },
    "eventPhase": null,
    "isDefaultPrevented": "[Function functionThatReturnsFalse]",
    "isPropagationStopped": "[Function functionThatReturnsFalse]",
    "isTrusted": null,
    "nativeEvent": {
      "changedTouches": [],
      "identifier": 0,
      "locationX": 93.71428680419922,
      "locationY": 98.66666412353516,
      "pageX": 222.85714721679688,
      "pageY": 403.047607421875,
      "target": 707,
      "timestamp": 260103852,
      "touches": []
    },
    "target": {
      "_children": [],
      "_internalFiberInstanceHandleDEV": "[FiberNode]",
      "_nativeTag": 707,
      "viewConfig": {}
    },
    "timeStamp": 1718462366862,
    "touchHistory": {
      "indexOfSingleActiveTouch": 0,
      "mostRecentTimeStamp": 260103852,
      "numberActiveTouches": 0,
      "touchBank": []
    },
    "type": null
  }
]

```



没问题的：

```shell
[
  {
    "_dispatchInstances": null,
    "_dispatchListeners": null,
    "_targetInst": {
      "_debugHookTypes": null,
      "_debugID": 2689,
      "_debugIsCurrentlyTiming": false,
      "_debugNeedsRemount": false,
      "_debugOwner": "[FiberNode]",
      "_debugSource": {},
      "actualDuration": 0,
      "actualStartTime": 661,
      "alternate": null,
      "child": "[FiberNode]",
      "childExpirationTime": 0,
      "dependencies": null,
      "effectTag": 0,
      "elementType": "RCTText",
      "expirationTime": 0,
      "firstEffect": null,
      "index": 0,
      "key": null,
      "lastEffect": null,
      "memoizedProps": {},
      "memoizedState": null,
      "mode": 8,
      "nextEffect": null,
      "pendingProps": {},
      "ref": null,
      "return": "[FiberNode]",
      "selfBaseDuration": 0,
      "sibling": null,
      "stateNode": "[ReactNativeFiberHostComponent]",
      "tag": 5,
      "treeBaseDuration": 0,
      "type": "RCTText",
      "updateQueue": null
    },
    "bubbles": null,
    "cancelable": null,
    "currentTarget": null,
    "defaultPrevented": null,
    "dispatchConfig": {
      "dependencies": [],
      "registrationName": "onResponderEnd"
    },
    "eventPhase": null,
    "isDefaultPrevented": "[Function functionThatReturnsFalse]",
    "isPropagationStopped": "[Function functionThatReturnsFalse]",
    "isTrusted": null,
    "nativeEvent": {
      "changedTouches": [],
      "identifier": 0,
      "locationX": 59.80952453613281,
      "locationY": 100.19047546386719,
      "pageX": 168.38095092773438,
      "pageY": 225.14285278320312,
      "target": 707,
      "timestamp": 260247809,
      "touches": []
    },
    "target": {
      "_children": [],
      "_internalFiberInstanceHandleDEV": "[FiberNode]",
      "_nativeTag": 707,
      "viewConfig": {}
    },
    "timeStamp": 1718462510841,
    "touchHistory": {
      "indexOfSingleActiveTouch": 0,
      "mostRecentTimeStamp": 260247809,
      "numberActiveTouches": 0,
      "touchBank": []
    },
    "type": null
  },
  {
    "_dispatchInstances": {
      "_debugHookTypes": null,
      "_debugID": 2689,
      "_debugIsCurrentlyTiming": false,
      "_debugNeedsRemount": false,
      "_debugOwner": "[FiberNode]",
      "_debugSource": {},
      "actualDuration": 0,
      "actualStartTime": 661,
      "alternate": null,
      "child": "[FiberNode]",
      "childExpirationTime": 0,
      "dependencies": null,
      "effectTag": 0,
      "elementType": "RCTText",
      "expirationTime": 0,
      "firstEffect": null,
      "index": 0,
      "key": null,
      "lastEffect": null,
      "memoizedProps": {},
      "memoizedState": null,
      "mode": 8,
      "nextEffect": null,
      "pendingProps": {},
      "ref": null,
      "return": "[FiberNode]",
      "selfBaseDuration": 0,
      "sibling": null,
      "stateNode": "[ReactNativeFiberHostComponent]",
      "tag": 5,
      "treeBaseDuration": 0,
      "type": "RCTText",
      "updateQueue": null
    },
    "_dispatchListeners": "[Function onResponderRelease]",
    "_targetInst": {
      "_debugHookTypes": null,
      "_debugID": 2689,
      "_debugIsCurrentlyTiming": false,
      "_debugNeedsRemount": false,
      "_debugOwner": "[FiberNode]",
      "_debugSource": {},
      "actualDuration": 0,
      "actualStartTime": 661,
      "alternate": null,
      "child": "[FiberNode]",
      "childExpirationTime": 0,
      "dependencies": null,
      "effectTag": 0,
      "elementType": "RCTText",
      "expirationTime": 0,
      "firstEffect": null,
      "index": 0,
      "key": null,
      "lastEffect": null,
      "memoizedProps": {},
      "memoizedState": null,
      "mode": 8,
      "nextEffect": null,
      "pendingProps": {},
      "ref": null,
      "return": "[FiberNode]",
      "selfBaseDuration": 0,
      "sibling": null,
      "stateNode": "[ReactNativeFiberHostComponent]",
      "tag": 5,
      "treeBaseDuration": 0,
      "type": "RCTText",
      "updateQueue": null
    },
    "bubbles": null,
    "cancelable": null,
    "currentTarget": null,
    "defaultPrevented": null,
    "dispatchConfig": {
      "dependencies": [],
      "registrationName": "onResponderRelease"
    },
    "eventPhase": null,
    "isDefaultPrevented": "[Function functionThatReturnsFalse]",
    "isPropagationStopped": "[Function functionThatReturnsFalse]",
    "isTrusted": null,
    "nativeEvent": {
      "changedTouches": [],
      "identifier": 0,
      "locationX": 59.80952453613281,
      "locationY": 100.19047546386719,
      "pageX": 168.38095092773438,
      "pageY": 225.14285278320312,
      "target": 707,
      "timestamp": 260247809,
      "touches": []
    },
    "target": {
      "_children": [],
      "_internalFiberInstanceHandleDEV": "[FiberNode]",
      "_nativeTag": 707,
      "viewConfig": {}
    },
    "timeStamp": 1718462510841,
    "touchHistory": {
      "indexOfSingleActiveTouch": 0,
      "mostRecentTimeStamp": 260247809,
      "numberActiveTouches": 0,
      "touchBank": []
    },
    "type": null
  }
]

```

_targetInst中的elementType不同



往以下日志打印：

```javascript
var ResponderEventPlugin = {
  /* For unit testing only */
  _getResponder: function () {
    return responderInst;
  },
  eventTypes: eventTypes,

  /**
   * We must be resilient to `targetInst` being `null` on `touchMove` or
   * `touchEnd`. On certain platforms, this means that a native scroll has
   * assumed control and the original touch targets are destroyed.
   */
  extractEvents: function (
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags
  ) {
    console.log(`new topLevelType: ${topLevelType}, nativeEvent:`, nativeEvent);
    if (isStartish(topLevelType)) {
      console.log(`isStartish true`);
      console.log(`isStartish true, targetInst elementTypes:`, targetInst.elementType);
      trackedTouchCount += 1;
    } else if (isEndish(topLevelType)) {
      console.log(`isEndish true trackedTouchCount: ${trackedTouchCount}`);
      if (trackedTouchCount >= 0) {
        trackedTouchCount -= 1;
      } else {
        {
          warn(
            "Ended a touch event which was not counted in `trackedTouchCount`."
          );
        }

        return null;
      }
        
     //第二步
    var extracted = canTriggerTransfer(topLevelType, targetInst, nativeEvent)
      ? setResponderAndExtractTransfer(
        topLevelType,
        targetInst,
        nativeEvent,
        nativeEventTarget
      )
      : null;

       console.log(`incrementalTouch ${incrementalTouch}`);
        if (incrementalTouch) {
          var gesture = ResponderSyntheticEvent.getPooled(
            incrementalTouch,
            responderInst,
            nativeEvent,
            nativeEventTarget
          );
          console.log(`incrementalTouch accumulateDirectDispatches before gesture`);
          extractAndPrintElementType("before gesture", gesture);
          gesture.touchHistory = ResponderTouchHistoryStore.touchHistory;
          accumulateDirectDispatches(gesture);
          console.log(`incrementalTouch accumulateDirectDispatches end gesture`);
          extractAndPrintElementType("end gesture", gesture);


          console.log(`incrementalTouch before extracted :`);
          extractAndPrintElementType("before extracted", extracted);
          extracted = accumulate(extracted, gesture);
          console.log(`incrementalTouch end extracted :`);
          extractAndPrintElementType("end extracted", extracted);
        }
        
        if (finalTouch) {
          var finalEvent = ResponderSyntheticEvent.getPooled(
            finalTouch,
            responderInst,
            nativeEvent,
            nativeEventTarget
          );
          finalEvent.touchHistory = ResponderTouchHistoryStore.touchHistory;
          accumulateDirectDispatches(finalEvent);
          extracted = accumulate(extracted, finalEvent);
           //打印最终的elementType
          console.log(`extracted _targetInst.elementType:`, extracted[0]._targetInst.elementType);
          changeResponder(null);
        }
    } 
        
        
  }
    
    function extractAndPrintElementType(tag, extracted) {
      if (extracted == null) {
        console.log(`${tag} extracted is null`);
        return;
      }
      if (Array.isArray(extracted)) {
        // 如果是数组，遍历每个对象并打印 elementType 的值
        extracted.forEach(item => {
          if (item && item.elementType) {
            console.log(`${tag} elementType:`, item.elementType);
          }
        });
      } else if (typeof extracted === 'object') {
        // 如果是对象，遍历对象的所有属性值并打印 elementType 的值
        Object.values(extracted).forEach(item => {
          if (item && item.elementType) {
            console.log(`${tag} elementType:`, item.elementType);
          }
        });
      } else {
        console.error('Invalid input. Expected an object or an array.');
      }
    }


function setResponderAndExtractTransfer(
  topLevelType,
  targetInst,
  nativeEvent,
  nativeEventTarget
) {
   if (responderInst) {
        if (shouldSwitch) {
          console.log(`wantsResponderInst shouldSwitch: ${shouldSwitch}`);
          //...省略代码
        } 
      } else {
         //无嵌套情况走的分支
        console.log(`wantsResponderInst responderInst: is null`);
        extracted = accumulate(extracted, grantEvent);
        changeResponder(wantsResponderInst, blockHostResponder);
      }

      return extracted;
    } 
  }

```





异常日志：

```shell
[Sun Jun 16 2024 11:19:11.800]  LOG      new topLevelType: topTouchStart, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 99.42857360839844, "locationY": 88, "pageX": 228.57142639160156, "pageY": 507.80950927734375, "target": 707, "timestamp": 271427931, "touches": [[Circular]]}
[Sun Jun 16 2024 11:19:11.129]  LOG      isStartish true
[Sun Jun 16 2024 11:19:11.129]  LOG      isStartish true, targetInst elementTypes: RCTText
[Sun Jun 16 2024 11:19:11.130]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sun Jun 16 2024 11:19:11.130]  LOG      skipOverBubbleShouldSetFrom: false
[Sun Jun 16 2024 11:19:11.130]  LOG      wantsResponderInst responderInst: is null
[Sun Jun 16 2024 11:19:11.130]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:19:11.131]  LOG      incrementalTouch : {"dependencies": ["topTouchStart"], "registrationName": "onResponderStart"}
[Sun Jun 16 2024 11:19:11.131]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:19:11.131]  LOG      before gesture elementType: RCTText
[Sun Jun 16 2024 11:19:11.131]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:19:11.131]  LOG      end gesture elementType: RCTText
[Sun Jun 16 2024 11:19:11.132]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:19:11.132]  LOG      before extracted elementType: RCTText
[Sun Jun 16 2024 11:19:11.164]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:19:11.165]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
        
[Sun Jun 16 2024 11:19:11.165]  LOG      new topLevelType: topTouchStart, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 99.42857360839844, "locationY": 88, "pageX": 208, "pageY": 212.95237731933594, "target": 707, "timestamp": 271427951, "touches": [[Circular]]}
[Sun Jun 16 2024 11:19:11.165]  LOG      isStartish true
[Sun Jun 16 2024 11:19:11.166]  LOG      isStartish true, targetInst elementTypes: RCTText
[Sun Jun 16 2024 11:19:11.166]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sun Jun 16 2024 11:19:11.166]  LOG      skipOverBubbleShouldSetFrom: true
[Sun Jun 16 2024 11:19:11.166]  LOG      wantsResponderInst shouldSwitch: true
[Sun Jun 16 2024 11:19:11.167]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:19:11.167]  LOG      incrementalTouch : {"dependencies": ["topTouchStart"], "registrationName": "onResponderStart"}
[Sun Jun 16 2024 11:19:11.167]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:19:11.167]  LOG      before gesture elementType: RCTView
[Sun Jun 16 2024 11:19:11.168]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:19:11.168]  LOG      end gesture elementType: RCTView
[Sun Jun 16 2024 11:19:11.168]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:19:11.168]  LOG      accumulate curent isArray concat next
[Sun Jun 16 2024 11:19:11.168]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:19:11.169]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
        
        
[Sun Jun 16 2024 11:19:11.183]  LOG      new topLevelType: topTouchMove, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 99.42857360839844, "locationY": 88, "pageX": 208, "pageY": 212.95237731933594, "target": 707, "timestamp": 271427975, "touches": [[Circular]]}
[Sun Jun 16 2024 11:19:11.183]  LOG      setResponderAndExtractTransfer: topTouchMove
[Sun Jun 16 2024 11:19:11.183]  LOG      skipOverBubbleShouldSetFrom: true
[Sun Jun 16 2024 11:19:11.184]  LOG      incrementalTouch : {"dependencies": ["topTouchMove"], "registrationName": "onResponderMove"}
[Sun Jun 16 2024 11:19:11.184]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:19:11.184]  LOG      before gesture elementType: RCTView
[Sun Jun 16 2024 11:19:11.184]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:19:11.185]  LOG      end gesture elementType: RCTView
[Sun Jun 16 2024 11:19:11.185]  LOG      end gesture elementType: RCTView
[Sun Jun 16 2024 11:19:11.185]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:19:11.185]  LOG      before extracted extracted is null
[Sun Jun 16 2024 11:19:11.186]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:19:11.186]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:19:11.186]  LOG      end extracted elementType: RCTView
[Sun Jun 16 2024 11:19:11.186]  LOG      end extracted elementType: RCTView
[Sun Jun 16 2024 11:19:11.187]  LOG      isResponderTerminate： false，
        finalTouch： null
        
        
        
[Sun Jun 16 2024 11:19:11.195]  LOG      new topLevelType: topTouchEnd, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 99.42857360839844, "locationY": 88, "pageX": 228.57142639160156, "pageY": 507.80950927734375, "target": 707, "timestamp": 271427984, "touches": []}
[Sun Jun 16 2024 11:19:11.195]  LOG      isEndish true trackedTouchCount: 2
[Sun Jun 16 2024 11:19:11.196]  LOG      incrementalTouch : {"dependencies": ["topTouchCancel", "topTouchEnd"], "registrationName": "onResponderEnd"}
[Sun Jun 16 2024 11:19:11.196]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:19:11.196]  LOG      before gesture elementType: RCTView
[Sun Jun 16 2024 11:19:11.196]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:19:11.197]  LOG      end gesture elementType: RCTView
[Sun Jun 16 2024 11:19:11.197]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:19:11.197]  LOG      before extracted extracted is null
[Sun Jun 16 2024 11:19:11.197]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:19:11.197]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:19:11.201]  LOG      end extracted elementType: RCTView
[Sun Jun 16 2024 11:19:11.201]  LOG      isResponderTerminate： false，
        finalTouch： {"dependencies": ["topTouchCancel", "topTouchEnd"], "registrationName": "onResponderRelease"}
[Sun Jun 16 2024 11:19:11.202]  LOG      finalTouch extracted _targetInst.elementType: RCTView




[Sun Jun 16 2024 11:19:11.304]  LOG      new topLevelType: topTouchEnd, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 99.42857360839844, "locationY": 88, "pageX": 208, "pageY": 212.95237731933594, "target": 707, "timestamp": 271427991, "touches": []}
[Sun Jun 16 2024 11:19:11.304]  LOG      isEndish true trackedTouchCount: 1
[Sun Jun 16 2024 11:19:11.305]  LOG      incrementalTouch : null
[Sun Jun 16 2024 11:19:11.305]  LOG      isResponderTerminate： null，
        finalTouch： null
```



正常日志：

```javascript
[Sun Jun 16 2024 11:22:45.635]  LOG      new topLevelType: topTouchStart, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 100.19047546386719, "locationY": 73.9047622680664, "pageX": 229.3333282470703, "pageY": 398.0952453613281, "target": 707, "timestamp": 271642539, "touches": [[Circular]]}
[Sun Jun 16 2024 11:22:45.647]  LOG      isStartish true
[Sun Jun 16 2024 11:22:45.647]  LOG      isStartish true, targetInst elementTypes: RCTText
[Sun Jun 16 2024 11:22:45.648]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sun Jun 16 2024 11:22:45.648]  LOG      skipOverBubbleShouldSetFrom: false
[Sun Jun 16 2024 11:22:45.648]  LOG      wantsResponderInst responderInst: is null
[Sun Jun 16 2024 11:22:45.648]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:22:45.648]  LOG      incrementalTouch : {"dependencies": ["topTouchStart"], "registrationName": "onResponderStart"}
[Sun Jun 16 2024 11:22:45.649]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:22:45.649]  LOG      before gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.649]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:22:45.650]  LOG      end gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.650]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:22:45.650]  LOG      before extracted elementType: RCTText
[Sun Jun 16 2024 11:22:45.650]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:22:45.651]  LOG      isResponderTerminate： false，
        finalTouch： null



[Sun Jun 16 2024 11:22:45.651]  LOG      new topLevelType: topTouchCancel, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 100.19047546386719, "locationY": 73.9047622680664, "pageX": 208.76190185546875, "pageY": 198.85714721679688, "target": 707, "timestamp": 271642544, "touches": []}
[Sun Jun 16 2024 11:22:45.651]  LOG      isEndish true trackedTouchCount: 1
[Sun Jun 16 2024 11:22:45.651]  LOG      incrementalTouch : {"dependencies": ["topTouchCancel", "topTouchEnd"], "registrationName": "onResponderEnd"}
[Sun Jun 16 2024 11:22:45.651]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:22:45.652]  LOG      before gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.652]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:22:45.665]  LOG      end gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.665]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:22:45.665]  LOG      before extracted extracted is null
[Sun Jun 16 2024 11:22:45.665]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:22:45.666]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:22:45.666]  LOG      end extracted elementType: RCTText
[Sun Jun 16 2024 11:22:45.666]  LOG      isResponderTerminate： true，
        finalTouch： {"dependencies": [], "registrationName": "onResponderTerminate"}
[Sun Jun 16 2024 11:22:45.666]  LOG      finalTouch extracted _targetInst.elementType: RCTText




[Sun Jun 16 2024 11:22:45.667]  LOG      new topLevelType: topTouchStart, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 100.19047546386719, "locationY": 73.9047622680664, "pageX": 208.76190185546875, "pageY": 198.85714721679688, "target": 707, "timestamp": 271642561, "touches": [[Circular]]}
[Sun Jun 16 2024 11:22:45.667]  LOG      isStartish true
[Sun Jun 16 2024 11:22:45.667]  LOG      isStartish true, targetInst elementTypes: RCTText
[Sun Jun 16 2024 11:22:45.667]  LOG      setResponderAndExtractTransfer: topTouchStart
[Sun Jun 16 2024 11:22:45.667]  LOG      skipOverBubbleShouldSetFrom: false
[Sun Jun 16 2024 11:22:45.671]  LOG      wantsResponderInst responderInst: is null
[Sun Jun 16 2024 11:22:45.671]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:22:45.671]  LOG      incrementalTouch : {"dependencies": ["topTouchStart"], "registrationName": "onResponderStart"}
[Sun Jun 16 2024 11:22:45.672]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:22:45.679]  LOG      before gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.679]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:22:45.679]  LOG      end gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.679]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:22:45.680]  LOG      before extracted elementType: RCTText
[Sun Jun 16 2024 11:22:45.680]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:22:45.681]  LOG      isResponderTerminate： false，
        finalTouch： null




[Sun Jun 16 2024 11:22:45.681]  LOG      new topLevelType: topTouchMove, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 100.19047546386719, "locationY": 73.9047622680664, "pageX": 208.76190185546875, "pageY": 198.85714721679688, "target": 707, "timestamp": 271642574, "touches": [[Circular]]}
[Sun Jun 16 2024 11:22:45.681]  LOG      setResponderAndExtractTransfer: topTouchMove
[Sun Jun 16 2024 11:22:45.681]  LOG      skipOverBubbleShouldSetFrom: true
[Sun Jun 16 2024 11:22:45.682]  LOG      incrementalTouch : {"dependencies": ["topTouchMove"], "registrationName": "onResponderMove"}
[Sun Jun 16 2024 11:22:45.682]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:22:45.682]  LOG      before gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.682]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:22:45.683]  LOG      end gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.683]  LOG      end gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.683]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:22:45.683]  LOG      before extracted extracted is null
[Sun Jun 16 2024 11:22:45.684]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:22:45.684]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:22:45.684]  LOG      end extracted elementType: RCTText
[Sun Jun 16 2024 11:22:45.684]  LOG      end extracted elementType: RCTText
[Sun Jun 16 2024 11:22:45.685]  LOG      isResponderTerminate： false，
        finalTouch： null



[Sun Jun 16 2024 11:22:45.685]  LOG      new topLevelType: topTouchEnd, nativeEvent: {"changedTouches": [[Circular]], "identifier": 0, "locationX": 100.19047546386719, "locationY": 73.9047622680664, "pageX": 208.76190185546875, "pageY": 198.85714721679688, "target": 707, "timestamp": 271642578, "touches": []}
[Sun Jun 16 2024 11:22:45.685]  LOG      isEndish true trackedTouchCount: 1
[Sun Jun 16 2024 11:22:45.685]  LOG      incrementalTouch : {"dependencies": ["topTouchCancel", "topTouchEnd"], "registrationName": "onResponderEnd"}
[Sun Jun 16 2024 11:22:45.835]  LOG      incrementalTouch accumulateDirectDispatches before gesture
[Sun Jun 16 2024 11:22:45.836]  LOG      before gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.836]  LOG      incrementalTouch accumulateDirectDispatches end gesture
[Sun Jun 16 2024 11:22:45.836]  LOG      end gesture elementType: RCTText
[Sun Jun 16 2024 11:22:45.837]  LOG      incrementalTouch before extracted :
[Sun Jun 16 2024 11:22:45.837]  LOG      before extracted extracted is null
[Sun Jun 16 2024 11:22:45.837]  LOG      accumulate curent is null, retuern  next
[Sun Jun 16 2024 11:22:45.837]  LOG      incrementalTouch end extracted :
[Sun Jun 16 2024 11:22:45.838]  LOG      end extracted elementType: RCTText
[Sun Jun 16 2024 11:22:45.838]  LOG      isResponderTerminate： false，
        finalTouch： {"dependencies": ["topTouchCancel", "topTouchEnd"], "registrationName": "onResponderRelease"}
[Sun Jun 16 2024 11:22:45.838]  LOG      finalTouch extracted _targetInst.elementType: RCTText

```

可以看到失败的核心原因是：

两次的topTouchStart事件，造成了点击事件响应对象的切换

想办法不让该中情况switch可解决问题

```javascript
function setResponderAndExtractTransfer(
  topLevelType,
  targetInst,
  nativeEvent,
  nativeEventTarget
) {
   if (responderInst) {
        if (shouldSwitch) {
          console.log(`wantsResponderInst shouldSwitch: ${shouldSwitch}`);
          //提前return
          return
          //忽略代码
        } 
      } else {
         //去嵌套情况走的分支
        console.log(`wantsResponderInst responderInst: is null`);
        extracted = accumulate(extracted, grantEvent);
        changeResponder(wantsResponderInst, blockHostResponder);
      }

      return extracted;
    } 
  }
```



native解法：

```java
ReactRootView reactRootView = new ReactRootView(context) {
  @Override
  public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
      ViewParent parent = getParent();
      while (parent != null) {
        if (parent instanceof RootView) {
          break;
        }
        parent = parent.getParent();
      }
      if (parent != null && parent instanceof RootView) {
        Log.d("Awille", "disable parent dispatch");
        ((RootView) parent).onChildStartedNativeGesture(ev);
      }
    }
    return super.onInterceptTouchEvent(ev);
  }
};
```







##  附录

### ReactZIndexedViewGroup

查看关于ReactZIndexedViewGroup的实现，发现返回的Z-Index实际是JS层设置的，这里可以了解一下概念，但暂时不深究。

在 React Native 中,z-index 是一个既存在于原生平台(iOS、Android)层面,也存在于 JavaScript 层面的概念。

1. 原生平台层面的 z-index:
   - 在原生 UI 系统中,z-index 属性用于控制视图的层叠顺序。
   - 这是一个底层的概念,由原生平台的渲染引擎来负责管理和应用。
2. JavaScript 层面的 z-index:
   - 在 React Native 的 JavaScript 层面,也存在 z-index 的概念和使用。
   - 这主要是为了让开发者能够在 JavaScript 代码中声明和控制视图的层次关系。
   - 开发者可以在 React Native 组件的样式中设置 z-index 属性,这会影响组件在视图树中的渲染顺序。
3. 两个层面的集成:
   - `ReactZIndexedViewGroup` 这个组件就是负责将 JavaScript 层面的 z-index 概念与原生平台层面的 z-index 机制进行集成和协调。
   - 它会根据 JavaScript 层面设置的 z-index 值,动态地维护原生平台层面的视图层次关系。
   - 这样就保证了 JavaScript 代码中设置的 z-index 能够正确地映射到原生平台的渲染结果。

所以总的来说,z-index 在 React Native 中是一个跨越原生平台和 JavaScript 层面的概念。`ReactZIndexedViewGroup` 扮演着连接这两个层面的关键角色,确保开发者能够在 JavaScript 层面上自然地控制视图的层次关系。



IntegrationTestsApp

### ReactCompoundView

有关`ReactCompoundView` 是 React Native 中一种特殊的视图类型,它具有以下特点:

1. 它既是一个 `ViewGroup`(可以包含子视图),又是一个 `ReactCompoundView`。
2. 它表示一个复合的 React 视图,通常由多个子视图组成。
3. 它可以接收指针事件(touch事件),并且可以将这些事件转发给内部的子视图。

那么,什么时候会使用 `ReactCompoundView` 呢?主要有以下几种情况:

1. **自定义复合视图**: 在 React Native 开发中,开发者经常需要构建一些复杂的视图组件,由多个子视图组成。这时就可以使用 `ReactCompoundView` 来实现,把这些子视图组织在一起,并提供统一的事件处理机制。
2. **原生视图封装**: 当需要将原生平台(Android/iOS)上的一些复杂视图组件封装到 React Native 中时,也会使用 `ReactCompoundView`。这样可以将原生视图的行为和事件处理逻辑封装在 `ReactCompoundView` 中,提供统一的 React 组件接口。
3. **优化事件处理**: 如果一个视图树中存在设置了 `PointerEvents.BOX_NONE` 的视图,但其子视图仍然需要接收事件,那么使用 `ReactCompoundView` 可以帮助正确地将事件分发到子视图。之前介绍的那段代码就是处理这种情况的。

总的来说, `ReactCompoundView` 是 React Native 中一种特殊的视图类型,用于构建复合视图组件,并提供统一的事件处理机制。它在自定义视图开发和原生视图封装等场景中都有重要应用。



