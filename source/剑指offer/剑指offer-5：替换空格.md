## 题目

请实现一个函数，把字符串 `s` 中的每个空格替换成"%20"。

```java
示例 ：
输入：s = "We are happy."
输出："We%20are%20happy."
```

## 思路历程

由于Java中字符串不可修改，需要拼装到另一个字符串。遍历整个字符串，遇到空格转换为%20，使用StringBuilder来拼接，最后调用toString返回即可。

但是这道题不可能这么简单，肯定另有玄机。题目的背景是在C++语言下，所以字符串是可以在原字符串上修改的，并不是一个常量，如何在原字符串上进行修改是这道题的难点。

官解中采用的方法是：

- 先遍历整个字符串，找到空格的个数，确定新的字符串有多长
- 扩展原来的字符串长度
- **从后往前**遍历原字符串进行迁移

他这里的核心就在于：从后往前，这样就不需要用移动整个数组，使得时间复杂度为O(N)

## 代码

```java
class Solution {
    public String replaceSpace(String s) {
		if(s==null) return "";
        
		StringBuilder sb = new StringBuilder();
        for(int i=0;i<s.length();i++){
            if(s.charAt(i)==' '){
                sb.append("%20");
                
            }else{
                
                sb.append(s.charAt(i));
            }
        }
        return sb.toString();
    }
}
```

