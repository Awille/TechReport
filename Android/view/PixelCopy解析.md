# PixelCopy解析 

***

> 探究为何PixelCopy.request() 为何能显示多个surface的截图

***

## 1、简介

```java

    /**
     * Requests a copy of the pixels from a {@link Window} to be copied into
     * a provided {@link Bitmap}.
     *
     * The contents of the source will be scaled to fit exactly inside the bitmap.
     * The pixel format of the source buffer will be converted, as part of the copy,
     * to fit the the bitmap's {@link Bitmap.Config}. The most recently queued buffer
     * in the Window's Surface will be used as the source of the copy.
     *
     * Note: This is limited to being able to copy from Window's with a non-null
     * DecorView. If {@link Window#peekDecorView()} is null this throws an
     * {@link IllegalArgumentException}. It will similarly throw an exception
     * if the DecorView has not yet acquired a backing surface. It is recommended
     * that {@link OnDrawListener} is used to ensure that at least one draw
     * has happened before trying to copy from the window, otherwise either
     * an {@link IllegalArgumentException} will be thrown or an error will
     * be returned to the {@link OnPixelCopyFinishedListener}.
     *
     * @param source The source from which to copy
     * @param dest The destination of the copy. The source will be scaled to
     * match the width, height, and format of this bitmap.
     * @param listener Callback for when the pixel copy request completes
     * @param listenerThread The callback will be invoked on this Handler when
     * the copy is finished.
     */
    public static void request(@NonNull Window source, @NonNull Bitmap dest,
            @NonNull OnPixelCopyFinishedListener listener, @NonNull Handler listenerThread) {
        request(source, null, dest, listener, listenerThread);
    }
```



## 2、渲染解析

大概流程为：

1. 从当前window中获取decorview
2. 从当前decorview中获取viewRootImpl
3. 从viewRoorImpl中获取surface
4. 调用 ThreadedRenderer.copySurfaceInto(source, srcRect, dest);

 ThreadedRenderer.copySurfaceInto：

结论是不能显示，优化的其实是用renderThread来渲染surface，只是更快的。