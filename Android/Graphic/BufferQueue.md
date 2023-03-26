# BufferQueue源码解析

https://juejin.cn/post/7033402333128032286

## 1、BufferQueue生产消费者关系建立：

```c++
void BufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
        sp<IGraphicBufferConsumer>* outConsumer,
        bool consumerIsSurfaceFlinger) {
    LOG_ALWAYS_FATAL_IF(outProducer == nullptr,
            "BufferQueue: outProducer must not be NULL");
    LOG_ALWAYS_FATAL_IF(outConsumer == nullptr,
            "BufferQueue: outConsumer must not be NULL");

    sp<BufferQueueCore> core(new BufferQueueCore());
    LOG_ALWAYS_FATAL_IF(core == nullptr,
            "BufferQueue: failed to create BufferQueueCore");

    sp<IGraphicBufferProducer> producer(new BufferQueueProducer(core, consumerIsSurfaceFlinger));
    LOG_ALWAYS_FATAL_IF(producer == nullptr,
            "BufferQueue: failed to create BufferQueueProducer");

    sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
    LOG_ALWAYS_FATAL_IF(consumer == nullptr,
            "BufferQueue: failed to create BufferQueueConsumer");

    *outProducer = producer;
    *outConsumer = consumer;
}
```

## 2、BufferQueueCore

### 2.1、BufferSlot：缓冲槽，会与buffer进行绑定。

bufferState:

```c++
// BufferState tracks the states in which a buffer slot can be.
struct BufferState {

    // All slots are initially FREE (not dequeued, queued, acquired, or shared).
    BufferState()
    : mDequeueCount(0),
      mQueueCount(0),
      mAcquireCount(0),
      mShared(false) {
    }

    uint32_t mDequeueCount;
    uint32_t mQueueCount;
    uint32_t mAcquireCount;
    bool mShared;

    // A buffer can be in one of five states, represented as below:
    //一个buffer可以是下面五个状态
    //
    //         | mShared | mDequeueCount | mQueueCount | mAcquireCount |
    // --------|---------|---------------|-------------|---------------|
    // FREE    |  false  |       0       |      0      |       0       |
    // DEQUEUED|  false  |       1       |      0      |       0       |
    // QUEUED  |  false  |       0       |      1      |       0       |
    // ACQUIRED|  false  |       0       |      0      |       1       |
    // SHARED  |  true   |      any      |     any     |      any      |
    //
    // FREE indicates that the buffer is available to be dequeued by the
    // producer. The slot is "owned" by BufferQueue. It transitions to DEQUEUED
    // when dequeueBuffer is called.
    //FREE 代表buffer可以被生产者dequeue, 缓冲槽是bufferQueue持有的， 当deququeBuffer被调用时，状态会调转为DEQUEUED
    //
    // DEQUEUED indicates that the buffer has been dequeued by the producer, but
    // has not yet been queued or canceled. The producer may modify the
    // buffer's contents as soon as the associated release fence is signaled.
    // The slot is "owned" by the producer. It can transition to QUEUED (via
    // queueBuffer or attachBuffer) or back to FREE (via cancelBuffer or
    // detachBuffer).
    //Dequque 标识buffer已经被producer获取，但是并未返还回来或取消。 生产者可能修改buffer的内容当fence发出信号的时候。 
    //
    // QUEUED indicates that the buffer has been filled by the producer and
    // queued for use by the consumer. The buffer contents may continue to be
    // modified for a finite time, so the contents must not be accessed until
    // the associated fence is signaled. The slot is "owned" by BufferQueue. It
    // can transition to ACQUIRED (via acquireBuffer) or to FREE (if another
    // buffer is queued in asynchronous mode).
    // QUEUED 代表buffer已经被生产者填满，并且被提交到bufferqueue准备被消费者消费，buffer中的内存可能会被持续修改在一个有限 的时间内，所以只有当fence通知时，buffer中的内从才能被获取。
    //
    // ACQUIRED indicates that the buffer has been acquired by the consumer. As
    // with QUEUED, the contents must not be accessed by the consumer until the
    // acquire fence is signaled. The slot is "owned" by the consumer. It
    // transitions to FREE when releaseBuffer (or detachBuffer) is called. A
    // detached buffer can also enter the ACQUIRED state via attachBuffer.
    //ACQUIRED表示buffer已经被消费者获取，当在QUEUED状态时，缓冲区内容被持续消费知道fence信号通知。此时缓冲区被消费者所有。
    //
    // SHARED indicates that this buffer is being used in shared buffer
    // mode. It can be in any combination of the other states at the same time,
    // except for FREE (since that excludes being in any other state). It can
    // also be dequeued, queued, or acquired multiple times.
    //SHARED状态表示buffer进入共享模式
};
```

这里面有个SHARED状态比较奇怪，SHARED状态一般发生以下模式下：

在 Android 中，BufferQueue 生产者消费者模型是一个用于处理图像和视频帧的常见模型。在该模型中，生产者将帧数据写入缓冲区，消费者从缓冲区中读取并显示帧数据。

BufferQueue 支持两种缓冲区状态：共享（SHARED）和非共享（NON-SHARED）。

**在共享状态下，多个生产者可以将帧数据写入同一个缓冲区，多个消费者可以从同一个缓冲区中读取数据。这种情况通常发生在多个应用程序共享相同相机源的情况下。这可以提高系统资源的利用率，但需要确保不会发生多个生产者同时写入相同缓冲区的情况，否则可能会导致数据不一致性和其他问题。**

在非共享状态下，每个生产者都有自己的缓冲区，每个消费者也有自己的缓冲区。这种情况通常用于单个应用程序处理视频或图像的情况下。这种情况下，系统资源的利用率相对较低，但不需要考虑数据一致性问题。

在使用 BufferQueue 时，必须注意正确设置缓冲区状态，以确保正确处理数据并避免潜在的问题。

BufferSlot

```c++
struct BufferSlot {

    BufferSlot()
    : mGraphicBuffer(nullptr),
      mEglDisplay(EGL_NO_DISPLAY),
      mBufferState(),
      mRequestBufferCalled(false),
      mFrameNumber(0),
      mEglFence(EGL_NO_SYNC_KHR),
      mFence(Fence::NO_FENCE),
      mAcquireCalled(false),
      mNeedsReallocation(false) {
    }

    // mGraphicBuffer points to the buffer allocated for this slot or is NULL
    // if no buffer has been allocated. 该slot指向的GraphicBuffer
    sp<GraphicBuffer> mGraphicBuffer;

    // mEglDisplay is the EGLDisplay used to create EGLSyncKHR objects.
    // EGLDisplay , 用于创建 EGLSyncKHR 
    EGLDisplay mEglDisplay;

    // mBufferState is the current state of this buffer slot.
    // 当前 buffer slot 的状态
    BufferState mBufferState;

    // mRequestBufferCalled is used for validating that the producer did
    // call requestBuffer() when told to do so. Technically this is not
    // needed but useful for debugging and catching producer bugs.
    // 告知 procducer 是否被调用 requestBuffer
    // 用于 debug 和查找bug
    bool mRequestBufferCalled;

    // mFrameNumber is the number of the queued frame for this slot.  This
    // is used to dequeue buffers in LRU order (useful because buffers
    // may be released before their release fence is signaled).
    // mFrameNumber 是此 slot 的 Frame 队列编号, 这是用于 buffer 队列的 LRU排序
    // 因为 buffer 可能这发出 release fence 信号之前 release
    uint64_t mFrameNumber;

    // mEglFence is the EGL sync object that must signal before the buffer
    // associated with this buffer slot may be dequeued. It is initialized
    // to EGL_NO_SYNC_KHR when the buffer is created and may be set to a
    // new sync object in releaseBuffer.  (This is deprecated in favor of
    // mFence, below.)
    EGLSyncKHR mEglFence;

    // mFence is a fence which will signal when work initiated by the
    // previous owner of the buffer is finished. When the buffer is FREE,
    // the fence indicates when the consumer has finished reading
    // from the buffer, or when the producer has finished writing if it
    // called cancelBuffer after queueing some writes. When the buffer is
    // QUEUED, it indicates when the producer has finished filling the
    // buffer. When the buffer is DEQUEUED or ACQUIRED, the fence has been
    // passed to the consumer or producer along with ownership of the
    // buffer, and mFence is set to NO_FENCE.
    sp<Fence> mFence;

    // Indicates whether this buffer has been seen by a consumer yet
    // 指示消费者是否看到了此缓冲区
    bool mAcquireCalled;

    // Indicates whether the buffer was re-allocated without notifying the
    // producer. If so, it needs to set the BUFFER_NEEDS_REALLOCATION flag when
    // dequeued to prevent the producer from using a stale cached buffer.
    // 指示是否在没有通知生产者的情况下重新分配了 buffer 
    // 如果是的, 那么当这个 buffer 出队时它需要设置为 BUFFER_NEEDS_REALLOCATION , 以防止生产者使用了一个过时的 buffer
    bool mNeedsReallocation;
};
```

这里我们可以看到，BufferSlot最终的缓冲区为GraphicBuffer





### 2.2、BufferQueueCore持有的四个数据结构：

```c++
// mFreeSlots contains all of the slots which are FREE and do not currently
// have a buffer attached. [FREE]状态 未绑定buffer
std::set<int> mFreeSlots;

// mFreeBuffers contains all of the slots which are FREE and currently have
// a buffer attached.  [FREE]状态，已绑定buffer
std::list<int> mFreeBuffers;

// mUnusedSlots contains all slots that are currently unused. They should be
// free and not have a buffer attached. [FREE]状态 未绑定buffer 并且未使用
std::list<int> mUnusedSlots;

// mActiveBuffers contains all slots which have a non-FREE buffer attached.
//非[FREE]状态buffer 并且被buffer绑定
std::set<int> mActiveBuffers;
```





## 3、dequeuBuffer

生产者会进行deququeBuffer调用，我们先找到IGraphicBufferProducer.h看是否有相关的函数：

```c++
该方法请求一个给客户端使用的新的buffer。 请求后缓冲槽的使用权被移交到客户端，这意味着服务端不会使用该缓冲槽对应的graphicBuffer的内容。  在客户端的角度，返回的槽位index，可能不包含BufferSlot，如果槽是空的，客户端应该调用requestBuffer去申请一个缓冲槽到该槽位。  客户端填充完这个缓冲区后，应该使用取消缓冲区或填充其关联缓冲区内容并调用queueBuffer将缓冲区所有权归还给服务器。如果dequeueBuffer返回BUFFER_NEEDS_REALLOCATION，客户端应该立即调用requestBuffer。 
fence参数将会被更新，以持有与缓冲区相关联的fence。在fence信号之前，不得覆盖缓冲区的内容。如果fence是Fence::NO_FENCE，则可以立即写入缓冲区。
宽度和高度参数必须不大于GL_MAX_VIEWPORT_DIMS和GL_MAX_TEXTURE_SIZE的最小值（参见：glGetIntegerv）。
由于无效的尺寸导致的错误可能要等到调用updateTexImage()才会报告。
如果宽度和高度都为零，则使用setDefaultBufferSize()指定的默认值。
如果格式为0，则将使用默认格式。

usage参数指定gralloc缓冲区使用标志。这些值在<gralloc.h>中被枚举，例如GRALLOC_USAGE_HW_RENDER。它们将与IGraphicBufferConsumer::setConsumerUsageBits指定的使用标志合并。

此调用将阻塞，直到有一个可用于出列的缓冲区。如果生产者和消费者都由应用程序控制，则此调用永远不会阻塞，并且如果没有可用的缓冲区，它将返回WOULD_BLOCK。

成功时将返回设置了标志（见上文）的非负值。

返回负值表示发生了错误：
* NO_INIT - 缓冲区队列已被放弃或生产者未连接。
* BAD_VALUE - 在异步模式下，缓冲区计数小于可同时分配的最大缓冲区数。
* INVALID_OPERATION - 无法附加缓冲区，因为它会导致太多的缓冲区出列，要么是因为生产者已经出列了一个单独的缓冲区并且没有设置缓冲区计数，要么是因为已经设置了缓冲区计数，而这个调用会导致其超过限制。
* WOULD_BLOCK - 当前没有可用的缓    
    
    
// dequeueBuffer requests a new buffer slot for the client to use. Ownership
    // of the slot is transfered to the client, meaning that the server will not
    // use the contents of the buffer associated with that slot.
    //
    // The slot index returned may or may not contain a buffer (client-side).
    // If the slot is empty the client should call requestBuffer to assign a new
    // buffer to that slot.
    //
    // Once the client is done filling this buffer, it is expected to transfer
    // buffer ownership back to the server with either cancelBuffer on
    // the dequeued slot or to fill in the contents of its associated buffer
    // contents and call queueBuffer.
    //
    // If dequeueBuffer returns the BUFFER_NEEDS_REALLOCATION flag, the client is
    // expected to call requestBuffer immediately.
    //
    // If dequeueBuffer returns the RELEASE_ALL_BUFFERS flag, the client is
    // expected to release all of the mirrored slot->buffer mappings.
    //
    // The fence parameter will be updated to hold the fence associated with
    // the buffer. The contents of the buffer must not be overwritten until the
    // fence signals. If the fence is Fence::NO_FENCE, the buffer may be written
    // immediately.
    //
    // The width and height parameters must be no greater than the minimum of
    // GL_MAX_VIEWPORT_DIMS and GL_MAX_TEXTURE_SIZE (see: glGetIntegerv).
    // An error due to invalid dimensions might not be reported until
    // updateTexImage() is called.  If width and height are both zero, the
    // default values specified by setDefaultBufferSize() are used instead.
    //
    // If the format is 0, the default format will be used.
    //
    // The usage argument specifies gralloc buffer usage flags.  The values
    // are enumerated in <gralloc.h>, e.g. GRALLOC_USAGE_HW_RENDER.  These
    // will be merged with the usage flags specified by
    // IGraphicBufferConsumer::setConsumerUsageBits.
    //
    // This call will block until a buffer is available to be dequeued. If
    // both the producer and consumer are controlled by the app, then this call
    // can never block and will return WOULD_BLOCK if no buffer is available.
    //
    // A non-negative value with flags set (see above) will be returned upon
    // success.
    // If the format is 0, the default format will be used.
    //
    // The usage argument specifies gralloc buffer usage flags.  The values
    // are enumerated in <gralloc.h>, e.g. GRALLOC_USAGE_HW_RENDER.  These
    // will be merged with the usage flags specified by
    // IGraphicBufferConsumer::setConsumerUsageBits.
    //
    // This call will block until a buffer is available to be dequeued. If
    // both the producer and consumer are controlled by the app, then this call
    // can never block and will return WOULD_BLOCK if no buffer is available.
    //
    // A non-negative value with flags set (see above) will be returned upon
    // success.
    //
    // Return of a negative means an error has occurred:
    // * NO_INIT - the buffer queue has been abandoned or the producer is not
    //             connected.
    // * BAD_VALUE - both in async mode and buffer count was less than the
    //               max numbers of buffers that can be allocated at once.
    // * INVALID_OPERATION - cannot attach the buffer because it would cause
    //                       too many buffers to be dequeued, either because
    //                       the producer already has a single buffer dequeued
    //                       and did not set a buffer count, or because a
    //                       buffer count was set and this call would cause
    //                       it to be exceeded.
    // * WOULD_BLOCK - no buffer is currently available, and blocking is disabled
    //                 since both the producer/consumer are controlled by app
    // * NO_MEMORY - out of memory, cannot allocate the graphics buffer.
    // * TIMED_OUT - the timeout set by setDequeueTimeout was exceeded while
    //               waiting for a buffer to become available.
    //
    // All other negative values are an unknown error returned downstream
    // from the graphics allocator (typically errno).
   
    virtual status_t dequeueBuffer(int* slot, sp<Fence>* fence, uint32_t w, uint32_t h,
                                   PixelFormat format, uint64_t usage, uint64_t* outBufferAge,
                                   FrameEventHistoryDelta* outTimestamps) = 0;
```



查看其实现dequeueBuffer，只截取核心代码

```c++
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<android::Fence>* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
    //省略一些状态检查的代码
    //返回结果初始化
    status_t returnFlags = NO_ERROR;
    EGLDisplay eglDisplay = EGL_NO_DISPLAY;
    EGLSyncKHR eglFence = EGL_NO_SYNC_KHR;
    bool attachedByConsumer = false;

    { // Autolock scope
        //给bufferQueueCore的mMutex对象上锁
        std::unique_lock<std::mutex> lock(mCore->mMutex);

        // If we don't have a free buffer, but we are currently allocating, we wait until allocation
        // is finished such that we don't allocate in parallel.
        //如果bufferQueueCore当中mFreeBuffers为空 并且当前有别的生产者正在分配buffer ，等待分配完成
        if (mCore->mFreeBuffers.empty() && mCore->mIsAllocating) {
            mDequeueWaitingForAllocation = true;
            //阻塞等待
            mCore->waitWhileAllocatingLocked(lock);
            //重置状态
            mDequeueWaitingForAllocation = false;
            //唤醒因为该状态阻塞的线程
            mDequeueWaitingForAllocationCondition.notify_all();
        }

        if (format == 0) {
            //未指定format的话 使用默认format
            format = mCore->mDefaultBufferFormat;
        }

        // Enable the usage bits the consumer requested
        usage |= mCore->mConsumerUsageBits;
        //如果宽度与高度都为0，使用默认值
        const bool useDefaultSize = !width && !height;
        if (useDefaultSize) {
            width = mCore->mDefaultWidth;
            height = mCore->mDefaultHeight;
            if (mCore->mAutoPrerotation &&
                (mCore->mTransformHintInUse & NATIVE_WINDOW_TRANSFORM_ROT_90)) {
                std::swap(width, height);
            }
        }

        int found = BufferItem::INVALID_BUFFER_SLOT;
        while (found == BufferItem::INVALID_BUFFER_SLOT) {
            //循环查找客可用的缓冲槽
            status_t status = waitForFreeSlotThenRelock(FreeSlotCaller::Dequeue, lock, &found);
            if (status != NO_ERROR) {
                return status;
            }

            // This should not happen
            if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
                BQ_LOGE("dequeueBuffer: no available buffer slots");
                return -EBUSY;
            }
            
            //将找到的图形缓冲区复制给buffer， 但是这个缓冲区可能未分配好
            const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);

            // If we are not allowed to allocate new buffers,
            // waitForFreeSlotThenRelock must have returned a slot containing a
            // buffer. If this buffer would require reallocation to meet the
            // requested attributes, we free it and attempt to get another one.
            if (!mCore->mAllowAllocation) {
                //如果当前不允许分配缓冲区
                if (buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) {
                    //如果buffer需要重新分配图形缓冲区 并且 当前找到的缓冲槽是一个共享缓冲槽，报错
                    if (mCore->mSharedBufferSlot == found) {
                        BQ_LOGE("dequeueBuffer: cannot re-allocate a sharedbuffer");
                        return BAD_VALUE;
                    }
                    mCore->mFreeSlots.insert(found);
                    mCore->clearBufferSlotLocked(found);
                    found = BufferItem::INVALID_BUFFER_SLOT;
                    //开始下次查找
                    continue;
                }
            }
        }
        
        //将找到的缓冲槽的图形缓冲区赋值给buffer，当前要不是已经分配了图形缓冲区，要不是可以分配缓冲区
        const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);
        if (mCore->mSharedBufferSlot == found &&
                buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) {
            //不允许是共享缓冲区
            BQ_LOGE("dequeueBuffer: cannot re-allocate a shared"
                    "buffer");

            return BAD_VALUE;
        }

        if (mCore->mSharedBufferSlot != found) {
            mCore->mActiveBuffers.insert(found);
        }
        *outSlot = found;
        ATRACE_BUFFER_INDEX(found);

        attachedByConsumer = mSlots[found].mNeedsReallocation;
        mSlots[found].mNeedsReallocation = false;

        mSlots[found].mBufferState.dequeue();
        if ((buffer == nullptr) ||
                buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage))
        {
            //如果图形缓冲区为null 并且需要重新分配
            mSlots[found].mAcquireCalled = false;
            mSlots[found].mGraphicBuffer = nullptr;
            mSlots[found].mRequestBufferCalled = false;
            mSlots[found].mEglDisplay = EGL_NO_DISPLAY;
            mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
            mSlots[found].mFence = Fence::NO_FENCE;
            mCore->mBufferAge = 0;
            mCore->mIsAllocating = true;
            //标记需要重新分配缓冲区的FLAG
            returnFlags |= BUFFER_NEEDS_REALLOCATION;
        } else {
            // We add 1 because that will be the frame number when this buffer
            // is queued
            mCore->mBufferAge = mCore->mFrameCounter + 1 - mSlots[found].mFrameNumber;
        }

        eglDisplay = mSlots[found].mEglDisplay;
        eglFence = mSlots[found].mEglFence;
        // Don't return a fence in shared buffer mode, except for the first
        // frame.
        *outFence = (mCore->mSharedBufferMode &&
                mCore->mSharedBufferSlot == found) ?
                Fence::NO_FENCE : mSlots[found].mFence;
        mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
        mSlots[found].mFence = Fence::NO_FENCE;

        // If shared buffer mode has just been enabled, cache the slot of the
        // first buffer that is dequeued and mark it as the shared buffer.
        if (mCore->mSharedBufferMode && mCore->mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore->mSharedBufferSlot = found;
            mSlots[found].mBufferState.mShared = true;
        }

        if (!(returnFlags & BUFFER_NEEDS_REALLOCATION)) {
            if (mCore->mConsumerListener != nullptr) {
                mCore->mConsumerListener->onFrameDequeued(mSlots[*outSlot].mGraphicBuffer->getId());
            }
        }
    } // Autolock scope

    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
        //开始分配图形缓冲区
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(
                width, height, format, BQ_LAYER_COUNT, usage,
                {mConsumerName.string(), mConsumerName.size()});
        //图形缓冲区初始化检查
        status_t error = graphicBuffer->initCheck();

        { // Autolock scope
            std::lock_guard<std::mutex> lock(mCore->mMutex);

            if (error == NO_ERROR && !mCore->mIsAbandoned) {
                graphicBuffer->setGenerationNumber(mCore->mGenerationNumber);
                //赋值给缓冲槽
                mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
                if (mCore->mConsumerListener != nullptr) {
                    mCore->mConsumerListener->onFrameDequeued(
                            mSlots[*outSlot].mGraphicBuffer->getId());
                }
            }

            mCore->mIsAllocating = false;
            mCore->mIsAllocatingCondition.notify_all();

            if (error != NO_ERROR) {
                mCore->mFreeSlots.insert(*outSlot);
                mCore->clearBufferSlotLocked(*outSlot);
                BQ_LOGE("dequeueBuffer: createGraphicBuffer failed");
                return error;
            }

            if (mCore->mIsAbandoned) {
                mCore->mFreeSlots.insert(*outSlot);
                mCore->clearBufferSlotLocked(*outSlot);
                BQ_LOGE("dequeueBuffer: BufferQueue has been abandoned");
                return NO_INIT;
            }

            VALIDATE_CONSISTENCY();
        } // Autolock scope
    }

    if (attachedByConsumer) {
        returnFlags |= BUFFER_NEEDS_REALLOCATION;
    }

    if (eglFence != EGL_NO_SYNC_KHR) {
        EGLint result = eglClientWaitSyncKHR(eglDisplay, eglFence, 0,
                1000000000);
        // If something goes wrong, log the error, but return the buffer without
        // synchronizing access to it. It's too late at this point to abort the
        // dequeue operation.
        if (result == EGL_FALSE) {
            BQ_LOGE("dequeueBuffer: error %#x waiting for fence",
                    eglGetError());
        } else if (result == EGL_TIMEOUT_EXPIRED_KHR) {
            BQ_LOGE("dequeueBuffer: timeout waiting for fence");
        }
        eglDestroySyncKHR(eglDisplay, eglFence);
    }

    BQ_LOGV("dequeueBuffer: returning slot=%d/%" PRIu64 " buf=%p flags=%#x",
            *outSlot,
            mSlots[*outSlot].mFrameNumber,
            mSlots[*outSlot].mGraphicBuffer->handle, returnFlags);

    if (outBufferAge) {
        *outBufferAge = mCore->mBufferAge;
    }
    addAndGetFrameTimestamps(nullptr, outTimestamps);

    return returnFlags;
}
```



### 3、BufferSlot查找



# 应用程序上层调用

当应用进程接收到 VSync 信号后，会回调到 `onVSync()` 方法。在 `onVSync()` 方法中，应用程序可以通过调用 `Choreographer` 类的 `postFrameCallback()` 方法向消息队列中添加一个回调，该回调会在下一个 `VSync` 时刻触发。该回调中可以调用 `SurfaceView` 的 `getHolder()` 方法获取到 `SurfaceHolder` 对象，并通过调用 `SurfaceHolder` 的 `getSurface()` 方法获取到 `Surface` 对象。然后，通过调用 `Surface` 的 `lockCanvas()` 方法锁定画布，并在画布上绘制图像。

在 `lockCanvas()` 方法中，应用程序可以通过调用 `SurfaceFlinger` 的 `dequeueBuffer()` 方法从图形缓冲区队列中取出一个空闲的缓冲区。`dequeueBuffer()` 方法返回一个 `BufferItem` 结构体，其中包含了该缓冲区的索引、时间戳、裁剪矩形等信息。应用程序可以将要绘制的图像绘制在该缓冲区上，并通过调用 `Surface` 的 `unlockCanvasAndPost()` 方法将缓冲区释放，并将缓冲区提交到 SurfaceFlinger 的图形缓冲区队列中等待后续的处理。

需要注意的是，在 Android 中，Surface 对象通常是由 SurfaceView 或 TextureView 创建的，并且 Surface 对象通常与一个或多个图形缓冲区相关联。当应用程序绘制图像时，必须首先从图形缓冲区队列中获取一个空闲的缓冲区，并在该缓冲区上绘制图像。完成绘制后，应用程序必须释放该缓冲区，并将其提交到图形缓冲区队列中等待 SurfaceFlinger 进程进行处理。

E:\AOSP\frameworks\base\core\java\android\view\ViewRootImpl.java

```java
private void performDraw() {
        try {
            boolean canUseAsync = draw(fullRedrawNeeded);
            if (usingAsyncReport && !canUseAsync) {
                mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
                usingAsyncReport = false;
            }
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

private boolean draw(boolean fullRedrawNeeded) {
    Surface surface = mSurface;
    mAttachInfo.mTreeObserver.dispatchOnDraw();

    int xOffset = -mCanvasOffsetX;
    int yOffset = -mCanvasOffsetY + curScrollY;
    final WindowManager.LayoutParams params = mWindowAttributes;

    boolean useAsyncReport = false;
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        } else {
            //draw
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                    scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }
    }

    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
    return useAsyncReport;
}


private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {

    // Draw with software renderer.
    final Canvas canvas;

    try {
        dirty.offset(-dirtyXOffset, -dirtyYOffset);
        final int left = dirty.left;
        final int top = dirty.top;
        final int right = dirty.right;
        final int bottom = dirty.bottom;
        //核心逻辑
        canvas = mSurface.lockCanvas(dirty);
        // TODO: Do this in native
        canvas.setDensity(mDensity);
    } catch (Surface.OutOfResourcesException e) {
        handleOutOfResourcesException(e);
        return false;
    } catch (IllegalArgumentException e) {
        return false;
    } finally {
        dirty.offset(dirtyXOffset, dirtyYOffset);  // Reset to the original value.
    }

    try {
        canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);

        mView.draw(canvas);

        drawAccessibilityFocusedDrawableIfNeeded(canvas);
    } finally {
        try {
            //draw结束后 unlock
            surface.unlockCanvasAndPost(canvas);
        } catch (IllegalArgumentException e) {
            return false;
        }
    }
    return true;
}
```

E:\AOSP\frameworks\base\core\java\android\view\Surface.java

```java
public Canvas lockCanvas(Rect inOutDirty)
        throws Surface.OutOfResourcesException, IllegalArgumentException {
    synchronized (mLock) {
        checkNotReleasedLocked();
        if (mLockedObject != 0) {
            // Ideally, nativeLockCanvas() would throw in this situation and prevent the
            // double-lock, but that won't happen if mNativeObject was updated.  We can't
            // abandon the old mLockedObject because it might still be in use, so instead
            // we just refuse to re-lock the Surface.
            throw new IllegalArgumentException("Surface was already locked");
        }
        mLockedObject = nativeLockCanvas(mNativeObject, mCanvas, inOutDirty);
        return mCanvas;
    }
}

/**
 * Posts the new contents of the {@link Canvas} to the surface and
 * releases the {@link Canvas}.
 *
 * @param canvas The canvas previously obtained from {@link #lockCanvas}.
 */
public void unlockCanvasAndPost(Canvas canvas) {
    synchronized (mLock) {
        checkNotReleasedLocked();

        if (mHwuiContext != null) {
            mHwuiContext.unlockAndPost(canvas);
        } else {
            unlockSwCanvasAndPost(canvas);
        }
    }
}
```

E:\AOSP\frameworks\base\core\jni\android_view_Surface.cpp

```c++
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));

    Rect dirtyRect(Rect::EMPTY_RECT);
    Rect* dirtyRectPtr = NULL;

    if (dirtyRectObj) {
        dirtyRect.left   = env->GetIntField(dirtyRectObj, gRectClassInfo.left);
        dirtyRect.top    = env->GetIntField(dirtyRectObj, gRectClassInfo.top);
        dirtyRect.right  = env->GetIntField(dirtyRectObj, gRectClassInfo.right);
        dirtyRect.bottom = env->GetIntField(dirtyRectObj, gRectClassInfo.bottom);
        dirtyRectPtr = &dirtyRect;
    }

    ANativeWindow_Buffer outBuffer;
    //实际是native的surface进行lock
    status_t err = surface->lock(&outBuffer, dirtyRectPtr);
    return (jlong) lockedSurface.get();
}
```

E:\AOSP\frameworks\native\libs\gui\Surface.cpp

```c++
status_t Surface::lock(
        ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    .......
    //dequeueBuffer 流程结束
    status_t err = dequeueBuffer(&out, &fenceFd);
    ......
    return err;
}


status_t Surface::unlockAndPost()
{
    if (mLockedBuffer == nullptr) {
        ALOGE("Surface::unlockAndPost failed, no locked buffer");
        return INVALID_OPERATION;
    }

    int fd = -1;
    status_t err = mLockedBuffer->unlockAsync(&fd);
    ALOGE_IF(err, "failed unlocking buffer (%p)", mLockedBuffer->handle);

    err = queueBuffer(mLockedBuffer.get(), fd);
    ALOGE_IF(err, "queueBuffer (handle=%p) failed (%s)",
            mLockedBuffer->handle, strerror(-err));

    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = nullptr;
    return err;
}
```



# 附录-专有名词解释

## 1、BP(Binder Proxy) and BN(Binder Native)

Binder是C-S架构的，服务端也称为Binder进程，通过Binder驱动程序(Binder Driver) 来提供服务。

* BP（Binder Proxy）客户端进程中持有的，与服务端进程中的binder对象通信的代理对象。本地通过该代理对象与binder服务端进行通信。
* BN（Proxy Native）服务端进程中持有的，与客户端进程中的BP对应的本地Binder对象。Bn负责处理与客户端进程的BP对象之间的通信。

#  附录-C++ 基础语法点记录：

## 1、sp:智能指针：template<typename T> using sp = std::shared_ptr<T>

`std::shared_ptr`是C++标准库中的智能指针类，它可以管理堆上的动态内存资源，自动进行内存回收。`std::shared_ptr`通过计数器来追踪当前有多少个`std::shared_ptr`指向同一个对象，当计数器归零时，`std::shared_ptr`会自动释放所管理的对象的内存。

`std::shared_ptr`还具有拷贝构造函数和赋值操作符，它们会自动增加计数器并共享同一个内存资源。此外，`std::shared_ptr`还支持自定义删除器，这使得用户可以指定在内存释放时要执行的操作。

由于`std::shared_ptr`可以自动管理内存资源，避免了手动释放内存时可能出现的问题，因此使用它可以有效地避免内存泄漏等问题。

## 2、inline

c++的inline用于请求编译器将代码插入到调用该函数的代码中，以减少函数调用的开销，inline仅是对编译器的请求，编译器不一定会实现内联。

## 3、unit64_t

`uint64_t` 是 C++ 标准库中的一种固定大小的整数类型，它是无符号的 64 位整数，范围从 0 到 18,446,744,073,709,551,615（即 2 的 64 次方减 1）。

在 C++ 中，`uint64_t` 是在 `<cstdint>` 头文件中定义的类型别名，它是基于实现的具体类型（例如 `unsigned long long`）的别名。这意味着，在不同的平台和编译器下，`uint64_t` 可能有不同的底层类型，但它的大小和范围是固定的。

`uint64_t` 可以用于需要大量存储空间和精度的计算中，例如大整数算法、哈希函数、加密算法等。它还可以用于处理位掩码和位字段，以及在需要表示较大整数范围的计算中。

## 4、status_t

`status_t` 是一种在 Android 开发中常见的数据类型，它实际上是一个整数类型，通常用于表示函数或方法的执行状态。在 Android 开发中，通常将 `status_t` 声明为一个 `int32_t` 类型的别名，其定义在 `<utils/Errors.h>` 中。

## 5、Virtual 虚函数

在 C++ 中，虚函数是一种特殊的成员函数，可以在派生类中覆盖基类中的同名函数，并在运行时确定要调用哪个函数。使用虚函数可以实现多态，允许使用基类的指针或引用调用派生类的函数，提高代码的灵活性和可扩展性。

## 6、std::lock_guard<std::mutex> & std::unique_lock<std::mutex>

std::mutex是C++11标准库中的一个类，用于提供线程安全的互斥访问。它是一个可移植的方式来避免多个线程同时访问共享资源，以避免数据竞争和不一致性问题。当一个线程正在访问临界资源时，其他线程将被阻塞直到该线程释放锁。

`std::lock_guard<std::mutex>` 和 `std::unique_lock<std::mutex>` 都是 C++ 中用于实现互斥锁的类。它们的区别在于：

- `std::lock_guard` 是一个轻量级的 RAII（Resource Acquisition Is Initialization）包装器，使用起来更加简洁，适用于快速加锁解锁的场景。它的构造函数会立即获取所传入的互斥量的所有权，并在析构函数中释放该所有权，因此不需要显式调用 unlock() 方法。由于 `std::lock_guard` 没有提供 unlock() 方法，因此不能在获取锁的同时，将锁的所有权转移给另一个线程。
- `std::unique_lock` 是一个更加灵活的互斥锁包装器，提供了比 `std::lock_guard` 更多的功能。它的构造函数可以指定锁的类型（默认是可重入锁），并支持手动加锁和解锁操作。此外，`std::unique_lock` 还支持锁的所有权转移，即一个线程可以将锁的所有权转移到另一个线程，从而避免了线程切换的开销。

在代码中，如果只是需要加锁解锁，没有特殊的要求，推荐使用 `std::lock_guard<std::mutex>`，因为它使用起来更加简洁。如果需要更灵活的锁操作，比如手动加锁和解锁，或者锁的所有权转移等，可以使用 `std::unique_lock<std::mutex>`。

