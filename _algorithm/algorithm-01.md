---
title: "回逆算法的思考"
collection: algorithm
permalink: /algorithm/algorithm-01
date: 2024-05-30
---



来看看这道题， 三数之和，问题描述：

给你一个整数数组 nums ，判断是否存在三元组 [nums[i], nums[j], nums[k]] 满足 i != j、i != k 且 j != k ，同时还满足 nums[i] + nums[j] + nums[k] == 0
leetcode:https://leetcode.cn/problems/3sum/description/

三数之和问题可以使用多种策略来解决，包括基于排序与双指针的方法。这里我将解释如何采用回逆的思想来解决这个问题来达到算法思想的迁移解决问题。


## 三数之和的回逆实现思路

在解决“三数之和”问题时，我们通常会面对一个数组，需要找出数组中所有满足三个数之和等于给定目标值的唯一三元组。为了有效地解决这个问题，
我们可以采用一种基于排序和回溯的回逆实现思路。

## 问题概述

给定一个包含 n 个整数的数组 nums 和一个目标值 target，找出 nums 中的所有三元组，使得三元组 [nums[i], nums[j], nums[k]] 满足 i != j，i != k，j != k，
且 nums[i] + nums[j] + nums[k] == target。

## 回逆实现思路

### 1. 排序

首先，对数组 nums 进行排序。排序后，相同的数字会相邻，这有助于我们跳过重复的三元组，提高算法的效率。

### 2. 回溯框架

使用回溯算法来遍历所有可能的三元组。回溯算法通过递归和状态重置（回溯）来搜索所有可能的解。

### 3. 递归函数

定义一个递归函数 getSum，它接受以下参数：
nums：原始数组。
index：当前搜索的起始位置。
target：目标值。
tmp：临时存储当前找到的三元组中的数字。
ans：存储所有满足条件的三元组的集合。

### 4. 递归终止条件

当 tmp 中的数字数量达到 3 时，表示已经找到一个满足条件的三元组，将其加入 ans 中，并返回。

### 5. 剪枝优化

在递归过程中，我们可以使用剪枝技巧来提前终止不必要的搜索，提高算法效率：
如果 index 已经超出数组范围，说明没有更多的数字可以加入 tmp，直接返回。
如果当前数字 nums[i] 大于目标值 target，或者当 tmp 中已经有两个数字时 nums[i] 不等于目标值，则跳过该数字，因为后面的数字会更大，无法满足条件。

### 6. 状态重置（回溯）

在将当前数字 nums[i] 加入 tmp 后，递归调用 getSum 函数继续搜索剩余的数字。搜索完成后，需要将 tmp 中的最后一个数字移除，以便尝试其他可能的组合。
这就是回溯的过程。

### 7. 跳过重复数字

由于数组已经排序，相邻的数字可能相同。为了避免产生重复的三元组，我们需要跳过重复的数字。具体地，在遍历过程中，如果当前数字与前一个数字相同，则跳过该数字。

### C++代码实现
```
// 递归辅助函数，用于查找和为目标值的三个数
void getSum(vector<int>& nums, int index, int target, vector<int>& tmp, vector<vector<int>>& ans) {  
    if (tmp.size() == 3) {  
        ans.emplace_back(tmp);  
        return;  
    }  
  
    if (index >= nums.size())  
        return;  
  
    for (int i = index; i < nums.size(); ++i) {  
        // 跳过重复的数字  
        if (i > index && nums[i] == nums[i -1])  
            continue;  
        // 如果当前数字已经大于目标值，或者当tmp中有两个数时，当前数字不等于目标值，则跳过  
        if (nums[i] > target || (tmp.size() == 2 && nums[i] != target))  
            continue;  
        // 将当前数字加入tmp  
        tmp.emplace_back(nums[i]);  
        // 递归查找剩余的两个数  
        getSum(nums, i + 1, target - nums[i], tmp, ans);  
        // 回溯，将当前数字从tmp中移除  
        tmp.pop_back();  
    }  
}

// 三数之和
vector<vector<int>> threeSum(vector<int>& nums) {  
    int n = nums.size();  
    if(n < 3)  
        return {{}};  
    sort(nums.begin(), nums.end());  
    vector<vector<int>> ans;  
    vector<int> tmp;  
    getSum(nums, 0, 0, tmp, ans);  
    return ans;  
} 
```

## 总结

通过排序、回溯、剪枝和跳过重复数字等技巧，我们可以有效地解决“三数之和”问题。这种回逆实现思路不仅适用于此问题，还可以扩展到其他类似的组合搜索问题中。
它非常适合这种从全局到局部，逐渐缩小范围的这种解决思想，与动态规划恰恰相反，每一步都是局部最优解来获取最终解。
在实际应用中，我们可以根据问题的具体需求进行调整。