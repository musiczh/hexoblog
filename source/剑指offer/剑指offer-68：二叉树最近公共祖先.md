## 题目

给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。如下图：

![UTOOLS1606465037173.png](https://img01.sogoucdn.com/app/a/100520146/bea7364b3757e477e4c1945b387c5042)

1,5的公共祖先是3，而5,4的公共祖先是5。

## 思路历程

这道题其实还有一个简单版，就是这棵二叉树并不是普通二叉树，而是搜索二叉树，这样寻找起来会更加方便一点，但是思想是没有变化的。

遇到二叉树的第一想法就是：**递归**。

一开始的思路是：递归到当前节点，如果当前节点是两个节点之一，那么返回1，如果不是，则递归并获取左右子树的返回值。如果左右子树都返回1 ，则说明当前节点就是所求节点，把当前节点赋值给全局变量result；如果有一个2，那么直接返回2；如果都是0返回0；剩下的返回1。

首次分析遗漏了5,4这种情况，只分析了5，1 这种左右节点的情况。后续改为先获取左右子树的情况，再判断当前节点的情况作出条件判断。

这种思路可以完成，但是需要一个全局变量，而且代码复杂，判断条件很多，代码可读性也很差。需要有一个更加合理的代码思路

## 新思路

观察官方解法可以发现，我和他的思路是比较接近的，但是他采用了和我不同的返回值。我才用了int类型的返回值，所以有很多种组合的情况需要判断，而他采用了TreeNode作为返回值，这样只需要判断是否是null就可以了，且不需要额外的一个全局变量。

作者的思路是直接拿当前节点作为分析根基。如果当前节点是所求节点，那么有以下情况：

- 左右各有一个目标节点
- 当前节点是目标节点，且另一个节点在他的左右子树中

如果当前不是目标节点，那么有以下情况：

- 全部都在右子树
- 全部都在左子树

由于返回的是目标节点引用，那么可以直接采用返回引用的方式。遇到目标节点直接返回。

如果左右都不是null，那么就是本节点。如果一边null一边不是null，那么就是返回的引用本身。这样可以极大地简化逻辑优化代码。

## 代码

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if(root == null || root == p || root == q) return root;
        TreeNode left = lowestCommonAncestor(root.left, p, q);
        TreeNode right = lowestCommonAncestor(root.right, p, q);
        if(left == null && right == null) return null; // 1.
        if(left == null) return right; // 3.
        if(right == null) return left; // 4.
        return root; // 2. if(left != null and right != null)
    }
}

作者：jyd
链接：https://leetcode-cn.com/problems/er-cha-shu-de-zui-jin-gong-gong-zu-xian-lcof/solution/mian-shi-ti-68-ii-er-cha-shu-de-zui-jin-gong-gon-7/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```



