---
title: 'The Go Memory Model'
tags:
  - Golang
  - Mutithread
categories:
  - Golang
date: 2020-10-22 00:14:36
---

# Memory Model
引自wiki: 
> [In computing, a memory model describes the interactions of threads through memory and their shared use of the data.](https://en.wikipedia.org/wiki/Memory_model_(programming))

如上所述，Memory Model是定义**多个线程**之间穿过内存交互影响（或者叫”干扰“），以及**多个线程**如何共享的使用数据。
首先，这是一个多线程的概念，具体来说，比如编译器在编译的时候做的一些编译优化，可能会导致程序的编写顺序和执行顺序不一致，在单线程模型下，这种优化对于程序的最后执行结果是没有任何影响的；但是在多线程模型下就可能会导致一些问题。
<!-- more -->
比如下面这段代码：
```golang
var s string
var b bool

func f1() {
    s = "Hello World"
    b = true
}

func f2() {
    for !b {
        
    }
    print(s)
}

func hello() {
    go f1()
    go f2()
}

func main() {
    hello()
}
```
f1函数给s赋值"Hello World"，然后把b置为true；f2函数，在自旋等待b被置为true，然后输出s，理想情况下应该输出"Hello World"。但是假如编译器在f1这里做了编译优化，把s和b的赋值顺序对调，虽然从单独从f1的角度看，s和b的最终赋值结果没有改变，执行结果和优化前是一致的，但是f2会因为b先被置为true而输出""。

这就是在多线程模型下，某些优化可能会造成程序执行结果出现不一致现象。而Memory Model就是为了保证程序的一致性，用来定义和约束这种在多线程模型下的优化行为的。

许多支持多线程的编程语言比如Java、C++都有Memory Model的相关定义，同时需要约束的优化行为也有很多，比如上面提到的编译器优化，还有硬件优化(CPU指令)。而Go作为天生支持并发的编程语言，也有自己的Memory Model定义，下面主要是总结Go的Memory Model的定义。

# [The Go Memeory Model](https://golang.org/ref/mem)
这是一篇Go 2014年发的博文，详细描述了Go的Memory Model，下面做一个简单的总结（翻译）。

## 定义
Go的并发编程使用的是goroutine，Memory Model就是定义多个goroutine之间的执行行为：
> 一个goroutine中对某个变量的读，被保证能够观察到在不同goroutine上对同一个变量的写所产生的值
即
> 在修改通过多个goroutine并发访问的数据时，必须保证访问的串行执行

注：这里Go建议使用channel保证串行并发访问，这也是Go的一个CSP模型的实现：**Do not communicate by sharing memory; instead, share memory by communicating.**，**通过通信来共享内存，而不是通过共享内存来进行通信**
## Happen Before
为了保证这种读写顺序的执行，Go定义了一个叫做Happen Before的概念：
> 在Go程序中一部分执行内存操作的执行顺序
* e1如果happens efore e2，则表示e2 happens after e1
* 如果e2即不在e1之后发生，也不再e1之前发生，则表示e1和e2同时发生

为了保证一个对变量v的读操作r，能够观察到另一个对变量v的写操作w，需要满足下面两个约束：
1. w happens before r
2. 任何其它对于共享变量v的读操作，要么happens before w, 要么 happen after r.

在单个goroutine里执行的程序，因为没有并发，happen before顺序就是程序的的书写顺序。（这里可能会对程序做reorder，但是不会影响happen before的条件）。
在多个goroutine访问一个共享变量时，就需要一些同步事件，来保证happen before。

## 同步控制

### Init函数
Go的init函数执行是在单个goroutine中，但是这个goroutine可能会创建其它的goroutine，然后并发执行。因此init会保证happen before：

* 如果package p引入了package q，则q的init函数一定happen before p的init函数
* main.main happen after 所有init函数完成

这里主要是保证，在一个package的init里执行的代码，必须能观测到它引入的package的init函数里的一些操作。

### Goroutine
开启一个goroutine的```go```语句，一定happen before这个goroutine的执行开始
比如下面这段代码一定输出"hello, world"
```golang
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

此外，goroutine的退出，不保证happen before任何程序中的事件。比如下面这段代码，go的编译器甚至可能会优化掉整个```go```语句：
```golang
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

### Channel
channel中每个发送都有一个对应的接收，通常是在不同的goroutine
* 一个在channel上的send，happen before与之对应的从channel上的receive的完成
* 一个channel的关闭一定happen before一个从channel中receive到零值的完成
* 一个容量为C的channel上的第k个receive，一定happen before在channel上第k+C次send的完成（当C为0时，代表channel是unbuffered）

这里都写的是complete，表示完成一定是按顺序的，满足happen before的。

### Locks

* 对于一个并发锁sync.Mutex或者读写锁sync.RWMutex 变量l, 第n次l.Unlock()的调用一定happen before第m次l.Lock()调用的返回，这里n小于m。比如下面这段代码,```a = "hello, world"```happen before于第一次l.Unlock()的调用，而第一次.Unlock()的调用happen before 于第二次l.Lock()的调用完成，第二次l.Lock()的调用完成happen before于```print(a)```，所以可以保证输出"hello, world"。
```golang
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```
* 对于一个读写锁sync.RWMutex 变量l，它的某一次l.RLock()的调用完成，happen after第n次l.Unlock()的调用，则与之对应的l.RUnlock happen before第n+1次l.lock()的调用。意思就是某一个读锁的完成和读解锁的发生之间，没有其他的写锁操作

### Once
Once是sync包提供的用于多个goroutine对于同一个函数只执行一次的机制：
```golang
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.

	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}
func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```
* 单个f的调用的完成happen before其他任何Do(f)的完成

这里表示如果当前f()已经在执行了，那么任何其它goroutine对于Do(f)的调用，会阻塞等到这个f的调用完成之后才返回，这样能确保其它goroutine对于f的完成的有效性。

在上面代码段的注释也可以看到，Go使用sync.Mutex来对保证f()的happen before，而不是用比如CAS：假如f已经开始执行，当另一个goroutine访问到时，CAS失败会直接返回，这样就没办法保证这个goroutine能观察到f()执行完的结果，也就不能保证sync.Once的happen before。这里用sync.Mutex来保证这个happen before：即**当某个goroutine在执行f时，会拿到锁，之后其它的goroutine会阻塞，等待直到f完成释放锁。**

此外这里还用到了atomic来保证对于o.done的读写的happen before关系：可以看到对o.done的读并不在sync.Mutex的作用域内，因此必须要保证对于o.done的写happen before o.done的读，来保证其他goroutine能观察到f()的完成。这里Go的issue里有一个关于这个的[讨论](https://github.com/golang/go/issues/38320)

另外可以发现，尽管sync.atomic的用法是假设是保证一致性的，但是官方目前(go 1.15)并没有sync.atomic Load/Store关于Memory Model的定义和保证，这里有一些相关的讨论，感兴趣可以翻一翻。

* [doc: define how sync/atomic interacts with memory model
](https://github.com/golang/go/issues/5045)
* [https://groups.google.com/g/golang-dev/c/vVkH_9fl1D8/m/azJa10lkAwAJ](https://groups.google.com/g/golang-dev/c/vVkH_9fl1D8/m/azJa10lkAwAJ)

# EOF
总结一句话：在并发访问上，显示的使用上面陈述的同步控制。

# 参考资料

- [1] https://golang.org/ref/mem
- [2] http://nil.csail.mit.edu/6.824/2016/notes/gomem.pdf
