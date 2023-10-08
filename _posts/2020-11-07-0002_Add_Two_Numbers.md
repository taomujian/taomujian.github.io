---
layout: post
title: "0002__Add_Two_Numbers"
subtitle: '0002__Add_Two_Numbers'
author: "taomujian"
header-style: text
tags:
  - leetcode
---




## 题目

> You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse 
> order, and each of their nodes contains a single digit. Add the two numbers and return the sum as a linked list.
> You may assume the two numbers do not contain any leading zero, except the number 0 itself.

> 有2个非空的链表,链表中没有负整数,这些数字是倒序存在链表中,每个结点存在一个数字,每次分别取这二个链表的一个结点的值进行相加,然后逆序把结果存放在另一个链表中

## 思路

> 首先python中没有链表结构,代码并不能运行,只能抽象的写出来算法.
> 每次取链表中的值,并相加就行了,不过从第二次循环开始要加上上一次循环的进位,最后还要检查最后一次相加是否进位,进位了则要再加一个结点

## 代码

```python
#!/usr/bin/env python3

class ListNode:
    def __init__(self, val):
        '''
        初始化一个单链表结点

        :param str val: 结点的值
        
        :return:
        '''

        self.val = val
        self.next = None

class Solution:

    def addTwoNumbers(self, l1, l2):
        '''
        完成2个链表的相加,并倒序输出

        :param ListNode l1: 链表结点
        :param ListNode l2: 链表结点
        
        :return ListNode head.next: 相加结果的链表
        '''

        carry = 0
        head = curr = ListNode(0) #初始化2个链表结点,存放相加后的结果,head和curr都指向同一个链表
        while l1 or l2:
            if l1:
                carry += l1.val
                l1 = l1.next
            if l2:
                carry += l2.val
                l2 = l2.next
            curr.next = ListNode(carry % 10)
            curr = curr.next
            carry = carry // 10
        
        # 如果最后一次相加进位了,则要新开一个结点进1
        if carry > 0:
            curr.next = ListNode(carry)
        return head.next

#if __name__ == "__main__":
    #solution = Solution()
    #solution.addTwoNumbers([2,4,3], [5,6,4])
```