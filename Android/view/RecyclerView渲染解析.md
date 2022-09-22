# RecyclerView渲染解析

## 1、prefetch逻辑

LinearLayoutManager控制逻辑：

假设0,1,2三个卡片，全部是全屏的。 初始的时候，0, 和1的 onCreateViewHolder跟 onBindViewHolder都会执行，RecyclerView本身是一个ViewGroup,  0是可见的，被attachViewToParent绑定到ViewGroup上，0的绘制三联会被执行。

接着从0往1滑动，滑动过程中，2的onCreateViewHolder onBindViewHolder会被执行(在没有View复用的场景下)， 由于1可见，1会被attachViewToParent绑定到ViewGroup上。

androidx.recyclerview.widget.RecyclerView.LayoutManager#addView(android.view.View)

androidx.recyclerview.widget.ChildHelper#addView(android.view.View, int, boolean)

androidx.recyclerview.widget.ChildHelper.Callback#attachViewToParent

RecyclerTextActivity|MyTextView