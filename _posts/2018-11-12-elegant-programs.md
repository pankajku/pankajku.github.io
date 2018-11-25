---
layout: post
title: "Elegant Programs"
date: 2018-11-12
---
# Linked Lists
```
# Definition for singly-linked list.
# class ListNode(object):
#     def __init__(self, x):
#         self.val = x
#         self.next = None
```
## Merge Two Sorted Lists
Merge two sorted linked lists and return it as a new list. The new list should be made by splicing together the nodes of the first two lists.

Example:

Input: 1->2->4, 1->3->4<br>
Output: 1->1->2->3->4->4

### Recursive Solution
```
class Solution(object):
    def mergeTwoLists(self, l1, l2):
        """
        :type l1: ListNode
        :type l2: ListNode
        :rtype: ListNode
        """
        if l1 and l2:
            if l1.val > l2.val:
                l1, l2 = l2, l1
            l1.next = self.mergeTwoLists(l1.next, l2)
            return l1
        return l1 or l2
```

### Iterative Solution
```
class Solution(object):
    def mergeTwoLists(self, l1, l2):
        head = cn = ListNode(0)
        while l1 and l2:
            if l1.val > l2.val:
                l1, l2 = l2, l1
            cn.next = l1
            cn = cn.next
            l1 = l1.next
        cn.next = l1 or l2
        return head.next
```
## Remove Duplicates from Sorted List

Given a sorted linked list, delete all nodes that have duplicate numbers, leaving only distinct numbers from the original list.

Example 1:

Input: 1->2->3->3->4->4->5<br>
Output: 1->2->5

Example 2:

Input: 1->1->1->2->3<br>
Output: 2->3

### Solution
```
class Solution(object):
    def deleteDuplicates(self, head):
        """
        :type head: ListNode
        :rtype: ListNode
        """
        dn = pn = ListNode(0)
        dn.next = cn = head
        while cn and cn.next:
            if cn.val == cn.next.val:
                while cn.next and cn.val == cn.next.val:
                    cn = cn.next
                cn = cn.next
                pn.next = cn
            else:
                pn = cn
                cn = cn.next
        return dn.next
```
## Merge k Sorted Lists
Merge k sorted linked lists and return it as one sorted list. Analyze and describe its complexity.

Example:

Input: [1->4->5, 1->3->4, 2->6]<br>
Output: 1->1->2->3->4->4->5->6

### Simple (but inefficient) Solution
```
class Solution(object):
    def mergeKLists(self, lists):
        """
        :type lists: List[ListNode]
        :rtype: ListNode
        """
        minNode, k, minNodeIndex = None, 0, -1
        for list in lists:
            if list:
                if minNode:
                    if list.val < minNode.val:
                        minNode, minNodeIndex = list, k
                else:
                    minNode, minNodeIndex = list, k
            k += 1
        if minNode:
            lists[minNodeIndex] = minNode.next
            minNode.next = self.mergeKLists(lists)
        return minNode
```
### Efficient Solution
```
class MinHeap:
    def __init__(self, capacity = 0):
        self.elems = capacity*[None]
        self.n = 0

    def size(self):
        return self.n
    
    def add(self, tpl):
        eIdx = self.n
        self.elems[eIdx] = tpl
        self.n += 1
        pIdx = (eIdx-1)//2 if eIdx else 0

        while self.elems[eIdx][0] < self.elems[pIdx][0]:
            self.elems[eIdx], self.elems[pIdx] = self.elems[pIdx], self.elems[eIdx]
            eIdx = pIdx
            pIdx = (eIdx-1)//2 if eIdx else 0

    def extract(self):
        tpl = self.elems[0]
        self.elems[0] = self.elems[self.n-1]
        self.n -= 1
        pIdx = 0
        lcIdx, rcIdx = 2*pIdx+1, 2*pIdx+2
        while lcIdx < self.n:
            minIdx = lcIdx
            if rcIdx < self.n and self.elems[rcIdx][0] < self.elems[lcIdx][0]:
                minIdx = rcIdx
            if self.elems[minIdx][0] < self.elems[pIdx][0]:
                self.elems[minIdx], self.elems[pIdx] = self.elems[pIdx], self.elems[minIdx]
                pIdx = minIdx
                lcIdx, rcIdx = 2*pIdx+1, 2*pIdx+2
            else:
                break
        return tpl
                

class Solution(object):
    def mergeKLists(self, lists):
        """
        :type lists: List[ListNode]
        :rtype: ListNode
        """
        mh = MinHeap(len(lists))
        for list in lists:
            if list:
                mh.add((list.val, list))

        dummy = curn = ListNode(0)
        while mh.size() > 0:
            curn.next = mh.extract()[1]
            curn = curn.next
            if curn.next:
                mh.add((curn.next.val, curn.next))

        return dummy.next
```
