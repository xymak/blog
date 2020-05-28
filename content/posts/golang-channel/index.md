---
title: "《Go 语言设计与实现》读书笔记：Channel"
date: 2020-05-28T16:14:54+08:00
---

今天看了[《Go 语言设计与实现》](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#fn:6)Channel相关的介绍。当初有遇到相关的问题，答得不是很好，这里记录一下。
其实golang channel的设计还是很简单的，结构体如下：
```go
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	elemtype *_type
	sendx    uint  
	recvx    uint
	recvq    waitq
	sendq    waitq

	lock mutex
}
```
* qcount为当前元素个数
* dataqsiz为总容量
* buf为缓冲区数据指针
* sendx为发送的标记位置
* recvx为接收的标记位置
* sendq为缓冲区不足的情况下阻塞的goroutine队列
* recvq为不存在数据的情况下阻塞的goroutine队列

大体流程是这样的：
* 发送情况：
    * 如果是缓冲区空间不足或者不存在缓冲区，这里有两种情况：
        * 有goroutine在接收队列等待，会直接将数据拷贝到接收goroutine的变量里面，sendq出队，并且唤醒接收goroutine
        * 没有等待的goroutine，则通过gopark将当前goroutine设置为wait，并且初始化一个sudog入队recvq
    * 如果缓存区空间充足：
        * 有goroutine在接收队列等待，会直接将数据拷贝到接收goroutine的变量里面，sendq出队，并且唤醒接收goroutine
        * 没有等待的goroutine，则写入buf
* 接收情况：
    跟发送类似，镜像操作，这里不做多余陈述
    
另外，从底层也能发现，channel里面只有一个锁，接收和发送都要通过这一个锁做并发控制，所以用channel替代一些无锁的写法，性能是很差的。