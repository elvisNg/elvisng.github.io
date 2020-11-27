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



#### 20200716

In an election, the i-th vote was cast for persons[i] at time times[i].

Now, we would like to implement the following query function: TopVotedCandidate.q(int t) will return the number of the person that was leading the election at time t.  

Votes cast at time t will count towards our query.  In the case of a tie, the most recent vote (among tied candidates) wins.

 

Example 1:

```
Input: 
["TopVotedCandidate","q","q","q","q","q","q"], [[[0,1,1,0,0,1,0],[0,5,10,15,20,25,30]],[3],[12],[25],[15],[24],[8]]

Output: [null,0,1,1,0,0,1]
```

```
Explanation: 
At time 3, the votes are [0], and 0 is leading.
At time 12, the votes are [0,1,1], and 1 is leading.
At time 25, the votes are [0,1,1,0,0,1], and 1 is leading (as ties go to the most recent vote.)
This continues for 3 more queries at time 15, 24, and 8.


Note:

1 <= persons.length = times.length <= 5000
0 <= persons[i] <= persons.length
times is a strictly increasing array with all elements in [0, 10^9].
TopVotedCandidate.q is called at most 10000 times per test case.
TopVotedCandidate.q(int t) is always called with t >= times[0].

```

Run:

```go
type TopVotedCandidate struct {
    voters    []int
    ts      []int
}

// 预处理数据 更新时间对应的person元素为最大得票者
func Constructor(persons []int, times []int) TopVotedCandidate {
    topPersons := make([]int, len(persons)+1)
    maxCount := 0
    p := persons[0]
    for i := range(persons) {
        topPersons[persons[i]]++
        //考虑平票情况
        if topPersons[persons[i]] >= maxCount {
            maxCount = topPersons[persons[i]]
            p = persons[i]
        }
        persons[i] = p
    }
    return TopVotedCandidate{persons, times}
}

// 二分查找
func (this *TopVotedCandidate) Q(t int) int {
    start := 0
    end := len(this.times)-1
    for start < end {
        mid := (start + end+1) >> 1
        if this.times[mid] > t {
            end = mid-1
        }else {
            start = mid
        }
    }
    return this.persons[start]
}

```



#### Algorithm-Project

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

