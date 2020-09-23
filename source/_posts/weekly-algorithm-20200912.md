---
title: '[Weekly Algorithm]20200912: Dynamic programming II'
tags:
  - algorithm
categories:
  - Weekly Algorithm
keywords:
  - Ugly Number
  - Ugly Number II
  - Perfect Squares
  - Range Sum Query - Immutable
  - Rnage Sum Query 2D - Imuutable
  - Best Time to Buy and Sell Stock
  - Best Time to Buy and Sell Stock II
  - Best Time to Buy and Sell Stock with Cooldown
  - Coin Change
  - Integer Break
  - Count Numbers with Unique Digits
date: 2020-09-13 00:02:30
---


# 0x00 本周刷题总结
* 一共刷了11道题，主要还是DP专题，有一些题比如买股票的题目类似，为了理解题意和其中的差距，把之前的题也一起做了
* 收获：理解DP的常见套路 
* 这周的部分分析不会这么细，比较浪费时间，相同题型或者某些优化点不再赘述。

# 0x01 [Ugly Number](https://leetcode.com/problems/ugly-number/) and [Ugly Number II](https://leetcode.com/problems/ugly-number-ii/)

## [Ugly Number](https://leetcode.com/problems/ugly-number/)
```
检查输入的正整数num是否是Ugly数，Ugly数的定义是一个质因子只有2、3、5的正整数。
```
### 分析与求解
难度是Easy，分析一下可以得出解题思路：不断的尝试用2、3、5整除数字num，知道2、3、5都无法被整除或者num被除到1，代码如下：
```golang
func isUgly(num int) bool {
    if num <= 0 {
        return false
    } 
    for ; num>1;  {
        if num%5 == 0 {
            num /= 5
        } else if num%3 == 0 {
            num /= 3
        } else if num%2 == 0 {
            num /= 2
        } else {
            return false
        }
    }
    return true
}
```
<!-- more -->
## [Ugly Number II](https://leetcode.com/problems/ugly-number-ii/)
```
输入n，输出第n个Ugly Number，Ugly数的定义是一个质因子只有2、3、5的正整数。
```

### 思路分析
* 常规的暴力方法是，从1开始步长为1遍历，分别计算每个数是否是Ugly Number并累计，直到第n个。但是对于每个数字，计算它是否是Ugly Number的时候，都要不断的用2、3、5除它，时间复杂度很高；之后我也尝试把计算过程中每个数是否是Ugly Number的计算结果存在数组里，给之后的数使用，结果是[out of memory](https://leetcode.com/submissions/detail/391586808/)。

* 思考了一下根本原因， 还是求Ugly Number的时候都是乘法关系，这样越往后就会越稀疏，步长为1的遍历效率过于低了。

* 生产者模式：我们可以换一个角度思考问题，遍历加校验的方式，其实是消费者的角度，不断的消费每个数，从中过滤出Ugly Number。既然消费者的角度行不通，我们可以从生产者的角度去求解，即尝试去**生产Ugly Number**。

* 我们用已有的Ugly Number，乘以2、3、5，就可以产生新的Ugly Number，然后存下来，再用这些数继续生成新的Ugly Number这样的时间复杂度和空间复杂度都降到了O(n)。

### 问题求解

* 我们用一个数组dp[n]来存储已经得到的Ugly Number，一开始我们只有一个Ugly Number即，取出1乘以2，3，5生成了三个Ugly Number，因为题目是求的第n个Ugly Number，所以我们要保证存在dp中的数组的有序性，这里取2、3、5其中最小的数2添加到dp数组的末尾。

* 下一次我们取其中最小的数2然后分别乘以2、3、5得到4、6、10，这是可以发现，3和5还没有被放进dp数组中，因为上一个Ugly Number即1所生产的Ugly Number还没有全部被消费到dp数组里，即当前用来生产乘2的种子是2，但是生产乘3和乘5的种子还是1。这里我们取出2乘以2，取出1乘以3和5，得到3个4、3、5，我们取出最小的数3添加到dp数组的末尾。以此类推下次我们应该消费种子2乘以2和3，种子1乘以5，得到4、6、5三个数，取出最小的数4添加到dp数组的末尾……

* 归纳一下上述的过程，我们需要一个dp数组存储所有已经生产的Ugly Number；其次我们需要三个index，分别记录应该用来乘2、3、5的种子的位置，这样才能保证所有Ugly Number被有序的添加到dp数组中

实现代码如下：
```golang
func min(a, b int) int {
    if a > b { return b }
    return a
}

func nthUglyNumber(n int) int {
    dp := make([]int, n)
    //idx记录2、3、5应该使用的种子的位置
	idx2 := 0
	idx3 := 0
	idx5 := 0
	dp[0] = 1

	for i := 1; i<n; i++ {
		c2 := dp[idx2] * 2
		c3 := dp[idx3] * 3
		c5 := dp[idx5] * 5

        dp[i] = min(min(c2, c3), c5)
        
		if c2 == dp[i] { idx2++ }
		if c3 == dp[i] { idx3++ } 
		if c5 == dp[i] { idx5++ }
	}
	return dp[n-1]
}
```
 注意这里迭代idx的方式，我们是判断c2、c3和c5是否和新添加进dp数组的数相等来更新对应的idx。因为对于一些数就是2、3、5相乘的结果的时候，比如6=2x3=3x2，10=2x5=5x2，这些数在计算过程中会重复出现，即c2、c3、c5可能会相等，通过这种方式刚好可以避免重复添加，比如乘2的种子3和乘3的种子2结果相等，那么idx2和idx3都会向后移，代表生产的数已经被消费。

# 0x02 [Perfect Squares](https://leetcode.com/problems/perfect-squares/)
输入正整数n，求最少需要多少个完美平方数能组成它

## 思路分析
一道典型的DP题，计算n最少需要多少个完美平方数组成（之后简写为f(n)），我们可以尝试用n先减去一个完美平方数，得到n'，那么n可以由f(n')+1个完美平方数组成。要求f(n)，只需要尝试所有可以用来减的完美平方数，然后求出其中的最小值，即为f(n)。状态转移方程如下：
<center>f(n) = min(f(n - i*i)+1)

其中i*i最大为n</center>

## 求解
用dp[n]=f(n)记录中间解，其中dp[0]=0代表需要0个数组成，从1遍历至n，实现代码如下
```golang
func numSquares(n int) int {
    if n == 1 { return 1 }
    dp := make([]int32, n+1)
    
    for i := 1; i<=n; i++ {
        var min int32 = int32(i)
        for j := 1; j*j<=i; j++ {
            t := 1 + dp[i-j*j]
            if t < min {
                min = t
            }
        }
        dp[i] = min
    }
    return int(dp[n])
}
```

# 0x03 [Range Sum Query](https://leetcode.com/problems/range-sum-query-immutable/) and [Rnage Sum Query 2D](https://leetcode.com/problems/range-sum-query-2d-immutable/)
题型和题意都相同，放在一起做和分析。

## [Range Sum Query](https://leetcode.com/problems/range-sum-query-immutable/)
```
给一个正数数组，求i到j的数的和(i<=j)。注意数组是不可变的，且会有多次query求和
```

### 分析求解
Easy难度，用f(n)代表当前数组0到n的数的和，则i到j的和等于f(j)-f(i-1)，因此我们只需要先求f(n)即可，而f(n)是最简单的求和，状态转移方程是`f(n) = f(n-1) + nums[n]`。实现代码如下：
```golang
type NumArray struct {
    dp []int
}
func Constructor(nums []int) NumArray {
    n := len(nums)
    
    dp := make([]int, n+1)
    
    for i := 1; i<=n; i++ {
        dp[i] = dp[i-1] + nums[i-1]
    }
    
    
    r := NumArray{
        dp: dp,
    }
    
    return r
}
func (this *NumArray) SumRange(i int, j int) int {
    return this.dp[j+1] - this.dp[i]
}
```
这里为了简化f(0)的计算，将dp数组长度声明为n+1，即`f(i) = dp[i+1]`，在构造过程中求解dp数组，query时使用。

## [Rnage Sum Query 2D](https://leetcode.com/problems/range-sum-query-2d-immutable/)
```
给定一个m*n矩阵，求其中任意矩形组成的矩阵中的数字和，输入是左上角和右下角的坐标
```
<div align=center> <img src ="https://leetcode.com/static/images/courses/range_sum_query_2d.png"/> </div>

### 分析求解
* 和上一题类似，只是从一维变成了二维，思路还是一样，任意矩形的sum，等于以（0，0）为左上角的大矩形的sum，减去左边的矩形sum，减去上面的矩形sum，再加上多减了一次的左上角的矩形的sum。

* 假设输入(l,t,r,b)代表矩形，s(l,t,r,b)代表矩阵的sum，则有公式`s(l,t,r,b) = (0,0,r,b) - s(0,0,l-1,b) - s(0,0,r, t-1) + s(0,0,l-1, t-1)`

* 接下来只需要在构造过程中计算存储s(0,0,i,j) = f(i,j) = dp[i,j]的值，和上一题类似的思路可以得到状态转移方程`f(i,j) = f(i-1,j) + f(i,j-1) - f(i-1,j-1) + matrix[i][j]`

实现代码如下（同样，这里为了简化计算过程中的边界case处理，把dp数组大小声明为m+1*n+1，即`f(i,j) = dp[i+1][j+1]`）：
```golang
type NumMatrix struct {
    dp [][]int
}

func Constructor(matrix [][]int) NumMatrix {
    m := len(matrix)
    if m == 0 { return NumMatrix{}}
    n := len(matrix[0])
    
    dp := make([][]int, m+1)
    for i:= 0; i<=m; i++ {
        dp[i] = make([]int, n+1)
    }
    
    for i := 1; i<=m; i++ {
        for j :=1; j<=n; j++ {
            dp[i][j] = dp[i-1][j] + dp[i][j-1] - dp[i-1][j-1] + matrix[i-1][j-1]
        } 
    }

    return NumMatrix {
        dp: dp,
    }
}

func (this *NumMatrix) SumRegion(row1 int, col1 int, row2 int, col2 int) int {
    return this.dp[row2+1][col2+1] - this.dp[row1][col2+1] - this.dp[row2+1][col1] + this.dp[row1][col1]
}
```

# 0x04 [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) / [II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/) / [with cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)
三道买卖股票的题。

## [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)
```
给定数组prices，长度为n，代表股票每天的价格，可以在任意天买入和卖出股票各一次。问能得到的最大利润是多少？
```

### 分析求解
最低点买入，最高点卖出即可，代码如下：
```golang
func maxProfit(prices []int) int {
    res := 0
    n := len(prices)
    if n == 0 { return res }
    min := prices[0]
    
    for i := 1; i <n ; i++ {
        if prices[i] < min { // 尝试更新股票最小价格
            min = prices[i] 
        } 
        t := prices[i] - min // 尝试更新最大利润
        if t > res {
            res = t
        }
    }
    return res
}
```

## [Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)
```
给定数组prices，长度为n，代表股票每天的价格，可以在任意天买入和卖出股票多次。问能得到的最大利润是多少？
```

### 思路分析
比上一题多了一个条件：可以多次买入和卖出，我们来结合上一题分析一下。

* 上一题最大利润的计算其实就是找到最大最小值，然后求差。即我们把股票的价格波动当做一个个山峰和山谷，上一题就是找到最高的山峰和最低的山谷，然后求高度之差。（当然这里都是要求山峰在后）
* 这一题可以买卖多次，很直观的可以想到，我们只需要在低点买入，高点卖出，然后不断重复这个操作，就可以获得最大值。类比之下，就相当于现在要找的不是股票价格的最大值和最小值，而是股票价格的极大值和极小值。
* 但是当我们开始写代码时候会发现，求极大极小值过于麻烦，我们再重新抽象分析问题，寻找更利于代码实现的解题方式：
    1. 当今天我们处于空仓时（手上没有股票），首先要做的是买入股票
    2. 当今天我们处于持仓时（已买入股票），依据明天股票的价格，我们有两种选择：
        * 明天的股票价格比今天的低，那么今天的持仓必须在今天卖掉，这样我就能保证手上持有的股票收益达到最高
        * 明天的股票价格比今天的高，那么至少在明天卖我可以赚钱，所以继续持有
        * 特殊的，当今天是最后一天时，必须卖出股票
    3. 重复12步骤

### 问题求解
我们用一个数记录当前持仓的股票买入时的价格，-1代表空仓，用res记录当前的利润，实现代码如下：
```golang
func maxProfit(prices []int) int {
    n := len(prices)
    
    res := 0
    
    buy := -1 // 持仓价格，-1代表空仓
    
    for i := 0; i<n; i++ {
        if buy == -1 {
            buy = prices[i] // 空仓时必须买入
        }
        if i == n-1 || prices[i] >= prices[i+1]  { // 今天是最后一天或者明天价格会降，以今天的价格卖出，更新收益
            res += (prices[i] - buy)
            buy = -1
        }
        // 否则，继续持有买入价为buy的股票
    }
    return res
}
```

## [Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)
```
给定数组prices，长度为n，代表股票每天的价格，可以在任意买入和卖出股票多次，但是卖出之后需要有一天的冷冻时间，然后才能买入。问能得到的最大利润是多少？
```

### 思路分析
加入了一个cooldown之后，买入的条件是，前一天不能有卖出，那么卖出的时机就会影响到下一次买入的时机，也就会影响到下一次的获利。状态变得复杂了。下面我们来分析一下怎么通过dp来解这道题：

* 买入的时机受卖出时机的影响
* 卖出的时机受买入的影响
* 卖出之后需要有至少一天的空仓期

综上我们分析出现在有三种状态：
1. 空仓（可买入）
2. 持仓
3. 刚清仓 （需要cooldown）

三种状态可能的转移方式分别是：
* 1 -> 1，继续空仓
* 3 -> 1，刚过cooldown的空仓
* 1 -> 2，空仓状态下买入，转移为持仓
* 2 -> 2，继续持仓
* 2 -> 3，卖出

根据状态转移，`我们可以遍历价格数组，计算每天是每个状态的最大收益，最后一天求三个状态最大收益的最大值，就是最后的最大收益`。假设：
* e(n)代表第n天空仓的最大收益
* b(n)代表第n天是持仓的最大收益
* c(n)代表第n天是cooldown状态的最大收益

根据上面的状态转移方式，我们可以得到三个状态转移方程：

* e(n) = max(e(n-1), c(n-1))
* b(n) = max(e(n-1), b(n-1) - prices[n])
* c(n) = b(n-1) + prices[n]

### 问题求解
根据上面的分析我们维护三个长度为n的dp数组分别代表上面三种状态的最大值，并且每次转移只依赖上一次的状态，我们可以压缩空间复杂度至O(1)，实现代码如下：
```golang
func max(a, b int) int {
    if a > b { return a }
    return b
}

func maxProfit(prices []int) int {
    n := len(prices)
    
    if n == 0 { return 0 }
    
    // 记录当前进行操作的最大收益
    dpEmpty := 0 // 当前是空仓
    dpBuyed := -prices[0] // 当前是持仓
    dpSelled := 0 // 当前处于刚清仓状态（需要cooldown冷静）
    
    for i := 1; i < n; i++ {
        dpEmptyT := max(dpEmpty, dpSelled) // 等于前一个是空仓或者selled的最大值
        dpBuyedT := max(dpEmpty - prices[i], dpBuyed) // 等于今天新买入或者前一天是持仓的最大值
        dpSelledT := dpBuyed + prices[i] // 等于前一个是持仓状态加上今天的卖出收益
        
        dpEmpty = dpEmptyT
        dpBuyed = dpBuyedT
        dpSelled = dpSelledT
    }
    
    return max(dpEmpty, max(dpBuyed, dpSelled))
}
```

关于这道题的解题思路，非常建议看一下讨论区的这个[回答](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/discuss/75928/Share-my-DP-solution-(By-State-Machine-Thinking))，看图理解可能会更清晰，我就是看了他的图瞬间找到了解题的实现方法~~

# 0x04 [Coin Change](https://leetcode.com/problems/coin-change/)
```
给定一组不同面额的硬币coins和一个钱的总金额amount，计算最少需要多少枚硬币能组成这个总金额。如果不能组成，返回-1
```

## 分析求解
f(n)代表组成n所需要的最少硬币个数，求解f(n)时，可以遍历coins数组，用n减去硬币的面额得到n'，则f(n)可能等于f(n')+1。我们可以得到状态转移方程：
<center>f(n)=min(f(n-coins[i]))

其中i从0到len(coins)</center>
需要注意的是n可能小于coins[i]，这时候不应该被计算。
实现代码如下：
```golang
func min(a int, b int) int {
    if a > b { return b }
    return a
}

func coinChange(coins []int, amount int) int {
    dp := make([]int, amount + 1)
    dp[0] = 0 
    sort.Ints(coins)
    for i := 1; i<=amount; i++ {
        dp[i] = 1<<32
        for _, coin := range coins {
            if coin <= i {
                dp[i] = min(dp[i], dp[i-coin] + 1)
            } else {
                break
            }
        }
    }
    r := dp[amount]
    if r > amount { // 无法组成amount
        r = -1
    }
    return r
}
```

# 0x05 [Integer Break](https://leetcode.com/problems/integer-break/)
```
给定整数n，分成至少两个正整数的和，求这些数的乘积的最大值。其中2<= n <=58
```

## 分析求解
用f(n)代表n能求得的最大乘积，求f(n)的时候，可以先用n减去一个数i，则f(n)可能等于f(n-i)*i，遍历i从1到n-1，求出最大值，即状态转移方程是：
<center>f(n)=max(f(n-i)*i)

其中i从1到n-1</center>

实现代码如下：
```golang
func max(a int, b int) int {
    if a > b { return a }
    return b
}

func integerBreak(n int) int {
    dp := make([]int, n+1)
    if n == 2 {
        return 1
    }

    for i := 2; i<=n; i++ {
        if i < n {
            dp[i] = i // if i is less than n, it can be used as a single num, which mean it's max product at least is i
        }
        for j := i-1; j >= 1; j-- {
            dp[i] = max(dp[i], dp[i-j]*j)    
        }
    }
    return dp[n]
}
```
需要注意注释那里，对于所有小于n的数，可以单独的被拆成一个数来参与乘积，即**dp[i]至少是i，当i<n时**。

# 0x05 [Count Numbers with Unique Digits](https://leetcode.com/problems/count-numbers-with-unique-digits/)
```
给定非负整数n，计算所有不包含重复数字的数x的个数，其中0<=x<10^n
```

## 思路分析
比较有意思的一道题，一开始思路一直是在找重复的数字，然后用所有的数的个数减去重复的数字的个数，但是重复数字的生成，由于新插入的位数的位置变化过于复杂。正确的思路应该是去生成不包含重复数字的个数，下面来分析一下：
* i为0时，只有一个一位数字0，即`f(0)=0`
* i为1时，有0到9十个一位数字，即`f(1)=10`
* i为2时，除了n为1时的10个一位数字，还有新的可以生成的两位数字：
    * 其中两位数的第一位不能为0，所以是1到9共9种选择
    * 第二位不能与第一位相同，则有10-1=9种选择
    * 即两位数共有9*(10-1)种
    * 即n等于2时，不包含重复数字的数共有所有的一位数加两位数：`f(2) = f(1) + 9*(10-1)`
* i为3时，除了n为2时的一位数和二位数，还可以生成新的三位数，为9*(10-1)*(10-2)，即`f(3) = f(2) + 9*(10-1)*(10-2)`
* ……
* i为n时，除了i为n-1时的一位数和二位数……n-1位数，还可以生成新的n位数，为9*(10-1)*(10-2)*……(10-(n-1))，即`f(n) = f(n-1) + 9*(10-1)*(10-2)*……(10-(n-1))`

至此我们得出来状态转移方程。

## 问题求解
维护dp数组记录f(n)，由于这里每次只依赖前一位的状态，可以优化空间复杂度为O(1)。此外注意这里每次新生成的n位数的个数，等于之前的n-1位数的个数乘以10-(n-1)，可以同时记录这个变量，实现代码如下：
```golang
func countNumbersWithUniqueDigits(n int) int {
    
    if n == 0 { return 1 }
    
    r := 10
    t := 9 // 第一位是只有9中选择
    
    for i := 2; i<=n; i++ {
        t *= (10 - i + 1) // 之后每位有10 - (i - 1)种
        r += t
    }
    return r
}
```

# 0x06 总结
本周又遇到了新的题型，有了新的解题思路，其中我非常详细的写了三道题的思路分析，我认为解题的思路比较有意思或者比较新颖，可以仔细的看一下。
* [Ugly Number II](https://leetcode.com/problems/ugly-number-ii/)
* [Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)
* [Count Numbers with Unique Digits](https://leetcode.com/problems/count-numbers-with-unique-digits/)