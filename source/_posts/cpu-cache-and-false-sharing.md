---
title: CPU cache and False sharing
tags:
  - OS
  - How&Why
categories:
  - How&Why
date: 2020-08-30 19:00:44
---


# 例子
我们先看两个例子，操作对象是一个MATRIXSIZE*8的整型矩阵。
（具体Benchmark代码已经上传到[GitHub](https://github.com/furthergo/cache-line-test)上，有兴趣的可以拉下来自己测试一下）

## 一、遍历矩阵
分别以行优先和列优先来遍历：
```go
func TraversalMatrixSumsByRow() int {
	sum := 0
	for i:= 0; i< MATRIXSIZE; i++ {
		for j := 0; j<8; j++ {
			sum += matrixs[i][j]
		}
	}
	return sum
}

func TraversalMatrixSumsByColumn() int {
	sum := 0
	for i:= 0; i< 8; i++ {
		for j := 0; j<MATRIXSIZE; j++ {
			sum += matrixs[j][i]
		}
	}
	return sum
}
```
测试结果如下：
<!-- more -->

```shell
 BenchmarkTraversalSumByRow-12       	   22184	     53765 ns/op
 BenchmarkTraversalSumByColumn-12    	   12649	     94891 ns/op
```
可以看出行优先遍历比列优先遍历快了40%多。

## 二、计算矩阵中偶数的个数
为了加速计算过程，我们通过协程来分块遍历矩阵，将结果存在一个整形数组中，在所有协程完成之后，遍历res数组取和。两个函数的区别是，在协程的循环里，一个是每次直接操作res数组对应的元素；而另一个是操作一个临时变量，循环结束后赋值给res数组对应的元素：
```go

func TraversalMatrixEven() int64 {
	var count int64 = 0
	res := make([]int64, GOROUTINENUM)
	c := make(chan int, GOROUTINENUM)
	for g := 0; g<GOROUTINENUM; g++ {
		g := g
		go func() {
			chunkSize := MATRIXSIZE / GOROUTINENUM
			start := g * chunkSize
			end := start + chunkSize
			if end > MATRIXSIZE {
				end = MATRIXSIZE
			}
			for i := start; i < end; i++ {
				for j := 0; j < 8; j++ {
					if matrixs[i][j] %2 == 0 {
						res[g]++ // use res[g]
					}
				}
			}
			c <- g
		}()
	}
	for i := 0; i< GOROUTINENUM; i++ {
		g := <- c
		count += res[g]
	}
	return count
}

func TraversalMatrixEvenUseLocal() int64 {
	var count int64 = 0
	res := make([]int64, GOROUTINENUM)
	c := make(chan int, GOROUTINENUM)
	for g := 0; g<GOROUTINENUM; g++ {
		g := g
		go func() {
			chunkSize := MATRIXSIZE / GOROUTINENUM
			start := g * chunkSize
			end := start + chunkSize
			var localCount int64 = 0
			if end > MATRIXSIZE {
				end = MATRIXSIZE
			}
			for i := start; i < end; i++ {
				for j := 0; j < 8; j++ {
					if matrixs[i][j] %2 == 0 {
						localCount++ // use local variable
					}
				}
			}
			res[g] = localCount
			c <- g
		}()
	}
	for i := 0; i< GOROUTINENUM; i++ {
		g := <- c
		count += res[g]
	}
	return count
}

```
测试结果如下：
```shell
BenchmarkMatrixEven-12              	   26013	     45509 ns/op
BenchmarkMatrixEvenByLocal-12       	   46527	     25937 ns/op
```
可以看出使用了local variable的比直接操作res数组快了将近一半。

下面我们来分析一下为什么几乎完全一样的代码，会产生上面两种差异呢？在分析差异之前，先介绍一些基础知识。

# CPU cache

## 什么是CPU cache？ 
摘抄一段wiki的解释：
`A CPU cache is a hardware cache used by the central processing unit (CPU) of a computer to reduce the average cost (time or energy) to access data from the main memory.` 即CPU cache是用来节省CPU访问主存时的平均消耗的一种硬件缓存。事实上CPU cache有三种通用的类型：
1. Data cache (D-Cache): 存储内容为数据的cache
2. Instruction cache (I-Cache): 存储内容为指令的cache
3. Translation lookaside buffer (TLB): 存储VA到PA映射的cache，主要是用来加速地址翻译

这里我们主要讨论D-Cache和I-Cache。

## CPU Cache hierarchy
通常CPU的cache是分多个等级：L1，L2和L3，其中L1 cache分为L1 D-cache和L1 I-Cache。其中L1和L2 cache是每个CPU核独享的，L3 cache是多核共享的。以MBP上的6-Core Intel Core i7为例，我们使用 `sysctl -a | grep cache` 可以看到，L1的D-cache和I-cache分别是32KB，L2 cache是256KB，L3 cache是12MB：
```shell
hw.cachelinesize: 64
hw.l1icachesize: 32768
hw.l1dcachesize: 32768
hw.l2cachesize: 262144
hw.l3cachesize: 12582912
```
访问速度上看，L1 cache访问时间大概是4个时钟周期； L2 cache的访问时间大概是12个始终周期；L3 cache的访问时间大概是100个时钟周期。
因此，当CPU访问的内容在L1 cache中时，时间消耗会非常少，当落到L3 cache上时，就需要花费25倍左右的时间来访问。

## Cache line
接下来是我们今天的主角Cache line，从上面的系统参数我们可以看到有一行 `hw.cachelinesize: 64`，代表的就是Cache line的大小是64B。Cache是由很多个Cache line组成的，每一个Cache line是许多个相邻的数据组成，通常CPU的Cache line size是64B。

在填充Cache的时候，是以Cache line为单位，比如读取主存中的一个元素，会同时把相邻位置的元素当成一个Cache line填充到Cache中，甚至会prefetch相邻的cache line到cache中（HardWare Prefetching），以提高之后的访问速度。

这就解释了第一个例子：为什么矩阵的行优先遍历会比列优先遍历快这么多？因为矩阵也就是二维数组，在内存中是行优先存储的，当我们通过行优先遍历的时候，由于访问第一个元素，整行cache line会被读到L1 cache中，访问之后一个元素时，都是从L1 cache中拿到的；而列优先遍历则和矩阵的存储方式冲突，造成cache的命中率很低，增加访问时间。

# Cache coherence
Cache coherence: `In computer architecture, cache coherence is the uniformity of shared resource data that ends up stored in multiple local caches.` 即缓存一致性，比如Core 1的L1 cache和Core 2的L1 cache分别维护了一个内存块的副本，当一个副本发生更改时，必须反映到其它副本上。缓存一致性旨在管理和实现这种一致性。

## 举个例子
拿6-Core Intel Core i7为例，假如Core1和Core2都缓存了地址A处的内容副本，记住，这里副本都是以cache line为单位的。当Core1的线程对这个副本进行写操作，修改了A的内容，为了保证缓存一致性，此时Core2上的A的缓存副本会被硬件置为失效状态。当Core2的线程需要访问地址A的时候，假如Core1把这个更新写回到了L3或者主存中了，Core2线程会从L3和主存中再读到地址A的内容，并缓存一份副本再L1/L2中。这种多个Core（多个线程）对同一份地址内容的读写，可以通过锁或者原子操作来保证一致性，代价是多消耗一些时间。

## false sharing
结合缓存一致性和cache line，引出了今天的另一个主角：false sharing。什么是false sharing呢？我们想象这样一个场景：Core1的线程在读写地址A，Core2的线程在读写地址A+1，这是两个独立的地址，因此不存在任何读写竞争，也就不需要任何的锁或者原子操作来保证缓存一致性

但是，我们知道cache是以cache line为单位的，每个cache line通常是64B，因此地址A和地址A+1处的内容很大几率会落在同一个cache line上。当Core1读A地址时，会缓存cache line到Core1的L1/L2 cache上；当Core2读A+1地址时，也会缓存同一个cache line到Core2的L1/L2 cache上。根据上一个例子，当Core1写地址A的时候，会导致Core2的L1/L2 cache中对应的cache line失效；当Core2读写地址A+1的时候，会发生cache miss，并同时写A+1会导致Core1 cache中的cache line失效。类推下去……假如有多个线程在交替的读写位于同一个cache line上的相邻的地址，会导致各自的cache line频繁失效，这就是False sharing.

这就解释了第二个例子：为什么使用local variable会比直接操作res数组快这么多？就是因为多个goroutine在访问res数组时，造成了false sharing，大大降低了res数组的访问速度。

## 怎么避免False sharing

我们可以先分析False sharing产生的条件：
1. 相互独立的值落在了同一个cache line上
2. 多线程访问
3. 非常频繁
4. 至少有一个writer（会改变cache line的内容）

我们只需要打破任意一个产生条件，就可以避免产生False sharing。

比如第二个例子，就是增加一个local variable，只在每次循环结束时访问res数组，就是避免了频繁的访问同一个cache line，即使每个goroutine循环结束后会访问一次res数组，由于次数较少，对性能几乎无影响。

再比如更通用的做法，很多情况下可能无法控制访问者的具体使用场景，我们可以通过避免相互独立的值落在同一个cache line来避免False sharing的产生。比如下面这个例子：
```go
type paddedRWMutex struct {
	_  [8]uint64 // Pad by cache-line size to prevent false sharing.
	mu sync.RWMutex
}
type DRWMutex []paddedRWMutex
```
[DRWMutex](https://github.com/jonhoo/drwmutex) 是一个go实现的分布式读写锁，它维护了一个锁的数组，根据core id为每个核心提供独立的锁，提升读效率。这里在真正的锁sync.RWMutex前面加了一个长度为8的uint64数组，刚好64B，确保每个paddedRWMutex不会落在同一个cache line上，避免产生False sharing.

# 总结
CPU cache是CPU提升访问速度非常重要的硬件资源，根据程序的局部性原理，绝大多数的访问都会发生在cache上，并且由于访问速度的差异，应该发生在cache。因此编写cache友好的程序对于提升运行速度有重要的作用。另一方面，在多线程的场景里，避免False sharing是另一个提升程序性能的重要手段，需要很好的理解CPU cache的工作原理来避免False sharing的产生。

## What's more?
文章开头给出的例子里，MATRIXSIZE和GOROUTINENUM我都没有贴出来，感兴趣的可以自己试一下不同的值对于性能差异的影响，并思考一下为什么？

欢迎在评论区留言讨论~





#### Reference
1. [code::dive conference 2014 - Scott Meyers: Cpu Caches and Why You Care](https://www.youtube.com/watch?v=WDIkqP4JbkE&list=WL&index=2&t=1712s) 