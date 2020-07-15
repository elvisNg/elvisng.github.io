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





#### 20200715

Given an array nums of n integers, are there elements a, b, c in nums such that a + b + c = 0? Find all unique triplets in the array which gives the sum of zero.

Note:

The solution set must not contain duplicate triplets.

Example:

```
Given array nums = [-1, 0, 1, 2, -1, -4],

A solution set is:
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```

Run:

```Go
func threeSum(nums []int) [][]int {
    if nums == nil || len(nums) < 3 {
        return nil
    }
    sort.Ints(nums)
    len := len(nums)
    result := [][]int{}
    for i := 0 ; i < len-2 ; i++ {
        // sort array if nums[i]>0 sum always >0 break 
        if nums[i] > 0 {
            break
        }
        //base duplicate removal
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        j := i + 1
        k := len - 1
        for {
            if j >= k {
                break
            }
            sum := nums[i] + nums[j] + nums[k]
            if sum > 0 {
                k --
                continue
            }
            if sum < 0 {
                j ++
                continue
            }
            if sum == 0{
                result = append(result, []int{nums[i], nums[j], nums[k]})
                for {
                    //left pointer duplicate removal
                    if j < k && nums[j] == nums[j+1] {
                        j++
                    } else {
                        break
                    }
                }
                for {
                    //right pointer duplicate removal
                    if  j < k && nums[k] == nums[k-1] {
                        k--
                    } else {
                        break
                    }
                }
                j++
                k--
            }
        }
    }
    return result
}
```





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

