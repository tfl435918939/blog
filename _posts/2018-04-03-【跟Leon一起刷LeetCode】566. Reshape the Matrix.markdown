---
layout:     post
title:      "【跟Leon一起刷LeetCode】566. Reshape the Matrix"
subtitle:   " \"人生总是起起伏伏\""
date:       2018-04-03 22:27:00
author:     "Leon"
header-img: "img/2018-04-03-posts-bg.jpg"
catalog: true
tags:
    - LeetCode
---

> “Anyway, anyhow. ”


### Reshape the Matrix

##### Description：
In MATLAB, there is a very useful function called 'reshape', which can reshape a matrix into a new one with different size but keep its original data.

You're given a matrix represented by a two-dimensional array, and **two** positive integers **r** and **c** representing the **row** number and **column** number of the wanted reshaped matrix, respectively.

The reshaped matrix need to be filled with all the elements of the original matrix in the same **row-traversing** order as they were.

If the 'reshape' operation with given parameters is possible and legal, output the new reshaped matrix; Otherwise, output the original matrix.
###### **Example 1：**
```
Input: 
nums = 
[[1,2],
 [3,4]]
r = 1, c = 4
Output: 
[[1,2,3,4]]
Explanation:
The row-traversing of nums is [1,2,3,4]. The new reshaped matrix is a 1 * 4 matrix, fill it row by row by using the previous list.
```
###### **Example 2：**
```
Input: 
nums = 
[[1,2],
 [3,4]]
r = 2, c = 4
Output: 
[[1,2],
 [3,4]]
Explanation:
There is no way to reshape a 2 * 2 matrix to a 2 * 4 matrix. So output the original matrix.
```
###### **Note：**
1. The height and width of the given matrix is in range [1, 100].
2. The given r and c are all positive.

##### **问题描述**:
给定一个矩阵表示为一个二维数组,和两个正整数r和c分别代表希望重塑矩阵的行数和列数。重塑矩阵需要充满原始矩阵的所有元素在同一个row-traversing顺序。 如果给出的“重塑”操作参数是可能的,合法的,输出新的重塑矩阵;否则,输出原始矩阵。
##### Solution：
首先得保证重塑前后矩阵的个数相等，假定原来矩阵行数row，列数col，col*row = r*c。每个元素在重塑前后的位置有等式，a*col + b = c*r + d  => newNums[i/c][i%c] = nums[i/col][i%col];
```
class Solution {
    public int[][] matrixReshape(int[][] nums, int r, int c) {
        int row = nums.length;
        int col = nums[0].length;
        if(r*c > row*col) return nums;
        int[][] newNums = new int[r][c];
        for(int i = 0; i < row*col; i++){
            newNums[i/c][i%c] = nums[i/col][i%col];
        }
        return newNums;
        
    }
}
```
如果先将原矩阵展开成一维数组再进行填充，就会遍历两次，时间复杂度较高。



