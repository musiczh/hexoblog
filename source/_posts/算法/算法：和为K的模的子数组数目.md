---
title: 算法：和可被k整除的子数组数目 #标题
date: 2020/5/27 17:30:00 #建立日期
updated: 2020/5/27 17:30:00 #更新日期
comments: true #开启评论
tags:  #标签
 - 算法 
 - 前缀和

categories:  #分类
 - 算法

---

### 题目：和可被k整除的子数组数目

给定一个整数数组 A，返回其中元素之和可被 K 整除的（连续、非空）子数组的数目。

```
示例：

输入：A = [4,5,0,-2,-3,1], K = 5
输出：7
解释：
有 7 个子数组满足其元素之和可被 K = 5 整除：
[4, 5, 0, -2, -3, 1], [5], [5, 0], [5, 0, -2, -3], [0], [0, -2, -3], [-2, -3]
```

```
提示：

1 <= A.length <= 30000
-10000 <= A[i] <= 10000
2 <= K <= 10000
```

> 来源：力扣（LeetCode）
> 链接：https://leetcode-cn.com/problems/subarray-sums-divisible-by-k

### 分析

这道题的思路重点在前缀和+同余定理。

拿到这道题的时候我一直在想着一个子数组如果符合条件的话，怎样推出他周边的子数组是否符合条件？想了半天还是只有O（N^2）的思路。这里就要用到这个同余定理。

同余定理：当一个数组，当从0到x-1位置所有数加起来取K的模，和0到Y位置所有数加起来取K的模相等，那么X到Y所有数加起来满足被K整除。这个定理很好证明，这里就不证明了。

所以我们就顺然想到了前缀和。我们可以统计该数组中所有的前缀和，那么就可以得到满足的子数组数目了。

这里的实现思路是在遍历的时候维护一个hashMap，记录每种余数的前缀和有多少个。然后当遍历到i位置的时候，假如当前位置的前缀和取K的模是 M，那么查询hashMap，M对应的数目有多少个，那么就有多少个符合条件的数组了。

注意：这里因为和可能为负数，取模后可能出现负数，所以要进行矫正。不是简单的取绝对值，因为假如模是6,-2实际上等于4而不是2。

### 解答

```kotlin
fun subarraysDivByK(A: IntArray?, K: Int): Int {
    val hashMap = HashMap<Int,Int>()
    hashMap[0] = 1
    var num = 0
    var sum = 0
    for (elem in A!!){
        sum += elem
        val remainder = (sum % K+K)%K
        println(remainder)
        val indexNum = hashMap.getOrDefault(remainder,0)
        num += indexNum
        hashMap[remainder] = indexNum+1
    }
    return num
}
```



### 复杂度分析

##### 时间复杂度

遍历一次数组

- 时间复杂度：O(n)

##### 空件复杂度

维护一个哈希表。长度是Max（K，A.size）

- O(Max（K，A.size）)