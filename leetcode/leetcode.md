# leetcode

## 双指针

### leetcode 11、盛水最多的容器

链接：https://leetcode.cn/problems/container-with-most-water

给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。

找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。

返回容器可以储存的最大水量。

说明：你不能倾斜容器。

<img src="https://raw.githubusercontent.com/Awille/MyBlog/main/img/2022/09/upgit_20220918_1663436862.png" alt="upgit_20220918_1663436862.png" style="zoom: 67%;" />

解法：双指针
从两头开始，位置矮的指针先移动，期间不断更新最大容量。



