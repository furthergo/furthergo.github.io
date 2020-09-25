---
title: weekly-algorithm-20200926
tags:
  - algorithm
categories:
  - Weekly Algorithm
keywords:
  - Ones And Zeros
  - Predict the Winner
  - Target Sum
  - Longest Palindromic Subsequence
  - Out of Boundary Paths - Minimax - Dynamic Programming - knapsack Algorithm
date: 2020-09-25 22:12:14
---


# 总结
本周一共刷了5道medium，巩固了一下背包算法，以及了解了一下Minimax算法的含义和使用场景。

# [Ones And Zeros](https://leetcode.com/problems/ones-and-zeroes/)
```
给一个字符串数组strs，其中的字符串只包含'0'或者'1'，输入m和n分别代表可以使用的0和1的个数，问这m个0和n个1，最多能从数组里取出多少个字符串来组成？

比如输入：strs = ["10","0001","111001","1","0"], m = 5, n = 3
输出： 4
解释: 用5个0和3个1最多可以取出"10","0001","1","0"四个字符串
```
## 分析
这是一道典型的背包问题，和那道取硬币的题类似，区别在于背包问题只有重量一个限制，假如限制重量是C，可以看做是C个1。而这道题是有0和1两个限制，相比于背包问题多了一个维度，但是解题的思路还是类似的。

类比背包问题的解题思路，我们用一个三维数组来存储取到第i个字符串时，消耗j个0和k个1最多能取到字符串的个数，记作f(i, j, k)。则有状态转移方程：
<center>
f(i, j, k) = max(f(i-1, j, k), f(i-1, j - zero(strs[i]), k - one(strs[i]))))

其中zero(strs[i])和one(strs[i])分别代表第i个字符串包含的0和1的个数
</center>

## 实现

根据之前常用的优化空间复杂度的套路，观察状态转移方程可以看出：
1. f(i, j, k)只依赖f(i-1)的状态，所以可以优化dp为二维数组;
2. 只有当j大于等于当前0的个数，且k大于等于当前1的个数时，才有第二种可能，否则f(i,j,k)=f(i-1,j,k)，即不需要更新;
3. 根据2可以得出，我们只需要遍历j和k到当前字符串的0和1的个数处即可

实现如下：
```golang

func zerosAndOnes(str string) (z, o int) {
	l := len(str)
	for _, v := range str {
		if v == '0' {
			z++
		}
	}
	return z, l - z
}
func findMaxForm(strs []string, m int, n int) int {
	l := len(strs)
	if l == 0 {
		return 0
	}
	dp := make([][]int, m+1)
	for i := 0; i <= m; i++ {
		dp[i] = make([]int, n+1)
	}
	for i := 0; i < l; i++ {
		z, o := zerosAndOnes(strs[i])
		for j := m; j >= z; j-- {
			for k := n; k >= o; k-- {
				if (dp[j-z][k-o] + 1) > dp[j][k] {
					dp[j][k] = dp[j-z][k-o] + 1
				}
			}
		}
	}
	return dp[m][n]
}
```



# [Predict the Winner](https://leetcode.com/problems/predict-the-winner/)
```
给定一个只包含非负整数的数组，有两个人会依次交替从这个数组的两端任意取走一个数字，取走的原则是最后自己能取到的所有的数的和比另一个人取走的数的和更大。假设两个人都能做出最优的判断，问先取的人最后能否取得胜利？

比如：
输入: [1, 5, 2]
输出: false
解释: 因为第一个人取走1或者2之后，第二个人会取5，5 > 2 + 1，所以第一个人会输
```

## 分析：递归解法
最直观的想这道题，我需要选择的数最后和尽可能大，那就是是让另一个人可选择的数的和尽可能的小，可以用递归求解。

1. 对于第一个人的每一次选择来说，有两种选法：选择当前剩下的数的第一个数或者最后一个数；
2. 应该选哪个数的原则，是让另一个人在剩下的数中能**选择的数的最大和**尽可能小。
3. 假如当前在index为i到j的范围内选，选择index为i的数，那么另一个人就是能在i+1到j的范围选择；选择index为j的数，那么另一个人就是能在i到j-1的范围选择；
4. 当另一个人从i+1到j或者i到j-1选择的时候，从他的视角看，他还是会重复1-3的步骤
5. 对于第一个人来说，另一个人的两种选择

我们用f(i, j)代表当前选择的人，能从数组的index为i到index为j的子数组里，能取到数的和的最大值，那么我们可以推断出`f(i, j) = sum(i, j) - min(f(i+1, j), f(i, j-1))`，我们对f(i, j)递归求解，直到数据里只有两个数的时候，一定是选更大的那一个。

## 递归实现
```golang
func max(a int, b int) int {
	if a > b {
		return a
	}
	return b
}
func maxPickSum(sum int, nums []int) int {
	n := len(nums)
	if n == 0 {
		return 0
	}
	if n == 1 {
		return nums[0]
	}
	if n == 2 {
		return max(nums[0], nums[1])
	}

	t1 := sum - maxPickSum(sum-nums[0], nums[1:])
	t2 := sum - maxPickSum(sum-nums[n-1], nums[:n-1])
	return max(t1, t2)
}
func PredictTheWinner(nums []int) bool {
	sum := 0
	for _, v := range nums {
		sum += v
	}
	m := maxPickSum(sum, nums)
	if m*2 >= sum {
		return true
	}
	return false
}
```
注意这里需要有一个sum变量代表当前的总和，并且不断更新。

一开始以为递归解法会超时，没想到尝试提交了一些居然过了，但是有点慢，我们再从DP的角度求解。

## 分析：DP解法
还是上面推导出的等式 `f(i, j) = sum(i, j) - min(f(i+1, j), f(i, j-1))`。

* 很明显的可以看到假如我们用一个数组存储f(i, j)。
* 注意这道题有一点不一样的是，f(i, j)依赖的是f(i+1,j)，所以在遍历求解dp的时候，需要最后一行开始往上遍历。
* 再观察一下，f(i, j) 只依赖当前行或者后一行，常见套路，空间复杂度可以优化到O(n)
* 注意这里依赖

## DP实现
```golang

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}

func PredictTheWinner(nums []int) bool {
	n := len(nums)
	dp := make([]int, n)
	sum := 0
	dp[n-1] = nums[n-1]
	for i := n - 2; i >= 0; i-- {
		dp[i] = nums[i]
		sum = nums[i]
		for j := i + 1; j < n; j++ {
			sum += nums[j]
			// 状态转移方程
			// dp[i][j] = sum[i][j] - min(dp[i][j-1], dp[i+1][j])
			dp[j] = sum - min(dp[j-1], dp[j])
		}
	}
	return dp[n-1]*2 >= sum
}
```

同样的，这里需要sum变量，代表当前sum(i, j)。i等于n-1时，只有一个数就是nums[n-1]，所以dp[n-1]=nums[n-1]。然后i从n-2即最后一行开始遍历。

# [Target Sum](https://leetcode.com/problems/target-sum/)
```
给定一个不包含负数的整数数组，和一个目标值S，用数组里的每一个数加权求和，问有多少种加权的方式最后和为S？这里每个数都可以加+1或者-1的权。

有一些条件限制：
1. 数组长度不超过20
2. 数组的元素之和不超过1000


比如输入： [1, 1, 1, 1, 1], S 是 3
输出: 5
解释: 
-1+1+1+1+1 = 3
+1-1+1+1+1 = 3
+1+1-1+1+1 = 3
+1+1+1-1+1 = 3
+1+1+1+1-1 = 3
共有5中加权方式。
```

## 分析
又双叒叕是一道背包的题型：
```
从数组中以某种姿势选择元素，然后以某种姿势组成一个有限制的输出。
问：一共有多少种可能的姿势？
```

* 常见的思路，就是求解取到第i个元素时，组成j的可能的姿势的个数，而这个个数往往可以由到i-1的某个j的个数决定，我们只需要遍历从第一个元素到最后一个元素；
* 比如这道题，目标值是S，每个数可以加+1权或者-1权立刻可以得出等式：`f(i, j) = f(i-1, j-nums[i]) + f(i-1, j+nums[i])`

但是，注意这道题，加权是正或者负，也就是说:
```
之前的数的和可能存在负数，而负数是不能作为数组下标的
```

为了继续使用DP，我们可以看到题目有个限制是**数组的所有元素和不超过1000**，就表示最小的和是-1000（所有的权都是-1）。我们可以给数组下标加个1000的偏移，0代表和为-1000， 1000代表0，2000代表1000… j代表和为j-1000

## 实现
用dp[i][j]代表数组从0-i的范围内，能组成和为j-1000的可能的方式，则`dp[i][j] = dp[i-1][j-nums[i]] + dp[i-1][j+nums[i]]`。
* 注意这里j的范围是0到1000*2，所以需要判断当前的j-nums[i]和j+nums[i]在这个范围内。
* 由于只依赖dp[i-1]，优化为两个一位数组，O(n)的空间复杂度

实现代码如下：
```golang
func findTargetSumWays(nums []int, S int) int {
    MAXS := 1000
    if S >  MAXS || S < -MAXS {
        return 0
    }
    n := len(nums)
    dp := make([]int, MAXS*2+1)
    preDp := make([]int, MAXS*2+1)
    
    preDp[MAXS] = 1
    
    for i := 1; i<=n; i++ {
        for j := 0; j<=MAXS*2; j++ {
            // dp[i][j]代表从0-i，能组成和为j-1000的所有可能的个数
            // 等于从0-i-1，能组成和为j-1000-nums[i]的个数加上能组成和为j-1000+nums[i]的个数
            // 即 j-1000-nums[i]的个数等于dp[i-1][j-1000-nums[i]+1000] = dp[i-1][j-nums[i]]
            // 即状态转移方程： dp[i][j] = dp[i-1][j-nums[i-1]] + dp[i-1][j+nums[i-1]]
            // 防止越界
            // 优化空间复杂度
            if j >= nums[i-1] {
                dp[j] = preDp[j-nums[i-1]]
            }
            if j + nums[i-1] <=MAXS*2 {
                dp[j] += preDp[j+nums[i-1]]
            }
        }
        for j := 0; j<=MAXS*2; j++ {
            preDp[j] = dp[j]
        }
    }
    return dp[S+1000]
}
```

这里当S超过了最大或最小的可能时，直接返回0。

# [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence/)
```
给定一个字符串str，求这个字符串的最长回文子序列的长度是多少？
比如输入bbbab
输入：4
最长回文子序列是bbbb
```

## 分析
DP经典题目，最重要的还是寻找状态转移方程，我们用`s(i,j)代表字符串从index i到j的范围内最长回文子序列`，`f(i,j)代表s(i,j)的长度`，下面来分析求解s(i,j)和f(i,j)的过程

1. 假如`str[i]==str[j]`，那么表示对于要求的s(i,j)，我们可以在s(i+1,j-1)的两端分别添加str[i]和str[j]，这样能组成最长回文子序列即s(i,j)
2. 可以得出从i到j的字符串的最长回文子序列的长度 `f(i,j) = f(i+1,j-1) + 2`

以上是str[i]==str[j]的情况，假如str[i] != str[j]呢？

3. 即对于我们要求的s(i, j)来说，由于`str[i] != str[j]`，那么`不可能同时在s(i+1,j-1)的两端添加str[i]和str[j]`来构成s(i,j)
4. 要么str[i]添加进s(i+1,j-1)的左边，要么str[j]添加进s(i+1,j-1)的右边，同时我们不知道这种添加是否会破坏s(i+1,j-1)的回文特性，所以不能直接用f(i+1, j-1)+1
5. 则对于这两种情况我们分别需要重新计算s(i, j-1)和s(i+1, j)。

最终可以得出 f(i,j) = max{f(i+1, j-1)&&str[i]==str[j], f(i,j-1), f(i+1,j)}，其中当str[i]==str[j]时，f(i+1, j-1)+2一定是最大的

## 求解
用dp[i][j]代表f(i,j)，由于dp[i][j]依赖dp[i+1][j-1]和dp[i+1][j]，我们i从右开始遍历，j从左开始遍历，实现代码如下：
```golang
//省略max函数定义
func longestPalindromeSubseq(s string) int {
	n := len(s)
	dp := make([][]int, n)
	for i := 0; i < n; i++ {
		dp[i] = make([]int, n)
		dp[i][i] = 1
	}
	for i := n - 2; i >= 0; i-- {
		for j := i + 1; j < n; j++ {
			if s[i] == s[j] {
				dp[i][j] = dp[i+1][j-1] + 2
			} else {
				dp[i][j] = max(dp[i+1][j], dp[i][j-1])
			}
		}
	}
	return dp[0][n-1]
}
```
其中dp[i][i]=1，代表只有str[i]一个字符。

## 优化
观察dp[i][j]的求解可以看出，依赖当前行j-1的状态，同时依赖下一行相同位置j和左边j-1的状态，因此我们需要两行数组来存储下一行和当前行的状态。优化代码如下:
```golang
// 省略max函数定义
func longestPalindromeSubseq(s string) int {
	n := len(s)
	if n == 1 {
		return 1
	}
	dp := make([]int, n)
	nextlineDp := make([]int, n)
	nextlineDp[n-1] = 1
	for i := n - 2; i >= 0; i-- {
		dp[i] = 1
		for j := i + 1; j < n; j++ {
			if s[i] == s[j] {
				dp[j] = nextlineDp[j-1] + 2
			} else {
				dp[j] = max(nextlineDp[j], dp[j-1])
			}
		}
		for j := i; j < n; j++ { // 更新下一行的状态
			nextlineDp[j] = dp[j]
		}
	}
	return dp[n-1]
}
```
最近发现通过观察代码来优化空间复杂度往往更简单，因为i+1,i-1这种表示更直观，不过还是想从原理的角度来分析一下优化的原理，以这道题为例：
```golang
// f(i,j) = max{f(i+1, j-1)&&str[i]==str[j], f(i,j-1), f(i+1,j)}
if s[i] == s[j] {
	dp[i][j] = dp[i+1][j-1] + 2
} else {
	dp[i][j] = max(dp[i+1][j], dp[i][j-1])
}
```
dp是一个矩阵，当求dp[i][j]时，很容易知道只依赖i+1行的状态，所以第一步space从O(n^2)到O(2n)，再其次
1. 第i行依赖当前行左边的数据
2. 第i行依赖i+1左边的数据和当前位置的数据

由1我们知道当前行比如从左往右开始求解，再加上2，还依赖i+1左边的数据，则不能用当前行来保存i+1行的左边的状态，因为会被当前行覆盖掉，所以这道题最少需要两个一维数组来存储求解的子状态。

对于一些状态转移方程比如`dp[i][j] = max(dp[i-1][j-1], dp[i-1][j])`和`dp[i][j] = max(dp[i][j-1], dp[i-1][j])`这种，我们只单个依赖当前行或者上一行左边的状态，就可以用当前行来存储上一行左边的状态，即`dp[j]=max(dp[j-1],dp[j])`当前行从右边开始遍历和`dp[j]=max(dp[j-1],dp[j])`当前行从左边开始遍历。

总结一下就是当不会依赖两行的同一位置的状态时，space可以优化到O(n)一个一维数组。

# [Out of Boundary Paths](Out of Boundary Paths)
```
给定一个m*n的表格和一个足球，足球一开始放在表格的（i，j）坐标处，足球可以往上下左右四个方向移动，给定总的可以走的步数N，问足球有多少种走出表格边界的方式？最后的结果取模1000000007。

其中 1<=m,n <=50, 0<N<50

比如：输入m = 2, n = 2, N = 2, i = 0, j = 0
输出：6
```

## DFS
很简单的一道题，DFS即可，根据函数原型就能迅速的写出递归代码，实现如下：
```golang
func findPaths(m int, n int, N int, i int, j int) int {
	if i < 0 || j < 0 || i >= m || j >= n {
		return 1
	}
	if N <= 0 {
		return 0
	}
	return findPaths(m, n, N-1, i-1, j) + findPaths(m, n, N-1, i+1, j) + findPaths(m, n, N-1, i, j-1) + findPaths(m, n, N-1, i, j+1)
}
```
分别往四个方向深度优先搜索所有可能的路径：
1. i和j超出边界时N到0的时候表示当前这条路径可以走出表格边界，返回1；
2. 否则N-=0时表示没有可以消耗的步数了，返回0；
3. N != 0时，继续搜索。

当然，不出意外超时了。。。。😂

仔细分析一下：
1. 我们从(i,j)出发，搜索向上(i-1,j)、向左(i, j-1)、向下搜索(i+1,j)和向右搜索(i,j+1)
2. 向上(i-1,j)和向左(i, j-1)时：(i-1,j)会向左搜索(i-1,j-1)，(i,j-1)会向上搜索(i-1,j-1)，这里会产生重复的搜索(i-1,j-1)
3. 综上所述搜索过程中会有很多的重复搜索

## DFS With Memo
根据上面DFS的分析，搜索过程中会有很多重复搜索，那么我们就要优化这些不必要的重复搜索，假如之前搜索过这条路径了，把结果存储起来，下一次有其它搜索需要这个路径的搜索结果时，直接取存储即可，这样就避免了重复的搜索。

当然，*从(i,j)点搜索，剩余不同的步数，结果是不一样的*，所以我们需要一个三维数组来存储这个搜索结果：**即用dp[N][i][j]表示，从(i,j)出发，有N步可用，走出边界的方式个数。**

同时，需要知道当前dp[N][i][j]是否是搜索过的，这一点我们可以在初始化数组的时候把值初始化为-1，由于dp[N][i][j]最小是0，当dp[N][i][j]的值大于等于0即表示dp[N][i][j]已经搜索过。

实现代码如下：

```golang
func mod1097(num int) int {
	mod := 1000000007
	return num % mod
}
func findPathsWithDP(m int, n int, N int, i int, j int, dp [][][]int) (int, [][][]int) {
	if i < 0 || j < 0 || i >= m || j >= n {
		return 1, dp
	}
	if N == 0 {
		dp[N][i][j] = 0
		return 0, dp
	}
	if dp[N][i][j] != -1 {
		return dp[N][i][j], dp
	}
	t1, dp := findPathsWithDP(m, n, N-1, i-1, j, dp)
	t2, dp := findPathsWithDP(m, n, N-1, i+1, j, dp)
	t3, dp := findPathsWithDP(m, n, N-1, i, j-1, dp)
	t4, dp := findPathsWithDP(m, n, N-1, i, j+1, dp)
	t := mod1097(t1) + mod1097(t2) + mod1097(t3) + mod1097(t4)
	dp[N][i][j] = mod1097(t)
	return dp[N][i][j], dp
}
func findPaths(m int, n int, N int, i int, j int) int {
	if N == 0 {
		return 0
	}
	dp := make([][][]int, N+1)
	for t := 0; t <= N; t++ {
		dp[t] = make([][]int, m)
		for i := 0; i < m; i++ {
			dp[t][i] = make([]int, n)
			for j := 0; j < n; j++ {
				dp[t][i][j] = -1
			}
		}
	}
	t, _ := findPathsWithDP(m, n, N, i, j, dp)
	return mod1097(t)
}
```

* 这里做的时候卡了很久，一直不知道应该怎么样保证计算过程结果不溢出，后来看题解才知道模运算是满足分配律，搜索过程中用模的结果当做dp值完全没有问题：`a%m + b%m = (x*m + c)%m + (y*m + d)%m = (x+y)*m + (c + d) = ((x+y)*m + (c+d))%m = (a+b)%m`
* 做的时候一直把这个解法叫做DFS with DP，因为我们是用一个数组去存储求解的子问题，只不过求解的过程是DFS，而不是常见的遍历

## DP
观察上面的DFS解法，可以看到其实对于N来说，dp[N][i][j]依赖dp[n-1]的状态，我们可以顺序遍历N，用非递归的解法求解。实现代码如下：

```golang
func findPaths(m int, n int, N int, i int, j int) int {
	if N == 0 {
		return 0
	}
	dp := make([][][]int, N+1)
	for t := 0; t <= N; t++ {
		dp[t] = make([][]int, m)
		for i := 0; i < m; i++ {
			dp[t][i] = make([]int, n)
		}
	}
	for t := 1; t <= N; t++ {
		for i := 0; i < m; i++ {
			for j := 0; j < n; j++ {
				k := 0
				if i == 0 {
					k += 1
				} else {
					k += dp[t-1][i-1][j]
				}
				if i == m-1 {
					k += 1
				} else {
					k += dp[t-1][i+1][j]
				}
				if j == 0 {
					k += 1
				} else {
					k += dp[t-1][i][j-1]
				}
				if j == n-1 {
					k += 1
				} else {
					k += dp[t-1][i][j+1]
				}
				k = mod1097(k)
				dp[t][i][j] = k
			}
		}
	}
	return dp[N][i][j]
}
```

再优化一下，其实只需要两个二维数组即可，实现如下：
```golang
func findPaths(m int, n int, N int, i int, j int) int {
	if N == 0 {
		return 0
	}
	dp := make([][]int, m)
	preDp := make([][]int, m)
	for i := 0; i < m; i++ {
		dp[i] = make([]int, n)
		preDp[i] = make([]int, n)
	}
	for t := 1; t <= N; t++ {
		for i := 0; i < m; i++ {
			for j := 0; j < n; j++ {
				k := 0
				if i == 0 {
					k += 1
				} else {
					k += preDp[i-1][j]
				}
				if i == m-1 {
					k += 1
				} else {
					k += preDp[i+1][j]
				}
				if j == 0 {
					k += 1
				} else {
					k += preDp[i][j-1]
				}
				if j == n-1 {
					k += 1
				} else {
					k += preDp[i][j+1]
				}
				k = mod1097(k)
				dp[i][j] = k
			}
		}
		for i := 0; i < m; i++ {
			for j := 0; j < n; j++ {
				preDp[i][j] = dp[i][j]
			}
		}
	}
	return dp[i][j]
}
```

# EOF
写的头晕了，最近总是在挑选点赞高的medium题做，又陷入了“舒适区“……下周加油，继续跳出舒适区！