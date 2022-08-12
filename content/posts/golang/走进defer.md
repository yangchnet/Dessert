---
author: "李昌"
title: "走进defer"
date: "2022-08-12"
tags: ["golang"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

## 1. defer是什么

`defer`，是一种特殊的机制，在调用普通函数或方法前加上关键字defer，就完成了defer所需要的语法。当defer语句被执行时，跟在defer后面的函数会被延迟执行。直到包含该defer语句的函数执行完毕时，defer后的函数才会被执行，多个defer的执行顺序与声明顺序相反。

对于defer的使用及需要注意的地方，可参考[defer用法](https://yangchnet.github.io/Dessert/posts/golang/defer%E7%94%A8%E6%B3%95/)。这里不再讨论。

在golang runtime中，defer被描述为一个结构体：
```go
type _defer struct {
	started bool      // 是否开始执行defer函数
	heap    bool      // 是否分配在堆上

	openDefer bool    // 是否经过开放编码（open-coded）的优化
	sp        uintptr // 调用defer时的栈指针 stack pointer
	pc        uintptr // 调用defer函数时的pc值
	fn        func()  // defer关键字传入的函数， 当使用open-coded defer时可为空
	_panic    *_panic // defer运行时的panic
	link      *_defer // 在goroutine中的下一个defer，可指向堆或栈

    // 如果openDefer为true，则下面的字段将记录具有open-code defer的栈帧和相关的函数。
    // 上面的sp将为帧的sp，pc将为defer调用的地址。
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr        // value of varp for the stack frame

	framepc uintptr     // 当前栈帧的pc
}
```

一个`_defer`结构体是defer调用的一环，一个函数中的多个defer被组织为链表的形式。defer有的分配在栈上，有的分配在堆上，但逻辑上他们都是属于栈的，因此在进行访问不需要加写屏障。

可以看到， `_defer`中保存了很多字段，主要可分为三类，一是指示defer自身状态的标志位，如`started`,`heap`等；二是保存defer上下文，如`sp`, `pc`等；三是有关开放编码的一些字段。

### 1.1 open-coded defer

在`_defer`中，有一个字段`openDefer`指示是否这个defer经过开放编码的优化，那么，什么是`open code defer`?

对于一个defer函数来说，将其放入defer链表调用与直接调用是存在性能差异的，例如直接调用的耗时可能在6ns左右，而从defer链表中调用的耗时在35ns左右<sup>[[1]](https://github.com/golang/proposal/blob/master/design/34481-opencoded-defers.md)</sup>，其中的主要原因在于，将函数放入defer链表或将其取出执行时，需要对上下文环境做保存和重做。因此，在go1.14中对其进行了优化，对于满足一定条件的defer，会进行open-coded优化。

例如，对于如下代码：
```go
defer f1(a)
if cond {
 defer f2(b)
}
body...
```

将被编译为：
```go
// 设置标识位
deferBits |= 1<<0
tmpF1 = f1
tmpA = a
if cond {
 deferBits |= 1<<1
 tmpF2 = f2
 tmpB = b
}
body...
exit:
// 在退出时检查标记位，以判断某个defer函数是否需要被执行
if deferBits & 1<<1 != 0 {
 deferBits &^= 1<<1
 tmpF2(tmpB)
}
if deferBits & 1<<0 != 0 {
 deferBits &^= 1<<0
 tmpF1(tmpA)
}
```

这里不再将defer简单的放入defer链表了事，而是将其添加到了函数的底部，并通过标志位进行检查。

### 1.2 newdefer

```go
// 每个p维护一个defer池

// Allocate a Defer, usually using per-P pool.
// Each defer must be released with freedefer.  The defer is not
// added to any defer chain yet.
func newdefer() *_defer {
	var d *_defer
	mp := acquirem() // 获取当前g的m, m.locks++
	pp := mp.p.ptr() // m上的p
	if len(pp.deferpool) == 0 && sched.deferpool != nil { // p的deferpool为空，但sched的deferpool不为空
		lock(&sched.deferlock) // 加锁
		for len(pp.deferpool) < cap(pp.deferpool)/2 && sched.deferpool != nil { // p的deferpool长度小于其容量的一半，且sched的deferpool不为空
			d := sched.deferpool // 从sched.deferpool取出一个_defer结构体
			sched.deferpool = d.link // 将取出的_defer的下一个_defer重新链接到sched.deferpool
			d.link = nil // 断链
			pp.deferpool = append(pp.deferpool, d) // 取出的这个_defer加入到p的deferpool中
		}
		unlock(&sched.deferlock) // 解锁
	}
	if n := len(pp.deferpool); n > 0 { // 从p的deferpool中取出一个_defer
		d = pp.deferpool[n-1]
		pp.deferpool[n-1] = nil
		pp.deferpool = pp.deferpool[:n-1]
	}
	releasem(mp) // m.locks--
	mp, pp = nil, nil

	if d == nil { // 如果从sched的deferpool和p的deferpool中都没有取到现成的_defer，只要新构建一个
		// Allocate new defer.
		d = new(_defer)
	}
	d.heap = true // 默认defer分配在堆上
	return d
}
```

从newdefer函数中中可以看出，`sched`和`p`上均有deferpool，里面均保存了若干空的`_defer`对象以便复用，在想要创建一个新的`_defer`时，如果`p`的deferpool为空，会尝试从`sched`的deferpool中去取，然后放在`p`的deferpool中。如果两个地方都没有空`_defer`对象，那就只要新建一个了，而新建的这个`_defer`对象，默认分配在堆上。

既然在newdefer时会从`sched.deferpool`中取，那么在释放`_defer`，响应的应该也会将用过的空`_defer`放入`sched`或`p`的deferpool中去。

### 1.3 freedefer

```go
func freedefer(d *_defer) {
	d.link = nil // 断链
	// After this point we can copy the stack.

	if d._panic != nil {
		freedeferpanic()
	}
	if d.fn != nil {
		freedeferfn()
	}
	if !d.heap { // _defer不在堆上，不需要单独进行释放
		return
	}

	mp := acquirem() // 获取当前g的m
	pp := mp.p.ptr() // 获取m的p
	if len(pp.deferpool) == cap(pp.deferpool) { // p的deferpool满了
		// Transfer half of local cache to the central cache.
		// 将一半的_defer从p.deferpool移动到sched.deferpool中
		var first, last *_defer
		// 这个循环取p.deferpool中的一半_defer组成一个链表，first、last分别为其第一、最后一个元素
		for len(pp.deferpool) > cap(pp.deferpool)/2 {
			n := len(pp.deferpool)
			d := pp.deferpool[n-1] // d是p.deferpool中的最后一个_defer
			pp.deferpool[n-1] = nil
			pp.deferpool = pp.deferpool[:n-1] // p.deferpool长度-1
			if first == nil {
				first = d
			} else {
				last.link = d
			}
			last = d
		}
		lock(&sched.deferlock) // 加锁
		last.link = sched.deferpool // 头插
		sched.deferpool = first
		unlock(&sched.deferlock) // 释放锁
	}


	*d = _defer{} // 清空内部字段

	pp.deferpool = append(pp.deferpool, d) // 直接放入p.deferpool

	releasem(mp)
	mp, pp = nil, nil
}
```

当一个`_defer`要被释放时，会尝试将其放入当前`g`所属`p`的deferpool中去，如果`p`的deferpool满了，则将`p`的deferpool中的一半元素放到`sched`的deferpool中，然后再将`_defer`放到`p.deferpool`

## 2. 调用defer

从`_defer`的结构我们可以看到，`_defer`可能会分配到栈上，也可能会分配在堆上。

### 2.1 栈上分配

对于大部分场景来说，在不开启`open-coded defer`的情况下会使用栈上分配。

对于如下代码：
```go
package main

import "fmt"

func main() {
        defer fmt.Println(1)
        defer fmt.Println(2)
        defer fmt.Println(3)
        defer func() {
                fmt.Println(4)
        }()
}
```
对其进行编译:
```sh
go tool compile -S -N main.go  # -N参数禁止编译时优化，如果不加-N，会使用open-coded defer
```

```s
...
0x01e2 00482 (main.go:9)        LEAQ    "".main.func1·f(SB), CX
0x01e9 00489 (main.go:9)        MOVQ    CX, ""..autotmp_12+112(SP)
0x01ee 00494 (main.go:9)        LEAQ    ""..autotmp_12+88(SP), AX
0x01f3 00499 (main.go:9)        CALL    runtime.deferprocStack(SB)
...
```

0x01e2处，将`"".main.func1·f(SB)`函数加载到CX，接下来将CX中的内容放置在`""..autotmp_12+112(SP)`。下一步将`""..autotmp_12+88(SP)`作为`runtime.deferprocStack`的参数。

这里可以简单分析一下，从`""..autotmp_12+88(SP)`到`""..autotmp_12+112(SP)`，这中间有24个字节的宽度，而`_defer`的前几个字段如下：
```go
type _defer struct {
	started bool      // 1字节
	heap    bool      // 1字节

	openDefer bool    // 1字节
	sp        uintptr // 8字节
	pc        uintptr // 8字节
	fn        func()  // defer关键字传入的函数， 当使用open-coded defer时可为空
	...
}
```

在fn之前共有1+1+1+8+8=19个字节的宽度，但实际上在进行内存分配时，在3个bool型字段后，需要添加5字节进行内存对齐，因此从`_defer`的开始地址到fn的地址，这中间的宽度就变成了1+1+1+5+8+8=24。

因此我们可以知道这几行汇编的前三行，是将函数赋值到`_defer`的fn字段上。

知道了这些，我们再来看runtime.deferprocStack`：

```go
func deferprocStack(d *_defer) {
	gp := getg() // 获取当前g
	if gp.m.curg != gp {
		// go code on the system stack can't defer
		throw("defer on system stack")
	}

	// fn已被赋值过，其他的字段在这里赋值
	d.started = false
	d.heap = false
	d.openDefer = false
	d.sp = getcallersp()
	d.pc = getcallerpc()
	d.framepc = 0
	d.varp = 0

	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.fd)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
}
```

将`_defer`分配在栈上，并将其放入`g`的defer链表中。

### 2.2 堆上分配

堆上分配是默认的兜底方案。

编译如下代码：
```go
package main

import "fmt"

func main() {
        for i := 0; i < 10; i++ {
                defer func(i int) {
                        fmt.Println(i)
                }(i)
        }
}
```

```sh
go tool compile -S -N  main.go
```

可看到其调用了`runtime.deferproc`:
```go
func deferproc(fn func()) {
	gp := getg() // 获取g
	if gp.m.curg != gp {
		// go code on the system stack can't defer
		throw("defer on system stack")
	}

	d := newdefer() // 获取一个_defer
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	d.link = gp._defer // 将_defer以头插法插入到g的_defer链表中
	gp._defer = d
	d.fn = fn            // 设置_defer函数
	d.pc = getcallerpc() // 保存上下文环境
	d.sp = getcallersp()

	return0()
}
```

相对于栈上分配，在堆上分配设计到`newdefer`的调用，而通过上文的分析我们可知，在调用`newdefer`时，会涉及到`sched`、`p`以及对`m`的加锁，因此性能上不如栈上分配。且，栈上分配不需要写屏障。

## 3. 执行defer

```go
package main

import "fmt"

func main() {
        defer fmt.Println(1)
        defer fmt.Println(2)
        defer fmt.Println(3)
        defer func() {
                fmt.Println(4)
        }()
}
```

查看以上代码的汇编代码时，可看到：
```s
0x0200 00512 (main.go:12)       CALL    runtime.deferreturn(SB)
0x0205 00517 (main.go:12)       MOVQ    440(SP), BP
0x020d 00525 (main.go:12)       ADDQ    $448, SP
0x0214 00532 (main.go:12)       RET
0x0215 00533 (main.go:9)        CALL    runtime.deferreturn(SB)
0x021a 00538 (main.go:9)        MOVQ    440(SP), BP
0x0222 00546 (main.go:9)        ADDQ    $448, SP
0x0229 00553 (main.go:9)        RET
0x022a 00554 (main.go:8)        CALL    runtime.deferreturn(SB)
0x022f 00559 (main.go:8)        MOVQ    440(SP), BP
0x0237 00567 (main.go:8)        ADDQ    $448, SP
0x023e 00574 (main.go:8)        RET
0x023f 00575 (main.go:8)        NOP
0x0240 00576 (main.go:7)        CALL    runtime.deferreturn(SB)
0x0245 00581 (main.go:7)        MOVQ    440(SP), BP
0x024d 00589 (main.go:7)        ADDQ    $448, SP
0x0254 00596 (main.go:7)        RET
0x0255 00597 (main.go:6)        CALL    runtime.deferreturn(SB)
```
可以看到会defer的调用是按照其定义的顺序反向调用的。

在调用了defer的函数中，编译器会自动在函数尾部插入对`runtime.deferreturn`的调用。

```go
func deferreturn() {
	gp := getg() // 获取当前g
	for {
		d := gp._defer // defer链表的第一个defer，也是最后一个被定义的defer
		if d == nil {
			return
		}
		sp := getcallersp()
		if d.sp != sp {
			return
		}
		if d.openDefer { // 如果开启了open-coded defer
			done := runOpenDeferFrame(gp, d)
			if !done {
				throw("unfinished open-coded defers in deferreturn")
			}
			gp._defer = d.link //// 删除链表的第一个元素
			freedefer(d)
			// If this frame uses open defers, then this
			// must be the only defer record for the
			// frame, so we can just return.
			return
		}

		fn := d.fn // 取出defer函数
		d.fn = nil
		gp._defer = d.link // 删除链表的第一个元素
		freedefer(d) // 释放_defer
		fn() // 调用defer函数
	}
}
```

## 4. defer的三种处理机制

经过以上的分析，我们可以总结一下，defer的三种处理机制：
1. open-coded defer
2. 栈上分配的defer
3. 堆上分配的defer

其执行效率为：`open-coded defer > 栈上分配的defer > 堆上分配的defer`

### 4.1 处理机制的选择

- 在defer语句出现在了循环语句里，或者无法执行更高阶的编译器优化时，亦或者同一个函数中使用了过多的defer时，会使用堆上分配<sup>[[2]](https://cloud.tencent.com/developer/article/1596802)</sup>

- 满足以下三种情况时<sup>[[3]](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#535-%e5%bc%80%e6%94%be%e7%bc%96%e7%a0%81)</sup>，会使用open-coded defer
  	1. 没有禁用编译器优化，即没有设置 -gcflags "-N"；
  	2. 函数内 defer 的数量不超过 8 个，且返回语句与延迟语句个数的乘积不超过 15；
  	3. defer 不是在循环语句中。

- 其他大部分情况下，会使用栈上分配

**END**

## References

https://github.com/golang/proposal/blob/master/design/34481-opencoded-defers.md

https://cloud.tencent.com/developer/article/1596802

https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#535-%e5%bc%80%e6%94%be%e7%bc%96%e7%a0%81

https://juejin.cn/post/6844904078569373710

https://xargin.com/go-1-13-defer-change/

https://juejin.cn/post/6975686540601245709

https://zhuanlan.zhihu.com/p/401339057