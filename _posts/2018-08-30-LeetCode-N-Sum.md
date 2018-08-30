---
layout: post
title: N Sum问题
date: 2018-08-29 17:23:01
tags: [LeetCode]
categories: 算法
mathjax: true
---

* content
{:toc}

本文对 *LeetCode* 的 N Sum 问题做了一下总结，包括算法思路及源码。






## 1. 两数之和
> 题目链接：https://leetcode-cn.com/problems/two-sum/description/

* 使用 Hash Table 在遍历的同时将已遍历的元素索引保存，可在 O(n) 时间内解决

```python
class Solution:
    def twoSum(self, nums, target):
        """
        :type nums: List[int]
        :type target: int
        :rtype: List[int]
        """
        numIndex = dict()
        for i, num in enumerate(nums):
            need = target - num
            if need in numIndex:
                return [numIndex[need], i]
            numIndex[num] = i
        return []
```
## 15. 三数之和
> 题目链接：https://leetcode-cn.com/problems/3sum/description/

* 通过遍历结合 Two Sum 中的 Hash Table 方法会导致超时，通过率：311/313
    * 原因：本题包含重复元素，同时需要返回所有满足的结果
* 解决方案：通过首尾指针逼近的方法得到结果

```python
class Solution:
    def threeSum(self, nums):
        """
        :type nums: List[int]
        :rtype: List[List[int]]
        """
        nums.sort()
        n, res = len(nums), []
        
        for i in range(n):
            if i > 0 and nums[i] == nums[i - 1]:
                continue
            target = nums[i]*-1
            l, r = i + 1, n - 1
            while l < r:
                if nums[l] + nums[r] == target:
                    res.append([nums[i], nums[l], nums[r]])
                    l += 1
                    while l < r and nums[l] == nums[l - 1]:
                        l += 1
                    while l < r and nums[r] == nums[r - 1]:
                        r -= 1
                elif nums[l] + nums[r] < target:
                    l += 1
                else:
                    r -= 1
        return res
```
## 16. 最接近的三数之和
> 题目链接：https://leetcode-cn.com/problems/3sum-closest/description/

* 和 Three Sum 类似，使用前后指针逼近的方法
* 在指针移动的同时保证当前 result 与 target 最近

```python
class Solution:
    def threeSumClosest(self, nums, target):
        """
        :type nums: List[int]
        :type target: int
        :rtype: int
        """
        nums.sort()
        result = nums[0] + nums[1] + nums[2]
        for i in range(len(nums) - 2):
            j, k = i + 1, len(nums) - 1
            while j < k:
                sum = nums[i] + nums[j] + nums[k]
                if sum == target:
                    return sum
                
                if abs(sum - target) < abs(result - target):
                    result = sum
                
                if sum < target:
                    j += 1
                elif sum > target:
                    k -= 1
            
        return result
```
## 18. 四数之和
> 题目链接：https://leetcode-cn.com/problems/4sum/description/

* 结合 Three Sum 的算法，多一层遍历
* 使用 set 对最后的结果去重

```python
class Solution:
    def threeSum(self, nums, target):
        n, res = len(nums), []
        
        for i in range(n):
            if i > 0 and nums[i] == nums[i - 1]:
                continue
            currentTarget = target - nums[i]
            l, r = i + 1, n - 1
            while l < r:
                if nums[l] + nums[r] == currentTarget:
                    res.append([nums[i], nums[l], nums[r]])
                    l += 1
                    while l < r and nums[l] == nums[l - 1]:
                        l += 1
                    while l < r and nums[r] == nums[r - 1]:
                        r -= 1
                elif nums[l] + nums[r] < currentTarget:
                    l += 1
                else:
                    r -= 1
        return res
    
    def fourSum(self, nums, target):
        """
        :type nums: List[int]
        :type target: int
        :rtype: List[List[int]]
        """
        nums.sort()
        tmpSet, res = set(), []
        for i, num in enumerate(nums):
            for tmpRes in self.threeSum(nums[i + 1:], target - num):
                tmp = '#'.join(list(map(lambda x: str(x), tmpRes)))
                if tmp not in tmpSet:
                    tmpRes.insert(0, num)
                    res.append(tmpRes)
                tmpSet.add(tmp)
        return res
```
## 167. 两数之和 II - 输入有序数组
> 题目链接：https://leetcode-cn.com/problems/two-sum-ii-input-array-is-sorted/description/

* 通过前后两指针逼近的方法获得符合条件的 index 数组

```python
class Solution:
    def twoSum(self, numbers, target):
        """
        :type numbers: List[int]
        :type target: int
        :rtype: List[int]
        """
        l, r = 0, len(numbers) - 1
        
        while l < r:
            if numbers[l] + numbers[r] == target:
                return [l + 1, r + 1]
            elif numbers[l] + numbers[r] < target:
                l += 1
            else:
                r -= 1
        return []
```
## 454. 四数相加 II
> 题目链接：https://leetcode-cn.com/problems/4sum-ii/description/

* 分为两个双层的 for 循环
* 由于需要统计所有符合要求的元祖数量，因此重复的也要计算
* 通过一个 dict 存储前两个列表中和为某个值的数量，在之后两个列表遍历的同时更新结果

```python
from collections import defaultdict

class Solution:
    def fourSumCount(self, A, B, C, D):
        """
        :type A: List[int]
        :type B: List[int]
        :type C: List[int]
        :type D: List[int]
        :rtype: int
        """
        count, sumDict = 0, defaultdict(int)
        
        for a in A:
            for b in B:
                sumDict[a + b] += 1
        
        for c in C:
            for d in D:
                if (c + d)*-1 in sumDict:
                    count += sumDict[(c + d)*-1]
        return count
```
## 653. 两数之和 IV - 输入 BST
> 题目链接：https://leetcode-cn.com/problems/two-sum-iv-input-is-a-bst/description/

* 首先将二叉树转换为序列，采用中序遍历
* 由于 BST 的中序遍历结果为有序数组，因此结合有序的 Two Sum 解决

```python
class Solution:
    
    def twoSum(self, numbers, target):
        l, r = 0, len(numbers) - 1
        
        while l < r:
            if numbers[l] + numbers[r] == target:
                return True
            elif numbers[l] + numbers[r] < target:
                l += 1
            else:
                r -= 1
        return False
    
    def middleTraversal(self, root, numbers):
        if root:
            self.middleTraversal(root.left, numbers)
            numbers.append(root.val)
            self.middleTraversal(root.right, numbers)
    
    def findTarget(self, root, k):
        """
        :type root: TreeNode
        :type k: int
        :rtype: bool
        """
        numbers = []
        self.middleTraversal(root, numbers)
        return self.twoSum(numbers, k)
```

> *我的 LeetCode-cn 主页：https://leetcode-cn.com/lca1826/*