---
title: '[Weekly Algorithm]20201003: Dynamic programming V'
date: 2020-10-02 09:39:50
tags:
    - algorithm
categories:
    - Weekly Algorithm
keywords:
    - Unique Paths II
    - Unique Binary Search Trees II
    - Triangle
    - Longest Increasing Subsequence
    - Continuous Subarray Sum
    - Largest Divisible Subset
---

# 前言
本周一共刷了6道medium，重点是学习了后面两道比较巧妙的解题思路，以及一些老问题的新题型，总体来说比较简单。

# [Unique Paths II](https://leetcode.com/problems/unique-paths-ii/)
```
给定一个带障碍的mxn的方格和一个初始位置在方格左上角的机器人，机器人每次可以从当前位置往右或者下方移动，遇到障碍会停止移动，问一共有多少走法能让机器人走到方格的右下角？其中1代表是障碍，0代表无障碍

输入:
[
  [0,0,0],
  [0,1,0],
  [0,0,0]
]
输出: 2
解释:
1. Right -> Right -> Down -> Down
2. Down -> Down -> Right -> Right
```
<!-- more -->
## 分析求解
和[Unique Paths](https://furthergo.github.io/weekly-algorithm-20200905/#Unique-Paths)类似，多了一个在方格中的障碍，沿用[Unique Paths](https://furthergo.github.io/weekly-algorithm-20200905/#Unique-Paths)的dp解法，我们只需要再加上当前点是否是障碍点的判断即可：当前点是障碍点即obstacleGrid[i][j]=1时，dp[i][j]=0；否则沿用之前的状态转移方程：
<center>dp[i][j] = dp[i-1][j] + dp[i][j-1]</center>

同时观察得到我们只需要一行子状态即可，优化空间复杂度为O(n)。实现代码如下：
```golang
func uniquePathsWithObstacles(obstacleGrid [][]int) int {
	m := len(obstacleGrid)
	n := len(obstacleGrid[0])
	dp := make([]int, n+1)
    for i := 0; i<m; i++ {
        for j := 1; j<=n; j++ {
            if i==0 && j==1 {
                dp[j] = obstacleGrid[0][0]^1
            } else {
                if obstacleGrid[i][j-1] == 1 {
                    dp[j] = 0
                } else {
                    dp[j] += dp[j-1]
                }
            }
        }
    }
    return dp[n]
}
```

## 有趣的优化
解完这道题之后，照例去讨论区看了看，发下了一个有趣的优化点：用原数组当做dp数组，因为对于每次计算来说，我们只依赖原数组的(i,j)点处的值，所以之前的值都可以被覆盖，我们刚好用它来存储dp的值，优化空间复杂度为O(1)。实现代码如下：
```golang
func uniquePathsWithObstacles(obstacleGrid [][]int) int {
	m := len(obstacleGrid)
	n := len(obstacleGrid[0])
    for i := 0; i<m; i++ {
        for j := 0; j<n; j++ {
            if i+j == 0 || obstacleGrid[i][j] == 1 {
                obstacleGrid[i][j] ^= 1
            } else {
                t := 0
                if i != 0 {
                    t += obstacleGrid[i-1][j]   
                }
                if j != 0 {
                    t += obstacleGrid[i][j-1]
                }
                obstacleGrid[i][j] = t
            }
        }
    }
    return obstacleGrid[m-1][n-1]
}
```
不过不知道lc是怎么计算内存占用的，事实上提交结果内存占用并没有比O(n)的小…
# [Unique Binary Search Trees II](https://leetcode.com/problems/unique-binary-search-trees-ii/)
```
给定一个整数n，输出所有用1到n的整数能组成的所有二叉搜索树(BST, Binary Search Tree)。其中BST用中序遍历输出，0<=n<=8。
```

## 分析求解
和[Unique Binary Search Trees](https://furthergo.github.io/weekly-algorithm-20200905/#Unique-Binary-Search-Trees)类似，区别在于之前是输出所有可组成的BST的个数，现在是需要输出所有BST。

解题思路还是可以参照之前的思想：
1. 从1~n任意选择一个数i做根节点组成BST，则它的左子树是1-i~1区间的数组成的BST数组，右子树是i+1~n区间的数组成的BST数组。
2. 左右子树的BST数组求笛卡尔积，和当前根节点i一起组成当前i做根节点时可以组成的BST
3. 递归的求左右子树
4. 当前区间为空时，返回空数组
5. i取值从1~n，执行1-4步骤

实现代码如下：
```golang
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func generateTreesN(l, r int) []*TreeNode {
    res := make([]*TreeNode, 0)
    if l > r {
        res = append(res, nil)
        return res
    } 
    for i := l; i<=r; i++ {
        leftBSTs := generateTreesN(l, i-1)
        rightBSTs := generateTreesN(i+1, r)
        for _, lt := range leftBSTs {
            for _, rt := range rightBSTs {
                root := &TreeNode{
                    Val: i,
                    Left: lt,
                    Right: rt,
                }
                res = append(res, root)
            }
        }
    }
    return res
    
}
func generateTrees(n int) []*TreeNode {
    if n == 0 { return []*TreeNode{} }
    return generateTreesN(1, n)
}
```

## 优化
上面的解法其实是DFS的思路，这是一道dp的题目，我们从dp的角度思考，看能不能优化一下。

对比[Unique Binary Search Trees](https://furthergo.github.io/weekly-algorithm-20200905/#Unique-Binary-Search-Trees)的实现我们知道：`从i到j的数所能构成的所有BST，结构是固定的，和i,j的值大小无关，仅和i,j的大小关系相关，即f(i,j) = f(1, j-i+1)，即n为j-i+1。`

那这道题能不能用同样的思路呢？下面来分析一下：
* dp[n]代表从1~n能组成的所有BST; f(i,j)代表i~j区间能组成的所有BST，即dp[n] = f(1,n)。
* 从1~n任意选择一个数i做根节点组成BST，则它的左子树是1-i~1区间的数组成的BST数组，右子树是i+1~n区间的数组成的BST数组。
* 而f(i+1,n)的所有BST的个数等于f(1,n-i)的所有BST的个数，并且是一一对应的。
* 区别f(i+1,n)的每一个BST，都在f(1,n-i)中有一个完全一样的结构的BST，并且每个对应位置节点的数相差是i
* 因此当我们求f(i+1,n)时，我们可以用预先求好的f(1,n-i)，然后对每个节点做一个i的偏移，即得到f(i+1,n)

实现代码如下：
```golang
func shiftBST(r *TreeNode, os int) *TreeNode {
	if r == nil {
		return r
	}
	return &TreeNode{
		Val: r.Val + os,
		Left: shiftBST(r.Left, os),
		Right: shiftBST(r.Right, os),
	}
}

func shiftBSTs(rs []*TreeNode, os int) []*TreeNode {
	res := make([]*TreeNode, len(rs))
	for i, r := range rs {
		res[i] = shiftBST(r, os)
	}
	return res
}

func generateTrees(n int) []*TreeNode {
	if n == 0 { return []*TreeNode{} }
	r1 := []*TreeNode{
		&TreeNode{
			Val: 1,
		},
	}
	if n == 1 {
		return r1
	}
	dp := make([][]*TreeNode, n+1)
	dp[0] = []*TreeNode{nil}
	dp[1] = r1
	for i := 2; i<=n; i++ {
		dpi := make([]*TreeNode, 0)
		for j := 1; j<=i; j++ {
			leftBSTs := dp[j-1]
			rightBSTs := shiftBSTs(dp[i-j], j)
			for _, lt := range leftBSTs {
				for _, rt := range rightBSTs {
					root := &TreeNode{
						Val: j,
						Left: lt,
						Right: rt,
					}
					dpi = append(dpi, root)
				}
			}
		}
		dp[i] = dpi
	}
	return dp[n]
}
```
需要注意的是，go的slice实际上是一个struct，其中有指向时机数组的指针array，结构是：
```golang
// runtime/slice.go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
dp是一个切片，所以shiftBSTs(dp[i-j], j)的时候，dp[i-j]是array指针指向的数组的一个元素，如果我们直接修改dp[i-j]，即shiftBSTs的rs，会发现dp[i-j]的值也被修改了，导致计算出错，所以我们需要重新创建一个res，来做shift操作。对于shiftBST(r, os)也是同理。

# [Triangle](https://leetcode.com/problems/triangle/)
```
给定一个三角形，输出从顶点到底边的最小路径和，每次只能往下一行相邻的的数字移动。

比如输入：
[
     [2],
    [3,4],
   [6,5,7],
  [4,1,8,3]
]
输出是11，等于2 + 3 + 5 + 1.
```

## 分析求解
典型的dp题，假设要求从顶点移动到当前位置点(i,j)的最小路径和dp[i][j]，我们只需要计算它上一行相邻的两个数的最小路径和，然后取最小值，加上当前位置的值即可。同时我们可以看到上面的三角形其实是
```
[
[2],
[3,4],
[6,5,7],
[4,1,8,3]
]
```
这种形式的，(i,j)的上一行的相邻点是(i-1,j)和(i-1, j-1)。即：
<center>
    dp[i][j] = min(dp[i-1][j], dp[i-1][j-1]) + nums[i][j]
</center>

优化空间复杂度后实现如下：
```golang
func minimumTotal(triangle [][]int) int {
    n := len(triangle)
    dp := make([]int, n)
    dp[0] = triangle[0][0]
    res := dp[0]
    for i := 1;i<n; i++ {
        dp[i] = dp[i-1] + triangle[i][i]
        res = dp[i]
        for j :=i-1; j>=0; j-- {
            if j > 0 && dp[j-1] < dp[j] {
                dp[j] = dp[j-1]
            }
            dp[j] += triangle[i][j]
            if dp[j] < res {
                res = dp[j]
            }
        }
    }
    return res
}
```
时间复杂度是O(n^2)，空间复杂度是O(n)

## 优化

用triangle存储dp值，优化space为O(n)，实现如下：
```golang
func minimumTotal(triangle [][]int) int {
    n := len(triangle)
    res := triangle[0][0]
    for i := 1;i<n; i++ {
        triangle[i][i] += triangle[i-1][i-1]
        res = triangle[i][i]
        for j :=i-1; j>=0; j-- {
            if j > 0 && triangle[i-1][j-1] < triangle[i-1][j] {
                triangle[i][j] += triangle[i-1][j-1]
            } else {
                triangle[i][j] += triangle[i-1][j]
            }
            if triangle[i][j] < res {
                res = triangle[i][j]
            }
        }
    }
    return res
}
```

上面的解法都是从上往下遍历，有个`j=0时该点没有左上角的相邻节点`这个边界case，我们可以从最后一行开始从下往上遍历，简化计算过程，即：
<center>dp[i][j] = min(dp[i+1][j], dp[i+1][j+1]) + nums[i][j]</center>

实现如下：
```golang
func min(a, b int) int {
    if a < b { return a }
    return b
}
func minimumTotal(triangle [][]int) int {
    n := len(triangle)
    for i := n-2;i>=0; i-- {
        for j :=0; j<=i; j++ {
            triangle[i][j] += min(triangle[i+1][j], triangle[i+1][j+1])
        }
    }
    return triangle[0][0]
}
```

# [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/)
```
给定一个无序的整数数组，输出数组的最长升序子序列的长度。

比如输入: [10,9,2,5,3,7,101,18]
输入: 4 
解释: 最长升序子序列是[2,3,7,101], 长度为4. 
```

## 分析求解
我们遍历数组，记录每个以元素nums[i]为升序子序列结尾的子序列的最长长度，则有:
<center>
dp[i] = max(dp[j]&(nums[i] >= nums[j])) + 1

其中j从0到i-1,
</center>

实现代码如下:
```golang
func max(a, b) int {
    if a > b { return a }
    return b
}
func lengthOfLIS(nums []int) int {
    n := len(nums)
    if n <=1 { return n }
    dp := make([]int, n)
    dp[0] = 1
    res := 1
    for i := 1; i<n; i++ {
        m := 0
        for j :=0; j<i; j++ {
            if nums[j] < nums[i] && dp[j] > m {
                m = dp[j]
            }
        }
        dp[i] = m+1
        res := max(res, dp[i])
    }
    return dp[n-1]
}
```

# [	Continuous Subarray Sum](https://leetcode.com/problems/continuous-subarray-sum/)
```
给定一个非负整数数组和一个整数k，输出当前数组是否存在一个长度至少为2的连续子数组的和，是k的倍数。
比如输入: [23, 2, 4, 6, 7],  k=6
输出: True
解释: 子数组[2, 4]的和是6 = 1*k.
```

## 分析求解
常规思路，遍历所有的子数组，求和然后判断是不是k的倍数即可，这里求子数组的和可以应用前缀和，同时注意k=0的情况，k是不能用作被模的数。实现代码如下：
```golang
func checkSubarraySum(nums []int, k int) bool {
    n := len(nums)
    if n < 2 { return false }
    sum := make([]int, n+1)
    for i := 1; i<=n; i++ {
        sum[i] = sum[i-1] + nums[i-1]
    }
    for i := 0; i<n-1; i++ {
        for j := i+1; j<n; j++ {
            t := sum[j+1] - sum[i]
            if k == 0 {
                if t == 0 {
                    return true
                }
                
            } else {
                if t%k == 0 {
                    return true
                }
            }
        }
    }
    return false
}
```

时间复杂度是O(n^2)，空间复杂度是O(n)。

## 优化
本以为这样已经够快了，提交发现faster than 38.67%，于是去讨论区学习了一下，发现了下面这个[非常非常巧妙的方法](https://leetcode.com/problems/continuous-subarray-sum/discuss/99499/Java-O(n)-time-O(k)-space)：

* 遍历数组，求解到当前位置i的子数组的和sum[i]
* 创建一个map，存储sum[i]和i，表示和为sum[i]的子数组是从下标0到下标i
* 对于sum[i]，我们尝试用k对sum[i]取模，得到mod
* 假如在map里存在mod，即存在j = map[mod]，则说明sum[j] =  mod，而sum[i] = nxk + mod
* 所以从j+1到i的子数组的和等于sum[i] - sum[j] = nxk + mod - mod = nxk。即存在子数组i~j的和是k的倍数。

这样我们只需要遍历数组一遍，同时存储一个sum和index的映射，时间复杂度和空间复杂度都是O(n)。

实现如下：
```golang
func checkSubarraySum(nums []int, k int) bool {
    n := len(nums)
    if n < 2 { return false }
    sumMap := make(map[int]int)
    sumMap[0] = -1
    sum := 0
    for i := 0; i<n-1; i++ {
        sum += nums[0]
        if k != 0 {
            sum %= k
        }
        if preI, ok := sumMap[sum]; ok {
            if preI < i-1 {
                return true
            }
        } else {
            sumMap[sum] = i
        }
    }
    return false
}
```
有三个点需要注意：

* 这里需要判断preI < i-1，因为要求子数组的长度至少是2，当preI = i-1时，子数组的长度为1，不满足条件
* sumMap[0] = -1，当sum[i]是k的倍数时，即sum[i]%k=0，代表0-i的子数组就满足是k的倍数，这时我们取sumMap[sum%k] = sumMap[0]一定要存在并且满足子数组长度至少为2，考虑极限情况i等于1时需要满足，i等于0时长度为1不满足，即 0-1<= sumMap[0] < 1-1，即-1<=sumMap[0]<0，即sumMap[0] = -1
* 在k不为0时，我们直接让sum %= k，然后sumMap[sum] = i，等同于sumMap[sum[i]%k] = i。这是因为a%m = b => a%m = b%m，即假如sum[j]%k = sum[i]，则sum[j]%k = sum[i]%k。

# [ Largest Divisible Subset](https://leetcode.com/problems/largest-divisible-subset/)
```
给定一个不含重复数字的正整数集合，输出它的最大可除子集，其中最大可除子集必须满足，子集中任意两个数，至少有一个可以被另一个数除尽。
比如输入: [1,2,3]
输出: [1,2] (或者[1,3])
```

## 分析求解
常规解法，首先我们对数组按升序排序，然后遍历数组尽可能的往最大可除子集的末尾添加新的数，因为是升序的，那么新的数可以添加进最大可除子集的条件是，它是当前最大可除子集里最大数的倍数。

我们用一个数组dp存储从下标0到下标i，以nums[i]为最大数的最大可除子集，则有状态转移方程：
<center>
dp[i] = { max(dp[j]&&(nums[i]%nums[j]==0)), nums[i] }
</center>

实现代码如下：
```golang
func largestDivisibleSubset(nums []int) []int {
    sort.Ints(nums)
    n := len(nums)
    if n == 0 { return []int{} }
    dp := make([][]int, n)
    dp[0] = []int{nums[0]}
    res := dp[0]
    for i:=1; i<n; i++ {
        maxLen := 0 // 记录最大的子集长度
        maxIdx := 0 // 记录对应的index
        for j := 0; j<i; j++ {
            if nums[i]%nums[j] == 0 && len(dp[j]) > maxLen { // 判断nums[i]可以除尽nums[j]，并且长度比maxLen长，则更新maxLen
                maxLen =len(dp[j])
                maxIdx = j
            }
        }
        dp[i] = make([]int, maxLen+1) // 根据maxIdx设置dp[i]
        for k, t := range dp[maxIdx] {
            dp[i][k] = t
        }
        dp[i][maxLen] = nums[i]
        maxLen += 1
        if maxLen > len(res) { // 尝试更新结果
            res = dp[i]
        }
    }
    return res
}
```

## 优化
上述解法的时间复杂度和空间复杂度都是O(n^2)，然后再次翻看了讨论区，叕发现了[非常精妙的解法](https://leetcode.com/problems/largest-divisible-subset/discuss/84006/Classic-DP-solution-similar-to-LIS-O(n2))。下面我们来分析一下：

* 上述解法中，我们记录了maxLen和maxIdx，以及dp[maxIdx]代表从0到maxIdx处，以nums[maxIdx]为最大值的最长可除子集
* 然后我们用{ dp[maxIdx], nums[i] } 更新dp[i]
* 通过观察可以发现，在求dp[i]时：
    * 根据dp[i]的定义，nums[i]一定在最后的dp[i]中
    * 当我们找到了maxIdx，dp[i] = {dp[maxIdx], nums[i]}则表明nums[maxIdx]也在dp[i]中
    * 同理对于之前我们求解dp[maxIdx]时，一定也有一个preMaxIdx的下标，即dp[maxIdx] = {dp[preMaxIdx], nums[maxIdx]}，即nums[preMaxIdx]在dp[maxIdx]中；即nums[preMaxIdx]在dp[i]中
    * ……
    * 以此类推，我们往前找，就可以找到最后所有在dp[i]中的数
* 因此，在我们遍历的过程中，只记录dp[i]对应的长度，以及当前i对应的maxIdx，然后当遍历结束后，maxIdx一步步的构建最后的最长可除子集

实现代码如下：
```golang
func largestDivisibleSubset(nums []int) []int {
    n := len(nums)
    sort.Ints(nums)
    dp := make([]int, n) // 代表0~i的最长可除子集的长度
    preIdx := make([]int, n) // 用于存储每个i的maxIdx
    resLen := 0
    resLastIdx := -1
    for i:=0; i<n; i++ {
        dp[i] = 1 // 记录最大的子集长度
        preIdx[i] = -1
        for j := 0; j<i; j++ {
            if nums[i]%nums[j] == 0 && dp[j] + 1 > dp[i] { // 判断nums[i]可以除尽nums[j]，并且长度比maxLen长，则更新maxLen
                dp[i]= dp[j] + 1
                preIdx[i] = j
            }
        }
        if dp[i] > resLen {
            resLen = dp[i]
            resLastIdx = i
        }
    }

    res := make([]int, resLen)
    for i:=0; i<resLen;i++ {
        res[i] = nums[resLastIdx]
        resLastIdx = preIdx[resLastIdx]
    }
    return res
}
```

# EOF
周末有两场contest，很久没参加了，准备再挑战一下！