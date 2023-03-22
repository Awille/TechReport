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

生产者会进行deququeBuffer调用，我们先找到IGraphicBufferProducer看是否有相关的函数：

```c++
该方法请求一个给客户端使用的新的buffer。 请求后缓冲槽的使用权被移交到客户端，这意味着服务端不会使用该缓冲槽对应的graphicBuffer的内容。  在客户端的角度，返回的槽位index，可能不包含BufferSlot，如果槽是空的，客户端应该调用requestBuffer去申请一个缓冲槽到该槽位。  客户端填充完这个缓冲区后，应该使用取消缓冲区或填充其关联缓冲区内容并调用queueBuffer将缓冲区所有权归还给服务器。如果dequeueBuffer返回BUFFER_NEEDS_REALLOCATION，客户端应该立即调用requestBuffer。 
fence参数将会被更新，以持有与缓冲区相关联的fence。在fence信号之前，不得覆盖缓冲区的内容。如果fence是Fence::NO_FENCE，则可以立即写入缓冲区。
// 宽度和高度参数必须不大于GL_MAX_VIEWPORT_DIMS和GL_MAX_TEXTURE_SIZE的最小值（参见：glGetIntegerv）。
// 由于无效的尺寸导致的错误可能要等到调用updateTexImage()才会报告。
// 如果宽度和高度都为零，则使用setDefaultBufferSize()指定的默认值。
// 如果格式为0，则将使用默认格式。
//
// usage参数指定gralloc缓冲区使用标志。这些值在<gralloc.h>中被枚举，例如GRALLOC_USAGE_HW_RENDER。它们将与IGraphicBufferConsumer::setConsumerUsageBits指定的使用标志合并。
//
// 此调用将阻塞，直到有一个可用于出列的缓冲区。如果生产者和消费者都由应用程序控制，则此调用永远不会阻塞，并且如果没有可用的缓冲区，它将返回WOULD_BLOCK。
//
// 成功时将返回设置了标志（见上文）的非负值。
//
// 返回负值表示发生了错误：
// * NO_INIT - 缓冲区队列已被放弃或生产者未连接。
// * BAD_VALUE - 在异步模式下，缓冲区计数小于可同时分配的最大缓冲区数。
// * INVALID_OPERATION - 无法附加缓冲区，因为它会导致太多的缓冲区出列，要么是因为生产者已经出列了一个单独的缓冲区并且没有设置缓冲区计数，要么是因为已经设置了缓冲区计数，而这个调用会导致其超过限制。
// * WOULD_BLOCK - 当前没有可用的缓    
    
    
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
