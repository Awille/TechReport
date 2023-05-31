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



## 4、MediaCodec创建

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
    //获取CodecBase GetCodecBase会返回一个ACodec对象或者CCodec对象，我们先知关注ACodec
    mCodec = GetCodecBase(name, mCodecInfo->getOwnerName());

    if (mIsVideo) {
        // video codec needs dedicated looper
        if (mCodecLooper == NULL) {
            //如果是video会再开启一个线程
            mCodecLooper = new ALooper;
            mCodecLooper->setName("CodecLooper");
            mCodecLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
        }
		//注册handler
        mCodecLooper->registerHandler(mCodec);
    } else {
        mLooper->registerHandler(mCodec);
    }
    mLooper->registerHandler(this);
	//设置Callback
    mCodec->setCallback(
            std::unique_ptr<CodecBase::CodecCallback>(
                    new CodecCallback(new AMessage(kWhatCodecNotify, this))));
    //从Acodec中获取BufferChannle
    mBufferChannel = mCodec->getBufferChannel();
    mBufferChannel->setCallback(
            std::unique_ptr<CodecBase::BufferCallback>(
                    new BufferCallback(new AMessage(kWhatCodecNotify, this))));
	//创建init消息
    sp<AMessage> msg = new AMessage(kWhatInit, this);
    msg->setObject("codecInfo", mCodecInfo);
    // name may be different from mCodecInfo->getCodecName() if we stripped
    // ".secure"
    msg->setString("name", name);
    //发送init消息
    err = PostAndAwaitResponse(msg, &response);
    return err;
}

//init消息处理
case kWhatInit:
{
    sp<AReplyToken> replyID;
    CHECK(msg->senderAwaitsResponse(&replyID));

    if (mState != UNINITIALIZED) {
        PostReplyWithError(replyID, INVALID_OPERATION);
        break;
    }

    mReplyID = replyID;
    setState(INITIALIZING);

    sp<RefBase> codecInfo;
    CHECK(msg->findObject("codecInfo", &codecInfo));
    AString name;
    CHECK(msg->findString("name", &name));

    sp<AMessage> format = new AMessage;
    format->setObject("codecInfo", codecInfo);
    format->setString("componentName", name);
	//调用ACodec.initiateAllocateComponent方法
    mCodec->initiateAllocateComponent(format);
    break;
}
```

ACodec.cpp

```c++
std::shared_ptr<BufferChannelBase> ACodec::getBufferChannel() {
    if (!mBufferChannel) {
        mBufferChannel = std::make_shared<ACodecBufferChannel>(
                new AMessage(kWhatInputBufferFilled, this),
                new AMessage(kWhatOutputBufferDrained, this));
    }
    return mBufferChannel;
}

//ACodec.cpp
void ACodec::initiateAllocateComponent(const sp<AMessage> &msg) {
    msg->setWhat(kWhatAllocateComponent);
    msg->setTarget(this);
    msg->post();
}

//UninitializedState onHandleMessage方法：
case ACodec::kWhatAllocateComponent:
{
    onAllocateComponent(msg);
    handled = true;
    break;
}


//onAllocateComponent
bool ACodec::UninitializedState::onAllocateComponent(const sp<AMessage> &msg) {
    ALOGV("onAllocateComponent");
    sp<AMessage> notify = new AMessage(kWhatOMXMessageList, mCodec);
    notify->setInt32("generation", mCodec->mNodeGeneration + 1);
  	...
    sp<RefBase> obj;
    CHECK(msg->findObject("codecInfo", &obj));
    sp<MediaCodecInfo> info = (MediaCodecInfo *)obj.get();
    ...
    //notify 是一个AMessage对象，他持有AHandler引用，可以将消息转发到AHandler 这里的AHandler就是ACodec  
    sp<CodecObserver> observer = new CodecObserver(notify);
  //IOMX是一个系统服务
    sp<IOMX> omx;
    sp<IOMXNode> omxNode;

    status_t err = NAME_NOT_FOUND;
    OMXClient client;
  //通过OMXClient获得对应的系统服务
    omx = client.interface();

    //使用IOMX服务闯将IOMXNode实例，其中传入observer进行binder通信
    err = omx->allocateNode(componentName.c_str(), observer, &omxNode);
    androidSetThreadPriority(tid, prevPriority);
   //更新codec Generation
    ++mCodec->mNodeGeneration;

    mCodec->mComponentName = componentName;
    mCodec->mRenderTracker.setComponentName(componentName);
    mCodec->mFlags = 0;
  	//记录相关的变量
    mCodec->mOMX = omx;
    mCodec->mOMXNode = omxNode;
  	//上层回调onComponentAllocated
    mCodec->mCallback->onComponentAllocated(mCodec->mComponentName.c_str());
  	//状态机扭转 至mLoadedState
    mCodec->changeState(mCodec->mLoadedState);

    return true;
}
```



在 Android 中，`IOMX`、`IOMXNode` 和 `OMXNodeInstance` 之间有着密切的关系，并提供了编解码器相关的能力。下面是对它们之间关系和提供的能力的详细说明：

1. IOMX（接口）：
   - `IOMX` 是一个系统级的接口，用于与底层的 OpenMAX (OMX) 组件进行交互。
   - 通过 `IOMX` 接口，应用程序可以请求系统加载、管理和操作 OMX 组件，实现音视频编解码的功能。
   - `IOMX` 接口提供了一系列方法，如加载 OMX 组件库、查询可用的组件、创建组件实例、配置组件参数、提交输入数据、获取输出数据等。
2. IOMXNode（接口）：
   - `IOMXNode` 是 `IOMX` 接口的一部分，用于表示单个 OMX 组件的实例。
   - 通过 `IOMXNode` 接口，应用程序可以与具体的 OMX 组件进行交互，并管理编解码器的状态、参数和数据传输等。
   - `IOMXNode` 接口提供了一系列方法，如设置组件回调、配置参数、提交输入数据、获取输出数据、切换状态等。
3. OMXNodeInstance（实现类）：
   - `OMXNodeInstance` 是 `IOMXNode` 接口的实现类，表示底层 OMX 组件的实例。
   - `OMXNodeInstance` 类是在 Android 框架中实现的，用于与底层 OMX 组件进行直接的通信和操作。
   - 它负责底层 OMX 组件的加载、初始化、参数配置、状态切换、输入输出缓冲区管理等。
   - `OMXNodeInstance` 实现了 `IOMXNode` 接口定义的方法，提供了与底层 OMX 组件的交互和管理功能。

关系：

- `IOMX` 是顶层的接口，用于管理和操作 OMX 组件。
- `IOMX` 接口中的 `IOMXNode` 表示单个 OMX 组件的实例。
- `OMXNodeInstance` 是 `IOMXNode` 接口的实现类，负责与底层 OMX 组件进行交互和操作。

能力：

- 通过 `IOMX` 接口，应用程序可以加载和管理底层的 OMX 组件，进行音视频编解码的操作。
- `IOMXNode` 接口提供了与具体 OMX 组件的交互，包括配置参数、提交输入数据、获取输出数据等。
- `OMXNodeInstance` 类实现了 `IOMXNode` 接口，提供了与底层 OMX 组件直接通信和操作的能力。

总结： `IOMX`、`IOMXNode` 和 `OMXNodeInstance` 之间形成了一个层次关系，提供了与底层



Omx.cpp

```c++
Return<void> Omx::allocateNode(
        const hidl_string& name,
        const sp<IOmxObserver>& observer,
        allocateNode_cb _hidl_cb) {

    using ::android::IOMXNode;
    using ::android::IOMXObserver;

    sp<OMXNodeInstance> instance;
    {
        Mutex::Autolock autoLock(mLock);
        if (mLiveNodes.size() == kMaxNodeInstances) {
          	//codec 实例最大数限制
						//constexpr size_t kMaxNodeInstances = (1 << 16);
            _hidl_cb(toStatus(NO_MEMORY), nullptr);
            return Void();
        }
				//创建OMXNodeInstance对象
        instance = new OMXNodeInstance(
                this, new LWOmxObserver(observer), name.c_str());
				
        OMX_COMPONENTTYPE *handle;
      	//核心：创建OMX_COMPONENTTYPE  OMX组件实体
        OMX_ERRORTYPE err = mMaster->makeComponentInstance(
                name.c_str(), &OMXNodeInstance::kCallbacks,
                instance.get(), &handle);

        if (err != OMX_ErrorNone) {
            LOG(ERROR) << "Failed to allocate omx component "
                    "'" << name.c_str() << "' "
                    " err=" << asString(err) <<
                    "(0x" << std::hex << unsigned(err) << ")";
            _hidl_cb(toStatus(StatusFromOMXError(err)), nullptr);
            return Void();
        }
        instance->setHandle(handle);

        // Find quirks from mParser
        const auto& codec = mParser.getCodecMap().find(name.c_str());
        if (codec == mParser.getCodecMap().cend()) {
            LOG(WARNING) << "Failed to obtain quirks for omx component "
                    "'" << name.c_str() << "' "
                    "from XML files";
        } else {
            uint32_t quirks = 0;
            for (const auto& quirk : codec->second.quirkSet) {
                if (quirk == "quirk::requires-allocate-on-input-ports") {
                    quirks |= OMXNodeInstance::
                            kRequiresAllocateBufferOnInputPorts;
                }
                if (quirk == "quirk::requires-allocate-on-output-ports") {
                    quirks |= OMXNodeInstance::
                            kRequiresAllocateBufferOnOutputPorts;
                }
            }
            instance->setQuirks(quirks);
        }

        mLiveNodes.add(observer.get(), instance);
        mNode2Observer.add(instance.get(), observer.get());
    }
    observer->linkToDeath(this, 0);

    _hidl_cb(toStatus(OK), new TWOmxNode(instance));
    return Void();
}
```

OMXMaster.cpp

```c++
OMX_ERRORTYPE OMXMaster::makeComponentInstance(
        const char *name,
        const OMX_CALLBACKTYPE *callbacks,
        OMX_PTR appData,
        OMX_COMPONENTTYPE **component) {
    ALOGI("makeComponentInstance(%s) in %s process", name, mProcessName);
    Mutex::Autolock autoLock(mLock);
		//初始化指向OMX_COMPONENTTYPE的指针存储的值为NULL，即没有指向任何对象
    *component = NULL;
		
    ssize_t index = mPluginByComponentName.indexOfKey(String8(name));

    if (index < 0) {
        return OMX_ErrorInvalidComponentName;
    }
		//获取OMXPluginBase实例，该实例可以用于创建OMX组件
    OMXPluginBase *plugin = mPluginByComponentName.valueAt(index);
  	//使用别的OMXPluginBase创建OMX组件实例
    OMX_ERRORTYPE err =
        plugin->makeComponentInstance(name, callbacks, appData, component);

    if (err != OMX_ErrorNone) {
        return err;
    }
		//创建映射关系
    mPluginByInstance.add(*component, plugin);

    return err;
}
```

`OMXPluginBase`在Android的OpenMAX IL (Integration Layer)实现中起着重要的作用。它是OpenMAX组件库的基础接口，并由各种具体的插件类实现。在Android系统中，OpenMAX IL用于与多媒体硬件编解码器交互，如视频编解码器和音频编解码器等。

其主要的功能包括：

1. **实例化OMX组件**：通过其`makeComponentInstance`方法，OMXPluginBase可以创建特定类型的OMX组件。这些组件通常对应特定的硬件编解码器。
2. **枚举OMX组件**：通过`enumerateComponents`和`getComponentName`方法，OMXPluginBase能够列出其支持的所有OMX组件名称。这在初始化过程中，或者在需要确定设备支持哪些硬件编解码器时非常有用。
3. **销毁OMX组件**：通过其`destroyComponentInstance`方法，OMXPluginBase可以销毁已经创建的OMX组件实例。
4. **查询组件信息**：如角色（`getRolesOfComponent`），这些角色标识了OMX组件能处理的数据类型，例如音频解码器或视频编码器等。

因此，OMXPluginBase不仅仅是用来创建OMX组件，它还提供了查询组件信息和销毁组件的功能，同时也为OMX组件提供了基础接口。这使得不同的硬件厂商可以为其特定的硬件编解码器实现自己的OpenMAX IL插件，进而与Android的多媒体框架集成。



这里看到OMXMaster本身也继承自OMXPluginBase，但是他这边主要是做了中间的一层分发，将创建具体OMX组件的任务分发给了其他的OMXPluginBase实现类。

对于硬件的实现类我们没办法找到，先看一个软解的：

```c++
OMX_ERRORTYPE SoftOMXPlugin::makeComponentInstance(
        const char *name,
        const OMX_CALLBACKTYPE *callbacks,
        OMX_PTR appData,
        OMX_COMPONENTTYPE **component) {
    ALOGV("makeComponentInstance '%s'", name);

    for (size_t i = 0; i < kNumComponents; ++i) {
        if (strcmp(name, kComponents[i].mName)) {
            continue;
        }

        AString libName = "libstagefright_soft_";
        libName.append(kComponents[i].mLibNameSuffix);
        libName.append(".so");
      	//它在运行时打开一个动态链接库，并返回一个指向该库的句柄
        void *libHandle = dlopen(libName.c_str(), RTLD_NOW|RTLD_NODELETE);
      	//定义要动态加载的函数(*CreateSoftOMXComponentFunc) 函数指针，返回值为SoftOMXComponent *指针
        typedef SoftOMXComponent *(*CreateSoftOMXComponentFunc)(
                const char *, const OMX_CALLBACKTYPE *,
                OMX_PTR, OMX_COMPONENTTYPE **);
				//给定的动态链接库中的地址, 查找一个符号（通常是一个函数或变量）
        CreateSoftOMXComponentFunc createSoftOMXComponent =
            (CreateSoftOMXComponentFunc)dlsym(
                    libHandle,
                    "_Z22createSoftOMXComponentPKcPK16OMX_CALLBACKTYPE"
                    "PvPP17OMX_COMPONENTTYPE");

        //调用该函数闯将SoftOMXComponent实例
        sp<SoftOMXComponent> codec =
            (*createSoftOMXComponent)(name, callbacks, appData, component);
				//初始化
        OMX_ERRORTYPE err = codec->initCheck();
        if (err != OMX_ErrorNone) {
            dlclose(libHandle);
            libHandle = NULL;

            return err;
        }

        codec->incStrong(this);
        codec->setLibHandle(libHandle);

        return OMX_ErrorNone;
    }

    return OMX_ErrorInvalidComponentName;
}
```





## 5、SetSurface

首先在用户初始配置时：就需要传入surface，当然一开始用户上层surface可能并没有准备好，所以这个时候可能会设置一个surfacePlaceHolder

```java
@Override
public MediaCodecAdapter createAdapter(Configuration configuration) throws IOException {
      @Nullable MediaCodec codec = null;
      try {
        codec = createCodec(configuration);
        TraceUtil.beginSection("configureCodec");
        codec.configure(
            configuration.mediaFormat,
            configuration.surface,
            configuration.crypto,
            configuration.flags);
        TraceUtil.endSection();
        TraceUtil.beginSection("startCodec");
        codec.start();
        TraceUtil.endSection();
        return new SynchronousMediaCodecAdapter(codec);
      } catch (IOException | RuntimeException e) {
        if (codec != null) {
          codec.release();
        }
        throw e;
      }
    }
}
```

在这个过程中，会用这个surface初始化mNativeWindow对象。`ANativeWindow`是Android系统中一个非常重要的接口，它为应用程序提供了访问底层窗口系统的途径。这是一个抽象的接口，封装了与底层窗口系统进行交互的所有必要操作，如设置缓冲区的属性（如大小、格式等）、获取/释放缓冲区、锁定缓冲区进行写入等。

在Android中，`Surface`对象就是基于`ANativeWindow`的一个具体实现。`Surface`对象包含一个`ANativeWindow`指针，并且提供了更加易用的API供上层应用程序使用。

`ANativeWindow`作为一个接口，允许不同的实现，可以根据具体的需求来选择使用哪种实现。例如，一种实现可能更适合于在屏幕上直接绘制内容，而另一种实现可能更适合于后台处理图像数据。

总的来说，`ANativeWindow`是一个允许应用程序访问并操作底层窗口系统的关键接口。





接下来看我们设置Surface的接口。

MediaCodec.java

```c++
public void setOutputSurface(@NonNull Surface surface) {
    if (!mHasSurface) {
        throw new IllegalStateException("codec was not configured for an output surface");
    }
    native_setSurface(surface);
}


private native void native_setSurface(@NonNull Surface surface);
```

android_media_MediaCodec.cpp

```c++
static void android_media_MediaCodec_native_setSurface(
        JNIEnv *env,
        jobject thiz,
        jobject jsurface) {
    sp<JMediaCodec> codec = getMediaCodec(env, thiz);

    if (codec == NULL) {
        throwExceptionAsNecessary(env, INVALID_OPERATION);
        return;
    }
		//图形缓冲区生产者(比如应用程序)
    sp<IGraphicBufferProducer> bufferProducer;
    if (jsurface != NULL) {
      	//获取native的Surface对象
        sp<Surface> surface(android_view_Surface_getSurface(env, jsurface));
        if (surface != NULL) {
          	//获取当前输入缓冲区的生产者, 这样可以后续通过该producer 获取BufferQueue中的缓冲区
            bufferProducer = surface->getIGraphicBufferProducer();
        } else {
            jniThrowException(
                    env,
                    "java/lang/IllegalArgumentException",
                    "The surface has been released");
            return;
        }
    }
		//把生产者设置到JMediaCodec当中
    status_t err = codec->setSurface(bufferProducer);
    throwExceptionAsNecessary(env, err);
}


status_t JMediaCodec::setSurface(
        const sp<IGraphicBufferProducer> &bufferProducer) {
    sp<Surface> client;
    if (bufferProducer != NULL) {
      	//根据IGraphicBufferProducer创建一个新的Surface对象
        client = new Surface(bufferProducer, true /* controlledByApp */);
    }
  	//调用MediaCodec.setSurface方法设置surface
    status_t err = mCodec->setSurface(client);
    if (err == OK) {
        mSurfaceTextureClient = client;
    }
    return err;
}
```



MediaCodec.cpp

```c++
status_t MediaCodec::setSurface(const sp<Surface> &surface) {
  	//消息转发
    sp<AMessage> msg = new AMessage(kWhatSetSurface, this);
    msg->setObject("surface", surface);

    sp<AMessage> response;
    return PostAndAwaitResponse(msg, &response);
}

void MediaCodec::onMessageReceived(const sp<AMessage> &msg) {
        case kWhatSetSurface:
        {
            sp<AReplyToken> replyID;
            CHECK(msg->senderAwaitsResponse(&replyID));

            status_t err = OK;

            switch (mState) {
                case CONFIGURED:
                case STARTED:
                case FLUSHED:
                {
                    sp<RefBase> obj;
                    (void)msg->findObject("surface", &obj);
                    //转换成native surface对象
                    sp<Surface> surface = static_cast<Surface *>(obj.get());
                    if (mSurface == NULL) {
                        //忽略校验代码
                    } else {
                      	//因为这是新建的surface对象，要让新建的surface对象与底层的ANativeWindow建立联系
                        err = connectToSurface(surface);
                        if (err == ALREADY_EXISTS) {
                            // reconnecting to same surface
                            err = OK;
                        } else {
                            //将新生成的surface设置到ACodec当中。
                            err = mCodec->setSurface(surface)
                            if (err == OK) {
                                (void)disconnectFromSurface();
                                mSurface = surface;
                            }
                        }
                    }
                    break;
                }

                default:
                    err = INVALID_OPERATION;
                    break;
            }

            PostReplyWithError(replyID, err);
            break;
        } 
}


status_t MediaCodec::connectToSurface(const sp<Surface> &surface) {
    status_t err = OK;
    if (surface != NULL) {
        uint64_t oldId, newId;
      	//nativeWindowConnect方法在Android源代码中用于建立与ANativeWindow实例的连接，使得某一组件（比如一个媒体解码器）可以通过这个接口与窗口交互。
        err = nativeWindowConnect(surface.get(), "connectToSurface");
        if (err == OK) {
          	//设置surface的一些状态
            static uint32_t mSurfaceGeneration = 0;
            uint32_t generation = (getpid() << 10) | (++mSurfaceGeneration & ((1 << 10) - 1));
            surface->setGenerationNumber(generation);
            nativeWindowDisconnect(surface.get(), "connectToSurface(reconnect)");
            err = nativeWindowConnect(surface.get(), "connectToSurface(reconnect)");
        }
    }
    // do not return ALREADY_EXISTS unless surfaces are the same
    return err == ALREADY_EXISTS ? BAD_VALUE : err;
}
```

ACodec.cpp

```c++
status_t ACodec::setSurface(const sp<Surface> &surface) {
    sp<AMessage> msg = new AMessage(kWhatSetSurface, this);
    msg->setObject("surface", surface);

    sp<AMessage> response;
    status_t err = msg->postAndAwaitResponse(&response);

    if (err == OK) {
        (void)response->findInt32("err", &err);
    }
    return err;
}

bool ACodec::BaseState::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case ACodec::kWhatSetSurface:
        {
            sp<AReplyToken> replyID;
            CHECK(msg->senderAwaitsResponse(&replyID));

            sp<RefBase> obj;
            CHECK(msg->findObject("surface", &obj));
						//调用ACodec.handleSetSurface
            status_t err = mCodec->handleSetSurface(static_cast<Surface *>(obj.get()));

            sp<AMessage> response = new AMessage;
            response->setInt32("err", err);
            response->postReply(replyID);
            break;
        }
    }
}

//mNativeWindow ANativeWindow智能指针
private sp<ANativeWindow> mNativeWindow;


status_t ACodec::handleSetSurface(const sp<Surface> &surface) {
   

    // cannot switch from bytebuffers to surface
    if (mNativeWindow == NULL) {
      	//在mNativeWindow具体的赋值在初始化OMX组件时被赋值 这里做基本校验
        ALOGW("component was not configured with a surface");
        return INVALID_OPERATION;
    }
		//获得当前surface对应的ANativeWindow
    ANativeWindow *nativeWindow = surface.get();
    // if we have not yet started the codec, we can simply set the native window  我们SPV有预渲染，所以一般都会先开始codec，忽略该分支
    if (mBuffers[kPortIndexInput].size() == 0) {
        mNativeWindow = surface;
        return OK;
    }

    int usageBits = 0;
    // no need to reconnect as we will not dequeue all buffers
  	//调用setupNativeWindowSizeFormatAndUsage 将一些nativeWindow的属性设置到OMX组件里面去，解码会参考这些因素
    status_t err = setupNativeWindowSizeFormatAndUsage(
            nativeWindow, &usageBits, !storingMetadataInDecodedBuffers());
    
    // get min undequeued count. We cannot switch to a surface that has a higher
    // undequeued count than we allocated.
    int minUndequeuedBuffers = 0;
  	//从nativeWindow获取一些未出列最小buffer
  	//"未入队的最小缓冲区数量"（NATIVE_WINDOW_MIN_UNDEQUEUED_BUFFERS）指的是在生产者端需要保持未入队的（即暂未使用的，可供生产者随时写入新数据的）缓冲区的最小数量。这个数量是系统考虑了平衡内存占用和生产-消费效率后确定的。
    err = nativeWindow->query(
            nativeWindow, NATIVE_WINDOW_MIN_UNDEQUEUED_BUFFERS,
            &minUndequeuedBuffers);
    if (minUndequeuedBuffers > (int)mNumUndequeuedBuffers) {
      	//对surface的空闲的缓冲区数量有要求， mNumUndequeuedBuffers默认值是0
        ALOGE("new surface holds onto more buffers (%d) than planned for (%zu)",
                minUndequeuedBuffers, mNumUndequeuedBuffers);
        return BAD_VALUE;
    }

    // we cannot change the number of output buffers while OMX is running
    // set up surface to the same count
    Vector<BufferInfo> &buffers = mBuffers[kPortIndexOutput];
    ALOGV("setting up surface for %zu buffers", buffers.size());
		//设置nativeWindow的缓冲区数量，与OMX保持一致
    err = native_window_set_buffer_count(nativeWindow, buffers.size());

    // need to enable allocation when attaching
  	//当IGBProducer获取BufferQueue中的缓冲区时，当BufferQueue所有的缓冲都在使用时，允许BufferQueue新建一个缓冲区。
    surface->getIGraphicBufferProducer()->allowAllocation(true);

    // dequeueBuffer cannot time out
    surface->setDequeueTimeout(-1);

    // for meta data mode, we move dequeud buffers to the new surface.
    // for non-meta mode, we must move all registered buffers
    for (size_t i = 0; i < buffers.size(); ++i) {
        const BufferInfo &info = buffers[i];
        // skip undequeued buffers for meta data mode
        if (storingMetadataInDecodedBuffers()
                && info.mStatus == BufferInfo::OWNED_BY_NATIVE_WINDOW) {
            ALOGV("skipping buffer");
            continue;
        }
        ALOGV("attaching buffer %p", info.mGraphicBuffer->getNativeBuffer());
      	//将OMX中的缓冲区绑定到BufferQueue当中。
        err = surface->attachBuffer(info.mGraphicBuffer->getNativeBuffer());
        if (err != OK) {
            ALOGE("failed to attach buffer %p to the new surface: %s (%d)",
                    info.mGraphicBuffer->getNativeBuffer(),
                    strerror(-err), -err);
            return err;
        }
    }

    // cancel undequeued buffers to new surface
    if (!storingMetadataInDecodedBuffers()) {
      	//非源数据模式取消绑定
        for (size_t i = 0; i < buffers.size(); ++i) {
            BufferInfo &info = buffers.editItemAt(i);
            if (info.mStatus == BufferInfo::OWNED_BY_NATIVE_WINDOW) {
                ALOGV("canceling buffer %p", info.mGraphicBuffer->getNativeBuffer());
                err = nativeWindow->cancelBuffer(
                        nativeWindow, info.mGraphicBuffer->getNativeBuffer(), info.mFenceFd);
                info.mFenceFd = -1;
                if (err != OK) {
                    ALOGE("failed to cancel buffer %p to the new surface: %s (%d)",
                            info.mGraphicBuffer->getNativeBuffer(),
                            strerror(-err), -err);
                    return err;
                }
            }
        }
        // disallow further allocation
        (void)surface->getIGraphicBufferProducer()->allowAllocation(false);
    }

    // push blank buffers to previous window if requested
    if (mFlags & kFlagPushBlankBuffersToNativeWindowOnShutdown) {
        pushBlankBuffersToNativeWindow(mNativeWindow.get());
    }

    mNativeWindow = nativeWindow;
    mNativeWindowUsageBits = usageBits;
    return OK;
}
```

在Android中，Vsync信号是由SurfaceFlinger来处理的，当Vsync信号到达时，SurfaceFlinger会遍历所有的窗口表面（Surface）并提交在这个Vsync周期内准备好的帧。

在你的场景中，如果OMX组件的输出缓冲区和activity的渲染缓冲区都在BufferQueue中且都准备好了，它们都会在同一个Vsync周期内被SurfaceFlinger处理。

然而，应当注意的是，OMX解码后的输出缓冲区和activity的渲染缓冲区是两个独立的BufferQueue中的缓冲区，即使它们都指向同一个物理表面（Surface），也并不意味着它们在Vsync周期内的处理顺序是固定的。这主要取决于它们的缓冲区何时准备就绪以及SurfaceFlinger如何调度它们。





## 6、Codec缓冲区释放如何进行渲染

ACodec.cpp

```c++
std::shared_ptr<BufferChannelBase> ACodec::getBufferChannel() {
    if (!mBufferChannel) {
        mBufferChannel = std::make_shared<ACodecBufferChannel>(
                new AMessage(kWhatInputBufferFilled, this),
                new AMessage(kWhatOutputBufferDrained, this));
    }
    return mBufferChannel;
}
```

MediaCodec.cpp

```c++
status_t MediaCodec::onReleaseOutputBuffer(const sp<AMessage> &msg) {
    size_t index;
    CHECK(msg->findSize("index", &index));

    int32_t render;
    if (!msg->findInt32("render", &render)) {
        render = 0;
    }
  	//省略状态检查代码
    BufferInfo *info = &mPortBuffers[kPortIndexOutput][index];
		

    // synchronization boundary for getBufferAndFormat
    sp<MediaCodecBuffer> buffer;
    {
        Mutex::Autolock al(mBufferLock);
        info->mOwnedByClient = false;
        buffer = info->mData;
        info->mData.clear();
    }

    if (render && buffer->size() != 0) {
        int64_t mediaTimeUs = -1;
        buffer->meta()->findInt64("timeUs", &mediaTimeUs);

        int64_t renderTimeNs = 0;
        if (!msg->findInt64("timestampNs", &renderTimeNs)) {
            // use media timestamp if client did not request a specific render timestamp
            ALOGV("using buffer PTS of %lld", (long long)mediaTimeUs);
            renderTimeNs = mediaTimeUs * 1000;
        }
      	//忽略solf render
      	//MediaCodec通过ACodec.getBufferChannel 获取mBufferChannel
        mBufferChannel->renderOutputBuffer(buffer, renderTimeNs);
    } else {
        mBufferChannel->discardBuffer(buffer);
    }

    return OK;
}
```

ACodecBufferChannel.cpp

```c++
status_t ACodecBufferChannel::renderOutputBuffer(
        const sp<MediaCodecBuffer> &buffer, int64_t timestampNs) {
    std::shared_ptr<const std::vector<const BufferInfo>> array(
            std::atomic_load(&mOutputBuffers));
    BufferInfoIterator it = findClientBuffer(array, buffer);
    if (it == array->end()) {
        return -ENOENT;
    }

    ALOGV("renderOutputBuffer #%d", it->mBufferId);
    sp<AMessage> msg = mOutputBufferDrained->dup();
    msg->setObject("buffer", buffer);
    msg->setInt32("buffer-id", it->mBufferId);
    msg->setInt32("render", true);
    msg->setInt64("timestampNs", timestampNs);
    msg->post();
    return OK;
}
```

ACodec.cpp

```c++
bool ACodec::BaseState::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatInputBufferFilled:
        {
            onInputBufferFilled(msg);
            break;
        }

        case kWhatOutputBufferDrained:
        {
          	//转接到ACodec的线程
            onOutputBufferDrained(msg);
            break;
        }
    }
}


void ACodec::BaseState::onOutputBufferDrained(const sp<AMessage> &msg) {
    IOMX::buffer_id bufferID;
    CHECK(msg->findInt32("buffer-id", (int32_t*)&bufferID));
    sp<RefBase> obj;
    CHECK(msg->findObject("buffer", &obj));
    sp<MediaCodecBuffer> buffer = static_cast<MediaCodecBuffer *>(obj.get());
    int32_t discarded = 0;
    msg->findInt32("discarded", &discarded);

    ssize_t index;
    //当前是在BaseState方法里面，每次创建BaseState时传入了ACodec对象，mCodec为ACodec对应的引用
    BufferInfo *info = mCodec->findBufferByID(kPortIndexOutput, bufferID, &index);
  	//获取当前BufferInfo的BufferInfo::Status
    BufferInfo::Status status = BufferInfo::getSafeStatus(info);
    //忽略异常校验
    info->mData = buffer;
    int32_t render;
    if (mCodec->mNativeWindow != NULL
            && msg->findInt32("render", &render) && render != 0
            && !discarded && buffer->size() != 0) {
        ATRACE_NAME("render");
      	//当前存在mNativeWindow，即设置过surface 以及render=true, 输出缓冲区size不为0
        // The client wants this buffer to be rendered.

        android_native_rect_t crop;
        if (buffer->format()->findRect("crop", &crop.left, &crop.top, &crop.right, &crop.bottom)) {
            // NOTE: native window uses extended right-bottom coordinate
            ++crop.right;
            ++crop.bottom;
          	//比较crop信息是否相同，如果不同则进行设置
            if (memcmp(&crop, &mCodec->mLastNativeWindowCrop, sizeof(crop)) != 0) {
                mCodec->mLastNativeWindowCrop = crop;
                status_t err = native_window_set_crop(mCodec->mNativeWindow.get(), &crop);
                ALOGW_IF(err != NO_ERROR, "failed to set crop: %d", err);
            }
        }

        int32_t dataSpace;
        if (buffer->format()->findInt32("android._dataspace", &dataSpace)
                && dataSpace != mCodec->mLastNativeWindowDataSpace) {
            status_t err = native_window_set_buffers_data_space(
                    mCodec->mNativeWindow.get(), (android_dataspace)dataSpace);
            mCodec->mLastNativeWindowDataSpace = dataSpace;
            ALOGW_IF(err != NO_ERROR, "failed to set dataspace: %d", err);
        }
        if (buffer->format()->contains("hdr-static-info")) {
            HDRStaticInfo info;
            if (ColorUtils::getHDRStaticInfoFromFormat(buffer->format(), &info)
                && memcmp(&mCodec->mLastHDRStaticInfo, &info, sizeof(info))) {
                setNativeWindowHdrMetadata(mCodec->mNativeWindow.get(), &info);
                mCodec->mLastHDRStaticInfo = info;
            }
        }

        sp<ABuffer> hdr10PlusInfo;
        if (buffer->format()->findBuffer("hdr10-plus-info", &hdr10PlusInfo)
                && hdr10PlusInfo != nullptr && hdr10PlusInfo->size() > 0
                && hdr10PlusInfo != mCodec->mLastHdr10PlusBuffer) {
            native_window_set_buffers_hdr10_plus_metadata(mCodec->mNativeWindow.get(),
                    hdr10PlusInfo->size(), hdr10PlusInfo->data());
            mCodec->mLastHdr10PlusBuffer = hdr10PlusInfo;
        }

        // save buffers sent to the surface so we can get render time when they return
        int64_t mediaTimeUs = -1;
        buffer->meta()->findInt64("timeUs", &mediaTimeUs);
        if (mediaTimeUs >= 0) {
            mCodec->mRenderTracker.onFrameQueued(
                    mediaTimeUs, info->mGraphicBuffer, new Fence(::dup(info->mFenceFd)));
        }

        int64_t timestampNs = 0;
        if (!msg->findInt64("timestampNs", &timestampNs)) {
            // use media timestamp if client did not request a specific render timestamp
            if (buffer->meta()->findInt64("timeUs", &timestampNs)) {
                ALOGV("using buffer PTS of %lld", (long long)timestampNs);
                timestampNs *= 1000;
            }
        }

        status_t err;
        err = native_window_set_buffers_timestamp(mCodec->mNativeWindow.get(), timestampNs);
        ALOGW_IF(err != NO_ERROR, "failed to set buffer timestamp: %d", err);

        info->checkReadFence("onOutputBufferDrained before queueBuffer");
      	//调用queueBuffer 通知数据已经渲染好
        err = mCodec->mNativeWindow->queueBuffer(
                    mCodec->mNativeWindow.get(), info->mGraphicBuffer.get(), info->mFenceFd);
        info->mFenceFd = -1;
        if (err == OK) {
            info->mStatus = BufferInfo::OWNED_BY_NATIVE_WINDOW;
        } else {
            ALOGE("queueBuffer failed in onOutputBufferDrained: %d", err);
            mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
            info->mStatus = BufferInfo::OWNED_BY_US;
            // keeping read fence as write fence to avoid clobbering
            info->mIsReadFence = false;
        }
    } else {
        if (mCodec->mNativeWindow != NULL && (discarded || buffer->size() != 0)) {
            // move read fence into write fence to avoid clobbering
            info->mIsReadFence = false;
            ATRACE_NAME("frame-drop");
        }
        info->mStatus = BufferInfo::OWNED_BY_US;
    }

  	//渲染结束后：
    PortMode mode = getPortMode(kPortIndexOutput);

    switch (mode) {
        case KEEP_BUFFERS:
        {
            // XXX fishy, revisit!!! What about the FREE_BUFFERS case below?

            if (info->mStatus == BufferInfo::OWNED_BY_NATIVE_WINDOW) {
                // We cannot resubmit the buffer we just rendered, dequeue
                // the spare instead.
							 //如果是元数据共享模式：将Buffer从BufferQueue中取出并持有
                info = mCodec->dequeueBufferFromNativeWindow();
            }
            break;
        }

        case RESUBMIT_BUFFERS:
        {
            if (!mCodec->mPortEOS[kPortIndexOutput]) {
                if (info->mStatus == BufferInfo::OWNED_BY_NATIVE_WINDOW) {
                    // We cannot resubmit the buffer we just rendered, dequeue
                    // the spare instead.
										//如果是元数据共享模式：将Buffer从BufferQueue中取出并持有
                    info = mCodec->dequeueBufferFromNativeWindow();
                }

                if (info != NULL) {
                    ALOGV("[%s] calling fillBuffer %u",
                         mCodec->mComponentName.c_str(), info->mBufferID);
                    info->checkWriteFence("onOutputBufferDrained::RESUBMIT_BUFFERS");
                  	//告诉codec缓冲区已经消费完毕，将Buffer返回给Codec
                    status_t err = mCodec->fillBuffer(info);
                    if (err != OK) {
                        mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
                    }
                }
            }
            break;
        }

        case FREE_BUFFERS:
        {
          	//释放Buffer
            status_t err = mCodec->freeBuffer(kPortIndexOutput, index);
            if (err != OK) {
                mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
            }
            break;
        }

        default:
            ALOGE("Invalid port mode: %d", mode);
            return;
    }
}
```



## 7、CodecObserver  ACodec中对OMX与的信息监听

```c++
//定义在IOMX.h类当中
class BnOMXObserver : public BnInterface<IOMXObserver> {
public:
    virtual status_t onTransact(
            uint32_t code, const Parcel &data, Parcel *reply,
            uint32_t flags = 0);
};

//实现类 定义在IOMX.cpp当中
status_t BnOMXObserver::onTransact(
    uint32_t code, const Parcel &data, Parcel *reply, uint32_t flags) {
    switch (code) {
        case OBSERVER_ON_MSG:
        {
            CHECK_OMX_INTERFACE(IOMXObserver, data, reply);
            std::list<omx_message> messages;
            status_t err = FAILED_TRANSACTION; // must receive at least one message
            do {
                int haveFence = data.readInt32();
                if (haveFence < 0) { // we use -1 to mark end of messages
                    break;
                }
                omx_message msg;
                msg.fenceFd = haveFence ? ::dup(data.readFileDescriptor()) : -1;
                msg.type = (typeof(msg.type))data.readInt32();
                err = data.read(&msg.u, sizeof(msg.u));
                ALOGV("onTransact reading message %d, size %zu", msg.type, sizeof(msg));
                messages.push_back(msg);
            } while (err == OK);

            if (err == OK) {
              	//获取消息成功后 调用onMessages
                onMessages(messages);
            }

            return err;
        }

        default:
            return BBinder::onTransact(code, data, reply, flags);
    }
}

struct CodecObserver : public BnOMXObserver {
    explicit CodecObserver(const sp<AMessage> &msg) : mNotify(msg) {}

    // from IOMXObserver  OMX binder调用回到这里
    virtual void onMessages(const std::list<omx_message> &messages) {
        if (messages.empty()) {
            return;
        }

        sp<AMessage> notify = mNotify->dup();
        sp<MessageList> msgList = new MessageList();
        for (std::list<omx_message>::const_iterator it = messages.cbegin();
              it != messages.cend(); ++it) {
            const omx_message &omx_msg = *it;

            sp<AMessage> msg = new AMessage;
            msg->setInt32("type", omx_msg.type);
            switch (omx_msg.type) {
                case omx_message::EVENT:
                {
                    msg->setInt32("event", omx_msg.u.event_data.event);
                    msg->setInt32("data1", omx_msg.u.event_data.data1);
                    msg->setInt32("data2", omx_msg.u.event_data.data2);
                    break;
                }

                case omx_message::EMPTY_BUFFER_DONE:
                {
                    msg->setInt32("buffer", omx_msg.u.buffer_data.buffer);
                    msg->setInt32("fence_fd", omx_msg.fenceFd);
                    break;
                }

                case omx_message::FILL_BUFFER_DONE:
                {
                    msg->setInt32(
                            "buffer", omx_msg.u.extended_buffer_data.buffer);
                    msg->setInt32(
                            "range_offset",
                            omx_msg.u.extended_buffer_data.range_offset);
                    msg->setInt32(
                            "range_length",
                            omx_msg.u.extended_buffer_data.range_length);
                    msg->setInt32(
                            "flags",
                            omx_msg.u.extended_buffer_data.flags);
                    msg->setInt64(
                            "timestamp",
                            omx_msg.u.extended_buffer_data.timestamp);
                    msg->setInt32(
                            "fence_fd", omx_msg.fenceFd);
                    break;
                }

                case omx_message::FRAME_RENDERED:
                {
                    msg->setInt64(
                            "media_time_us", omx_msg.u.render_data.timestamp);
                    msg->setInt64(
                            "system_nano", omx_msg.u.render_data.nanoTime);
                    break;
                }

                default:
                    ALOGE("Unrecognized message type: %d", omx_msg.type);
                    break;
            }
            msgList->getList().push_back(msg);
        }
        notify->setObject("messages", msgList);
        notify->post();
    }

protected:
    virtual ~CodecObserver() {}

private:
    const sp<AMessage> mNotify;

    DISALLOW_EVIL_CONSTRUCTORS(CodecObserver);
};
```

`omx_message` 是 Android OpenMAX IL (OMX IL) 用来在编解码器和它的调用者之间进行通信的消息。在 Android 中，`OMX IL` 是一种多媒体硬件抽象层的协议，它允许底层多媒体硬件被上层应用以一种设备无关的方式进行使用。

`omx_message` 的各个枚举值代表了不同的消息类型：

1. `omx_message::EVENT`: 这个消息表示有一个 OMX 事件需要处理。OMX 事件可以是 OMX 组件的状态改变、端口设置改变等。
2. `omx_message::EMPTY_BUFFER_DONE`: 这个消息表示 OMX 组件已经处理完一个输入缓冲区并且它已经准备好被重新使用。通常，当你将一个带有数据的输入缓冲区传递给 OMX 组件进行处理后（例如，解码操作），OMX 组件在处理完这个缓冲区后会发送 `EMPTY_BUFFER_DONE` 消息，告知这个缓冲区已经空了，可以被重新填充数据。（输入缓冲区就绪）
3. `omx_message::FILL_BUFFER_DONE`: 这个消息表示 OMX 组件已经处理完一个输出缓冲区并且它已经准备好被读取。在你请求 OMX 组件填充一个输出缓冲区后（例如，编码操作），OMX 组件在填充完这个缓冲区后会发送 `FILL_BUFFER_DONE` 消息，告知这个缓冲区已经被填充了数据，可以被读取。 （输出缓冲区就绪）
4. `omx_message::FRAME_RENDERED`: 这个消息在一个帧被成功渲染后发送。这是在 Android P（9.0） 中添加的一个新特性，用于帮助应用更准确地测量帧的渲染时间。这对于需要精确同步音视频的应用来说是非常有用的。

以上这些消息都是由底层 OMX 组件发送给上层的应用或框架，以通知它们 OMX 组件的状态和行为。在收到这些消息后，上层的应用或框架需要根据消息类型进行相应的处理。



IOMX服务

```
class IOMX : public RefBase {
public:

    typedef uint32_t buffer_id;

    virtual status_t listNodes(List<ComponentInfo> *list) = 0;

    virtual status_t allocateNode(
            const char *name, const sp<IOMXObserver> &observer,
            sp<IOMXNode> *omxNode) = 0;

    virtual status_t createInputSurface(
            sp<IGraphicBufferProducer> *bufferProducer,
            sp<IGraphicBufferSource> *bufferSource) = 0;
};
```



