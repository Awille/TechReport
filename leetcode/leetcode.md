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

