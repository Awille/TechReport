# MediaCodec缓冲区管理

## 1、背景

最近在线上出现了一个问题，在小米10手机上，往codec里面送了6个packer的数据后，无法获取更多的输入缓冲区，无法再输入更多的数据，codec输出缓冲区也一直没有解码后的数据，如此过程往复循环，最终导致首帧超时，用户手机看到的视频是黑屏的状态。 对于出现问题的视频，我们在正常播放时，发现至少要送入9个packet的数据才出首帧。

目前这个问题只在mi10上出现，我们费了很大的功夫在demo上复现该问题，发现一个规律是当手机内存剩余量较少的时候，复现概率很高。   我们有几个怀疑点：

* 系统对所有codec实例的输入缓冲区大小有个总体的管控，有个内存最大限制，导致了后面创建的codec实例无法获取到更多的缓冲区内存。
* 系统codec实例的缓冲区数量与手机整体的内存情况有关，在内存比较紧张时容易出现该问题。

针对以上猜想，我们写了demo去验证，但都没办法很明确的确认就是哪个原因引起的，基于此，想对codec缓冲区的管理做一些大致的了解。 虽然各大手机厂商都对系统有一些魔改，但是还是尽量了解下底层的机制，尝试一些对应的解决方案。



以下源码分析基于android-10.0.0分支，基于AOSP的两个代码库

* framework/base：base代码主要包括一些java层代码定义以及JNI方法的声明。
* framework/av： MediaCodec native代码所在的仓库。



## 2、ALooper机制

由于codec内存时使用ALooper消息系统运行，这里先做一些必要的了解，该ALooper与我们日常使用的Looper很像，可以带着Looper的思想来一起看。

### 2.1、ALooper的简单使用

```c++
//以下为伪代码
mHandler = new AHandler{
    void onMessageReceived(const sp<AMessage> &msg) {
        switch (msg->what()) {
            case 1: {
                break;
            }
            case 2: {
                break;
            }
            default: {
                break;
            }
        }
    }
};
mLooper = new ALooper;
mLooper->setName("MediaCodec_looper");
mLooper->start(
            false,      // runOnCallingThread
            true,       // canCallJava
            ANDROID_PRIORITY_VIDEO);
mLooper->registerHandler(mHandler);

//开始发送消息
sp<AMessage> msg = new AMessage(kWhatDequeueInputBuffer, this);
msg->postAndAwaitResponse(response);
```



### 2.2、ALooper的启动

ALooper.cpp

```c++
//这里有个gLooperRoster实例，可以先记一下，后续要用到。
ALooperRoster gLooperRoster;

ALooper::ALooper()
    : mRunningLocally(false) {
    // clean up stale AHandlers. Doing it here instead of in the destructor avoids
    // the side effect of objects being deleted from the unregister function recursively.
    gLooperRoster.unregisterStaleHandlers();
}

//Looper的start方法调用
status_t ALooper::start(
        bool runOnCallingThread, bool canCallJava, int32_t priority) {
    //looper是否要运行在当前调用start方法的线程
    if (runOnCallingThread) {
        {
            //加锁 规避多线程问题
            Mutex::Autolock autoLock(mLock);
            //一个looper只允许运行在一个线程里面
            if (mThread != NULL || mRunningLocally) {
                return INVALID_OPERATION;
            }

            mRunningLocally = true;
        }
        //开始消息循环
        do {
        } while (loop());

        return OK;
    }

    Mutex::Autolock autoLock(mLock);

    //一个looper只允许运行在一个线程里面
    if (mThread != NULL || mRunningLocally) {
        return INVALID_OPERATION;
    }
    //自己开启一个looper线程
    mThread = new LooperThread(this, canCallJava);
    //开始运行线程
    status_t err = mThread->run(
            mName.empty() ? "ALooper" : mName.c_str(), priority);
    if (err != OK) {
        mThread.clear();
    }

    return err;
}
```

LooperThread，是Alooper内存类

```c++
struct ALooper::LooperThread : public Thread {
    LooperThread(ALooper *looper, bool canCallJava)
        : Thread(canCallJava),
          mLooper(looper),
          mThreadId(NULL) {
    }

    //线程启动时会被调用
    virtual status_t readyToRun() {
        mThreadId = androidGetThreadId();

        return Thread::readyToRun();
    }
    
    //线程运行时会调用
    virtual bool threadLoop() {
        return mLooper->loop();
    }

    bool isCurrentThread() const {
        return mThreadId == androidGetThreadId();
    }

protected:
    virtual ~LooperThread() {}

private:
    ALooper *mLooper;
    android_thread_id_t mThreadId;

    DISALLOW_EVIL_CONSTRUCTORS(LooperThread);
};
```

这里可以看到LooperThread重写了readyToRun与threadLoop， 我们之前在java层使用线程可能不太熟悉，可以先暂时不管，总之记住线程在运行时readyToRun与threadLoop会被顺序调用。





ok通过以上的逻辑可以发现，ALooper::start不论是在当前的线程中开启消息循环，还是新开启一个线程开启消息循环，都设计到最核心的调用mLooper->loop()。



### 2.3、ALooper的消息循环与数据分发

Alooper.cpp中的 loop方法

```c++
//成员变量 condition，负责线程的阻塞与唤醒
Condition mQueueChangedCondition;
//消息队列
List<Event> mEventQueue;
bool ALooper::loop() {
    Event event;

    {
        Mutex::Autolock autoLock(mLock);
        //状态检查
        if (mThread == NULL && !mRunningLocally) {
            return false;
        }
        if (mEventQueue.empty()) {
            //如果消息队列空的，调用wait 线程进入阻塞状态。
            mQueueChangedCondition.wait(mLock);
            return true;
        }
        //获取消息队列收个消息的消费时间
        int64_t whenUs = (*mEventQueue.begin()).mWhenUs;
        //获取当前的时间
        int64_t nowUs = GetNowUs();

        if (whenUs > nowUs) {
            int64_t delayUs = whenUs - nowUs;
            if (delayUs > INT64_MAX / 1000) {
                delayUs = INT64_MAX / 1000;
            }
            //如果当前的消费队列第一个消息的时间戳在当前时间后面，
            //算出要等待的时间，然后要线程阻塞相同的时间
            mQueueChangedCondition.waitRelative(mLock, delayUs * 1000ll);

            return true;
        }

        event = *mEventQueue.begin();
        //取出消息队列首个消息，然后从队列里面移除。
        mEventQueue.erase(mEventQueue.begin());
    }
    //开始分发消息
    event.mMessage->deliver();
    return true;
}
```



上面loop方法，如果消息队列有消息，最终会调用event.mMessage->deliver();开始消息分发。

看看Event的数据结构

```c++
struct Event {
    int64_t mWhenUs;
    sp<AMessage> mMessage;
};
```

这个数据结构很简单，event.mMessage的对象为AMessage的类型，所以我们最终调用了AMessage.deliver()消息进行分发。

AMessage.cpp

```c++
void AMessage::deliver() {
    //首先获取到具体的handler
    sp<AHandler> handler = mHandler.promote();
    if (handler == NULL) {
        ALOGW("failed to deliver message as target handler %d is gone.", mTarget);
        return;
    }
    //调用handler.deliverMessage开始具体的消息分发。 
    handler->deliverMessage(this);
}
```

这里先看看AHandler.deliverMessage方法

```c++
void AHandler::deliverMessage(const sp<AMessage> &msg) {
    //onMessageReceived这个方法就是我们创建AHandler对象时重写的方法。
    onMessageReceived(msg);
    mMessageCounter++;

    if (mVerboseStats) {
        uint32_t what = msg->what();
        ssize_t idx = mMessages.indexOfKey(what);
        if (idx < 0) {
            mMessages.add(what, 1);
        } else {
            mMessages.editValueAt(idx)++;
        }
    }
}

```



从以上过程中，消息最后怎么分发 到我们创建的Ahandler对象的onMessageReceived的 已经理清楚了。



### 2.4、消息系统中的AHandler注册与使用时的消息发送

还剩下一个点，就是AMessage::deliver中sp<AHandler> handler = mHandler.promote(); 这行代码，mHandler类型就是AHandler，该对象时怎么赋值到AMessage的。回到之前demo调用的mLooper->registerHandler(mHandler)方法

ALooper->registerHandler

```c++
ALooper::handler_id ALooper::registerHandler(const sp<AHandler> &handler) {
    return gLooperRoster.registerHandler(this, handler);
}

//其中gLooperRoster为ALooperRoster对象
ALooper::handler_id ALooperRoster::registerHandler(
        const sp<ALooper> &looper, const sp<AHandler> &handler) {
    Mutex::Autolock autoLock(mLock);

    //状态检查，一个handler只能被注册一次
    if (handler->id() != 0) {
        CHECK(!"A handler must only be registered once.");
        return INVALID_OPERATION;
    }

    //创建一个HandlerInfo对象
    HandlerInfo info;
    info.mLooper = looper;
    info.mHandler = handler;
    ALooper::handler_id handlerID = mNextHandlerID++;
    
    //关注下mHandlers的对象类型为：
    //KeyedVector<ALooper::handler_id, HandlerInfo> mHandlers;
    mHandlers.add(handlerID, info);

    handler->setID(handlerID, looper);
    return handlerID;
}
```

ALooper->registerHandler内部在ALooperRoster的KeyedVector<ALooper::handler_id, HandlerInfo> mHandlers成员变量中把Ahandler存了起来。



再看我们使用时发送消息的逻辑：

sp<AMessage> msg = new AMessage(kWhatDequeueInputBuffer, this);

AMessage::postAndAwaitResponse

```c++
AMessage::AMessage(uint32_t what, const sp<const AHandler> &handler)
    : mWhat(what),
      mNumItems(0) {
    //创建消息时，直接把handler 对象传进来，这样每个AMessage都会对应一个AMessage对象。
    setTarget(handler);
}

void AMessage::setTarget(const sp<const AHandler> &handler) {
    if (handler == NULL) {
        mTarget = 0;
        mHandler.clear();
        mLooper.clear();
    } else {
        mTarget = handler->id();
        mHandler = handler->getHandler();
        //记录对应的Looper
        mLooper = handler->getLooper();
    }
}
    

//发送消息的代码
status_t AMessage::postAndAwaitResponse(sp<AMessage> *response) {
    sp<ALooper> looper = mLooper.promote();
    if (looper == NULL) {
        ALOGW("failed to post message as target looper for handler %d is gone.", mTarget);
        return -ENOENT;
    }

    sp<AReplyToken> token = looper->createReplyToken();
    if (token == NULL) {
        ALOGE("failed to create reply token");
        return -ENOMEM;
    }
    setObject("replyID", token);
    //插入到Looper的消息队列中，等待对应的线程循环消息消费消息队列里面的消息。
    looper->post(this, 0 /* delayUs */);
    return looper->awaitResponse(token, response);
}
```



### 2.5、ALooper消息循环小结

通过以上消息，我们把ALooper的消息消费线程的运作方式与 使用过程中 消息怎么插入到looper线程消息队列中的消息理解清楚了，这些与我们应用层使用Looper消息循环原理是一样的，只是另一套实现而已。



## 3、MediaCodec

MediaCodec里面的代码逻辑很多，我没办法从初始化讲起，先直接获取我们最关心的逻辑

我们使用codec的一般范式为：

```java
- createByCodeName/createEncoderByType/createDecoderByType: （静态工厂构造MediaCodec对象）--Uninitialized状态
- configure：（配置） -- configure状态
- start        （启动）--进入Running状态
- while(1) {
    try{
       - dequeueInputBuffer    （从编解码器获取输入缓冲区buffer）
       - queueInputBuffer      （buffer被生成方client填满之后提交给编解码器）
       - dequeueOutputBuffer   （从编解码器获取输出缓冲区buffer）
       - releaseOutputBuffer   （消费方client消费之后释放给编解器）
    } catch(Error e){
       - error                   （出现异常 进入error状态）
    }

}
- stop                          （编解码完成后，释放codec）
- release
```



### 3.1、MediaCodec输入缓冲区是如何获取的

我们上层获取输入缓冲区时，调用的是

status_t dequeueInputBuffer(size_t *index, int64_t timeoutUs = 0ll);

其中index是我们期望的返回值，timeoutUs 为超时时间

```c++
status_t MediaCodec::dequeueInputBuffer(size_t *index, int64_t timeoutUs) {
    sp<AMessage> msg = new AMessage(kWhatDequeueInputBuffer, this);
    msg->setInt64("timeoutUs", timeoutUs);

    sp<AMessage> response;
    status_t err;
    if ((err = PostAndAwaitResponse(msg, &response)) != OK) {
        return err;
    }

    CHECK(response->findSize("index", index));

    return OK;
}
```

这里直接通过发送一个AMessage对象来获取输入缓冲区的Index, 直接看到处理该消息的逻辑：

MediaCodec本身继承自AHandler，他处理消息的逻辑也在MediaCodec.cpp类内

```c++
case kWhatDequeueInputBuffer:
{
    sp<AReplyToken> replyID;
    CHECK(msg->senderAwaitsResponse(&replyID));
    // ... 中间一些状态检查的代码先忽略
    if (handleDequeueInputBuffer(replyID, true /* new request */)) {
        break;
    }
    //... 后面的逻辑是timeoutUs不为0时的一些逻辑，也都先忽略 值考虑timeoutUs为0的情况，即要求缓冲区立马返回。
    break;
}

//handleDequeueInputBuffer
bool MediaCodec::handleDequeueInputBuffer(const sp<AReplyToken> &replyID, bool newRequest) {
    //前置的一些状态检查全部忽略
    ssize_t index = dequeuePortBuffer(kPortIndexInput);
    if (index < 0) {
        //未获取到缓冲区
        CHECK_EQ(index, -EAGAIN);
        return false;
    }
    //获取到缓冲区返回对应的输入缓冲区index
    sp<AMessage> response = new AMessage;
    response->setSize("index", index);
    response->postReply(replyID);

    return true;
}

//dequeuePortBuffer
ssize_t MediaCodec::dequeuePortBuffer(int32_t portIndex) {
    CHECK(portIndex == kPortIndexInput || portIndex == kPortIndexOutput);

    //从mAvailPortBuffers中获取一个缓冲区index
    List<size_t> *availBuffers = &mAvailPortBuffers[portIndex];

    if (availBuffers->empty()) {
        //没有缓冲区的情况
        return -EAGAIN;
    }
    //取出list中的首个index
    size_t index = *availBuffers->begin();
    //从可获取队列中移除。
    availBuffers->erase(availBuffers->begin());
    
    //根据index 获取到具体的缓冲区对象
    BufferInfo *info = &mPortBuffers[portIndex][index];
    CHECK(!info->mOwnedByClient);
    {
        Mutex::Autolock al(mBufferLock);
        info->mOwnedByClient = true;

        // set image-data
        if (info->mData->format() != NULL) {
            sp<ABuffer> imageData;
            //实际的缓冲区开辟内存
            if (info->mData->format()->findBuffer("image-data", &imageData)) {
                info->mData->meta()->setBuffer("image-data", imageData);
            }
            int32_t left, top, right, bottom;
            if (info->mData->format()->findRect("crop", &left, &top, &right, &bottom)) {
                info->mData->meta()->setRect("crop-rect", left, top, right, bottom);
            }
        }
    }
    return index;
}
```



我们看到缓冲区的管理是这样的，mAvailPortBuffers数组中有两个元素，一个是输入，一个是输出，他们index为：

```c++
enum {
    kPortIndexInput         = 0,
    kPortIndexOutput        = 1,
};
```

输入缓冲区使用的是mAvailPortBuffers[kPortIndexInput]的数据，该数据为List<size_t>结构，存储的是可用缓冲区的index 下标。 在dequeueInputBuffer的逻辑中，先从mAvailPortBuffers[kPortIndexInput]获取到 可用输入缓冲区的下标，再根据下标创建ABuffer对象，ABuffer对象才是实际的缓冲区对象。

我们目前bug的现象就是availBuffers是空的。 



看到这里，我们并没有什么头绪，虽然我们清楚里输入缓冲区是如何管理的，具体的输入缓冲区对应到native层是ABuffer对象， 但是为什么availBuffers是空的，我们可能要从别地方再找找。



### 3.2、mAvailPortBuffers是如何管理的

在一下代码中发现了mAvailPortBuffers 是如和被添加的

```c++
size_t MediaCodec::updateBuffers(
        int32_t portIndex, const sp<AMessage> &msg) {
    CHECK(portIndex == kPortIndexInput || portIndex == kPortIndexOutput);
    size_t index;
    CHECK(msg->findSize("index", &index));
    sp<RefBase> obj;
    CHECK(msg->findObject("buffer", &obj));
    //根据接收到的AMessage对象，获取到index与obj对象。并将obj强转为MediaCodecBuffer对象。
    sp<MediaCodecBuffer> buffer = static_cast<MediaCodecBuffer *>(obj.get());

    {
        Mutex::Autolock al(mBufferLock);
        if (mPortBuffers[portIndex].size() <= index) {
            //如果缓冲区的大小小于index 进行扩容
            mPortBuffers[portIndex].resize(align(index + 1, kNumBuffersAlign));
        }
        //将真正的缓冲区存下来
        mPortBuffers[portIndex][index].mData = buffer;
    }
    mAvailPortBuffers[portIndex].push_back(index);

    return index;
}


case kWhatFillThisBuffer:
{
    /* size_t index = */updateBuffers(kPortIndexInput, msg);

    if (mState == FLUSHING
            || mState == STOPPING
            || mState == RELEASING) {
        returnBuffersToCodecOnPort(kPortIndexInput);
        break;
    }

    if (!mCSD.empty()) {
        ssize_t index = dequeuePortBuffer(kPortIndexInput);
        CHECK_GE(index, 0);

        // If codec specific data had been specified as
        // part of the format in the call to configure and
        // if there's more csd left, we submit it here
        // clients only get access to input buffers once
        // this data has been exhausted.

        status_t err = queueCSDInputBuffer(index);

        if (err != OK) {
            ALOGE("queueCSDInputBuffer failed w/ error %d",
                  err);

            setStickyError(err);
            postActivityNotificationIfPossible();

            cancelPendingDequeueOperations();
        }
        break;
    }


```



### 3.1、MediaCodec创建

android.media.MediaCodec#createByCodecName

```java
@NonNull
public static MediaCodec createByCodecName(@NonNull String name)
        throws IOException {
    return new MediaCodec(
            name, false /* nameIsType */, false /* unused */);
}


private MediaCodec(
        @NonNull String name, boolean nameIsType, boolean encoder) {
    Looper looper;
    if ((looper = Looper.myLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else if ((looper = Looper.getMainLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else {
        mEventHandler = null;
    }
    //这两个EventHandler对象先忽略，后面在回来看
    mCallbackHandler = mEventHandler;
    mOnFrameRenderedHandler = mEventHandler;

    mBufferLock = new Object();
    //最终codec对象的创建是到native层实现的
    native_setup(name, nameIsType, encoder);
}

private native final void native_setup(
        @NonNull String name, boolean nameIsType, boolean encoder);
```



android_media_MediaCodec.cpp

```c++
static void android_media_MediaCodec_native_setup(
        JNIEnv *env, jobject thiz,
        jstring name, jboolean nameIsType, jboolean encoder) {
    ...
    //创建一个JMediaCodec对象
    sp<JMediaCodec> codec = new JMediaCodec(env, thiz, tmp, nameIsType, encoder);

    const status_t err = codec->initCheck();
    ...
    //调用JMediaCodec->registerSelf()方法
    codec->registerSelf();
    setMediaCodec(env,thiz, codec);
}

JMediaCodec::JMediaCodec(
        JNIEnv *env, jobject thiz,
        const char *name, bool nameIsType, bool encoder)
    : mClass(NULL),
      mObject(NULL) {
    jclass clazz = env->GetObjectClass(thiz);
    CHECK(clazz != NULL);

    mClass = (jclass)env->NewGlobalRef(clazz);
    mObject = env->NewWeakGlobalRef(thiz);

    cacheJavaObjects(env);

    mLooper = new ALooper;
    mLooper->setName("MediaCodec_looper");

    //codec内部会开启一个线程开始消息循环
    mLooper->start(
            false,      // runOnCallingThread
            true,       // canCallJava
            ANDROID_PRIORITY_VIDEO);
    if (nameIsType) {
        //ExoPlayer在初始化时，该值为false，我们暂时先忽略这个分支
        mCodec = MediaCodec::CreateByType(mLooper, name, encoder, &mInitStatus);
        if (mCodec == nullptr || mCodec->getName(&mNameAtCreation) != OK) {
            mNameAtCreation = "(null)";
        }
    } else {
        //关注CreateByComponentName如何创建codec
        mCodec = MediaCodec::CreateByComponentName(mLooper, name, &mInitStatus);
        mNameAtCreation = name;
    }
    CHECK((mCodec != NULL) != (mInitStatus != OK));
}

```

MediaCodec.cpp

```c++
// static
sp<MediaCodec> MediaCodec::CreateByComponentName(
        const sp<ALooper> &looper, const AString &name, status_t *err, pid_t pid, uid_t uid) {
    //通过构造方法创建codec对象
    sp<MediaCodec> codec = new MediaCodec(looper, pid, uid);
    //init方法调用
    const status_t ret = codec->init(name);
    if (err != NULL) {
        *err = ret;
    }
    return ret == OK ? codec : NULL; // NULL deallocates codec.
}
```

对于构造方，去源码里面扫一眼就行，里面的逻辑创建了ResourceManagerClient与ResourceManagerServiceProxy我们先不关心，后面有需要再回来看。

我们看下init方法：

```c++
status_t MediaCodec::init(const AString &name) {
    mResourceManagerService->init();
    //...
    const sp<IMediaCodecList> mcl = MediaCodecList::getInstance();
    if (mcl == NULL) {
        mCodec = NULL;  // remove the codec.
        return NO_INIT; // if called from Java should raise IOException
    }
    for (const AString &codecName : { name, tmp }) {
        //从MediaCodecList中根据codecName获取codec 下标
        ssize_t codecIdx = mcl->findCodecByName(codecName.c_str());
        if (codecIdx < 0) {
            continue;
        }
        //根据下标获取到mCodecInfo
        mCodecInfo = mcl->getCodecInfo(codecIdx);
        Vector<AString> mediaTypes;
        mCodecInfo->getSupportedMediaTypes(&mediaTypes);
        for (size_t i = 0; i < mediaTypes.size(); i++) {
            if (mediaTypes[i].startsWith("video/")) {
                mIsVideo = true;
                break;
            }
        }
        break;
    }
    if (mCodecInfo == nullptr) {
        return NAME_NOT_FOUND;
    }

    mCodec = GetCodecBase(name, mCodecInfo->getOwnerName());
    if (mCodec == NULL) {
        return NAME_NOT_FOUND;
    }

    if (mIsVideo) {
        // video codec needs dedicated looper
        if (mCodecLooper == NULL) {
            //如果是video会再开启一个线程
            mCodecLooper = new ALooper;
            mCodecLooper->setName("CodecLooper");
            mCodecLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
        }

        mCodecLooper->registerHandler(mCodec);
    } else {
        mLooper->registerHandler(mCodec);
    }

    mLooper->registerHandler(this);

    mCodec->setCallback(
            std::unique_ptr<CodecBase::CodecCallback>(
                    new CodecCallback(new AMessage(kWhatCodecNotify, this))));
    mBufferChannel = mCodec->getBufferChannel();
    mBufferChannel->setCallback(
            std::unique_ptr<CodecBase::BufferCallback>(
                    new BufferCallback(new AMessage(kWhatCodecNotify, this))));

    sp<AMessage> msg = new AMessage(kWhatInit, this);
    msg->setObject("codecInfo", mCodecInfo);
    // name may be different from mCodecInfo->getCodecName() if we stripped
    // ".secure"
    msg->setString("name", name);

    if (mAnalyticsItem != NULL) {
        mAnalyticsItem->setCString(kCodecCodec, name.c_str());
        mAnalyticsItem->setCString(kCodecMode, mIsVideo ? kCodecModeVideo : kCodecModeAudio);
    }

    if (mIsVideo) {
        mBatteryChecker = new BatteryChecker(new AMessage(kWhatCheckBatteryStats, this));
    }

    status_t err;
    Vector<MediaResource> resources;
    MediaResource::Type type =
            secureCodec ? MediaResource::kSecureCodec : MediaResource::kNonSecureCodec;
    MediaResource::SubType subtype =
            mIsVideo ? MediaResource::kVideoCodec : MediaResource::kAudioCodec;
    resources.push_back(MediaResource(type, subtype, 1));
    for (int i = 0; i <= kMaxRetry; ++i) {
        if (i > 0) {
            // Don't try to reclaim resource for the first time.
            if (!mResourceManagerService->reclaimResource(resources)) {
                break;
            }
        }

        sp<AMessage> response;
        err = PostAndAwaitResponse(msg, &response);
        if (!isResourceError(err)) {
            break;
        }
    }
    return err;
}
```

BpMediaCodecList.cpp

BpMediaCodecList为MediaCodecList服务的binder调用代理

```c++
virtual sp<MediaCodecInfo> getCodecInfo(size_t index) const
{
    Parcel data, reply;
    data.writeInterfaceToken(IMediaCodecList::getInterfaceDescriptor());
    data.writeInt32(index);
    //通过binder调用获取 codecInfo
    remote()->transact(GET_CODEC_INFO, data, &reply);
    status_t err = reply.readInt32();
    if (err == OK) {
        return MediaCodecInfo::FromParcel(reply);
    } else {
        return NULL;
    }
}
```

```c++
status_t BnMediaCodecList::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) {
    case GET_CODEC_INFO:
        {
            CHECK_INTERFACE(IMediaCodecList, data, reply);
            size_t index = static_cast<size_t>(data.readInt32());
            const sp<MediaCodecInfo> info = getCodecInfo(index);
            if (info != NULL) {
                reply->writeInt32(OK);
                info->writeToParcel(reply);
            } else {
                reply->writeInt32(-ERANGE);
            }
            return NO_ERROR;
        }
}
```

