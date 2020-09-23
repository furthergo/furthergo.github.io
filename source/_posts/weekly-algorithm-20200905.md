---
title: '[Weekly algorithm]20200905: Dynamic programming'
date: 2020-09-05 17:12:49
tags: 
    - algorithm
categories:
    - Weekly Algorithm
# keywords: algorithm,leetcode,Unique Paths,Decode Ways,Unique Binary Search Trees,Word Break,Maximum Product Subarray,House Robber,Maximal Square
keywords: 
    - algorithm
    - leetcode
    - Unique Paths
    - Decode Ways
    - Unique Binary Search Trees
    - Word Break
    - Maximum Product Subarray
    - House Robber
    - Maximal Square
---


# 0x00
本周刷题总结：
* 本周刷题集中在动态规划专题，语言是Golang
* 一共刷了八道题，全部是medium
* 收获：了解基础的动态规划题型，掌握了基本的解题套路和优化方式



# 0x01
下面就针对每一道题大概写一下解题思路，权当复习和巩固知识。
## [Unique Paths](https://leetcode.com/problems/unique-paths/)
`给一个m*n的矩阵，计算从左上角走到右下角共有多少种走法，每一步只能向右或者向下走。`

<div align=center> <img src ="https://assets.leetcode.com/uploads/2018/10/22/robot_maze.png"/> </div>

### 思路分析
* 状态方程：
对于走到矩阵第i,j点来说，有两种走法，从i-1,j或者从i,j-1，因此设f(i,j)为走到点i,j的走法，则：**f(i,j) = f(i-1, j) + f(i, j-1)**
* 求解：用一个m*n的dp数组存储走到点i,j的走法，遍历求解`dp[i][j] = dp[i-1][j] + dp[i][j-1]`，时间复杂度和空间复杂度为`O(mn)`。dp初始化全部为1，然后i,j从1开始遍历，因为第一列和第一行的每个点都只有一种走法
<!-- more -->
```golang
    func uniquePaths(m int, n int) int {
    dp := make([][]int, m)
    for i := 0; i<m; i++ {
        dp[i] = make([]int, n)
        for j := 0; j<n; j++ {
            dp[i][j] = 1
        }
	}

	for i := 1; i<m; i++ {
		for j := 1; j<n; j++ {
			dp[i][j] = dp[i-1][j] + dp[i][j-1]
		}
	}
	return dp[m-1][n-1]
    }
```
* 优化：观察代码可以知道，求dp[i][j]的时候，依赖dp[i-1][j]和dp[i][j-1]的值，因此可以只用两个长度为m的数组来存储这两行状态，空间复杂度为O(2n) = O(n)，这是一种常见的DP空间压缩方式
```golang
    func uniquePaths(m int, n int) int {
        dp := make([]int, n) // 前一行
        dp1 := make([]int, n) // 当前行
        for j := 0; j<n; j++ {
            dp[j] = 1
            dp1[j] = 1
        }

        for i := 1; i<m; i++ {
            for j := 1; j<n; j++ {
                dp1[j] = dp[j] + dp1[j-1] // dp[j]代表上边；dp1[j-1]代表左边	
            }
            dp = dp1 // 每行遍历结束更新前一行dp
        }
        return dp1[n-1]
    }
```
* 优化more： 在观察代码`dp1[j] = dp[j] + dp1[j-1]`，发现求dp1[j]时，dp[j]其实是在上一次行遍历结束时`dp = dp1`从dp1那里得到的，即dp[j] = dp1[j]，因此这里可以优化成 `dp1[j] = dp1[j] + dp1[j-1]`，这时就可以发现，这个状态转移方程里只有一个数组dp1，因此我们可以进一步优化内存占用
```golang
    func uniquePaths(m int, n int) int {
        dp := make([]int, n)
        for i := 0; i<n; i++ {
            dp[i] = 1
        }

        for i := 1; i<m; i++ {
            for j := 1; j<n; j++ {
                dp[j] += dp[j-1]
            }
        }
        return dp[n-1]
    }
```

### 小结
为什么用dp？如果这道题用递归，写法也很简单，但是在计算f(i,j-1)和f(i-1,j)时都会计算一遍f(i-1,j-1)，产生重复计算，。动态规划就是使用空间换时间的思想，在求解依赖子问题时，使用dp数组存储子问题解，避免重复计算。而在实现过程中，又可以根据我们所需要依赖的子问题的状态，进一步优化空间，达到最优解。

## [Minmium Path Sum](https://leetcode.com/problems/minimum-path-sum/)
`还是m*n的矩阵，每个点有一个非负整数，和上题一样的走法，计算左上角到右下角最小路径和。`

### 分析求解
这道题和上题的思路类似，加上了每次路径的和，因为要求最小值，可以得到状态转移方程`f(i,j) = min(f(i-1,j), f(i, j-1)) + v(i, j)`。实现代码如下, 时间复杂度为O(mn)，空间复杂度为O(n)
```golang
func minPathSum(grid [][]int) int {
	m := len(grid)
	n := len(grid[0])

	dp := make([]int, n)
    dp[0] = grid[0][0] // 出发点
    
	for j:= 1; j < n; j++ {
		dp[j] = dp[j-1] + grid[0][j] // 先处理第一行，第一行只有从左往右的走法
	}

	for i := 1; i < m; i++ { // 从第二行开始遍历
		dp[0] +=grid[i][0] // 每行第一个点，只有从上往下的走法
		for j:= 1; j < n; j++ {
			if dp[j-1] < dp[j] { // dp[j]代表的是从上往下的走法，尝试用从左往右的走法更新dp[j]
				dp[j] = dp[j-1]
			}
			dp[j] += grid[i][j]
		}
	}
	return dp[n-1]
}
```

## [Decode Ways](https://leetcode.com/problems/decode-ways/)
`一串数字组成的数字串，数字可以映射到字母，1-26代表字母A-Z，问字符串s映射到字母串有多少种映射方式？`

### 分析求解
这是一道一维dp，对于位置的数字s[i]来说，它的映射方式依赖s[i-1]的值，可以单独映射成一个字母，也可以和前一个数一起一起组成两位数映射到字母。有两种可能：
1. 它可以自己映射为一个字母，需要s[i]不为0
2. 它可以和前一个数字一起映射为一个字母，组成的数字是10到26，需要s[i-1]为1或者s[i-1]为2且s[i]小于等于6。

因此状态转移方程是
<center>f(i) = ff(i-1) + ff(i-2)</center>
其中
<center>ff(i-1) = f(i-1)*(s[i]!=0)</center>
<center>ff(i-2) = f(i-2) * (s(i-1)==1 || (s(i-1)==2&&s[i]<7))</center>

实现代码如下，时间复杂度和空间复杂度为O(n)
```golang
func numDecodings(s string) int {
    n := len(s)
    if s[0] == '0' {
        return 0
    }
    dp := make([]int, n+1)
    dp[0] = 1
    dp[1] = 1
    
    for i := 2; i<=n; i++ {
        if s[i-1] == '0' {
            dp[i] = 0
        } else {
            dp[i] = dp[i-1]
        }
        if s[i-2] == '1' || (s[i-2] == '2' && s[i-1] < '7') {
            dp[i] += dp[i-2]
        }
    }
    return dp[n]
}
```
因为对于每个i来说，需要依赖dp(i-1)和dp(i-2)的状态，为了简化和统一计算dp(1)，在将dp数组声明成n+1，且dp[0] = 1。

### 优化

```golang
func numDecodings(s string) int {
    n := len(s)
    if s[0] == '0' {
        return 0
    }
    
    pre := 1
    prepre := 1
    
    for i := 2; i<=n; i++ {
        t := 0
        if s[i-1] == '0' {
            t = 0
        } else {
            t = pre
        }
        if s[i-2] == '1' || (s[i-2] == '2' && s[i-1] < '7') {
            t += prepre
        }
        prepre = pre
        pre = t
    }
    
    return pre
}
```
因为dp[i]只依赖dp[i-1]和dp[i-2]，因此可以用两个变量来存储，优化空间复杂度为O(1)。

## [Unique Binary Search Trees](https://leetcode.com/problems/unique-binary-search-trees/)
`给定数字n，求1到n个数字能组成多少种二叉搜索树（BST, Binary Search Tree)？其中1<=n<=19`

### 分析求解
这道题想了很久，也没想明白怎么用DP解……看题解发现很巧妙也很简单：

1. 对于1-n个数，我们从1-n个数里任意拎出来一个数i作为BST根节点。可以得出
<center>f(n) = root(1) + root(2) + ... root(n)

其中root(i)代表选第i个数作为BST的根节点时，n个数能组成BST的个数</center>

2. BST的性质是：对于根节点，`所有的左子树的值都比根节点小，右子树的值都比根节点大`。那么当我们拎出i作为根节点时，它的左子树就是1到i-1的数组成的BST，右子树就是i+1到n组成的BST。
可以得出 

<center>root(m) = sub(1...m-1)*sub(m+1...n)

其中sub(i...j)代表i到j的数字序列能组成的BST的个数</center>

3. 对于从i到j的连续的自然数，这j-i个数能组成BST的个数和i,j的大小其实没有关系，因为它们之间的大小关系是固定的，而BST的组成依赖的就是数字之间的大小关系，因此同样个数的连续的自然数序列可以组成的BST的个数是相同的。
可以得出

<center>sub(i...j) = sub(1...j-i+1) = f(j-i+1)

即

root(m) = f(m-1)*f(n-m)

</center>

4. 最终得出状态转移方程
<center>f(n) = root(1) + root(2) + ... + root(n)

即

f(n) = f(0)*f(n-1) + f(1) * f(n-2) + ... + f(n-1)*f(0)

其中f(0) = 1, 代表子树为空子树</center>

我们从1到n一次计算f(n)，用一个数组dp[i]代表f(i)，时间复杂度是O(n*n), 空间复杂度是O(n), 具体实现代码如下：
```golang
func numTrees(n int) int {
    dp := make([]int, n+1)
    dp[0] = 1
    dp[1] = 1
    if n == 1 {
        return 1
    }
    for i := 2; i<=n; i++ {
        s := 0
        for j := 1;j <= i; j++ {
            s += dp[j-1] * dp[i-j]
        } 
        dp[i] = s
    } 
    return dp[n]
}
```

## [Word Break](https://leetcode.com/problems/word-break/)
`给一个字符串和一个单词组成的数组，求这个字符串是否可以划分为只由数组里的词组成，每个词可以使用多次。`

### 分析求解
思路比较直观，对于字符串s，假如它可以被分成一个满足条件的子串和一个在数组中存在的单词，那么这个字符串就满足题意。即 s = s' == true && (s-s') in dict。对于长度为n的字符串，根据划分的子串的长度可以是1到n-1，我们可以得到状态转移方程：

<center>f(n) = f(0) && c(0...n) || f(1) && c(1...n) || ... || f(-1) && c(n-1...n)

其中c(i...j)代表s的子串s(i:j)是否包含于单词数组。</center>

实现代码如下：
```golang
func wordContain(word string, wordDict []string) bool {
	for _, v := range wordDict {
		if word == v {
			return  true
		}
	}
	return false
}

// do[i]代表从0开始长度为i的子串是否是可以wordBreak
// dp[i] = dp[j] && wordDict.contain(s[j:i]),  j = 0...i-1
func wordBreak(s string, wordDict []string) bool {
	n := len(s)
	dp := make([]bool, n+1)
	dp[0] = true
	for i := 1; i<=n; i++ {
		for j := i-1; j>=0; j-- {
			if dp[j] && wordContain(s[j:i], wordDict) {
				dp[i] = true
				break
			}
		}
	}
	return dp[n]
}
```

## [Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/)
`给定一个整数数组，求其连续子数组的最大乘积。`

### 分析求解
首先直观思路，对于求字数组nums[0:i]的最大乘积，我们可以先求nums[0:i-1]的最大乘积，nums[0:i]的最大乘积为两情况：
1. nums[i]参与了乘积，即子数组是0...i
2. 仅nums[i]参与了乘积，即子数组是i...i

根据最终求得的乘积谁更大来确定。

但是这道题还有一个注意点，由于数组里存在负数，负数会把最大值变为最小值，因此我们需要存储子问题的最大最小值，遇到负数后，需要交换最大最小值来保证正确性即

* 首先我们需要两个变量存储当前的最大、最小乘积，然后遍历数组
* 当前nums[i]为正数时，乘以最大、最小值来更新
* 当前nums[i]为负数时，最大最小值会变换符号，所以我们交换最大最小值，然后继续计算
* 当仅有nums[i]参与的情况，尝试更新最大最小值

实现代码如下：
```golang
func maxProduct(nums []int) int {
    n := len(nums)
    
    res := nums[0]
    max := nums[0]
    min := nums[0]
    for i := 1; i<n; i++ {
        min = min * nums[i] // 分别计算当前位置累计乘积的最大和最小乘积
        max = max * nums[i] 
        if nums[i] < 0 { // 如果nums[i]小于零，交换max和min
            min, max = max, min
        }
        
        if nums[i] < min { 
            min = nums[i]
        }
        if nums[i] > max { // 尝试用当前数单个作为乘积更新累计乘积最大最小值
            max = nums[i]
        }
        if max > res { // 更新res
            res = max
        }
    }
    return res
}
```

## [House Robber](https://leetcode.com/problems/house-robber/) and [House Robber II](https://leetcode.com/problems/house-robber-ii/)
(这两道题类似所以放在一起写)
`给长度为n的非负整数数组，条件是不能从取相邻的两个数，问取出的数的和最大是多少？`

### 分析求解
对于数i，有取或者不取两种情况，取的话前一个数就不能取，不取的话就可以取前一个数，因此当我们用f(i)代表前i个数能取到的最大和时，有状态转移方程：

<center>f(i) = max(f(i-2) + nums[i], f(i-1))</center>

一个非常典型的dp，同时由于只依赖f(i-2)和f(i-1)，我们可以只用两个数来存储这两个状态。

实现代码如下：
```golang
func rob(nums []int) int {
    n := len(nums)
    
    if n == 0 { return 0}
    if n == 1 { return nums[0]}
    
    prepre := nums[0]
    pre := nums[1]
    if prepre > pre {
        pre = prepre
    }
    
    for i := 2; i<n; i++ {
        t := prepre + nums[i]
        if pre > t {
            t = pre
        }
        prepre = pre
        pre = t
    }
    if prepre > pre {
        pre = prepre
    }
    return pre 
}
```

### House Robber II
II改为了数组是一个环，第一个数和最后一个数也是相邻的，这里我们只需要加一种情况即是否选第一个数，即从第一个数开始，还是从第二个数开始，然后用上一个题的求解即可，即f(n) = fpre(0) + fpre(1)，[实现代码](https://leetcode.com/submissions/detail/390717793/)就不贴了。。

## [Maximal Square](https://leetcode.com/problems/maximal-square/)
`一个有0和1组成的二维数组，求其中由全1组成的正方形的最大面积`

### 分析求解
很直观的DP题，以i,j为正方形的右下角，则它能组成正方形的最大边长等于(i-1,j-1)、(i-1,j)、(i,j-1)的最小值加1，同时遍历时由于只依赖前一行的数据，可以用两个数组来存储状态求解。注意这里由于依赖(i-1,j-1)点，所以没办法优化成一个数组。
状态方程如下：
<center>f(i,j) = min(f(i-1,j-1), f(i-1,j), f(i,j-1)) + 1

这里有条件是nums[i][j]为1时</center>

实现代码如下:

```golang
func maximalSquare(matrix [][]byte) int {
	n := len(matrix)
	if n == 0 { return 0 }
	m := len(matrix[0])
	dp := make([]int, m+1)
    dp1 := make([]int, m+1)
    
	max := 0
	for i := 1; i<= n; i++ {
		for j := 1; j<= m; j++ {
			if matrix[i-1][j-1] == '1' {
				t := dp1[j-1]
				if dp1[j] < t {
					t = dp1[j]
				}
				if dp[j-1] < t {
					t = dp[j-1]
				}

				t += 1
				dp[j] = t
            } else {
                dp[j] = 0
            }


			if dp[j] > max {
				max = dp[j]
			}
		}
        
        //dp1 = dp
        for j := 1; j<= m; j++ {
            dp1[j] = dp[j]
		}
	}
	return max * max
}
```

### 坑
这里有几个不得不说的。。。
* 输入是个byte数组，但是判断的时候用的是'1'字符，我以为是用1，查了好久才查到问题
* go没有标准库的min/max函数，确实很蛋疼
* 我注释的那一行`dp1=dp`，也是一个坑，go的两个slice赋值时其实是底层数据是共享的，那样直接`dp1=dp`，修改dp1的值会影响到dp的值，造成子状态错误

# EOF
本周的总结完成了，全是DP的题，还是有一些收获，并且在写blog的过程中也会发现一些没有理解透彻的知识点，再接再厉！