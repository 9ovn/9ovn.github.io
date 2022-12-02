# 岛屿问题


![background](/blog/island-issue/bak.png "岛屿问题")
  
## 前言

 实在丢人，实在丢人，前两个月设定的目标一个也没有完成。本来风控在家这么久应该是一个刷题、学习的好机会，结果沉迷于WLK无法自拔，也许是上天想拉我一把，魔兽世界要停服了！:sweat_smile:
 于是，开始重新捡起数据结构学习。这篇文章主要记录一下岛屿数量这道题的学习过程，后续的岛屿类的题也会在这篇文章内更新。

## 正文

### 网格类和DFS

岛屿问题的底层就是网格类的DFS便利方法。那什么是网格？什么是DFS？

### 网格

网格就是m*n个小方格形成的网格。
![gird](/blog/island-issue/grid.jpg "网格")

如图所示，每个格子中的数字可能0也可能是1。我们把数字为0的视为海洋，1为陆地。而与1的上下左右就是他相邻的格子，当相邻的格子均为1的时候，那么它们就构成了一个岛屿。

### DFS

什么是DFS？
 DFS全称为深度优先搜索算法，DFS通常用在树或者图结构上。
![dfs](/blog/island-issue/dfs.png "DFS图解")
如图的树结构，如果用dfs遍历的方式，那么顺序就是0->1->3->4->2->5->6。代码的模板是：

``` go
func travel(root treeNode){
 // 判空退出递归
 if root == nil {
  return
 }
 travel(root.left)
 travel(root.right)
}
```

根据模板我们能看出来，重要的地方就是==访问相邻节点== 和 ==判断叶子节点==。

- ==访问相邻节点== 在二叉树中比较简单，我们只需要递归调用左右子树即可。
- ==判断叶子节点== 判断叶子节点的跳出递归调用。

但是网格类型和树是不同，左右子树。所以我们需要对这个模板进行一些变动。

>网格结构dfs模板。

```go
func travel(grid [][]byte, row, col int){

    // 判断叶子节点
    if offGrid(){
    return 
    }

    // 访问相邻节点
    travel(grid, r + 1, c)
    travel(grid, r - 1, c)
    travel(grid, r, c - 1)
    travel(grid, r, c - 1)
 
}

// 两种判断方式，一个是判断是否在范围内（isInArea），一个是判断是否越级(offGrid)
func offGrid(grid [][]byte, row, col int){
    return row >= len(grid) || col >= len(grid) || row < 0 || col < 0
}

func isInArea(grid [][]byte, row, col int){
    return row < len(grid) && col >= len(grid) && row <= 0 && col <= 0
}

```

- ==访问相邻子节点==与树结构的不同点：
  - 格子的上下左右均是他的相邻节点，所以我们需要递归四方方向的格子，直到格子触及边界（!sInArea, offGrid）。

 > 事情并没有简单的结束，我们需要考虑重复遍历的问题。如下图的情况就会出现循环遍历的情况出现。那么我们需要添加添加一个判断来保证不会出现下图的情况。
 >
``` go
 // 判断叶子节点
 if offGrid(){
  return 
 }
 // 
 grid[r][c] = "2"
 // 访问相邻节点
 travel(grid, r + 1, c)
 travel(grid, r - 1, c)
 travel(grid, r, c - 1)
 travel(grid, r, c - 1)
```

> 我们可以看这里我们加入了一行==grid[r][c] = "2"==的方法，这会显式的将该节点标记为以遍历，那么我们再次遇到这个节点就可以选择跳过这个节点。

![循环遍历](/blog/island-issue/circle_loop.jpg "循环遍历")

### 答案

通过上面的分析我们就可以得出代码为。

``` go
func numIslands(grid [][]byte) int {
    cnt := 0
    for r:=0;r<len(grid);r++ {
        for c:=0;c<len(grid[r]);c++{
         // 遍历陆地节点
            if grid[r][c] == '1'{
    dfs(grid, r, c)
    cnt++
            }
        }
    }

    return cnt

}

func dfs(grid [][]byte, r int, c int){

 // 判断叶子节点
    if offGrid(grid [][]byte, r int, c int){
        return
    }
    
 // 不是陆地就跳出递归
    if grid[r][c] != '1'{
        return
    }
    
 // 标记节点为遍历过的
    grid[r][c] = '2'
    
    dfs(grid, r-1, c)
    dfs(grid, r+1, c)
    dfs(grid, r, c-1)
    dfs(grid, r, c+1)

}

func offGrid(grid [][]byte, row, col int){
 return row >= len(grid) || col >= len(grid) || row < 0 || col < 0
}

```

## end

这就是岛屿数量的解法，岛屿问题后续还有几个类似题，陆陆续续也会在这个篇文章中更新。

参考资料： [200. 岛屿数量 - 力扣（Leetcode）](https://leetcode.cn/problems/number-of-islands/solutions/211211/dao-yu-lei-wen-ti-de-tong-yong-jie-fa-dfs-bian-li-/)

