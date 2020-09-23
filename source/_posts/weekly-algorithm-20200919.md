---
title: '[Weekly Algorithm]20200919: Dynamic programming III'
date: 2020-09-19 14:32:52
tags:
  - algorithm
categories:
  - Weekly Algorithm
keywords:
  - Combination Sum IV
  - Arithmetic Slices
  - Partition Equal Subset Sum
---

# 0x00 总结
* 本周一共刷了三道题，medium
* 两道常规dp题，一道背包问题，会重点介绍一下这道背包问题

# 0x01 [Combination Sum IV](https://leetcode.com/problems/combination-sum-iv/)
```
给定一个无重复数字的正整数数组和一个target，尝试从数组中取和为target的数字，问共有多少种数字的组合？
```
```
比如nums = [1, 2, 3]，target = 4
所有可能的组合有7种:
(1, 1, 1, 1)
(1, 1, 2)
(1, 2, 1)
(1, 3)
(2, 1, 1)
(2, 2)
(3, 1)

注意不同数字顺序算不同的组合，比如(1, 1, 2)和(1, 2, 1)是两种
```
<!-- more -->
## 分析
根据上面的例子分析，和为4的所有的数字的组合，等于

1. 和为3的所有的数字组合里添加1（假如数组里包含1），加上
2. 和为2的所有的数字组合里添加2（假如数组里包含2），加上
3. 和为1的所有的数字组合里添加3（假如数组里包含3），加上
4. 和为0的所有的数字组合里添加4（假如数组里包含4）.

假设f(n)为和为n的所有数字的组合个数，c(i)代表数组nums里包含数字i，则

<center>
f(n) = f(n-1)&&c(1) + f(n-2)&&c(2) + ... + f(0)&&c(n)

从0到n-1
</center>

这里的f(n)就是求解过程中的子问题，我们只需要从f(0)计算到f(target)即可，注意这里f(0)是边界case，由于都是正整数，f(0)就代表一个数组中的数也不选，有一种可能，即f(0)=1。

c(i)表示nums中是否存在i，如果按照上述的方法，我们需要从f(0)计算到f(target)，同时每一步都需要计算c(i)，时间复杂度太高了，可以换一个角度：**从nums里取数字**，然后用target减去这个数字。还是以上面的例子分析，和为4的所有的数字的组合，等于

1. 和为4-nums[0]=3的所有的数字组合，加上
2. 和为4-nums[1]=2的所有的数字组合，加上
2. 和为4-nums[2]=1的所有的数字组合.

即

<center>
f(n) = f(n-nums[0]) + f(n-nums[1])  + ... + f(n-nums[len(nums)-1]) 

注意这里需要nums[i]小于等于n
</center>

这里我们可以先对nums排一下序，遇到nums[i]>n就可以退出遍历了

## 求解
初始化dp[target+1]存储f(n)，其中`f(i) = dp[i], dp[0] = f(0) = 1`，实现代码如下:
```golang
func combinationSum4(nums []int, target int) int {
    dp := make([]int, target+1)
    dp[0] = 1
    sort.Ints(nums)
    for i := 1; i<=target; i++ {
        for _, v := range nums {
            if v <= i {
                dp[i] += dp[i-v]
            } else {
                break
            }
        }
    }
    return dp[target]
}
```

# 0x02 [Arithmetic Slices](https://leetcode.com/problems/arithmetic-slices/)
```
首先：
定义Arithmetic Sequence代表序列中任意两个相邻的元素的差值都相同。

给定一个数组A，求这个数组有多少个切片是Arithmetic的，注意这里切片是Arithmetic的前提是切片的长度大于等于3。

比如
A = [1, 2, 3, 4]

返回为3, 代表A有三个Arithmetic的切片: [1, 2, 3], [2, 3, 4] 和 [1, 2, 3, 4].
```

## 分析
首先最暴力的解法，求出A的所有的可能的切片，然后判断每个切片是不是Arithmetic的，A的长度为n，则共有 n-1 + n-2 + … + 1 = n(n-1)/2 个长度至少为2的切片，还需要判断每个切片是否是Arithmetic，时间复杂度是O(n^3)。下面我们从暴力解法出发，尝试优化一下时间复杂度：

1. 所有的切片可以分类为从index为0开始的切片，从index为1开始的切片，…… ， 从index为n-2开始的切片；
2.  假如从index为i开始，长度为l的切片是Arithmetic的，那么
    * 从index为i开始，长度为3、… 、l的切片都是Arithmetic的，一共有l-2个；
    * 从index为i+1开始，长度为3、… 、l-1的切片都是Arithmetic的，一共有l-3个；
    * 从index为i+2开始，长度为3、… 、l-2的切片都是Arithmetic的，一共有l-4个；
    * …
    * 从index为i+l-3开始，长度为3的切片都是Arithmetic的，一共有1个；
    * 即一共有(l-2) + (l-3) + … + 1 = (l-1)*(l-2)/2个
3. 在从index为i开始的切片里，假如长度为l的切片不是Arithmetic的，那么长度为l+1的切片一定也不是Arithmetic的，这里就没有再往下计算的必要了；

那么综合上面三条分析，当我们从index为i开始，找到满足Arithmetic的长度最长的切片，假设最长为l，则**从index为i，i+1，… ，i+l-2开始的所有满足Arithmetic的切片都已经被找到了**，接下来我们只需要从index为i+l-1开始，重新找满足Arithmetic的长度最长的切片即可，即求解步骤为
1. index初始化0
2. 从index，找到index=0开始满足Arithmetic的最长切片，假设长度为l
3. 根据上述2更新res += (l-1)*(l-2)/2
4. index更新为i+l-2，重复步骤234
5. 因为Arithmetic的切片长度至少为3，则index最大为n-3

## 求解
实现代码如下：
```golang
func numberOfArithmeticSlices(A []int) int {
    res := 0
    n := len(A)
    
    for i := 0; i<n-2; {
        if A[i+1] - A[i] == A[i+2] - A[i+1] {
            j := i+2 // j代表当前slice的最后一个数的index
            for ; j<n-1; j++ { // j最大为n-2
                if A[j+1]-A[j] != A[j] - A[j-1] { // 尝试扩充slice的长度
                    break
                }
            }
            l := j - i + 1 // 最长arithmetic的切片长度
            res += (l-1)*(l-2)/2
            i = j - 1 
        } else {
            i++
        }
    }
    return res
}
```
这里虽然是两层循环，但是第一层循环的步长是由第二层遍历到的位置更新的，所以时间复杂度是O(n)，空间复杂度是O(1)。

## DP解法
上面的解法其实是暴力解法的优化版，这道题是DP分类里的，所以我们再来从DP的角度分析和求解。

* 用f(i)代表以A[i]结尾的所有的满足Arithmetic的切片的个数
* 求解f(i)时：
    1. 假如A[i]-A[i-1] == A[i-1]-A[i-2]，即(A[i-2], A[i-1], A[i])满足Arithmetic
    2. 则对于所有的f(i-1)的切片，A[i]-A[i-1] == A[i-1]-A[i-2] == A[i-2]-A[i-3] == ...
    3. 即假如1满足，那么以A[i-1]结尾的所有满足Arithmetic的切片之后，添加一个A[i]，同样是Arithmetic的
    4. 同时有一个新的满足Arithmetic的切片(A[i-2], A[i-1], A[i])产生
* 即f(i) = f(i-1)+1

我们用dp[i]代表f(i)，则状态转移方程为：
<center> dp[i] = dp[i-1] + 1 

当A[i]-A[i-1] == A[i-1]-A[i-2]，否则

dp[i] = 0
</center>

同时观察到dp[i]只依赖dp[i-1]，则我们只需要一个变量来存储前一个状态即可，实现代码如下：
```golang
func numberOfArithmeticSlices(A []int) int {
    res := 0
    n := len(A)
    dp := 0
    for i := 2; i<n;i++ {
        if A[i] - A[i-1] == A[i-1] - A[i-2] {
            dp += 1
            res += dp
        } else {
            dp = 0
        }
    }
    return res
}
```

时间复杂度是O(n)，空间复杂度是O(1)

# 0x03 [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/)
```
给定一个只包含正整数的非空数组，判断数组是否能被划分成两个和相等的子集。

比如
输入: [1, 5, 11, 5]
输出: true
数组可以被划分为 [1, 5, 5] and [11].
```

## 分析
被划分成两个和相等的子集，首先要满足：
* 数组个数大于1
* 数组元素的和是偶数

在满足上面两个条件之后，我们可以先求出数组元素和的一半half，接下来问题就转换成，从数组中找子集，满足子集元素的和等于half。

这是一个典型的背包问题，找子集满足子集元素的和等于half，那么对于数组中的每个元素，我们可以把它放到最后的子集里，或者不放到最后的子集里，这里子集看成一个背包，就是一个0-1背包问题。暴力的解法就是DFS，对于每个元素尝试放或者不放到子集里，用暴力的方法写了一下，结果超时了：
```golang
func dfs(nums []int, target int) bool {
	if target == 0 {
		return true
	}
	n := len(nums)
	if n == 0 || target < 0 {
		return false
	}
	return dfs(nums[:n-1], target) || dfs(nums[:n-1], target-nums[n-1])
}
func canPartition(nums []int) bool {
	n := len(nums)
	sum := 0
	for i := 0; i < n; i++ {
		sum += nums[i]
	}
	if sum%2 != 0 {
		return false
	}
	half := sum / 2
	return dfs(nums, half)
}
```

既然直接DFS不行，还是从DP的角度来分析：求是否有满足和为target的子集，我们只需要从求和有为1的子集开始，和为2，… ，从这些子问题求最终是否存在和为target的子集。

假如我们知道了前i-1个数的子集能组成的和的结果，那么对于前i个数来说，求能否有一个子集的和为j时，有两种可能：
1. 元素i不在这个子集里，即前i-1个数里有一个和为j的子集
2. 元素i在这个子集里，即前i-1个数里有一个和为j-nums[i]的子集，这里需要nums[i]小于等于j
3. 12都不满足，则前i个数中不存在和为j的子集

我们用dp[i][j]代表0-i的数里有和为j的子集，则可以得出：

<center>dp[i][j] = dp[i-1][j] || dp[i-1][j-nums[i]]
</center>

我们可以用一个n行，target+1列的矩阵来存储计算结果，同时我们观察上述状态转移方程，dp[i][j]只依赖前一行的左边的计算结果，则可以优化dp矩阵为单行，这个优化方式之前的文章里写过，不再赘述。

因为这里依赖dp[i-1][j-nums[i]]，所以最多只能优化到一行，其次由于依赖左边的计算结果，我们需要倒序遍历更新dp数组，防止当前行的计算结果覆盖前一行的结果，影响当前行的计算

## 求解
实现代码如下：
```golang
func sumToTarget(nums []int, target int) bool {
	n := len(nums)
	dp := make([]bool, target+1)
	dp[0] = true

	for i := 0; i<n; i++ {
		for j := target; j>=1; j-- { // 因为dp[j]依赖前一行的j之前的元素的状态，反向遍历防止前一行元素的状态被覆盖
			// dp[j]代表0-i的元素里，有和为j的子集
			// dp[j] = dp[j] || dp[j-nums[i]]
			if nums[i] <= j {
				dp[j] = dp[j] || dp[j-nums[i]]
            } else {
                break // j是递减的，之后的元素都比nums[i]小
            }
		}
	}
	return dp[target]
}

func canPartition(nums []int) bool {
	n := len(nums)
	if n <= 1{ return false }

	sum := 0

	for i := 0; i<n; i++ {
		sum += nums[i]
	}
	if sum % 2 != 0 { return false }

	half := sum / 2
	return sumToTarget(nums, half)
}
```

时间复杂度为O(n*target)，空间复杂度为O(target)。

# EOF
* 背包问题，这个需要后面碰到后再总结
* 心血来潮的自律不如日积月累的好习惯，养成好的习惯很重要