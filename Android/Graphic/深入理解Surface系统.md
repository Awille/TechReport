# 深入理解Surface系统

## 1、概述

### 1.1、核心理解点：

* 应用程序与Surface的关系
* Surface与SurfaceFlinger的关系

### 1.2、源码依据

android10

两个Android源码地址：
**[platform_frameworks_native](https://github.com/Awille/platform_frameworks_native)**

**[platform_frameworks_base](https://github.com/aosp-mirror/platform_frameworks_base)**

大家自行下载两个库的Android源码，并切换到android10分支，跟着下面的流程一起看。

## 2、Activity的显示

### 2.1 Activity对象的创建

ActivityThread.handleResumeActivity：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
    	...
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        final Activity a = r.activity;
    	...
        if (r.window == null && !a.mFinished && willBeVisible) {
            //1、获取window对象
            r.window = r.activity.getWindow();
            //2、获取decorView
            View decor = r.window.getDecorView();
            ...
            //3、获取windowManger
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    //4、将decor加到viewManager当中
                    wm.addView(decor, l);
                } else {
                    ...
                }
            }
            ...
        } 
    }
```

以上有四步流程，在来了解四步流程以前，先看看Activity的setContentView或获取，上面四步中某些数据的来源。

```java
@UnsupportedAppUsage
private Window mWindow;

public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}

public Window getWindow() {
    return mWindow;
}
```

我们需要探究mWindow对象到底从何而来，Window的定义我们先看看官方的说明：

```
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 
 一个对于顶层窗口外观和行为策略的抽象基类。 该类的实例应作为加入到window manager中的顶层view来使用。他提供标准的ui策略，比如背景、标题栏、默认按键处理等。
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 该抽象类的唯一实现是android.view.PhoneWindow，当需要window时都使用的是PhoneWindow
 */
```

ok，其实认真读官方对于window的释义，还是挺清晰明了的，而且对于其用途也是清晰明了的。 通过这个介绍，我们知道在android中，window会承载着view，并添加到window Manger中。

那么接下来我们有两个小目标

* 了解window的实现类PhoneWindow，以及PhoneWindow的使用时机。
* 了解window manager是什么。

### 2.2、Window抽象类的唯一实现PhoneWindow

AcitivityThread.performLaunchActivity:

```java
/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    //创建activity的context
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //创建activity对象
        activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
        ...
    } catch (Exception e) {
        ...
    }

    try {
        //创建application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        if (activity != null) {
            ...
            Window window = null;
            appContext.setOuterContext(activity);
            //关键函数 绑定context token等各种信息
            activity.attach(appContext, this, getInstrumentation(), r.token,
                            r.ident, app, r.intent, r.activityInfo, title, r.parent,
                            r.embeddedID, r.lastNonConfigurationInstances, config,
                            r.referrer, r.voiceInteractor, window, r.configCallback,
                            r.assistToken);
            ...
            r.activity = activity;
        }
        r.setState(ON_CREATE);
    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
        ...
    }
    return activity;
}
```

Activity.attach:

```java
@UnsupportedAppUsage
final void attach(Context context, ActivityThread aThread,
                  Instrumentation instr, IBinder token, int ident,
                  Application application, Intent intent, ActivityInfo info,
                  CharSequence title, Activity parent, String id,
                  NonConfigurationInstances lastNonConfigurationInstances,
                  Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                  Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    attachBaseContext(context);
    //window实例化，类型为PhoneWindow
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ...
    //给window设置windowManger
    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    ...
}
```

到这里我们知道了window对象实例化的过程。 这里设置给window设置windowManger的过程也关注下：
mWindow.setWindowManager:

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                             boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}

//WindowManagerImple.createLocalWindowManager
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}

//WindowManagerImpl 构造类
private WindowManagerImpl(Context context, Window parentWindow) {
    mContext = context;
    mParentWindow = parentWindow;
}

```

这里我们可以看到mWindow中的windowManger实际为WindowManagerImpl对象。 这里需要先记一下，便于后面流程的解析。

### 2.4、HandleResumeActivity-将decorView添加到windowManger当中

从上面的解析中我们知道了Window的具体实现类，我们重回handleResumeActivity看一看：

```java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                                 String reason) {
    ...
    final Activity a = r.activity;
    if (r.window == null && !a.mFinished && willBeVisible) {
        //获取window，上面分析过，为phoneWindow对象
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        //获取viewManager，viewManger的接口在windowManger当中实现了
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        ...
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                //将decor添加到windowManager当中。
                wm.addView(decor, l);
            } else {
                ...
            }
        }
    } else if (!willBeVisible) {
        ...
    }
    ...
    Looper.myQueue().addIdleHandler(new Idler());
}

```

这里的的关键为wm.addView(decor, l); wm对象为activity当中的windowManger成员变量，该成员变量的具体类为WindowManagerImpl，这个在上一节分析过，我们继续追踪wm.addView(decor, l)，实际为WindowMangerImpl.addView

```java
//WindowMangerImpl.addView
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}

//这里的mGlobal为WindowManagerGlobal对象，我们继续看WindowManagerGlobal.addView:
public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        ...
            //1、创建ViewRootImpl对象
            root = new ViewRootImpl(view.getContext(), display);
        try {
            //2、调用setView 将view添加到ViewRootImpl当中
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            ...
        }
    }
}
```

以上过程我们主要关注两个，一个是创建ViewRootImpl， 一个是ViewRootImpl.setView， 下一节我们先分析new ViewRootImpl(view.getContext(), display)， 里面大有乾坤。



### 2.5、newViewRootImpl

构造函数

```java
@UnsupportedAppUsage
final IWindowSession mWindowSession;
final W mWindow;
public ViewRootImpl(Context context, Display display) {
    ...
    //mWindowSession创建
    mWindowSession = WindowManagerGlobal.getWindowSession();
    ...
    //mWindow对象创建
    mWindow = new W(this);
    ...
}
```

创建ViewRootImpl的构造函数设计很多成员变量的初始化，我们先只关注mWindowSession， 跟mWindow对象的创建，这两个对象创建与WMS有关。

* IWindowSession的创建

  ```java
  //WindowManagerGlobal.getWindowSession();
  @UnsupportedAppUsage
  public static IWindowSession getWindowSession() {
      synchronized (WindowManagerGlobal.class) {
          if (sWindowSession == null) {
              try {
                  ...
                  //IWindowManager为WMS的AIDL服务
                  IWindowManager windowManager = getWindowManagerService();
                  //调用WMS.opneSession获取一个IWindowSession
                  sWindowSession = windowManager.openSession(
                      new IWindowSessionCallback.Stub() {
                          @Override
                          public void onAnimatorScaleChanged(float scale) {
                              ValueAnimator.setDurationScale(scale);
                          }
                      });
              } catch (RemoteException e) {
                  throw e.rethrowFromSystemServer();
              }
          }
          return sWindowSession;
      }
  }
  
  //getWindowManagerService实现
  @UnsupportedAppUsage
  public static IWindowManager getWindowManagerService() {
      synchronized (WindowManagerGlobal.class) {
          if (sWindowManagerService == null) {
              sWindowManagerService = IWindowManager.Stub.asInterface(
                  ServiceManager.getService("window"));
              ...
          }
          return sWindowManagerService;
      }
  }
  ```
  我们看看WMS提供的openSession方法	
  ```java
  @Override
  public IWindowSession openSession(IWindowSessionCallback callback) {
    return new Session(this, callback);
  }
  
  //那么Session到底是什么，我们看看官方释义：
  /**
   * This class represents an active client session.  There is generally one
   * Session object per process that is interacting with the window manager.
   
  该类代表了一个活跃的客户端会话，通常每个应用继承会持有一个Session来用于与WMS交互。
   */
  class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
      
  }
  ```

  ok，通过上面的官方注释，我们已经很清晰知道Session的作用了，可以简单理解为应用程序进   程与WMS交互的工具。

  所以在创建ViewRootImpl时，会在WMS中申请一个Session，该Session可被             ViewRootImpl用于与WMS进行交互。
  

* mWindow对象的创建

  mWindow对象实际为W类型：

  W为ViewRootImpl的静态内部类：

  ```java
  static class W extends IWindow.Stub
  ```

  这里我先给到结论，W是ViewRootImpl创建时传递给WMS的AIDL服务，WMS持有该对象对Window的行为进行控制。具体什么时候传递给WMS我们后续再ViewRootImpl.setView中分析

### 2.6、ViewRootImpl.setView

```java
/**
     * We have one child
     */
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            ...
            //requestLayout
            requestLayout();
            ...
            try {
                ...
                //mWindowSession.addToDisplay
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                                                  getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                                                  mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                                                  mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                                                  mTempInsets);
                setFrame(mTmpFrame);
            } catch (RemoteException e) {
                ...
            } finally {
                ...
            }
    }
}

```

我们先关注mWindowSession.addToDisplay， 这里mWindowSession是ViewRootImpl在WMS中申请的Session, 可以调用WMS的服务，我们看看Session中addToDisplay的实现：

```java
//Session.addToDisplay
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
                        int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
                        Rect outStableInsets, Rect outOutsets,
                        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
                        InsetsState outInsetsState) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                              outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel,
                              outInsetsState);
}
```

//mService为WindowManagerService, WindowMangerService.addWindow里面的方法太繁琐了，我们就不看了，但可以关注到在addToDisplay时ViewRootImpl的mWindow对象，W类型，被传入到WMS当中，后续WMS可以通过该AIDL服务区操作客户端的window。

再看requstLayout，requestLayout会吧performTraversal任务发送给Choreographer，并请求在Vsync信号，在接收到Vsync信号时，performTraversal会被执行(这里面的过程这里就不分析了，外面很多文章有对应分析)。  我们关注下performTraversal里面的关键函数：

```java
@UnsupportedAppUsage
public final Surface mSurface = new Surface();
private final SurfaceControl mSurfaceControl = new SurfaceControl();

private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;
	...
    //这里有一个知识点，后续面试可以追踪下，八股之一
    // Execute enqueued actions on every traversal in case a detached view enqueued an action
    getRunQueue().executeActions(mAttachInfo.mHandler);
    if (mFirst || windowShouldResize || insetsChanged ||
        viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        ...
        try {
            ...
            //relayoutWindow 关键函数
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
            ...
        } catch (RemoteException e) {
        }
        //ThreadedRenderer构建
        final ThreadedRenderer threadedRenderer = mAttachInfo.mThreadedRenderer;
        ...
    }
    ...
    mIsInTraversal = false;
}
```

performTraversals中有绘制三联操作，老生常谈了，这里我们只关注与WMS的有关的逻辑，

relayoutWindow：

```java
    private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
        ...
        //注意这里传入了mSurfaceControl对象，mSurfaceControl是初始化时创建的
        int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
                insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
                mTmpFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
                mPendingMergedConfiguration, mSurfaceControl, mTempInsets);
        if (mSurfaceControl.isValid()) {
            //复制Surface
            mSurface.copyFrom(mSurfaceControl);
        } else {
            destroySurface();
        }
        ...
        return relayoutResult;
    }
```

这里调用了mWindowSession.relayout

```java
    @Override
    public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewFlags, int flags, long frameNumber,
            Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets,
            Rect outStableInsets, Rect outsets, Rect outBackdropFrame,
            DisplayCutout.ParcelableWrapper cutout, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState) {
        if (false) Slog.d(TAG_WM, ">>>>>> ENTERED relayout from "
                + Binder.getCallingPid());
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, mRelayoutTag);
        //传入outSurfaceControl对象
        int res = mService.relayoutWindow(this, window, seq, attrs,
                requestedWidth, requestedHeight, viewFlags, flags, frameNumber,
                outFrame, outOverscanInsets, outContentInsets, outVisibleInsets,
                outStableInsets, outsets, outBackdropFrame, cutout,
                mergedConfiguration, outSurfaceControl, outInsetsState);
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        if (false) Slog.d(TAG_WM, "<<<<<< EXITING relayout to "
                + Binder.getCallingPid());
        return res;
    }
```

好了，前面铺垫这么久，终于到了与Surface相关的逻辑了，我们可以看到mSurface与mSurfaceControl都是在创建ViewRootImpl时创建的对象，但是最终传给了WMS, 后续使用WMS返回的mSurfaceControl复制到当前的Surface当中。所以实际上看来，window创建的Surface真正的操作是在WMS当中。



### 2.7、阶段总结

经历了上面一长串逻辑，我们先理清一下我们得到的结论。

* 在ActivityThread.handleResumeActivity当中，会将decorView加入到windowManager当中
* 该windowManager的实现为WindowMangerImpl，WindoMangerImpl.addView由WindowMangerGlobal代理实现
* WindowManagerGlobal.addView方法中会创建ViewRootImpl
* ViewRootImpl创建时会新建surface, 与surfaceControl，但实质为空实现。 并且会通过AIDL调用WMS请求一个Session, 持有该Session可以调用WMS的一些服务。
* ViewRootImpl.setView方法中会调用addToDisplay，将自身暴露给WMS的AIDL服务(W类型)传递给WMS。
* ViewRootImpl.setView方法中会调用requestLayout，最终执行的performTravelsal当中会调用mSession.relayoutWindow, 调用时会传入surfaceControl，最终服务结束时，将返回的surfaceControl复制到Surface当中。



接下来，我们就针对Surface怎么创建的主题进行进一步的分析。

## 3、Surface与SurfaceControl的创建。



