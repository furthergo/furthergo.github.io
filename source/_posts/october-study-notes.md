---
title: 国庆学习笔记
tags:
  	- Golang
date: 2020-10-08 18:14:15
categories:
	- Golang
---


# 总结
假期针对性的学习了一下Golang和MySQL相关的知识，其中主要Golang，跟着网上的一个系列教程[7days-golang](https://github.com/geektutu/7days-golang)在学习，，三个用Golang实现的框架：

* gee-web：HTTP框架
* gee-cache：分布式缓存
* gee-orm：orm库

教程通俗易懂，并且对新手非常友好，强烈建议跟着学习一遍，学习到了很多以前不清楚或者理解很模糊的知识，受益颇深，写一篇博客总结和梳理一下学习到的知识点，加深理解。
<!--more-->
# HTTP框架：gee-web

## HTTP框架主要做的事

### **ListenAndServe**
`Engine`类，框架和用于交互的直接入口
* **实现HTTP Handle接口**，用于注册给http.Server，处理请求和响应

`Context`类，**抽象出请求上下文Context概念**
* 用于处理request和response
* 提供一些便利方法，解析request和构造response

### **路由能力**
`Router`和`Tree`类
* **提供Router路由能力**，即设置和存储path和HandlerFunc映射的功能，存储在Engine中
* **动态路由能力**
    * 支持/api/:name和/api/*的Router格式
    * Trie树用于路由匹配，主要是注册Router时Trie树的建立，以及处理请求时根据  Path在Trie树中搜索对应的pattern，并调用对应的HanderFunc
    * Trie树即前缀树，主要是用来匹配多个字符串前缀的，比如LeetCode的[最大公共前缀](https://leetcode.com/problems/longest-common-prefix/)就可以用Trie来解决

### RouterGroup和中间件
#### **RouterGroup**
`RouterGroup`类
提供**路由分组**能力RouterGroup，对于同一类接口，提供分组处理的能力，比如/api, /api/v1, /api/v2这种形式。

RouterGroup提供每个Group注册时的pattern，以及对于的中间件数组，这样在请求到来的时候，会从所有的Group里取出当前path满足pattern的中间件函数并调用。这里RouterGroup只支持静态路由。
```golang
type RouterGroup struct {
	prefix string // pattern
	handlers []HandlerFunc // 中间件函数
	engine *Engine // 用于注册到前缀树中
}
```
这里有意思的一个点是：gee-web把RouterGroup作为Engine类的嵌套子struct，类似面向对象中的继承，如下所示，这样做的好处是，Engine类可以直接调用RouterGroup的方法，接口友好。也可以看一下[Gin](https://github.com/gin-gonic/gin/blob/master/gin.go#L57)的实现。
```golang
type Engine struct {
	*RouterGroup // 嵌套Struct
	router *Router
	groups []*RouterGroup

	templates *template.Template
	funcMap template.FuncMap
}
```
上面RouterGroup又持有了一个指向当前Engine的指针，这样在Group里还是可以使用Engine的能力。

#### **中间件**

提供**设置中间件**的能力，对于很多接口，需要用同一个处理函数去处理，可以用RouterGroup分组，并抽象出中间件来处理。

如上所示中间件HandlerFunc是存储在对应的RouterGroup里的，这样请求到来的时候：
* 先到所有的RouterGroup取出需要调用的中间件，然后组合成一个[]HandlerFunc，再交给`Router`类处理
* Router类在Trie中取出Path的对应HandlerFunc，并append到中间件的末尾，然后调用`c.Next()`

`c.Next`的实现如下，非常有意思：
```golang
func (c *Context)Next() {
	c.index++
	for ; c.index < len(c.handlers); c.index++ {
		c.handlers[c.index](c)
	}
}
```
在Context中会存储一个当前执行到的HanderFunc的index，初始值为-1，调用c.Next时，会加一，并跑一个for循环顺序处理直到HanderFunc处理完成。

为什么要抽象出c.Next()这个函数呢，path对应的处理函数是被append到中间件函数数组的末尾的，也就是说所有的中间件总是先调用的，然后最后调用请求的处理函数。这样就有个问题，假如有些中间件的代码，*想在请求处理函数完成之后再调用该怎么处理*，比如一个统计请求处理时长的中间件，这个时候就可以在中间件函数里先调用c.Next()再执行其它的函数，这样在c.Next()就会处理完所有HanderFunc，再执行剩下的代码，比如Logger中间件这样：
```golang
// Logger中间件
func Logger() HandlerFunc {
	return func(c *Context) {
		st := time.Now()
		c.Next()
		fmt.Printf("\033[1;37;41m[%d]\u001B[0m %s: %d\n", c.StatusCode, c.Path, time.Since(st))
	}
}
```

# 缓存框架：gee-cache
一个分布式的内存缓存框架，参考[groupcache](https://github.com/golang/groupcache)。这篇学习到了很多以前只知道概念而没有时间过的知识。

* LRU实现
* 一致性哈希
* 节点注册与节点通信
* 缓存击穿

gee-cache只涉及缓存读操作，不涉及直接的缓存写的操作，缓存只会从一个用户设置的失败Getter中获取并更新。

## 梳理
gee-cache主要分为两个大部分

1. HTTPPool类，用于节点通信、分布式实现
2. Group类，cache对象，代表一组KV，暴露对外的Get接口，内部封装cache获取的逻辑

这里面包括一些子部分，比如LRU实现，并发安全的Cache类，一致性哈希实现，节点选择，HTTP通信等等。

## Group
一个Group就是一个命名空间，结构如下；
```golang
// 对应一组KV，每个Group有自己的分布式缓存peer
type Group struct {
	name string // namespace
	c cache // 本地缓存
	gt Getter // 缓存miss
	peerPicker PeerPicker // 分布式节点选择
	loadGroup flightGroup // 缓存miss时，load操作限制，防止缓存击穿：cache miss时，多个请求同时打到DB，造成DB压力瞬间增大
}
```

cache是一个读写锁和LRU实现的并发安全KV结构，如下：
```golang
type cache struct {
	mu sync.RWMutex
	lru *lru.Cache
	cacheBytes int64
}
```

lru.Cache是一个用map和双向链表实现的LRU淘汰策略的map，如下：
```golang
type Cache struct {
	maxBytes int64 // 最大Size
	cBytes int64 // 当前大小
	OnEvicted func(k string, v Value) // 淘汰回调
	ll *list.List // 双向链表，用于存储element之前的顺序
	cc map[string]*list.Element // map，存储key到element节点的映射
}
```
map用于映射KV，而双向链表用于缓存淘汰。查找和淘汰的时间复杂度都是O(1)。

## HTTPPool
HTTPPool代表分布式缓存中的一个节点，结构如下：
```golang
type HTTPPool struct {
	self string // 当前节点名
	basePath string // api path
	mu *sync.Mutex
	peers *consistenthash.Map // 一致性hash
	httpGetters map[string]*httpGetter // 节点名和HTTPGetter的映射
}
```
实现http.Handle接口，用于处理跑一个http server，响应其它节点的请求。

consistenthash.Map是一个一致性哈希的实现，可以在其中增加和删除节点，支持虚拟节点，结构如下：
```golang
type Map struct {
	hash Hash // hash算法
	replicas int // 虚拟节点倍数
	keys []int // 所有的节点
	hashMap map[int]string // 虚拟节点和真实节点的映射
}
```
这里实现方式是：
1. 对于所有真实节点，生成replicas个虚拟节点，key是编号加真实节点名
2. 所有虚拟节点计算哈希值并顺序存在keys数组中，keys数组代表哈希环（最后一个节点的下一个节点是第一个节点）
3. hashMap是虚拟节点和真实节点的映射
4. 当一个请求的key到来时，先计算哈希值，然后在keys环中二分搜索找到应该落到的虚拟节点，然后根据hashMap得到真实的节点

这里需要注意的是，因为有节点的更新，更适合一致性哈希环的数据结构应该是双向链表加跳表，双向链表用于新增节点，跳表用于节点搜索。

httpGetter是向其它节点发送请求的数据结构，主要是封装请求和发送，这里响应数据用的是PB编码。

## 缓存获取的步骤
1. Group的Get(key)接口
2. 从缓存中获取
3. 根据一致性哈希的结果，从远端节点获取，发送调用HTTPGetter
4. 节点落到本地，调用Getter从本地获取，比如文件或者DB中
5. 本地获取会更新缓存

## 缓存击穿

这里有一个比较有意思的Package叫singleflight，主要是使用`sync.waitGroup`来用来防止缓存击穿。
缓存击穿：对于一个不在缓存中的key，同一时间发送了大量的请求，全部打到DB，导致DB压力过大。
解决方案：对于同一个key，只请求一次。
```golang
type call struct {
	wg sync.WaitGroup
	val interface{}
	err error
}
type Group struct {
	mu sync.Mutex
	m map[string]*call
}
func (g *Group)Do(k string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[k]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		log.Printf("Do key %s, hit loaderGroup", k)
		return c.val, c.err
	}
	c := &call{}
	g.m[k] = c
	g.mu.Unlock()

	c.wg.Add(1)
	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, k)
	g.mu.Unlock()
	return c.val, c.err
}
```
这里对于每个key生成一个call，存储在Group的map里，当同一个key重复请求，取到已经发送的call，就会在waitGroup中等待，知道`waitGroup.Done`，从call中取回结果。

还有一些相关概念：
1. 缓存穿透:
    * 大量请求不存在的Key，由于缓存中没有数据，会在DB上发生大量的查询请求，造成DB压力过大。
    * 解决方案是使用布隆过滤器，在缓存中存储数据是否存在的信息，过滤掉不存在key的请求
2. 缓存雪崩：
    * 分布式存储中，一个或几个节点突然崩溃，导致大量请求落到后一个节点上，造成节点压力过大
    * 解决方案是，一使用一致性缓存的虚拟节点，这样单个节点崩溃时，它的请求会fallback到不同节点上；二，这种情况下仍然会存在一整个节点缓存失效的情况，可以在哈希环中的每个节点都缓存一份前一个节点的缓存，这样即使前一个节点fallback到当前节点，也不会产生大量的DB请求。

## 其它

第一次接触这种分布式缓存的框架，发现一个点是groupcache的节点通信，用的是HTTP这种文本协议，虽然编码用的是PB，但是HTTP头之类的数据，还是有冗余。
上网查了一下资料redis的CS通信协议是RESP，是一种基于TCP的文本协议；而Redis节点间通信用的是Gossip，基于TCP二进制协议。

# ORM框架：gee-orm
使用面向对象的方法，来操作数据库。MySQL语句和Go的标准库，都是直接执行SQL语句的。比如"SELECT * FROM User WHERE Name=`A` and Age > 8"，编写比较麻烦。ORM库就是封装一些常用的数据库操作，方便使用。ORM库本身的难点如何整合在于不同的关系型数据库甚至其它非关系型数据库的差异，抽象出统一的接口。

主要难点是：涉及到大量的反射reflect

## 梳理

### Engine
`Engine`类
代表一个DB连接，封装一些数据库级别的操作，比如打开和关闭数据库连接，事务、迁移等
```golang
type Engine struct {
	db *sql.DB
	dialect dialect.Dialect
}
```
`dialect`类是根据传入的不同数据库生成不同的SQL语句，是一个接口类型，不同类型的数据库实现自己的差异部分。gee-orm只实现了sqlite3
```
type Dialect interface {
	DataTypeOf(tpe reflect.Value) string
	TableExistSQL(tableName string) (string, []interface{})
}
```

### Session
`Session`类，代表一个操作对象，主要是表级别的操作，比如创建、删除表，记录读取插入删除等等，同时提供hooks和链式调用

```golang
type Session struct {
	db *sql.DB
	tx *sql.Tx
	dialect dialect.Dialect
	refTable *schema.Schema
	clause clause.Clause
	sql strings.Builder
	sqlVars []interface{}
}
```

其中`schema.Schema`代表一个表，用来映射Model到表结构，包括表名，列名等
```golang
type Schema struct {
	Model interface{}
	Name string
	Fields []*Field
	FieldsName []string
	FieldsMap map[string]*Field
}
```

### Clause
Clause package封装各种SQL语句的子句构建实现
```golang
type Clause struct {
	sql map[Type]string
	sqlVars map[Type][]interface{}
}
```

## 其它
gee-orm库写下来的感觉是非常繁琐，需要处理大量的细节，同时对MySQL和反射本身也不太熟悉，主要是在插入和查找操作中，涉及到Model反射到Schema和rows，以及rows反射到Model上。

# EOF
后续需要再仔细学习一下的知识点：

1. reflect
2. go标准库的http package
3. 其它web/分布式cache/orm库的实现


PS: 从gee-cache实现一致性哈希和节点选择那里开始，这两天又看了一些比如Redis的分布式集群、gossip、Raft协议等等，也尝试看了一下MIT的[分布式系统](https://www.bilibili.com/video/BV1R7411t71W?t=1943)，以后会多看一下这方面的知识，感觉分布式相关知识的实现比起概念更有意思，同时和OS、网络、存储这些计算机基础的相关性也比较大。
