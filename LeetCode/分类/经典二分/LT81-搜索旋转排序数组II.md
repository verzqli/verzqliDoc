#### 题目：[搜索旋转排序数组II](https://leetcode-cn.com/problems/search-in-rotated-sorted-array)

> 假设按照升序排序的数组在预先未知的某个点上进行了旋转。( 例如，数组 `[0,0,1,2,2,5,6]` 可能变为 `[2,5,6,0,0,1,2]` )。编写一个函数来判断给定的目标值是否存在于数组中。若存在返回 `true`，否则返回 `false`。
>

#### 示例

```java
输入: nums = [2,5,6,0,0,1,2], target = 0
输出: true

输入: nums = [2,5,6,0,0,1,2], target = 3
输出: false
```

#### 分析

这题是[LT33-搜索旋转排序数组](LT33-搜索旋转排序数组.md)的进阶版，这里主要增加的难度是存在相同的数字，这就会导致这种情况``[0,0,0,0,0,1,1]``旋转后变成``[0,0,0,0,1,1,0]``这时候你会发现旋转后start mid end三个位置的地方全是0，你根本没法判断mid处在前面的递增序列还是后面的递增序列，所以出现这种情况的时候需要加一个顺序遍历。在上题中使用递归，这题使用while循环来做。





