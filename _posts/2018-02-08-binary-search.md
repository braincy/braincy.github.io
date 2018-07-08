---
layout: post
title: 二分法及其应用
date: 2018-02-08 09:48:44
tags: 二分法
categories: 算法
mathjax: true
---

* content
{:toc}

本文对 *二分法* 及其变体进行介绍并用程序实现。




## 算法介绍
### 典型的二分法
#### 基本思想
* 假设数据是按升序排序的，对于给定值 $key$，从序列的中间位置 $k$ 开始比较，如果当前位置 $arr[k]$ 的值等于 $key$，则查找成功。
* 若 $key$ 小于当前位置值 $arr[k]$，则在数列的前半段 $arr[low,mid-1]$ 中查找。
* 若 $key$ 大于当前位置值 $arr[k]$，则在数列的后半段 $arr[mid+1,high]$ 中继续查找。

*直到找到为止，时间复杂度: $O(logN)$。*

#### 算法实现
```C++
int search(int *arr, int n, int key)
{
    int left = 0, right = n - 1;
    while(left <= right) {//慎重截止条件，根据指针移动条件来看，这里需要将数组判断到空为止
        int mid = left + ((right - left) >> 1);//防止溢出
        if (arr[mid] == key)//找到了
            return mid; 
        else if(arr[mid] > key) 
            right = mid - 1;//给定值key一定在左边，并且不包括当前这个中间值
        else 
            left = mid + 1;//给定值key一定在右边，并且不包括当前这个中间值
    }
    return -1;
}
```
### 二分法变种
数组中的数据可能重复，要求返回匹配的数据的最小（或最大）的下标；更进一步，需要找出数组中第一个大于 $key$ 的元素（也就是最小的大于 $key$ 的元素的）下标，等等。

#### 变种a：找出第一个与key相等的元素的位置
#### 算法实现
```C++
int searchFirstEqual(int *arr, int n, int key)
{
    int left = 0, right = n - 1;
    while(left < right) {
        int mid = left + ((right - left) >> 1);
        if(arr[mid] < key)
            left = mid + 1;
        else
            right = mid;
    }
    if(arr[right] == key) 
        return right;
    return -1;
}
```

#### 变种b：找出最后一个与key相等的元素的位置
##### 算法实现
```C++
int searchLastEqual(int *arr, int n, int key) {
    int left = 0, right = n - 1;
    while(left < right) {
        int mid = left + ((right - left + 1) >> 1);
        if(arr[mid] > key)
            right = mid - 1;
        else
            left = mid;
    }
    if(arr[left] == key) 
        return left;
    return -1;
}
```
#### 变种c：查找第一个大于Key的元素的位置
##### 算法实现
```C++
int searchFirstLarger(int *arr, int n, int key) {
    int left = 0, right = n - 1;
    while(left < right) {
        int mid = left + ((right - left) >> 1);
        if(arr[mid] <= key) 
            left = mid + 1;
        else
            right = mid;
    }
    if(arr[right] > key)
        return right;
    return -1;
}
```
#### 变种d：查找最后一个小于Key的元素的位置
##### 算法实现
```C++
int searchLastSmaller(int *arr, int n, int key) {
    int left = 0, right = n - 1;
    while(left < right) {
        int mid = left + ((right - left + 1) >> 1);
        if(arr[mid] >= key) 
             right = mid - 1;
        else
             left = mid;
    }
    if(arr[left] < key)
        return left;
    return -1;
}
```
## 算法应用
### Leetcode例题
#### [35. Search Insert Position](https://leetcode.com/problems/search-insert-position/description/)
```C++
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1;
        while(left <= right) {
            int mid = left + ((right - left) >> 1);
            if(nums[mid] == target)
                return mid;
            else if(nums[mid] > target)
                right = mid - 1;
            else
                left = mid + 1;
        }
        return left;
    }
};
```
#### [162. Find Peak Element](https://leetcode.com/problems/find-peak-element/description/)

*注：题目说了相邻元素不会相等，这个条件很重要。*

* $nums[mid] < nums[mid + 1]$，说明 $mid$ 与后一个位置形成递增区间，则 $mid$ 后面一定存在峰且当前 $mid$ 一定不是峰，则 $left=mid+1$。
* $nums[mid] > nums[mid + 1]$，说明 $mid$ 与后一个位置形成递减区间，则当前位置 $mid$ 就有可能是峰（也可能在其前面），则 $right=mid$。

```C++
class Solution {
public:
    int findPeakElement(vector<int>& nums) {
        int left = 0, right = nums.size() - 1;
        while(left < right) {
            int mid = left + ((right - left) >> 1);
            if(nums[mid] < nums[mid + 1])
                left = mid + 1;
            else
                right = mid;   
        }
        return left;
    }
};
```
#### [278. First Bad Version](https://leetcode.com/problems/first-bad-version/description/)
* 该题可抽象为`找出第一个与key相等的元素的位置`，即变种 $a$。

```C++
// Forward declaration of isBadVersion API.
bool isBadVersion(int version);

class Solution {
public:
    int firstBadVersion(int n) {
        int left = 1, right = n;
        while(left < right) {
            int mid = left + ((right - left) >> 1);
            if(isBadVersion(mid))
                right = mid;
            else
                left = mid + 1;
        }
        return right;
    }
};
```
