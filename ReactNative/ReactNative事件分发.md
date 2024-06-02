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

Libraries\Renderer\implementations\ReactFabric-dev.js



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



