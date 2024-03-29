# androidQ surfaceFlinger介绍

## 1 综述

从整体上分析Android图形显示系统的结构，更多的是使用结构图来让大家理解Android是如何绘制，合成图形并显示到屏幕上的。

参考文档：https://blog.changdong.ltd/2020/03/11/xue-xi-android-xian-shi-xi-tong/

https://ljd1996.github.io/2020/09/15/Android%E5%9B%BE%E5%BD%A2%E7%B3%BB%E7%BB%9F%E7%BB%BC%E8%BF%B0/



## 2 从下层往上层理解

思路：显示屏->SurfaceFlinger->上层绘制->Vsync以及三缓冲

### 2.1 显示屏

* 帧缓冲区简单理解：带有宽高和像素密度的内存区块

* 显示屏从帧缓冲区中开始读取内容，读取过程为：从buffer的其实地址开始，从上往下，从左往右扫描整个buffer，将内容映射到显示屏上。

* 由于屏幕在显示的内容在不同帧之间是要变化的，如果只有一个帧缓冲区，那么读取与合成操作很容易冲突，最终造成显示图像撕裂(同一个缓冲区有多个帧的内容)等问题。基于此考虑，硬件层提供了两个帧缓冲区，一个叫前缓冲区，用于屏幕显示，一个叫后缓冲区，用于屏幕图像合成。 前后缓冲区本质上没什么区别，他们会经常进行角色转换，比如当前帧A为前缓冲区，B为后缓冲区。当屏幕在显示A内容时，B在合成图像。当B合成图像结束后，B会变为前缓冲区，而A变为后缓冲区。

前后缓冲区能完美的运行，建立在当前一帧显示结束，后一帧合成完成的假设至上。显示情况肯定是有偏差的。

* 屏幕刷新率(HZ)：屏幕在1s内刷新屏幕的次数，Android手机一般为60HZ，当然现在市面上也出现了90HZ、120HZ的产品。
* 系统帧速率(FPS): 系统在1s内合成的帧数，该值大小有硬件、渲染任务的繁杂程度、算法等方方面面决定。

假设两种场景：

* 屏幕刷新率>系统帧速率：当前缓冲区内容显示结束后，后缓冲区数据还未合成完成，屏幕无法读取下一帧，造成一帧显示多次，产生卡顿。
* 系统帧速率>屏幕刷新率：当前缓冲区内容屏幕还未显示完全时，后缓冲区内容已经合成完成并要求屏幕读取下一帧，产生屏幕撕裂：屏幕部分显示上一帧，部分显示下一帧。

所以协调两者的速率是能正常显示的基础，我们一般的解决方案是让系统帧速率配合屏幕刷新率的节奏。节奏控制的法宝：

* VSync(垂直同步)：当屏幕从缓冲区扫描完一帧显示在屏幕上后，开始扫描下一帧前，发出一个同步信号，该信号用来切换前缓冲区和后缓冲区。

屏幕的显示节奏是固定的，比如是60HZ，那么每隔16.6ms，系统接收到Vsync信号，前后缓冲区切换，并系统要开始渲染下一帧，在下一个16.6ms内渲染完成，等Vsync到来给屏幕读取。

那么在android中是谁进行后缓冲区图像合成的工作，答案是surfaceFlinger。

* surfaceFlinger-图形合成者。 屏幕是消费者，那么SurfaceFlinger就是生产者。
  * 作为上层应用的消费者，硬件层的生产者。
  * 负责图形的合成
  * 与AMS一样，也是一个独立进行提供的系统服务。
  * 将多个surface上的内容进行合成。

我们应用中的每个窗口，或是顶部bar activity 都会有对应的window，每个windows会一一对应一个surface对象，SF的职责就是将上层对应的surface内的图像进行合成。所有的要当前帧要显示的surface会 合成(composite)一个 后缓冲区的 帧内容，最终显示在屏幕上。

SF通过后缓冲区与屏幕建立联系，通过surface与上层建立联系，从而起到一个承上启下的作用。

* Surface
  * 每个surface对应上层的一个window(对话框、activity、状态栏)
  * 作为上层图形绘制的花画板
  * canvas是画布，上层通过调用canvas的API向surface上绘制图形。
  * surface内部存在多个缓冲区，形成一个bufferQueue（这点需要好好理解）。

在surface这一侧，我们提到图形的传递是通过Buffer作为载体的，surface是对buffer的进一步分装，也就是说surface内部具有多个buffer供上层使用，如何管理buffer呢？

surface内存为了管理多个buffer，形成了一个Bufferqueue， 每个窗口对应的进程在renderThread中渲染结束以后，会将渲染完成的缓冲提交到buferqueue当中，后续等待SF将当前屏幕显示的所有surface对应Buffequeue里面的内容合成一帧，提交到后缓冲区当中。

每个buffer拥有四种状态：

* Free: 可被上层使用
* Dequeued: 出列，正在被上层使用
* Queued: 入列，已完成上层绘制，等待SF合成
* Acquired: 被获取, SF正在持有该buffer进行合成。

Buffer



















记录一下，后续看看吧内容插入到哪里：

* 你能具体讲讲android 主线程最终是 把什么数据结构传递给renderThread么？

* 在android中，主线程通常是负责处理应用程序逻辑和数据，而渲染线程则负责处理渲染逻辑。主线程和渲染线程之间通常是通过一些数据结构来进行通信的。

  具体来说，主线程会创建一些描述图形信息的数据结构，例如顶点数据、纹理数据、光照和材质等。然后通过某种方式（例如EglContext）将这些数据传递给渲染线程。

  渲染线程收到这些数据之后，会使用OpenGL ES或者Vulkan等图形API来渲染图形。这些API提供了各种函数来渲染图形，例如：

  - glVertexAttribPointer()：用于将顶点数据传递给GPU。
  - glTexImage2D()：用于将纹理数据传递给GPU。
  - glDrawArrays()：用于根据顶点数据和纹理数据来绘制三维图形。

  主线程传递的数据结构的类型和细节取决于应用程序的实现，它可能是通过EglContext或者其他方式来传递的，不过一般来说都会涉及到顶点数据和纹理数据,以及其他渲染所需的数据。
  

  在android中，RenderThread是由系统自动驱动的。RenderThread是在应用程序启动时由系统自动创建的一个线程，它主要负责处理渲染逻辑。

  具体来说，当应用程序首次启动时，系统会自动创建RenderThread线程，并且在这个线程中创建EglContext来管理OpenGL ES和GPU之间的交互。

  之后，主线程会通过EglContext将顶点数据和纹理数据传递给RenderThread，RenderThread再使用OpenGL ES来渲染图形。此外RenderThread会负责处理其它渲染相关的操作，例如渲染前的清除屏幕等。这样的话就能保证主线程的流畅性，并且渲染的效率。



