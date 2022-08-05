---
author: "李昌"
title: "走进chan"
date: "2022-08-05"
tags: ["channel"]
categories: ["golang"]
ShowToc: true
TocOpen: true
---

## 1. chan的结构

一个channel长这样：
```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16 // chan中元素大小
	closed   uint32 // 是否关闭
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	lock mutex
}
```

channel的字段中，主要可以分为三部分：

第一部分是标识channel自身的一些状态和性质，如`hchan.closed`标识chan是否关闭，`hchan.elemsize`标识chan中元素的大小、`hchan.elemtype`标识chan中元素类型；

第二部分是标识底层循环数组的状态的字段，如`hchan.qcount`标识当前数组中元素数量、`hchan.dataqsiz`标识循环数组的大小、`hchan.buf`是指向底层数组的指针、`hchan.sendx`标识待发送元素的下标、`hchan.recvx`标识待接收元素的下标；

第三部分是存储正等待当前chan的goroutine，如`hchan.recvq`存储等待接收的goroutine，`hchan.sendq`存储等待发送的gotoutine

最后是一个锁，保证了并发安全。

### 1.1 如何构造一个chan

通过汇编可以看到，在构造chan时，调用的是`runtime.makechan`函数，函数如下：
```go
type chantype struct {
	typ  _type
	elem *_type
	dir  uintptr
}

func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// ...
    // 省略部分检查代码

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```

首先对`mem`进行了计算，`mem`是chan中元素所占的空间

`runtime.makechan`函数中，受先是一些检查，然后根据chan的长度和elem的类型，会有不同的构造策略：
- 如果是构造一个无缓冲chan，即参数size为0，那么默认分配`hchanSize`=96字节空间并强制转换为*hchan类型，96字节是一个hchan的大小。然后
- 如果chan的elem不包含指针，则分配`hchanSize+mem`空间给chan，并将`hchanSize`之后的所有空间分配给`hchan.buf`。
- 如果chan的elel包含指针，那么直接new一个chan，并为`hchan.buf`分配`meme`大小的空间

最后将`hchan.elemsize`、`hchan.elemtype`、`hchan.dataqsiz`进行赋值。

返回一个`hchan`的指针，这保证了我们在对channel进行传递时不会进行复制。

## 2. chan的发送与接收

chan发送时调用了`runtime.chansend1`:
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil { // 如果chan为nil
		if !block { // 如果不可阻塞
			return false    // 发送失败
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2) // 可阻塞，goroutine挂起
		throw("unreachable")
	}

    // 省略部分代码

	if !block && c.closed == 0 && full(c) { // 不可阻塞，chan未关闭且已满，则发送失败
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock) // 加锁，并发安全

	if c.closed != 0 { // 如果chan已经被关闭了
		unlock(&c.lock) // 解锁
		panic(plainError("send on closed channel")) // 向已关闭的chan发送会panic
	}

    // 如果接收队列中存在goroutine，则不经过hchan.buf，而直接复制到接收端缓冲区
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

    // chan还没满
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx) // 计算存放元素的内存地址
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep) // 将元素复制到buf
		c.sendx++ // 待发送下标+1
		if c.sendx == c.dataqsiz { // 循环数组
			c.sendx = 0
		}
		c.qcount++ // 总元素数量+1
		unlock(&c.lock) // 解锁
		return true // 发送成功
	}

    // chan满了

	if !block { // chan满了，且要求不可阻塞，则直接失败
		unlock(&c.lock)
		return false
	}

    // 可以阻塞
    // 获取当前goroutine指针
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg) // 当前goroutine进入chan的待发送队列

	atomic.Store8(&gp.parkingOnChan, 1)

    // 挂起当前goroutine
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	KeepAlive(ep)

	// 当前goroutine被唤醒了
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed { // 被唤醒后发现chan关闭了
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

上述代码的主要逻辑是：
- 检查会发送失败的情况
- 加锁
	- 再次检查会发送失败的情况，失败则解锁返回失败
	- 如果有goroutine在等着接收，直接复制给它，解锁返回成功
	- chan的buf没满，复制到buf中，解锁返回成功
	- 如果chan的buf满了
		- 不可以阻塞，解锁返回失败
		- 可以阻塞，让goroutine进入等待队列并挂起，等待唤醒

如果在发送时发现有等待接收的goroutine，会调用`runtime.send`将元素复制给等待的goroutine，`runtime.send`如下：
```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if raceenabled {
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			racenotify(c, c.recvx, nil)
			racenotify(c, c.recvx, sg)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
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

TODO