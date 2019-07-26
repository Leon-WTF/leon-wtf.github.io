---
title: "LeetCode-数组中的第K个最大元素"
category: Algorithm
tag: algorithm
---
# 题目 #
在未排序的数组中找到第 k 个最大的元素。请注意，你需要找的是数组排序后的第 k 个最大的元素，而不是第 k 个不同的元素。

示例 1:

输入: [3,2,1,5,6,4] 和 k = 2
输出: 5
示例 2:

输入: [3,2,3,1,2,4,5,5,6] 和 k = 4
输出: 4
说明:

你可以假设 k 总是有效的，且 1 ≤ k ≤ 数组的长度。

> https://leetcode-cn.com/problems/kth-largest-element-in-an-array

# 方案 #
采用[QuickSort](https://leon-wtf.github.io/algorithm/2019/07/26/sort/#more)的思想来解决:
```java
import java.util.Random;
class Solution {
    private int[] nums;
    private int partition(int l, int r, int pivot_index) {
        int pivot = this.nums[pivot_index];
        if (pivot_index != (r - 1)) {
            this.nums[pivot_index] = this.nums[r -1];
            this.nums[r - 1] = pivot;
        }
        int j = l;
        for (int i = l; i < r; i++) {
            if (this.nums[i] > pivot) {
                if (i != j) {
                    int temp = this.nums[i];
                    this.nums[i] = this.nums[j];
                    this.nums[j] = temp;
                }
                j++;
            }
        }
        this.nums[r - 1] = this.nums[j];
        this.nums[j] = pivot;
        return j;
    } 
    private int quickSelect(int low, int high, int k) {
        if (low == high) {
            return this.nums[low];
        }
        Random random_num = new Random();
        int pivot_index = low + random_num.nextInt(high - low); 
        int p = partition(low, high, pivot_index);
        if ((p + 1) == k) {
            return nums[p];
        } else if ((p + 1) > k) {
            return quickSelect(low, p, k);
        } else {
            return quickSelect(p + 1, high, k);
        }        
    }    
    public int findKthLargest(int[] nums, int k) {
        this.nums = nums;
        return quickSelect(0, nums.length, k);
    }
}
```
从实验结果看，用随机的方式选择pivot比选择第一个/最后一个元素作为pivot用时要少一个量级左右。
