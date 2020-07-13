---
layout: post
title: "algorithm-project"
subtitle: '2020'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - algorithm
---



#### 20200713

You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.

You may assume the two numbers do not contain any leading zero, except the number 0 itself.

Example:

```
Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8
Explanation: 342 + 465 = 807.
```

Run:

```go
type ListNode struct {
    Val int 
    Next *ListNode
}

func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
    if l1 == nil{
        return l2
    }
    if l2 == nil{
        return l1
    }
    if l1 == nil && l2 == nil {
        return nil
    }
    sum := l1.Val + l2.Val
    nextNode := addTwoNumbers(l1.Next,l2.Next)
    if sum < 10 {
        return &ListNode{
            Val:sum,
            Next:nextNode,
        }
    }else{
        temp := &ListNode {
            Val:1,
            Next:nil,
        }
        return &ListNode{
            Val:sum-10,
            Next: addTwoNumbers(nextNode,temp),
        }
    }
}
```

