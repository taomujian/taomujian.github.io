---
layout: post
title: "0006_ZigZag_Conversion"
subtitle: '0006_ZigZag_Conversion'
author: "taomujian"
header-style: text
tags:
  - leetcode
---


## 题目

> The string "PAYPALISHIRING" is written in a zigzag pattern on a given number of rows like this: (you may want to display this pattern in a fixed font for better legibility)

> 给定一个字符串,按照特定的行数对这个字符串进行竖Z型输出

## 思路

> 保存结果的数组初始值只需定义numRows个空元素就行,依次开始对s进行取值进行保存,每满足一个循环周期后就开始逆方向循环保存值,保存结果的数组的每个值代表了哪些字符处于同一行.

## 代码

```python
#!/usr/bin/env python3

class Solution(object):

    def convert(self, s, numRows):

        '''
        把一个字符串按照一定行数进行竖Z字型排列,并输出.
        保存结果的数组初始值只需定义numRows个空元素就行,依次开始对s进行取值进行保存,
        每满足一个循环周期后就开始逆方向循环保存值,保存结果的数组的每个值代表了哪些字符处于同一行.

        :param str z: 字符串
        :param int numRows: 竖Z字型的行数

        '''

        # 如果数组长度为1,直接返回这个数组
        if numRows == 1:
            return s

        # 定义循环周期,即每循环一个周期,增长方向就要改变一次,用负数位置来进行数组方向的逆方向循环保存值
        p = 2 * (numRows - 1)

        # 定义一个空数组,用来存放结果
        result = [""] * numRows

        # 开始循环
        for i in range(len(s)):
            floor = i % p
            # r如果元素位置已经到了该逆向赋值的情况下,调转方向
            if floor >= p//2:
                floor = p - floor

            result[floor] += s[i]

        return "".join(result)


if __name__ == '__main__':
    s = Solution()
    print(s.convert("PAYPALISHIRING", 3))
```