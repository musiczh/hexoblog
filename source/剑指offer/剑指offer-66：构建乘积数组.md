## 题目

给定一个数组 A[0,1,…,n-1]，请构建一个数组 B[0,1,…,n-1]，其中 B 中的元素 B[i]=A[0]×A[1]×…×A[i-1]×A[i+1]×…×A[n-1]。不能使用除法。

>**示例:**
>
>```
>输入: [1,2,3,4,5]
>输出: [120,60,40,30,24]
>```

## 思路历程

很明显，他要求的是A数组中，除去当前项，其他所有项的乘积。如果可以使用除法，那么直接求出全部乘积，然后依次除去当前数字即可。

首先想到的就是暴力解法：直接把每一项都乘积一次。如B[0] = 2 * 3 * 4 * 5,直接相乘得出结果。空间复杂度是O(1)，时间复杂度是O(n^2)，因为一共有n项，每一项需要n-1次乘积。这显然太暴力了。

暴力时间复杂度太高的原因是重复计算。例如计算2x3x4x5，这里其实已经计算了一次3x4x5，如果下一次计算还是需要再重新计算一次。所以思路就是把这些计算记录起来，后面的利用前面的计算结果，利用空间换时间。

思路是记录第0个到[1,2,,,n-1]个元素的乘积，然后再记录第[n-1]个元素到[0,1,,,n-2]个元素的乘积。最后给B数组赋值的时候，只需要取出前后两个数字乘积即可。代码如下：

```java
class Solution {
    public int[] constructArr(int[] a) {
        // 使用+2的原因是这样可以不用考虑边界问题
        int[][] dp = new int[2][a.length+2];
        dp[0][0] = 1;
        dp[1][a.length+1] = 1;
        int currentSum = 1;
        int currentSum_ = 1;
        for(int i=0;i<a.length;i++){ 
            currentSum*=a[i];
            currentSum_*=a[a.length-1-i];
            dp[0][i+1] = currentSum;
            dp[1][a.length-i] = currentSum_;
        }

        int[] b = new int[a.length];
        for(int i=0;i<a.length;i++){
            b[i] = dp[0][i]*dp[1][i+2];
        }
        return b;
    }
}
```

我们需要两个一维数组，空间复杂度是O(N)，一共有两个循环，时间复杂度是O(N)，这样就已经优化的差不多了。

最后来检查一下代码：

- 输入全范围：空数组
- 非法输入：题目保证不会有其他对象输入
- 边界：递归和循环的条件

这样问题就解决了。

## 更优解

这里可以继续优化，把数组占用的空间也优化掉。

> 作者：jyd
> 链接：https://leetcode-cn.com/problems/gou-jian-cheng-ji-shu-zu-lcof/solution/mian-shi-ti-66-gou-jian-cheng-ji-shu-zu-biao-ge-fe/
> 来源：力扣（LeetCode）

本题的难点在于 不能使用除法 ，即需要 只用乘法 生成数组 BB 。根据题目对 B[i]B[i] 的定义，可列表格，如下图所示。

根据表格的主对角线（全为 11 ），可将表格分为 上三角 和 下三角 两部分。分别迭代计算下三角和上三角两部分的乘积，即可 不使用除法 就获得结果。

![](https://i.loli.net/2020/11/18/DufwnX48xKjUFQa.png)

算法流程：
初始化：数组 BB ，其中 B[0] = 1B[0]=1 ；辅助变量 tmp = 1tmp=1 ；
计算 B[i]B[i] 的 下三角 各元素的乘积，直接乘入 B[i]B[i] ；
计算 B[i]B[i] 的 上三角 各元素的乘积，记为 tmptmp ，并乘入 B[i]B[i] ；
返回 BB 。

代码：

```java
class Solution {
    public int[] constructArr(int[] a) {
        if(a.length == 0) return new int[0];
        int[] b = new int[a.length];
        b[0] = 1;
        int tmp = 1;
        for(int i = 1; i < a.length; i++) {
            b[i] = b[i - 1] * a[i - 1];
        }
        for(int i = a.length - 2; i >= 0; i--) {
            tmp *= a[i + 1];
            b[i] *= tmp;
        }
        return b;
    }
}
```

