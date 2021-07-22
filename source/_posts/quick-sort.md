---
title: 快速排序
date: 2021-07-22 13:12:10
excerpt: 快速排序和快速选择的算法介绍，以及常见面试题
categories:
- 算法
tags: 
- 算法
- 面试
- 快速排序
---

### 快速排序
经典的快速排序使用了分治的思想，平均复杂度为O(nlgn)，最坏的情况会退化为O(n^2)

快排的核心思想就是，找到一个点，先把比它小的排在左边，比它大的排在右边，然后一直递归排下去

[Leetcode 912](https://leetcode-cn.com/problems/sort-an-array/)

```java
class Solution {
    public int[] sortArray(int[] nums) {
        partition(nums, 0, nums.length - 1);
        return nums;
    }
    private void partition(int[] nums, int left, int right) {
        if (left >= right) return;
        int pivot = nums[left]; // 比较值
        int j = left; // 分界点
        for (int i = left + 1; i <= right; i++) {
            if (nums[i] < pivot) {
                j++;
                swap(nums, i, j);
            }
        }
        swap(nums, left, j);
        partition(nums, j + 1, right);
        partition(nums, left, j - 1);
    }

    private void swap(int[] nums, int i, int j) {
        int tmp = nums[j];
        nums[j] = nums[i];
        nums[i] = tmp;
    }
}
```

### 加上随机因子的快速排序

为了避免出现最坏的情况O(n^2)，我们可以引入随机因子，即在每一次分治的步骤中都加上随机的**pivot**，并交换位置

```java
class Solution {
    private void partition(int[] nums, int left, int right) {
        if (left >= right) return;
        // 加上随机因子，ran表示的是随机的位置
        int ran = random.nextInt(right - left + 1) + left;
        swap(nums, left, ran);
        int pivot = nums[left]; // 比较值
        int j = left; // 分界点
        for (int i = left + 1; i <= right; i++) {
            if (nums[i] < pivot) {
                j++;
                swap(nums, i, j);
            }
        }
        swap(nums, left, j);
        partition(nums, j + 1, right);
        partition(nums, left, j - 1);
    }
}
```

### 快速选择

和快速排序类似，快速选择不需要全部排完，只需要对K个元素排好序，然后取K个元素即可

[Leetcode 最小的K个数](https://leetcode-cn.com/problems/smallest-k-lcci/)

```java
class Solution {
    public int[] smallestK(int[] arr, int k) {
        partition(arr, 0, arr.length - 1, k);
        return Arrays.copyOfRange(arr, 0, k);
    }
    
    private void partition(int[] arr, int l, int r, int k) {
        if (l >= r) return;
        int j = l;
        int pivot = arr[l];
        for (int i = j + 1; i <= r; i++) {
            if (arr[i] < pivot) {
                j++;
                swap(arr, i, j);
            }
        }
        swap(arr, l, j);
        if (j == k) {
            return;
        } else if (j < k) {
            partition(arr, j + 1, r, k);
        } else {
            partition(arr, l, j - 1, k);
        }
    }

    private void swap(int[] arr, int i, int j) {
        int tmp = arr[j];
        arr[j] = arr[i];
        arr[i] = tmp;
    }
}
```

[Leetcode 最接近原点的 K 个点](https://leetcode-cn.com/problems/k-closest-points-to-origin/)
这道题把比较的值替换为到原点的距离，其他的解题思路是一样的
```java
class Solution {
    public int[][] kClosest(int[][] points, int k) {
        if (points.length <= k) {
            return points;
        }
        quickSelect(points, 0, points.length - 1, k);
        return Arrays.copyOfRange(points, 0, k);
    }
    private void quickSelect(int[][] points, int start, int end, int k) {
        int pivot = distance(points[start]);
        int j = start;
        for (int i = start + 1; i <= end; i++) {
            if (distance(points[i]) < pivot) {
                j++;
                swap(points, i, j);
            }
        }
        swap(points, start, j);
        if (j == k) {
            return;
        } else if (j < k) {
            quickSelect(points, j + 1, end, k);
        } else {
            quickSelect(points, start, j - 1, k);
        }
    }

    private int distance(int[] point) {
        return point[0] * point[0] + point[1] * point[1];
    }

    private void swap(int[][] points, int i, int j) {
        int[] tmp = points[i];
        points[i] = points[j];
        points[j] = tmp;
    }
}
```
