---
layout:     post
title:      "【跟Leon一起刷LeetCode】463. Island Perimeter"
subtitle:   " \"人生总是起起伏伏\""
date:       2018-04-03 16:57:00
author:     "Leon"
header-img: "img/2018-04-03-posts-bg.jpg"
catalog: true
tags:
    - LeetCode
---

> “Anyway, anyhow. ”


### Island Perimeter

##### Description：
You are given a map in form of a two-dimensional integer grid where 1 represents land and 0 represents water. Grid cells are connected horizontally/vertically (not diagonally). The grid is completely surrounded by water, and there is exactly one island (i.e., one or more connected land cells). The island doesn't have "lakes" (water inside that isn't connected to the water around the island). One cell is a square with side length 1. The grid is rectangular, width and height don't exceed 100. Determine the perimeter of the island.
###### **Example 1：**
```
[[0,1,0,0],
 [1,1,1,0],
 [0,1,0,0],
 [1,1,0,0]]
```
Answer: 16
Explanation: The perimeter is the 16 yellow stripes in the image below:
![island](http://img.blog.csdn.net/20180120171830671?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGVvbmZyZWU3Nzc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
##### **问题描述**:
给定一个二维数组表示一个网格，1代表岛，0代表水。网格是完全被水环绕的，只有一个岛（一个或多个接壤的陆地cell），对角是不相连接的。一个cell的边长为1。求小岛的周长。
##### Solution：
解法一：
找到每个陆地，并且计算出每个陆地接壤的水cell的个数（假设边界外面包了一圈水cell）。
```
class Solution {
    public int islandPerimeter(int[][] grid) {
        int row = grid.length;
        int col = grid[0].length;
        int perimeter = 0;
        for(int i = 0; i < row; i++){
           for(int j = 0; j < col; j++){
                if (grid[i][j] == 0){
                    continue;
                }else{
                    if (i - 1 < 0 ) perimeter++;
                    else if (grid[i-1][j] == 0) perimeter++;
                    if (i + 2 > row) perimeter++;
                    else if (grid[i+1][j] == 0) perimeter++;
                    
                    if (j - 1 < 0 ) perimeter++;
                    else if (grid[i][j-1] == 0) perimeter++;
                    if (j + 2 > col) perimeter++;
                    else if (grid[i][j+1] == 0) perimeter++; 

                }
            } 
        }
        return perimeter;
    }
}
```
解法二：
把它看成一个数学问题:
```
+--+     +--+                   +--+--+
|  |  +  |  |          ->       |     |
+--+     +--+                   +--+--+
```
``islands * 4 - neighbours * 2``，每次合并都会在总周长的基础上减2，形成环形的[[1,1],[1,1]]时候要减4.总结下来就是上面的公式。这个时候只要遍历所有的陆地cell，找到它们的邻居。遍历时，是找两个方向，左上或者右下邻居，因为两个方向即可遍历完这个网格，且不会重复。
```
public class Solution {
    public int islandPerimeter(int[][] grid) {
        int islands = 0, neighbours = 0;

        for (int i = 0; i < grid.length; i++) {
            for (int j = 0; j < grid[i].length; j++) {
                if (grid[i][j] == 1) {
                    islands++; // count islands
                    if (i < grid.length - 1 && grid[i + 1][j] == 1) neighbours++; // count down neighbours
                    if (j < grid[i].length - 1 && grid[i][j + 1] == 1) neighbours++; // count right neighbours
                }
            }
        }

        return islands * 4 - neighbours * 2;
    }
}
```



