## 题目

在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个高效的函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

```java
示例：
[
  [1,   4,  7, 11, 15],
  [2,   5,  8, 12, 19],
  [3,   6,  9, 16, 22],
  [10, 13, 14, 17, 24],
  [18, 21, 23, 26, 30]
]

给定 target = 5，返回 true。
给定 target = 20，返回 false。
```



 ## 思路历程

首先第一思路肯定是遍历整个数组，但是这样的复杂度非常的高，是O(n^2)，一般算法题遇到这种复杂度肯定可以进行优化。

利用题目讲到的性质，从左下角或者右上角开始，如果target>当前数字，则往右边移动，如果target<当前数字，则往左边移动。这样就可以在O(N)的时间复杂度找到目标数字。



## 代码

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        int i = matrix.length - 1, j = 0;
        while(i >= 0 && j < matrix[0].length)
        {
            if(matrix[i][j] > target) i--;
            else if(matrix[i][j] < target) j++;
            else return true;
        }
        return false;
    }
}


```

