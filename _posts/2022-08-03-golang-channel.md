---
title: 图解 Golang Channel 原理
author: neil
date: 2022-08-03 21:42:00 +0800
categories: [技术]
tags: [面试, Golang]
img_path: /posts/20220803/
---


## 基础概念

Channel 是 Golang 的核心类型，常用于多个 Goroutine 之间的通信。可以把 Channel 理解成是一个单向的管道，具有 FIFO 特性。

![image.png](https://raw.githubusercontent.com/neilzhang/blog-images/main/posts/20220803/image-20220803220239287.png)

Channel 是有容量限制的
1. 当容量是 0 时，称为无缓冲 Channel。发送和接收只有一方就绪时，就绪方会被阻塞直到另一方也就绪。
2. 当容量大于 0 时，称为有缓冲 Channel。当传输中的元素个数超过容量时，发送方将会被阻塞直到有可用的缓冲空间出现；当传输中的元素个数为 0 时，消费方将会被阻塞直到缓冲空间出现新的数据。


## 数据结构

```golang
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

- qcount，缓冲队列的大小，记录实际元素数量
- dataqsiz，缓冲队列的容量，记录最大可存储元素数量
- buf，指向环形缓冲队列的指针
- elemsize，每个元素的大小
- closed，记录 channel 的关闭状态
- elemtype，元素的类型
- sendx，缓冲队列中即将发送的数据下标
- recvx，缓冲队列中即将接收的数据下标
- recvq，等待从 channel 接收数据的 goroutine 双向链表
- sendq，等待向 channel 发送数据的 goroutine 双向链表
- lock，多 goroutine 读写的并发保护锁


## 图解发送数据


![image.png](https://raw.githubusercontent.com/neilzhang/blog-images/main/posts/20220803/image-20220803220324301.png)

注：无缓冲 Channel 原理类似不做赘述


## 图解接收数据

![image.png](https://raw.githubusercontent.com/neilzhang/blog-images/main/posts/20220803/image-20220803220345955.png)

注：无缓冲 Channel 原理类似不做赘述


## 源码解读

[在线源码](https://github.com/golang/go/blob/master/src/runtime/chan.go)


### 发送数据

```golang
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...
	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil { // 关键点1
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz { // 关键点2
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		...
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// 关键点3
	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)
	...
	return true
}

func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	...
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

关键点
1. 当 recvq 有等待的接收者时，说明缓冲队列是空的，则将数据直接发送给接收者，然后将接收者的 Goroutine 标记成可运行的状态，并加入到本地可运行队列中。
2. 当缓冲队列未满时，则将数据直接写入缓冲队列。
3. 当缓冲队列满了或者无缓冲队列时，则将发送数据的指针和当前 Goroutine 等信息组装成 sudog 并加入到 sendq 中，等待合适机会执行。

##### 接收数据

``` golang
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	...
	lock(&c.lock)
	...
	if sg := c.sendq.dequeue(); sg != nil { // 关键点1
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	if c.qcount > 0 { // 关键点2
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		...
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 关键点3
	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
	...
	return true, success
}

func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		...
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		...
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

关键点
1. 当 sendq 有等待的发送者时，如果是无缓冲队列，则直接从发送者获取数据；如果是缓冲队列满了，则从缓冲队列取出一个数据，然后将发送者的数据写入缓冲队列。最后将发送者的 Goroutine 标记成可运行的状态，并加入到本地可运行队列中。
2. 当缓冲队列有数据时，则直接从缓冲队列读取数据。
3. 当缓冲无数据时，则将接收数据的指针和当前 Goroutine 等信息组装成 sudog 并加入到 recvq 中，等待合适机会执行。
