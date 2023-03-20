# leetcode

## 双指针

### leetcode [11. 盛最多水的容器](https://leetcode.cn/problems/container-with-most-water/)

给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。

找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。

返回容器可以储存的最大水量。

说明：你不能倾斜容器。

<img src="https://raw.githubusercontent.com/Awille/MyBlog/main/img/2022/09/upgit_20220918_1663436862.png" alt="upgit_20220918_1663436862.png" style="zoom: 67%;" />

解法：双指针
从两头开始，位置矮的指针先移动，期间不断更新最大容量。



## 二叉树

### [剑指 Offer 28. 对称的二叉树](https://leetcode.cn/problems/dui-cheng-de-er-cha-shu-lcof/)

请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。

例如，二叉树 [1,2,2,3,4,4,3] 是对称的。

   1
   / \
  2   2
 / \ / \
3  4 4  3
但是下面这个 [1,2,2,null,3,null,3] 则不是镜像对称的:

   1
   / \
  2   2
   \   \
   3    3

```java
class Solution {
    public boolean isSymmetric(TreeNode root) {
        if (root == null) {
            return true;
        }
        return dfs(root.left, root.right);
    }

    public boolean dfs(TreeNode left, TreeNode right) {
        if (left == null && right == null) {
            return true;
        }
        if (left != null && right != null && left.val == right.val) {
            return dfs(left.left, right.right) && dfs(left.right, right.left);
        } 
        return false;
    }
}
```



### [剑指 Offer 32 - II. 从上到下打印二叉树 II](https://leetcode.cn/problems/cong-shang-dao-xia-da-yin-er-cha-shu-ii-lcof/)

[102. 二叉树的层序遍历](https://leetcode.cn/problems/binary-tree-level-order-traversal/)

从上到下按层打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。

解法BFS，主要是记一下queue的几个函数操作，offer、poll

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> result = new ArrayList<List<Integer>>();
        if (root == null) {
            return result;
        }
        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            List<Integer> level = new ArrayList<Integer>();
            int levelSize = queue.size();
            for (int i = 0 ; i < levelSize; i ++) {
                TreeNode node = queue.poll();
                level.add(node.val);
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }
            result.add(level);
        }
        return result;
    }
}
```



### [103. 二叉树的锯齿形层序遍历](https://leetcode.cn/problems/binary-tree-zigzag-level-order-traversal/)

跟上面一样，只是在每一层添加list时 判断要不要reverse一下。

### [207. 课程表](https://leetcode.cn/problems/course-schedule/) -  拓扑排序

你这个学期必须选修 numCourses 门课程，记为 0 到 numCourses - 1 。

在选修某些课程之前需要一些先修课程。 先修课程按数组 prerequisites 给出，其中 prerequisites[i] = [ai, bi] ，表示如果要学习课程 ai 则 必须 先学习课程  bi 。

例如，先修课程对 [0, 1] 表示：想要学习课程 0 ，你需要先完成课程 1 。
请你判断是否可能完成所有课程的学习？如果可以，返回 true ；否则，返回 false 。

思路：以prerequisites 构造有向图，然后寻找有向图中的拓扑排序(即寻找没有环的解)

```java
class Solution {
    private List<List<Integer>> edges;
    boolean valid = true;
    //访问记录数据 未访问0 访问中1 访问结束2
    int[] visited;
    public boolean canFinish(int numCourses, int[][] prerequisites) {
        edges = new ArrayList<List<Integer>>();
        visited = new int[numCourses];
        for (int i = 0; i < numCourses; i ++) {
            edges.add(i, new ArrayList<Integer>());
        }
        //构造边 注意指向
        for (int i = 0; i < prerequisites.length; i ++) {
            for (int j = 0; j < prerequisites[i].length; j ++) {
                edges.get(prerequisites[i][1]).add(prerequisites[i][0]);
            }
        }
        //时刻注意剪枝 如果已经invalid就不要再进行dfs了
        for (int i = 0; i < numCourses; i ++) {
            if (visited[i] == 0 && valid) {
                dfs(i);
            }
        }
        return valid;
    }

    private void dfs(int i) {
        //访问开始
        visited[i] = 1;
        for (int j = 0; j < edges.get(i).size(); j ++) {
            if (visited[edges.get(i).get(j)] == 1) {
                valid = false;
                return;
            } else if (visited[edges.get(i).get(j)] == 0 && valid) {
                dfs(edges.get(i).get(j));
            }
        }
        //访问结束
        visited[i] = 2;
    }
}
```



## 滑动窗口

### [53、最大子数组和](https://leetcode.cn/problems/maximum-subarray/)

最简单直接的解法：

两重for循环，复杂度为O(n^2)

更优化的解法：动态规划

比如数组[1,2, -1, -3]

* 子问题1：以1结尾的最大子数组是多少？
* 子问题2：以2结尾的最大子数组是多少？
* 子问题3：以-1结尾的最大子数组是多少？
* 子问题4：以-3结尾的最大子数组是多少？

这些子问题都联系的，比如 子问题2可以转化为子问题1的值加上2孰大孰小

我们建立数组dp[i], 代表以数组i位置结尾的最大子数组值，那么可以得到

dp[i+1] = dp[i] + value[i + 1] >= values[i+1] ? dp[i] + valuse[i+1] : valuse[i+1]

由此，我们可以从子问题1推到子问题2，再从子问题推到到子问题3

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int length = nums.length;
        int[] dp = new int[length];
        dp[0] = nums[0];
        int max = dp[0];
        for (int i = 1; i < length; i ++) {
            int sum = dp[i-1] + nums[i];
            if (sum >= nums[i]) {
                dp[i] = sum;
            } else {
                dp[i] = nums[i];
            }
            max = Math.max(max, dp[i]);
        }
        return max;
    }
}
```



### [121.买卖股票的最佳时机](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/)

假设在第i天卖出，则在i天前的历史最低价买入最合适

```java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices.length == 0 || prices.length == 1) {
            return 0;
        }
        int minPrice = prices[0];
        int maxProfit = 0;
        for (int i = 1; i < prices.length; i ++) {
            maxProfit = Math.max(maxProfit, prices[i] - minPrice);
            minPrice = Math.min(minPrice, prices[i]);
        }
        return maxProfit;
    }
}
```



### [3.无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/)

滑动窗口，从0开始进行窗口扩展，当向右延伸的元素包含窗口中的值时，将窗口左侧移动至不包含该元素之处，并在窗口滑动过程中不断记录最大的大小。

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> map = new HashMap<>();
        int maxLength = 0;
        int left = 0, right = 0;
        while(right < s.length()) {
            Integer charIndex = map.get(s.charAt(right));
            if (charIndex != null) {
                //滑动窗口左侧移动，并删除被移动的元素
                for(int i = left; i < charIndex + 1; i ++) {
                    map.remove(s.charAt(i));
                }
                left = charIndex + 1;
            }
            map.put(s.charAt(right), right);
            maxLength = Math.max(right - left + 1, maxLength);
            right ++;
        }
        return maxLength;
    }
}


//更优化的解法： 默认以i结尾的字串，每次更新left的值， map这里的作用只是标记坐标，不做最长字串的记录
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> map = new HashMap<>();
        int maxLength = 0;
        int left = 0;
        for (int i = 0; i < s.length(); i ++) {
            //以i位置结尾的最长子串
            Integer index = map.get(s.charAt(i));
            if (index != null) {
                left = Math.max(left, index + 1);
            } 
            map.put(s.charAt(i), i);
            maxLength = Math.max(maxLength, i - left + 1);
        }
        return maxLength;
    }
}
```



### [209. 长度最小的子数组](https://leetcode.cn/problems/minimum-size-subarray-sum/)

双指针加滑动窗口：

```java
class Solution {
    public int minSubArrayLen(int target, int[] nums) {
        if (nums.length == 1) {
            if (nums[0] >= target) {
                return 1;
            } else {
                return 0;
            }
        }
        int sum = 0;
        int minLength = nums.length;
        int left = 0, right = 0;
        while (right < nums.length) {
            sum += nums[right];
            while((sum - nums[left] >= target) && left < right) {
                sum -= nums[left];
                left ++;
            }
            if (sum >= target) {
                minLength = Math.min(minLength, right - left + 1);
            }
            right ++;
        }
        if (sum >= target) {
            return minLength;
        }
        return 0;
    }
}
```





### [239. 滑动窗口最大值](https://leetcode.cn/problems/sliding-window-maximum/)



### [567. 字符串的排列](https://leetcode.cn/problems/permutation-in-string/)

