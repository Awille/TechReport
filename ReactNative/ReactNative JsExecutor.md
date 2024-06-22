# ReactNative JsExecutor

## 背景

ReactNative C++ 调用JS的原理解析 



## 源码链路

ReactAndroid/src/main/jni/react/jni/CatalystInstanceImpl.cpp

```c++
void CatalystInstanceImpl::jniCallJSFunction(
    std::string module,
    std::string method,
    NativeArray *arguments) {
  // We want to share the C++ code, and on iOS, modules pass module/method
  // names as strings all the way through to JS, and there's no way to do
  // string -> id mapping on the objc side.  So on Android, we convert the
  // number to a string, here which gets passed as-is to JS.  There, they they
  // used as ids if isFinite(), which handles this case, and looked up as
  // strings otherwise.  Eventually, we'll probably want to modify the stack
  // from the JS proxy through here to use strings, too.
  instance_->callJSFunction(
      std::move(module), std::move(method), arguments->consume());
}
```

mModule = "RCTEventEmitter"

mMethod = “receiveTouches”

argument为writableArray: 
["topTouchStart",[{"identifier":0,"timestamp":513046596,"target":93,"locationY":10.285714149475098,"locationX":210.6666717529297,"pageY":287.23809814453125,"pageX":239.23809814453125}],[0]]



其中instance_d对象定义在头文件当中

```c++
std::shared_ptr<Instance> instance_;
```

ReactCommon/cxxreact/Instance.cpp

```c++
void Instance::callJSFunction(
    std::string &&module,
    std::string &&method,
    folly::dynamic &&params) {
  callback_->incrementPendingJSCalls();
  nativeToJsBridge_->callFunction(
      std::move(module), std::move(method), std::move(params));
}

std::shared_ptr<NativeToJsBridge> nativeToJsBridge_;
```

ReactCommon/cxxreact/NativeToJsBridge.cpp

```c++
void NativeToJsBridge::callFunction(
    std::string &&module,
    std::string &&method,
    folly::dynamic &&arguments) {
  int systraceCookie = -1;
  runOnExecutorQueue([this,
                      module = std::move(module),
                      method = std::move(method),
                      arguments = std::move(arguments),
                      systraceCookie](JSExecutor *executor) {
    executor->callFunction(module, method, arguments);
  });
}

void NativeToJsBridge::runOnExecutorQueue(
    std::function<void(JSExecutor *)> task) {
  if (*m_destroyed) {
    return;
  }

  std::shared_ptr<bool> isDestroyed = m_destroyed;
  m_executorMessageQueueThread->runOnQueue(
      [this, isDestroyed, task = std::move(task)] {
        if (*isDestroyed) {
          return;
        }
        task(m_executor.get());
      });
}
```

最终runOnExecutorQueue函数实现中，通过m_executor.get()回调了一个JSExecutor给调用方，调用方为callFunction，最终将m_executor.get()执行callFunction(module, method, arguments)操作

看看m_executor怎么来的

依旧在NativeToJsBridge.cpp当中，查看构造函数

```c++
NativeToJsBridge::NativeToJsBridge(
    JSExecutorFactory *jsExecutorFactory,
    std::shared_ptr<ModuleRegistry> registry,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<InstanceCallback> callback)
    : m_destroyed(std::make_shared<bool>(false)),
      m_delegate(std::make_shared<JsToNativeBridge>(registry, callback)),
      m_executor(jsExecutorFactory->createJSExecutor(m_delegate, jsQueue)),
      m_executorMessageQueueThread(std::move(jsQueue)),
      m_inspectable(m_executor->isInspectable()) {}
```

m_executor为jsExecutorFactory->createJSExecutor(m_delegate, jsQueue)生成， jsExecutorFactory为构造函数传入， NativeToJsBridge对象由Instance.cpp 进行构建

ReactCommon/cxxreact/Instance.cpp

```c++
void Instance::initializeBridge(
    std::unique_ptr<InstanceCallback> callback,
    std::shared_ptr<JSExecutorFactory> jsef,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<ModuleRegistry> moduleRegistry) {
  callback_ = std::move(callback);
  moduleRegistry_ = std::move(moduleRegistry);
  jsQueue->runOnQueueSync([this, &jsef, jsQueue]() mutable {
    nativeToJsBridge_ = std::make_shared<NativeToJsBridge>(
        jsef.get(), moduleRegistry_, jsQueue, callback_);

    nativeToJsBridge_->initializeRuntime();

    /**
     * After NativeToJsBridge is created, the jsi::Runtime should exist.
     * Also, the JS message queue thread exists. So, it's safe to
     * schedule all queued up js Calls.
     */
    jsCallInvoker_->setNativeToJsBridgeAndFlushCalls(nativeToJsBridge_);

    std::lock_guard<std::mutex> lock(m_syncMutex);
    m_syncReady = true;
    m_syncCV.notify_all();
  });

  CHECK(nativeToJsBridge_);
}
```

其中Instance.cpp由CatalystInstanceImpl.cpp来完成

ReactAndroid/src/main/jni/react/jni/CatalystInstanceImpl.cpp

```c++
void CatalystInstanceImpl::initializeBridge(
    jni::alias_ref<ReactCallback::javaobject> callback,
    // This executor is actually a factory holder.
    JavaScriptExecutorHolder *jseh,
    jni::alias_ref<JavaMessageQueueThread::javaobject> jsQueue,
    jni::alias_ref<JavaMessageQueueThread::javaobject> nativeModulesQueue,
    jni::alias_ref<jni::JCollection<JavaModuleWrapper::javaobject>::javaobject>
        javaModules,
    jni::alias_ref<jni::JCollection<ModuleHolder::javaobject>::javaobject>
        cxxModules) {
    //Js消息队列
  moduleMessageQueue_ =
      std::make_shared<JMessageQueueThread>(nativeModulesQueue);
    //NativeModule注册
  moduleRegistry_ = std::make_shared<ModuleRegistry>(buildNativeModuleList(
      std::weak_ptr<Instance>(instance_),
      javaModules,
      cxxModules,
      moduleMessageQueue_));
    //Instance对象实例化创建
  instance_->initializeBridge(
      std::make_unique<JInstanceCallback>(callback, moduleMessageQueue_),
      jseh->getExecutorFactory(),
      std::make_unique<JMessageQueueThread>(jsQueue),
      moduleRegistry_);
}

void CatalystInstanceImpl::registerNatives() {
  registerHybrid({
      //忽略无关代码，申明了 jni接口，由java进行调用
      makeNativeMethod(
          "initializeBridge", CatalystInstanceImpl::initializeBridge),
  });

  JNativeRunnable::registerNatives();
}
```

ReactAndroid/src/main/java/com/facebook/react/bridge/CatalystInstanceImpl.java

```java
  private CatalystInstanceImpl(
      final ReactQueueConfigurationSpec reactQueueConfigurationSpec,
      final JavaScriptExecutor jsExecutor,
      final NativeModuleRegistry nativeModuleRegistry,
      final JSBundleLoader jsBundleLoader,
      NativeModuleCallExceptionHandler nativeModuleCallExceptionHandler) {
    FLog.d(ReactConstants.TAG, "Initializing React Xplat Bridge.");
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "createCatalystInstanceImpl");

    mHybridData = initHybrid();

    mReactQueueConfiguration =
        ReactQueueConfigurationImpl.create(
            reactQueueConfigurationSpec, new NativeExceptionHandler());
    mBridgeIdleListeners = new CopyOnWriteArrayList<>();
    mNativeModuleRegistry = nativeModuleRegistry;
    mJSModuleRegistry = new JavaScriptModuleRegistry();
    mJSBundleLoader = jsBundleLoader;
    mNativeModuleCallExceptionHandler = nativeModuleCallExceptionHandler;
    mNativeModulesQueueThread = mReactQueueConfiguration.getNativeModulesQueueThread();
    mTraceListener = new JSProfilerTraceListener(this);
    initializeBridge(
        new BridgeCallback(this),
        jsExecutor,
        mReactQueueConfiguration.getJSQueueThread(),
        mNativeModulesQueueThread,
        mNativeModuleRegistry.getJavaModules(this),
        mNativeModuleRegistry.getCxxModules());
    mJavaScriptContextHolder = new JavaScriptContextHolder(getJavaScriptContext());
  }
```

com.facebook.react.bridge.CatalystInstanceImpl.Builder

```java
  public static class Builder {

    private @Nullable ReactQueueConfigurationSpec mReactQueueConfigurationSpec;
    private @Nullable JSBundleLoader mJSBundleLoader;
    private @Nullable NativeModuleRegistry mRegistry;
    private @Nullable JavaScriptExecutor mJSExecutor;
    private @Nullable NativeModuleCallExceptionHandler mNativeModuleCallExceptionHandler;

    public Builder setReactQueueConfigurationSpec(
        ReactQueueConfigurationSpec ReactQueueConfigurationSpec) {
      mReactQueueConfigurationSpec = ReactQueueConfigurationSpec;
      return this;
    }

    public Builder setRegistry(NativeModuleRegistry registry) {
      mRegistry = registry;
      return this;
    }

    public Builder setJSBundleLoader(JSBundleLoader jsBundleLoader) {
      mJSBundleLoader = jsBundleLoader;
      return this;
    }

    public Builder setJSExecutor(JavaScriptExecutor jsExecutor) {
      mJSExecutor = jsExecutor;
      return this;
    }

    public Builder setNativeModuleCallExceptionHandler(NativeModuleCallExceptionHandler handler) {
      mNativeModuleCallExceptionHandler = handler;
      return this;
    }

    public CatalystInstanceImpl build() {
      return new CatalystInstanceImpl(
          Assertions.assertNotNull(mReactQueueConfigurationSpec),
          //构造者模式构建
          Assertions.assertNotNull(mJSExecutor),
          Assertions.assertNotNull(mRegistry),
          Assertions.assertNotNull(mJSBundleLoader),
          Assertions.assertNotNull(mNativeModuleCallExceptionHandler));
    }
  }
```

ReactAndroid/src/main/java/com/facebook/react/bridge/CatalystInstanceImpl.java

```java
  private ReactApplicationContext createReactContext(
      JavaScriptExecutor jsExecutor, JSBundleLoader jsBundleLoader) {

    final ReactApplicationContext reactContext = new ReactApplicationContext(mApplicationContext);
    NativeModuleCallExceptionHandler exceptionHandler =
        mNativeModuleCallExceptionHandler != null
            ? mNativeModuleCallExceptionHandler
            : mDevSupportManager;
    reactContext.setNativeModuleCallExceptionHandler(exceptionHandler);

    NativeModuleRegistry nativeModuleRegistry = processPackages(reactContext, mPackages, false);

    CatalystInstanceImpl.Builder catalystInstanceBuilder =
        new CatalystInstanceImpl.Builder()
            .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
             //核心
            .setJSExecutor(jsExecutor)
            .setRegistry(nativeModuleRegistry)
            .setJSBundleLoader(jsBundleLoader)
            .setNativeModuleCallExceptionHandler(exceptionHandler);
    final CatalystInstance catalystInstance;
    //实际创建  
    catalystInstance = catalystInstanceBuilder.build();

    reactContext.initializeWithInstance(catalystInstance);

    //忽略无关代码

    return reactContext;
  }
```

ReactAndroid/src/main/java/com/facebook/react/bridge/CatalystInstanceImpl.java



com.facebook.react.ReactInstanceManager#runCreateReactContextOnNewThread该函数中

```java
final ReactApplicationContext reactApplicationContext =
  createReactContext(
      initParams.getJsExecutorFactory().create(),
      initParams.getJsBundleLoader());
```

ReactAndroid/src/main/java/com/facebook/react/ReactNativeHost.java

最终从这里传入

```java
  protected ReactInstanceManager createReactInstanceManager() {
    ReactMarker.logMarker(ReactMarkerConstants.BUILD_REACT_INSTANCE_MANAGER_START);
    ReactInstanceManagerBuilder builder =
        ReactInstanceManager.builder()
            .setApplication(mApplication)
            .setJSMainModulePath(getJSMainModuleName())
            .setUseDeveloperSupport(getUseDeveloperSupport())
            .setRedBoxHandler(getRedBoxHandler())
            //核心
            .setJavaScriptExecutorFactory(getJavaScriptExecutorFactory())
            .setUIImplementationProvider(getUIImplementationProvider())
            .setJSIModulesPackage(getJSIModulePackage())
            .setInitialLifecycleState(LifecycleState.BEFORE_CREATE);

    for (ReactPackage reactPackage : getPackages()) {
      builder.addPackage(reactPackage);
    }

    String jsBundleFile = getJSBundleFile();
    if (jsBundleFile != null) {
      builder.setJSBundleFile(jsBundleFile);
    } else {
      builder.setBundleAssetName(Assertions.assertNotNull(getBundleAssetName()));
    }
    ReactInstanceManager reactInstanceManager = builder.build();
    ReactMarker.logMarker(ReactMarkerConstants.BUILD_REACT_INSTANCE_MANAGER_END);
    return reactInstanceManager;
  }
```

getJavaScriptExecutorFactory一般为null

所以最终由这里创建

ReactAndroid/src/main/java/com/facebook/react/ReactInstanceManagerBuilder.java

```java
  public ReactInstanceManager build() {
    String appName = mApplication.getPackageName();
    String deviceName = getFriendlyDeviceName();

    return new ReactInstanceManager(
        mApplication,
        mCurrentActivity,
        mDefaultHardwareBackBtnHandler,
        //核心，一般为null
        mJavaScriptExecutorFactory == null
            ? getDefaultJSExecutorFactory(appName, deviceName, mApplication.getApplicationContext())
            : mJavaScriptExecutorFactory,
        (mJSBundleLoader == null && mJSBundleAssetUrl != null)
            ? JSBundleLoader.createAssetLoader(
                mApplication, mJSBundleAssetUrl, false /*Asynchronous*/)
            : mJSBundleLoader,
        mJSMainModulePath,
        mPackages,
        mUseDeveloperSupport,
        mBridgeIdleDebugListener,
        Assertions.assertNotNull(mInitialLifecycleState, "Initial lifecycle state was not set"),
        mUIImplementationProvider,
        mNativeModuleCallExceptionHandler,
        mRedBoxHandler,
        mLazyViewManagersEnabled,
        mDevBundleDownloadListener,
        mMinNumShakes,
        mMinTimeLeftInFrameForNonBatchedOperationMs,
        mJSIModulesPackage,
        mCustomPackagerCommandHandlers);
  }

  private JavaScriptExecutorFactory getDefaultJSExecutorFactory(
      String appName, String deviceName, Context applicationContext) {
    try {
      // If JSC is included, use it as normal
      initializeSoLoaderIfNecessary(applicationContext);
      SoLoader.loadLibrary("jscexecutor");
      return new JSCExecutorFactory(appName, deviceName);
    } catch (UnsatisfiedLinkError jscE) {
      
    }
  }
```

最终调用了一个jscexecutor的so

这个so是hermes来完成的

ReactAndroid/src/main/java/com/facebook/react/jscexecutor/JSCExecutorFactory.java

```java
public class JSCExecutorFactory implements JavaScriptExecutorFactory {
  private final String mAppName;
  private final String mDeviceName;

  public JSCExecutorFactory(String appName, String deviceName) {
    this.mAppName = appName;
    this.mDeviceName = deviceName;
  }

  @Override
  public JavaScriptExecutor create() throws Exception {
    WritableNativeMap jscConfig = new WritableNativeMap();
    jscConfig.putString("OwnerIdentity", "ReactNative");
    jscConfig.putString("AppIdentity", mAppName);
    jscConfig.putString("DeviceIdentity", mDeviceName);
    return new JSCExecutor(jscConfig);
  }
}
```

ReactAndroid/src/main/java/com/facebook/react/jscexecutor/JSCExecutor.java

```java
@DoNotStrip
/* package */ class JSCExecutor extends JavaScriptExecutor {
  static {
    SoLoader.loadLibrary("jscexecutor");
  }

  /* package */ JSCExecutor(ReadableNativeMap jscConfig) {
    super(initHybrid(jscConfig));
  }

  @Override
  public String getName() {
    return "JSCExecutor";
  }

  private static native HybridData initHybrid(ReadableNativeMap jscConfig);
}
```



另一个为Hermes版本的实现

ReactAndroid/src/main/java/com/facebook/hermes/reactexecutor/HermesExecutorFactory.java

```java
public class HermesExecutorFactory implements JavaScriptExecutorFactory {
  private static final String TAG = "Hermes";

  private final RuntimeConfig mConfig;

  public HermesExecutorFactory() {
    this(null);
  }

  public HermesExecutorFactory(RuntimeConfig config) {
    mConfig = config;
  }

  @Override
  public JavaScriptExecutor create() {
    return new HermesExecutor(mConfig);
  }

  @Override
  public void startSamplingProfiler() {
    HermesSamplingProfiler.enable();
  }

  @Override
  public void stopSamplingProfiler(String filename) {
    HermesSamplingProfiler.dumpSampledTraceToFile(filename);
    HermesSamplingProfiler.disable();
  }

  @Override
  public String toString() {
    return "JSIExecutor+HermesRuntime";
  }
}

```

ReactAndroid/src/main/java/com/facebook/hermes/reactexecutor/HermesExecutor.java

```java
public class HermesExecutor extends JavaScriptExecutor {
  private static String mode_;

  static {
    // libhermes must be loaded explicitly to invoke its JNI_OnLoad.
    SoLoader.loadLibrary("hermes");
    try {
      SoLoader.loadLibrary("hermes-executor-debug");
      mode_ = "Debug";
    } catch (UnsatisfiedLinkError e) {
      SoLoader.loadLibrary("hermes-executor-release");
      mode_ = "Release";
    }
  }

  HermesExecutor(@Nullable RuntimeConfig config) {
    super(
        config == null
            ? initHybridDefaultConfig()
            : initHybrid(config.heapSizeMB, config.es6Proxy));
  }

  @Override
  public String getName() {
    return "HermesExecutor" + mode_;
  }

  /**
   * Return whether this class can load a file at the given path, based on a binary compatibility
   * check between the contents of the file and the Hermes VM.
   *
   * @param path the path containing the file to inspect.
   * @return whether the given file is compatible with the Hermes VM.
   */
  public static native boolean canLoadFile(String path);

  private static native HybridData initHybridDefaultConfig();

  private static native HybridData initHybrid(long heapSizeMB, boolean es6Proxy);
}
```

一般实现为Herems





所以JsExcutor的实现在Hermes当中

hermes\android\intltest\java\com\facebook\hermes\test\JSRuntime.cpp

```c++
void JSRuntime::callFunction(std::string functionName) {
  jsi::Function function =
      runtime_->global().getPropertyAsFunction(*runtime_, functionName.c_str());

  function.call(*runtime_);
}
```

hermes\API\jsi\jsi\jsi.cpp

```c++
Function Object::getPropertyAsFunction(Runtime& runtime, const char* name)
    const {
  Object obj = getPropertyAsObject(runtime, name);
  return std::move(obj).getFunction(runtime);
}

Object Object::getPropertyAsObject(Runtime& runtime, const char* name) const {
  Value v = getProperty(runtime, name);

  return v.getObject(runtime);
}
```



最终是通过JSI来实现的

https://blog.csdn.net/m0_49508485/article/details/122622251