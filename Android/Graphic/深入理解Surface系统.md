# 深入理解Surface系统

## 1、概述

## 1.1、核心理解点：

* 应用程序与Surface的关系
* Surface与SurfaceFlinger的关系

## 1.2、源码依据

android10

## 2、Activity的显示

## 2.1 Activity对象的创建

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

到这里我们知道了window对象实例化的过程，下面看看windowManger

### 2.3、WindowManager

