---
title: 算法：合并k个有序链表 #标题
date: 2020/5/24 16:30:00 #建立日期
updated: 2020/5/24 16:30:00 #更新日期
comments: true #开启评论
tags:  #标签
 - 算法 
 - 链表

categories:  #分类
 - 算法

---

### 题目：合并k个有序链表

合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

示例:

```
输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6
```

> 来源：力扣（LeetCode）
> 链接：https://leetcode-cn.com/problems/merge-k-sorted-lists

### 分析

要解决这个问题，首先要解决合并2个有序链表的方法，并以此作为拓展来实现合并K个有序链表。关于合并两个有序链表这里不再描述，可以去看我的相关文章。

这里提供两个思路，一是不断通过两两合并的方式把全部链表合并起来；二是每次从所有链表中选出最小的那个加到链表尾部。

思路一：这个思路很清晰，但是要注意一点，不能够一条一条合进来，要进行分治法，先两两合并后，再把剩下的两两合并。例如1234，先2+1，再3+4，最后再3+7（这里的1234指的是4个链表）。

思路二：每次从所有节点中抽取一个节点出来。这里可以使用一个优先队列来维护k个数据。每次从队列中拿走一个节点后就把该节点的下一节点放进去。这样的好处就是，优先队列可以通过效率比较高的算法排序（堆算法），比我们一个个找的效率要高，坏处就是要占用额外的空间。下面的解答我没有使用队列，直接一个个去查找，读者可以自行尝试。

### 解答

```kotlin
class ListNode(var `val`: Int) {
    var next: ListNode? = null
}
```



##### 解答一

```kotlin
fun mergeKLists(lists: Array<ListNode?>): ListNode? {
    val listArray = ArrayList<ListNode>()
    for (node in lists){
        if (node!=null) listArray.add(node)
    }
    if (listArray.isEmpty()) return null
    val listHead = ListNode(0)
    var listCurrent = listHead
    while (listArray.isNotEmpty()){
        var num = listArray[0].`val`
        var index = 0
        for (i in 1 until listArray.size){
            if (listArray[i].`val`<=num){
                num = listArray[i].`val`
                index = i
            }
        }
        listCurrent.next = listArray[index]
        listCurrent = listCurrent.next!!
        if (listArray[index].next!=null) listArray[index] = listArray[index].next!!
        else listArray.removeAt(index)
    }
    listCurrent.next = null
    return listHead.next
}
```

##### 方法二

```kotlin
fun mergeKLists(lists: Array<ListNode?>): ListNode? {
        if (lists.isEmpty()) return null
        return merge(lists,0,lists.size-1)
    }

fun merge(lists: Array<ListNode?>,s:Int, e:Int):ListNode?{
    if (e<=s) return lists[s]
    return mergeTwoLists(merge(lists, s, s+(e-s)/2),merge(lists,s+(e-s)/2+1,e))
}

fun mergeTwoLists(l1: ListNode?, l2: ListNode?): ListNode? {
        when{
            l1==null&&l2==null -> return null
            l1==null -> return l2
            l2==null -> return l1
        }
        val listNode :ListNode? = ListNode(0)
        var current = listNode
        var point1 = l1
        var point2 = l2
        while (point1!=null&&point2!=null){
            if (point1.`val`<point2.`val`){
                current?.next = point1
                point1 = point1.next
            } else {
                current?.next = point2
                point2 = point2.next
            }
            current = current?.next
        }
        if (point1==null){
            current?.next = point2
        } else {
            current?.next = point1
        }
        return listNode?.next
    }
```



### 复杂度分析

假设一共有k个链表，每个链表的最长长度是n

##### 时间复杂度

第一种方法每次要从k个链表中查找最小值，一共需要找kn次

- 时间复杂度：O(kn*k)

第二种方法递归的层数是logk，（自底向上）第一层一共分为k/2组，每组需要的时间是O(2n)。第二层一共分为k/4组，每组需要的时间是O(4n),以此类推。

- 时间复杂度：O(logk*kn)

可以看到第二种的速度会更快点，但是如果使用先序队列，那么时间复杂度就一样了。

##### 空件复杂度

第一种方法不需要任何数组空间，只需要少量变量

- 空间复杂度：O(1)

第二种方法需要用到递归，栈的深度是logk，所以是

- 空间复杂度：O(logk)