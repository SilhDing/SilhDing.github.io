---
title: 'Magic Bitwise Operation: Benwick Tree'
date: 2020-08-12 23:30:21
tag:
    - "algorithm"
categories:
    - "technique"
mathjax: true
---

First, let's look at one problem on the LeetCode:

Given an integer array nums, find the sum of the elements between indices `i` and `j` (`i` ≤ `j`), inclusive. The `update(i, val)` function modifies nums by updating the element at index `i` to `val`.

Example:
```
Given nums = [1, 3, 5]

sumRange(0, 2) -> 9
update(1, 2)
sumRange(0, 2) -> 8
```

Fro this question, a simple and naive solution is to maintain a prefix sum array for this input array and then we can easily get the sum of any sub array. However, the array itself will be updated at some entries, and the prefix sum array will also be updated, which could be expensive. Is there any better solution for this problem?
