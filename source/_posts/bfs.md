---
title: 广度优先 
excerpt: 广度优先算法和最短路径 
date: 2021-07-27 14:31:36 
categories:
- 算法 
tags:
- 广度优先
- 算法
- 面试

---

### 广度优先

BFS 相对 DFS 的最主要的区别是：BFS找到的路径一定是最短的，但代价就是空间复杂度比 DFS 大很多

使用BFS可以求出最短路径

### 模版

```java
public class Solution {
    public int bfs(Node node) {
        Queue<Node> queue = new LinkedList<>();
        queue.add(node);
        int depth = 0; // 最短路径
        while (!queue.isEmpty()) {
            depth++;
            int size = queue.size();
            for (int i = 0; i < size; i++) {
                node = queue.poll();
                // 相邻节点加入队列
                if (// 某些条件) {
                    queue.add();
                }
            }
        }
}
```
[Leetcode 二叉树的最小深度](https://leetcode-cn.com/problems/minimum-depth-of-binary-tree/)

这道题就是典型的求最短路径，使用BFS解

```java
class Solution {
    public int minDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        Queue<TreeNode> queue = new LinkedList<>();
        queue.add(root);
        // 定义最短路径
        int depth = 0;
        while (!queue.isEmpty()) {
            depth++;
            int size = queue.size();
            for (int i = 0; i < size; i++) {
                TreeNode node = queue.poll();
                // 找到了叶子节点
                if (node.left == null && node.right == null) {
                    return depth;
                }
                if (node.left != null) {
                    queue.add(node.left);
                }
                if (node.right != null) {
                    queue.add(node.right);
                }
            }
        }
        return depth;
    }
}
```

[Leetcode 腐烂的橘子](https://leetcode-cn.com/problems/rotting-oranges/)
这道题采用了多源出发点，首先需要找到所有腐烂的橘子作为出发点，然后使用BFS找到最短的路径，
如果遍历结束仍然有新鲜的橘子，则返回-1，否则返回最短路径

注意终止条件需要加上新鲜橘子为0的情况，为0表示所有橘子都变为腐烂，已经找到了最短路径

```java
class Solution {
    public int orangesRotting(int[][] grid) {
        Queue<int[]> queue = new LinkedList<>();
        int count = 0; // 新鲜橘子的个数
        int M = grid.length;
        int N = grid[0].length;
        for (int i = 0; i < M; i++) {
            for (int j = 0; j < N; j++) {
                if (grid[i][j] == 1) {
                    count++;
                } else if (grid[i][j] == 2) {
                    queue.add(new int[]{i, j});
                }
            }
        }
        int depth = 0; // 表示腐烂的轮数，或者分钟数
        // 新鲜橘子个数为0，则不需要再遍历了
        while (count > 0 && !queue.isEmpty()) {
            depth++;
            int size = queue.size();
            for (int i = 0; i < size; i++) {
                int[] rotten = queue.poll();
                int r = rotten[0];
                int c = rotten[1];
                if (r > 0 && grid[r-1][c] == 1) {
                    grid[r-1][c] = 2;
                    count--;
                    queue.add(new int[]{r-1, c});
                }
                if (r + 1 < M && grid[r+1][c] == 1) {
                    grid[r+1][c] = 2;
                    count--;
                    queue.add(new int[]{r+1, c});
                }
                if (c > 0 && grid[r][c-1] == 1) {
                    grid[r][c-1] = 2;
                    count--;
                    queue.add(new int[]{r, c-1});
                }
                if (c + 1 < N && grid[r][c+1] == 1) {
                    grid[r][c+1] = 2;
                    count--;
                    queue.add(new int[]{r, c+1});
                }
            }
        }
        if (count != 0) {
            return -1;
        }
        return depth;
    }
}
```
