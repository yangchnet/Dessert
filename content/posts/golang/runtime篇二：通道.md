---
author: "李昌"
title: "runtime篇二：通道"
date: "2022-08-05"
tags: ["golang", "runtime"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

> 本系列代码基于golang1.19(1e5987635cc8bf99e8a20d240da80bd6f0f793f7)

- [runtime篇一：接口](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E4%B8%80%E6%8E%A5%E5%8F%A3/)
- [runtime篇二：通道](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E4%BA%8C%E9%80%9A%E9%81%93/)
- [runtime篇三：defer](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E4%B8%89defer/)
- [runtime篇四：panic](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E5%9B%9Bpanic/)

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

### 2.1 chan的发送

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

唤醒后，不会再进行发送或复制等，因为一个goroutine如果是从等待队列被唤醒，则是直接从发送goroutine将消息复制过来，然后才唤醒，因此可以直接结束。

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
	// sg.elem 指向接收到的值存放的位置，如 val <- ch，指的就是 &val
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

### 2.2 chan的接收

chan接收时调用的函数为`runtime.chanrecv`:

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil { // chan为nil
		if !block { // 且不可阻塞
			return	// 直接返回失败
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2) // 可阻塞，gotoutine挂起
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	if !block && empty(c) { // 不可阻塞且chan为空
		if atomic.Load(&c.closed) == 0 { // chan没关闭
			return
		}
		// The channel is irreversibly closed. Re-check whether the channel has any pending data
		// to receive, which could have arrived between the empty and closed checks above.
		// Sequential consistency is also required here, when racing with such a send.
		if empty(c) { // chan被关闭了，但其中可能还有未取出的值
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock) // 上锁

	if c.closed != 0 {	// 再次检查chan是否被关闭
		if c.qcount == 0 {
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			unlock(&c.lock)
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
		// The channel has been closed, but the channel's buffer have data.
	} else {
		// Just found waiting sender with not closed.
		if sg := c.sendq.dequeue(); sg != nil { // 有个chan等着发
			// Found a waiting sender. If buffer is size 0, receive value
			// directly from sender. Otherwise, receive from head of queue
			// and add sender's value to the tail of the queue (both map to
			// the same buffer slot because the queue is full).
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}

	if c.qcount > 0 { // 当前chan中有值
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)	// 将消息复制给接收者
		}
		typedmemclr(c.elemtype, qp)	// 清理发送者内存空间
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {	// 当前chan中没有值，又不能阻塞，只好失败
		unlock(&c.lock)
		return false, false
	}

	// 可以阻塞，先坐等一会
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
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// 被唤醒了
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```

总体来看，和发送的过程很相似，主要是检查chan是否关闭，是否可阻塞，是否有发送者等待，chan中是否有值，然后做出响应的处理。

在使用chan进行接收时，有时可以使用ok来标识是否真的从chan中接收到了值，go底层通过不同的函数做到这一点：
```go
// a := <-chan
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

// a, ok :=  <-chan
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}
```

## 3. chan的关闭

关闭chan时，调用的是`runtime.closechan`:
```go
func closechan(c *hchan) {
	if c == nil { // 不能关闭一个nilchan
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 { // 不能重复关闭chan
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1 // 设置chan状态为关闭

	var glist gList

	// 处理所有等待读取的goroutine
	for {
		sg := c.recvq.dequeue()
		if sg == nil { // 当recvq中没有goroutine时跳出
			break
		}
		if sg.elem != nil { // sg.elem不为空说明接受者未忽略接收值
			typedmemclr(c.elemtype, sg.elem) // 给等待读取的设置零值
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}

		// 把goroutine取出来
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp) // 将goroutine推入glist
	}

	// 所有想要发送的goroutine，对不起，你们panic吧
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp) // 将goroutine推入glist
	}
	unlock(&c.lock)

	// 遍历gList，把其中的所有goroutine唤醒
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

对于chan的关闭，主要有两点：
- 对于待接收的chan，如果chan缓冲区中没有值了，则返回其一个零值，如果缓冲区还有值，则把值交给它
- 对于待发送的chan，对不起了宝贝们，你们都给爷爷panic吧

## References

https://golang.design/go-questions/channel/struct/

https://codeburst.io/diving-deep-into-the-golang-channels-549fd4ed21a8