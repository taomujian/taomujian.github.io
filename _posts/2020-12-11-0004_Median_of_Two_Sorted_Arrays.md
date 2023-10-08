---
layout: post
title: "0004_Median_of_Two_Sorted_Arrays"
subtitle: '0004_Median_of_Two_Sorted_Arrays'
author: "taomujian"
header-style: text
tags:
  - leetcode
---




## 题目

> Given two sorted arrays nums1 and nums2 of size m and n respectively, return the median of the two sorted arrays.

>  给2个有序数组,把二个数组组成新的有序数组,并计算这个新数组中间二个元素的平均值,但其实并没有把这二个数组组合成新数组,只是抽象的想象成为一个新数组

## 思路

> 在二个数组中分别选择出两个元素,这两个元素分别是各自数组中间的元素以及相邻后面的元素,比如分别为L1,R1,L2,R2. L1是第一个数组选择的第一个元素,R1是L1相邻后面的元素.L2是第2个数组选择的第二个元素,R2是L2相邻后面的元素.通过比较L1和R2、L2和R1的大小,分别循环重新选择L1,R1,L2,R2即可

## 代码

```python
#!/usr/bin/env python3

import sys

class Solution(object):

    def findMedianSortedArrays(self, nums1, nums2):
        '''
        给2个有序数组,把二个数组组成新的有序数组,并计算这个新数组中间二个元素的平均值,但其实并没有把这二个数组组合成新数组,只是抽象的想象成为一个新数组

        :param list nums1: 数组1
        :param list nums2: 数组2

        :return str head.next: 新数组中间二个元素的平均值
        '''

        # 确保nums1是长度最短的数组
        ls1, ls2 = len(nums1), len(nums2)
        if ls1 < ls2:
            return self.findMedianSortedArrays(nums2, nums1)
        
        l, r = 0, ls2 * 2
        while l <= r:
            mid2 = (l + r) >> 1          # 第二个数组在新数组开始的位置      
            mid1 = ls1 + ls2 - mid2      # 第一个数组在新数组开始的位置

            # 在二个数组中分别选择出两个元素,这两个元素分别是各自数组中间的元素以及相邻后面的元素, L1, R1分别是第一个数组所选择的两个元素, L2, R2分别是第二个数组所选择的两个元素 
            L1 = -sys.maxsize - 1 if mid1 == 0 else nums1[(mid1 - 1) >> 1]  
            L2 = -sys.maxsize - 1 if mid2 == 0 else nums2[(mid2 - 1) >> 1]
            R1 = sys.maxsize if mid1 == 2 * ls1 else nums1[mid1 >> 1]
            R2 = sys.maxsize if mid2 == 2 * ls2 else nums2[mid2 >> 1]

            # 如果第一个数组选择的第一个元素大于第二个数组选择的第二个元素,则mid1向前移一个位置, mid2向后移一个位置
            if L1 > R2:
                l = mid2 + 1
            
            # 如果第二个数组选择的第一个元素大于第一个数组选择的第二个元素,则mid1向后移一个位置, mid2向前移一个位置
            elif L2 > R1:
                r = mid2 - 1

            else:
                return (max(L1, L2) + min(R1, R2)) / 2.0   # max(L1, L2), min(R1, R2)可看成是新数组的中间的两个元素,计算出结果并返回


if __name__ == '__main__':
    # begin
    s = Solution()
    print(s.findMedianSortedArrays([0], [1,3]))
```