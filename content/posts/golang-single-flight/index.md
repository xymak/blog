---
title: "《Go 语言设计与实现》读书笔记：SingleFlight"
date: 2020-05-25T20:44:54+08:00
---

今天读[《Go 语言设计与实现》](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#rwmutex)看到了一个感觉比较有用的golang同步原语，它能在一个服务中抑制多次重复的请求。

"一个常用的场景是 - 我们使用redis对数据库中的数据进行缓存，发生缓存击穿，大量流量都会达到数据上进而影响服务的延时"，这句话是作者的原话。从这句话我可以延伸到，还可以用来做合并请求，例如cdn上的合并请求。
之前我还因为缓存击穿扛过一个事故，就是mysql请求一直超时，redis缓存一直打不上，造成了恶性循环，所以我觉得这个同步原语非常有用。

它的结构体 x/sync/singleflight.Group 由一个映射表和一个互斥锁组成：
```go
type Group struct {
	mu sync.Mutex
	m  map[string]*call
}

type call struct {
	wg sync.WaitGroup

	val interface{}
	err error

	dups  int
	chans []chan<- Result
}
```
它对外提供了两个用于抑制请求的方法：
* x/sync/singleflight.Group.Do — 同步等待的方法 Do；
* x/sync/singleflight.Group.DoChan — 返回 Channel 异步等待的方法；
```go
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err, true
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}
```
原理很简单，通过资源的唯一key去map里面查找，如果找到了，引用+1，并且等待资源生成，如果没找到，就执行doCall。
```go
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	for _, ch := range c.chans {
		ch <- Result{c.val, c.err, c.dups > 0}
	}
	g.mu.Unlock()
}
```
doCall实际上就是把map里面的call中的value赋值，并且通过chan将数据同步给调用DoChan的协程。

DoChan方法其实跟Do方法差不多，只是返回给call增加了chan来做协程的同步。
```go
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
	ch := make(chan Result, 1)
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		c.chans = append(c.chans, ch)
		g.mu.Unlock()
		return ch
	}
	c := &call{chans: []chan<- Result{ch}}
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	go g.doCall(c, key, fn)

	return ch
}
```