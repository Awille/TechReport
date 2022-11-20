# 事件分发-mFirstTouchTarget

## 1、简介

在ViewGroup中，mFirstTouchTarget扮演了一个非常重要的角色，但是对于里面的设计我们可能不太清晰，比如为什么是个链表，源码中的pointerIdBits扮演了怎样的角色，本篇文章可以深入讨论下。



##  2、TouchTarget介绍

```java
// First touch target in the linked list of touch targets.
@UnsupportedAppUsage
private TouchTarget mFirstTouchTarget;
```

首先mFirstTouchTarget为TouchTarget类型，官方释义为 触摸链表中的第一个触摸目标。 何为触摸目标，触摸目标指的是接收当前点击事件的控件。 由此来看，接收当前点击事件的控件是可能是多个的，而且构成一个链表结构。

TouchTarget的结果为：

```java
	/* Describes a touched view and the ids of the pointers that it has captured.
     *
     * This code assumes that pointer ids are always in the range 0..31 such that
     * it can use a bitfield to track which pointer ids are present.
     * As it happens, the lower layers of the input dispatch pipeline also use the
     * same trick so the assumption should be safe here...
     */
/*
描述了一个触控的View以及该view补货的ids of pointers。
*/
private static final class TouchTarget {

    // The touched child view.
    @UnsupportedAppUsage
    public View child;

    // The combined bit mask of pointer ids for all pointers captured by the target.
    public int pointerIdBits;

    // The next target in the target list.
    public TouchTarget next;
}
```

观察其成员变量有三个：

* child 触控的view
* pointerIdBits  该view捕获的poniter ids的组合掩码。在android 中，为了区分多点触控时不同的触控点，每个触控点都会携带一个pointerId。 pointerIdBits  是该view消耗的所有触控点的poniterId的组合。
* next 下一个目标

TouchTarger中存在obtain和recycle两个方法，这两个方法在消息队列中也有用到，用于缓存复用，我们可以解析下这两个方法。

```java
private static final class TouchTarget {
    private static final int MAX_RECYCLED = 32;
    private static final Object sRecycleLock = new Object[0];
    private static TouchTarget sRecycleBin;
    private static int sRecycledCount;

    public static final int ALL_POINTER_IDS = -1; // all ones
    
    public static TouchTarget obtain(@NonNull View child, int pointerIdBits) {
        //入参为当前触控的view 和 其消费的每个触点的pointerId的组合-pointerIdBits
        if (child == null) {
            throw new IllegalArgumentException("child must be non-null");
        }

        final TouchTarget target;
        //对象锁处理并发情况
        synchronized (sRecycleLock) {
            if (sRecycleBin == null) {
                //为空代表第一次创建，直接new
                target = new TouchTarget();
            } else {
                //分析这里钱先去看recycle的实现
                //targert直接复用对象。
                target = sRecycleBin;
                //sRecycleBin记录为上一次复用的对象，在recycler时已经被记录到next节点。
                sRecycleBin = target.next;
                //已被复用，sRecycledCount计数-1
                sRecycledCount--;
                target.next = null;
            }
        }
        //获取到复用的或者新建的target，赋上新值
        target.child = child;
        target.pointerIdBits = pointerIdBits;
        return target;
    }

    public void recycle() {
        if (child == null) {
            throw new IllegalStateException("already recycled once");
        }
		//对象锁处理并发情况
        synchronized (sRecycleLock) {
            //假如这个缓存一直没有被重新使用，会记录sRecycledCount次数，并再超过32次时，直接把该缓存对象回收。
            if (sRecycledCount < MAX_RECYCLED) {
                //用next节点记录之前的sRecycleBin 
                next = sRecycleBin;
                //sRecycleBin记录为当前的对象，用于以后复用。
                sRecycleBin = this;
                //复用次数+1
                sRecycledCount += 1;
            } else {
                next = null;
            }
            //child赋为空值，供下次使用
            child = null;
        }
        //最后recycler后的结果是，之前被缓存的复用对象记录到next节点，然后当前的sRecycleBin记录为当前的view，并清空child
    }
}
```

小结：TouchTarget是把消耗事件的view以链表的形式存储，并且记录各个view对应的触控点链表。并且可以推断出：

* 单点触控场景：mFirstTouchTarget链表中只存在单个TouchTarget对象， pointerIdBits保存了该单个触控点的pointerId。
* 多点触控，且目标相同：mFirstTouchTarget链表中只存在单个TouchTarget对象， pointerIdBits保存了多个触控点的pointerId。
* 多点触控，且目标不同：`mFirstTouchTarget`成为链表，每个节点记录了touchView以及该view捕获的触控点的pointerId。

## 3、ViewGroup.dispatchTouchEvent流程分析

### 3.1、初始ACTION_DOWN事件处理

```java
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    //当新的手势开始时，丢弃之前所有的状态。
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
/**
     * Resets all touch state in preparation for a new cycle.
     */
private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}

/**
     * Clears all touch targets.
     */
private void clearTouchTargets() {
    TouchTarget target = mFirstTouchTarget;
    if (target != null) {
        do {
            TouchTarget next = target.next;
            target.recycle();
            target = next;
        } while (target != null);
        mFirstTouchTarget = null;
    }
}
```

可以关注到在ACTION_DOWN事件为所有触摸事件的起点，这里会调用resetTouchState()->clearTouchTargets() 把mFirstTouchTarget整个链表全部清空。当ACTION_DOWN事件开始时，mFirstTouchTarget=null。

### 3.2 ViewGroup是否拦截事件处理

```java
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
    || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}
```

当为ACTION_DOWN事件时, 根据disallowIntercept标志位判断是否调用onInterceptTouchEvent进行拦截。disallowIntercept该标志位可由子view调用，初始为false， 让父控件不允许拦截事件。

当为ACTION_DOWN以外的事件时，如果mFirstTouchTarget不为空才进行拦截判断逻辑，其他情况都默认拦截。

ok，看到这里，我们就找到了我们要追踪的主线，mFirstTouchTarget到底何时被创建，可以看到在当前viewGroup是否拦截事件的处理上，它发挥着至关重要的角色。

### 3.3、是否取消标志位处理& 是否支持多点触控

```java
// Check for cancelation.
//判断该事件是否要取消，如果事件本身为取消则直接取消，另一种情况是当该viewGroup持有PFLAG_CANCEL_NEXT_UP_EVENT标记时，会取消该事件。至于那种情况下回出现view持有PFLAG_CANCEL_NEXT_UP_EVENT 以后再去探究把，当前我们直接当没有这种场景。
final boolean canceled = resetCancelNextUpFlag(this)
    || actionMasked == MotionEvent.ACTION_CANCEL;

// Update list of touch targets for pointer down, if needed.
//是否支持多点触控，目前基本上都为true，我们也忽略掉不支持多点触控的场景。
final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;

//临时变量-记录最终消费的TouchTarget
TouchTarget newTouchTarget = null;
//临时变量-记录已经被分发的事件，避免重复分发。
boolean alreadyDispatchedToNewTouchTarget = false;
```

* viewGroup持有PFLAG_CANCEL_NEXT_UP_EVENT标记的场景(只做记录不要过分关心)
  设置该标签的方法之一为：performButtonActionOnTouchDown()，输入事件为鼠标右键，这种我们直接忽略，现在正常人不会接个鼠标来玩手机的。 方法二为onStartTemporaryDetach()，该函数在子控件与父控件临时分离时调用，临时分离在recyclerView或者listView中可能发生，当父空间与子控件临时分离时，点击事件传递到该viewGroup时，标记为取消。
* FLAG_SPLIT_MOTION_EVENTS： 是否支持拆解MotionEvents，即多点触控，在android3.0后默认支持，也可通过setMotionEventSplittingEnabled手动管理。



### 3.4、事件分发的核心流程

当事件没有被cancel 并且 当前viewGroup没有拦截该事件时，则进入ViewGroup事件分发的核心逻辑。

```java
if (actionMasked == MotionEvent.ACTION_DOWN
    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
    //当为ACTION_DOWN、ACTION_POINTER_DOWN时，进行处理。 MotionEvent.ACTION_HOVER_MOVE为鼠标事件忽略掉。
    //获取触控点下标，表明这是第几个触控点
    final int actionIndex = ev.getActionIndex(); // always 0 for down
    //如果支持多点触控的话用idBitsToAssign记录触控点id， 移位运算。 poniterId一般从0开始，每次+1. 
    //ponterId为0时，记录为0000 0001 为1时，记录为0000 0010 为5时，记录为00010 0000
    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
        : TouchTarget.ALL_POINTER_IDS;
    // Clean up earlier touch targets for this pointer id in case they
    // have become out of sync.
    //清楚touchTarget中的pointerIds
    removePointersFromTouchTargets(idBitsToAssign);
    ...
}

/**
     * Removes the pointer ids from consideration.
     */
private void removePointersFromTouchTargets(int pointerIdBits) {
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    //这里判断mFirstTouchTarget不为null时才操作，我们暂时追踪到mFirstTouchTarget何时构建，所以可以先忽略，后面再回过头来看。
    //待创建mFirstTouchTarget再来补充。 你们可以追到后面看到touchTarget创建再回过头来看。
    while (target != null) {
        final TouchTarget next = target.next;
        if ((target.pointerIdBits & pointerIdBits) != 0) {
            //如果当前target中的触控点包含此次的触控点 清楚记录
            target.pointerIdBits &= ~pointerIdBits;
            if (target.pointerIdBits == 0) {
                //如果当前target没有触控点了，将当前target从多点链表中移除。
                //由这步操作可以想到，每次的触控点发生时，都要检查之前的touchTarget是否包含该触控点poniterId,如果包含要移除，并且如果该touchTarget没有任何触控点信息了，要直接从链表中移除。
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
```

继续看removePointersFromTouchTargets下面的逻辑：

```java
final int childrenCount = mChildrenCount;
if (newTouchTarget == null && childrenCount != 0) {
    //当前viewGroup子view大于0时，进行进行一下逻辑
    final float x = ev.getX(actionIndex);
    final float y = ev.getY(actionIndex);
    // Find a child that can receive the event.
    // Scan children from front to back.
    
    //子控件布局排序机制---------忽略
    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
    final boolean customOrder = preorderedList == null
        && isChildrenDrawingOrderEnabled();
    //认为customOrder=false
    final View[] children = mChildren;
    for (int i = childrenCount - 1; i >= 0; i--) {
        //-----------------------------------------------
        final int childIndex = getAndVerifyPreorderedIndex(
            childrenCount, i, customOrder);
        final View child = getAndVerifyPreorderedView(
            preorderedList, children, childIndex);
        //------------------------------------------------
        //由于我们忽略子控件布局排序机制的场景，所以以上两行代码可以理解为：
        //** childIndex=i   child = children[childIndex];

        // If there is a view that has accessibility focus we want it
        // to get the event first and if not handled we will perform a
        // normal dispatch. We may do a double iteration but this is
        // safer given the timeframe.
        
       // accessibility focus的场景我们也可以先-------------------忽略
        if (childWithAccessibilityFocus != null) {
            if (childWithAccessibilityFocus != child) {
                continue;
            }
            childWithAccessibilityFocus = null;
            i = childrenCount - 1;
        }
        //----------------------------------------------------------

        if (!child.canReceivePointerEvents()
            || !isTransformedTouchPointInView(x, y, child, null)) {
            //child.canReceivePointerEvents()判断view是否可以接收点控，当visivle或者在执行动画时返回true
            //isTransformedTouchPointInView 判断当前view是否在当前触控的坐标中
            //所以这里当view为gong、invisivble或者view的返回不再点击事件坐标中都忽略掉，跳过该子view
            ev.setTargetAccessibilityFocus(false);
            continue;
        }
        
```

在上面的逻辑中，根据view的状态，以及是否在点击事件的坐标中，过滤掉了不接受事件的view。

```java
//对于可以接收点控的view开始下面的逻辑：
newTouchTarget = getTouchTarget(child);
//查找该子view是否已经在touchTarget链表中
if (newTouchTarget != null) {
    // Child is already receiving touch within its bounds.
    // Give it the new pointer in addition to the ones it is handling.
    //如果该view已经接收过之前的触摸事件，将这次新的触摸点加到该view的pointerIdBits当中，说明该view接收了多个触点。 
    newTouchTarget.pointerIdBits |= idBitsToAssign;
    //停止遍历逻辑，说明已经找到该消费该事件的view
    break;
}
/**
     * Gets the touch target for specified child view.
     * Returns null if not found.
     */
private TouchTarget getTouchTarget(@NonNull View child) {
    for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
        if (target.child == child) {
            return target;
        }
    }
    return null;
}
```

这里的逻辑可以看到，一个view是可能接收多个触点的，这里提出一个问题：

* 假如有个控件A，点用一个手指点击不要放，再用另一个手指一起点击，后面放开其中一个手指，点击事件并没有触发，再放开剩下一个手指时，点击事件才触发这是为什么？

  可以分析下，再第二个手指点击时，出现上述代码的逻辑，控件A的pointerIdBits中记录了两个触点，当某个手指放开时，收到的是ACTION_POINTER_UP事件，而不是ACTION_UP，所以第一只手指放开是不产生触控事件的。这里为什么是产生ACTION_POINTER_UP事件而不是ACTION_UP事件，具体逻辑是在dispatchTransformedTouchEvent当中。这也是接下来马上要分析的逻辑

我们继续回到主线往下看：

```java
//dispatchTransformedTouchEvent的逻辑我们以后再分析，先总结下逻辑：
//如果传入的子view消耗了此事件，返回true，则返回true，否则返回false。
if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
    // Child wants to receive touch within its bounds.
    //子view想在其范围内接收触摸
    mLastTouchDownTime = ev.getDownTime();
    //----------------------------------------忽略与子控件布局排序场景 即认为preorderedList=null
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
    //----------------------------------------
    mLastTouchDownX = ev.getX();
    mLastTouchDownY = ev.getY();
    //到这里就是我们追溯的终点 newTouchTarget的创建
    newTouchTarget = addTouchTarget(child, idBitsToAssign);
    alreadyDispatchedToNewTouchTarget = true;
    break;
}

/**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    //缓存复用创建TouchTarget
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    //当前TouchTarget被插到mFirstTouchTarget链表的头部
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

理一下dispatchTransformedTouchEvent的逻辑

```java
/**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
                                              View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        //如果是CANCEL的场景
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            //child传入null  父类未view，那么该viewGroup将退化成view，走view 的事件分发，会执行onTouchEvent等逻辑
            handled = super.dispatchTouchEvent(event);
        } else {
            //调用子类的dispatchTouchEvent
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    //------------------------------------------------
    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
        return false;
    }
    //以上一段是一致性校验，确保当前触控的poniterId跟调用该函数前 是保持一致的。 说实话，我也不知道出现不一致的场景有哪些我们就不关心了。
    //...............................................

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
        //我们只关注该场景：
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                //ViewGroup退化为view直接处理
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);
                //调用子view的dispatchTouchEvent
                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            //返回消费结果
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }

    // Perform any necessary transformations and dispatch.
    if (child == null) {
        //ViewGroup退化为view直接处理
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        //调用子view的dispatchTouchEvent
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```

总结下来就是，如果传入子view为null，则ViewGroup退化为view调用dispatchTouchEvent，否则调用子view的diapatchTouchEvent。

看到创建touchTarget的过程后我们继续往下看：

```java
//case1 ----------------------------------------------------------------
if (newTouchTarget == null && mFirstTouchTarget != null) {
    //该case发生下 发生了ACTION_POINTER_DOWN事件，即多点触控场景
    //遍历子view没找到接收该点控的view 但是 之前mFirstTouchTarget不为空(这个case只发生在非初始ACTION_DOWN事件中，mFirstTouchTarget在第一次的ACTION_DOWN事件中构建好了， 所以一般情况为无法找到消耗ACTION_POINTER_DOWN事件时产生。)
    // Did not find a child to receive the event.
    // Assign the pointer to the least recently added target.
    //将此点控(一般为ACTION_POINTER_DOWN)的pointerId记录到最先链表的尾部(即第一次创建的touchTarget，一般为消耗ACRION_DOWN的控件)
    //这里可以引入一个问题 按下A，按下VG（空白区域），为什么先释放A，却无法触发A的点击事件，继续释放VG，又会触发A的点击事件。ACTION_DOWN被A消耗，则mFirstTouchTarget的末尾元素为A，后续没有被消耗的ACTION_POINTER_DOWN事件都会传入A中，此时相当于A是2个触控点的目标元素。
    newTouchTarget = mFirstTouchTarget;
    while (newTouchTarget.next != null) {
        newTouchTarget = newTouchTarget.next;
    }
    newTouchTarget.pointerIdBits |= idBitsToAssign;
}

//case2 ---------------------------------------------------------------------------
// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    //进入该case时，代表该事件未找到排粉目标
    // No touch targets so treat this as an ordinary view.
    //mFirstTouchTarget为空， dispatchTransformedTouchEvent中view传入null代表自己。ViewGroup退化成View处理该点击事件。
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                            TouchTarget.ALL_POINTER_IDS);
} else {
    //case3 -------------------------------------------------------------
    // Dispatch to touch targets, excluding the new touch target if we already
    // dispatched to it.  Cancel touch targets if necessary.
    //发生在非ACTION_DOWN事件
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    while (target != null) {
        final TouchTarget next = target.next;
        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
            //用到了alreadyDispatchedToNewTouchTarget标记，用于过滤新建TouchTarget时已消耗事件的情况，避免重复派分。
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
```

可以做个小结：

以上部分可能导致的结果有：

| /                                    | mFirstTouchTarget | newTouchTarget       | alreadyDispatchedToNewTouchTarget |
| ------------------------------------ | ----------------- | -------------------- | --------------------------------- |
| `ACTION_DOWN` 无目标                 | `null`            | `null`               | `false`                           |
| `ACTION_DOWN` 有目标                 | `TouchTarget(0)`  | `new TouchTarget(0)` | `true`                            |
| `ACTION_POINTER_DOWN` 无目标         | `TouchTarget(0)`  | `TouchTarget(0)`     | `false`                           |
| `ACTION_POINTER_DOWN` 有目标(已存在) | 不变              | `find TouchTarget`   | `false`                           |
| `ACTION_POINTER_DOWN` 有目标(不存在) | `TouchTarget(n)`  | `new TouchTarget(n)` | `true`                            |

可以总结到如下性质：

1. 标记位`alreadyDispatchedToNewTouchTarget`只会在新建`TouchTarget`时设置`true`。
2. `ACTION_DOWN`无法找到目标时会导致后续所有的派分都直接传到`ViewGroup`本身。
3. `ACTION_POINTER_DOWN`无法找到目标时视为`ACTION_DOWN`目标接收派分。

对于ACTION_UP事件：

剩余的部分，是对`ACTION_UP`类型事件进行清理：

1. `ACTION_UP`：说明这是最后一个触控点抬起，通过`resetTouchState()`完全清理派分目标和状态。
2. `ACTION_POINTER_UP`：移除触控点对应的`TouchTarget`内的`pointerIdBits`记录，当移除后`pointerIdBits = 0`（即没有其他触控点记录），则把该`TouchTarget`从`mFirstTouchTarget`中移除。



好了总结了这么久 大家可以重新再过一遍事件分发流程，希望大家有新的理解。

* 只有ACTION_DOWN或者ACTION_POINTER_DOWN会产生新的touchTarget
* 如果ACTION_DOWN没有找到touchTarget， 那么后续的MOVE或者ACTION_POINTER_DOWN都不会进行事件传递了，ViewGroup会降级成View，直接消耗后续的事件。
* 继ACTION_DOWN之后，如果ACTION_POINTER_DOWN产生了新的touchTarget与ACTION_DOWN目标不同，新的touchTarget会加到链表当中。如果跟ACTION_DOWN是同一个目标，则把事件的pointerId记录下mask里面去。

