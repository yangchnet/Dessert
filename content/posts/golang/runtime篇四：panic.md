---
author: "李昌"
title: "runtime篇四：panic"
date: "2022-08-12"
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

## 1. panic的底层结构

panic在runtime中的底层表示是`runtime._panic`结构体。

```go
type _panic struct {
	argp      unsafe.Pointer // 指向defer调用时参数的指针
	arg       any            // panic参数
	link      *_panic        // 连接到更早的_panic
	pc        uintptr        // 程序计数器
	sp        unsafe.Pointer // 栈指针
	recovered bool           // 当前panic是否被recover恢复
	aborted   bool           // 当前panic是否被中止
	goexit    bool           // 是否调用了runtime.Goexit
}
```

类似于`_defer`，panic也被组织成链表结构，多个panic通过`link`字段连接成一个链表。

在`_panic`结构体中，pc、sp、goexit三个字段是为了修复`runtime.Goexit`带来的问题引入的<sup>[[1]](https://github.com/golang/go/commit/7dcd343ed641d3b70c09153d3b041ca3fe83b25e)</sup>.


## 2. 调用panic

在函数中调用panic时，底层会调用`runtime.gopanic`，其源码如下：
```go
func gopanic(e any) {
	gp := getg()         // 获取当前g

    // ...
    // 此处省略部分代码

	var p _panic
	p.arg = e          // panic参数
	p.link = gp._panic // 头插
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	// 省略defer调用部分

	// ran out of deferred calls - old-school panic now
	// Because it is unsafe to call arbitrary user code after freezing
	// the world, we call preprintpanics to invoke all necessary Error
	// and String methods to prepare the panic strings before startpanic.
	preprintpanics(gp._panic)

	fatalpanic(gp._panic) // should not return
	*(*int)(nil) = 0      // not reached
}
```

先看panic主干流程，首先获取当前发生了panic的`g`，然后新建了一个`_panic`，将其字段赋值后，以头插的形式插入到`g`的`_panic`链表中，在函数的最后，调用了`runtime.fatalpanic`，这个函数实现了无法被恢复的程序崩溃：

```go
func fatalpanic(msgs *_panic) {
	pc := getcallerpc()
	sp := getcallersp()
	gp := getg()
	var docrash bool
	// Switch to the system stack to avoid any stack growth, which
	// may make things worse if the runtime is in a bad state.

    // 切换到系统栈以避免用户栈增长
	systemstack(func() {
        // startpanic_m在应该打印panic信息时返回true
		if startpanic_m() && msgs != nil { // 
			atomic.Xadd(&runningPanicDefers, -1)

			printpanics(msgs) // 打印panic信息
		}

		docrash = dopanic_m(gp, pc, sp)
	})

	if docrash {
		crash()
	}

	systemstack(func() {
		exit(2)
	})

	*(*int)(nil) = 0 // not reached
}
```

`runtime.fatalpanic`最后调用`exit(2)`终止程序，返回值为2.

## 3. 在有defer调用时panic

上面介绍的情况是在函数运行时没有设置defer调用，然后直接panic，现在来看具有defer调用的函数发生panic时会怎样。

回顾[runtime篇三：defer]()我们知道，程序的defer调用以`_defer`链表的形式存储在`g`中。

先大致看下源码:

```go
func gopanic(e any) {
	gp := getg()         // 获取当前g

    // 省略部分代码

	var p _panic
	p.arg = e          // panic参数
	p.link = gp._panic // 头插
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p))) // 将当前这个panic赋值给当前defer

	atomic.Xadd(&runningPanicDefers, 1)

	// By calculating getcallerpc/getcallersp here, we avoid scanning the
	// gopanic frame (stack scanning is slow...)
	addOneOpenDeferFrame(gp, getcallerpc(), unsafe.Pointer(getcallersp())) // 这里添加了一个open-code defer

    // 检查g中是否还存在defer调用
	for {
		d := gp._defer // 尝试获取_defer
		if d == nil { // 如果没有设置_defer，则直接跳出
			break
		}

        // 如果defer被更早的panic或Goexit启动了（或者在程序到达这里之前，又触发了一个新的panic），
		// 则将当前defer移出defer链表，先前的panic将不再执行，但确保先前的Goexit继续执行

		if d.started { // defer已经被启动了
			if d._panic != nil { // defer函数中也存在panic
				d._panic.aborted = true // 终止defer的panic
			}
			d._panic = nil
			if !d.openDefer { // 未使用开放编码
				d.fn = nil
				gp._defer = d.link // 继续检查下一个defer
				freedefer(d)
				continue
			}
		}
		// Mark defer as started, but keep on list, so that traceback
		// can find and update the defer's argument frame if stack growth
		// or a garbage collection happens before executing d.fn.
		d.started = true // 将defer标记为启动

		// Record the panic that is running the defer.
		// If there is a new panic during the deferred call, that panic
		// will find d in the list and will mark d._panic (this panic) aborted.
		d._panic = (*_panic)(noescape(unsafe.Pointer(&p))) // 将当前panic赋值給defer

		done := true
		if d.openDefer {
			done = runOpenDeferFrame(gp, d)
			if done && !d._panic.recovered {
				addOneOpenDeferFrame(gp, 0, nil)
			}
		} else {
			p.argp = unsafe.Pointer(getargp())
			d.fn() // 调用defer函数
		}
		p.argp = nil

		// Deferred function did not panic. Remove d.
		if gp._defer != d {
			throw("bad defer entry in panic")
		}
		d._panic = nil

		// trigger shrinkage to test stack copy. See stack_test.go:TestStackPanic
		//GC()

		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		if done { // 如果完成了defer函数
			d.fn = nil
			gp._defer = d.link
			freedefer(d)
		}
		if p.recovered { // 如果panic被recover，则继续执行下一个panic
			// 省略recover部分
		}
	}

	preprintpanics(gp._panic)

	fatalpanic(gp._panic) // should not return
	*(*int)(nil) = 0      // not reached
}
```

有点复杂，结合具体程序来看这段代码：
```go
1│ package main
2│
3│ func main() {
4│     defer func() {
5│         panic("2")
6│     }()
7│     panic("1")
8│ }
```

对于这个程序，我们来分析它的运行过程，首先程序在运行到第4行时，会将这个defer放入`g`的`_defer`链表中，这个defer的fn字段指向`func(){panic("2")}`。然后程序继续执行，来到第7行，在这里调用了`runtime.gopanic`函数。

1. 新建了一个`_panic`结构体，并将其插入到`g`的`_panic`链表头部，这里称为`panic1`
2. 在`g`上又新增了一个open-coded defer，现在`g`上有两个defer了，第一个为我们调用defer产生的（暂称为`mydefer`），第二个为open-coded defer，是runtime添加的(暂称`openDefer`)
3. 当`g`上还有defer时，取出第一个defer，这里为`mydefer`
   1. `mydefer`没有在运行
   2. 标记`mydefer`为运行状态，将`panic1`放入`mydefer`的_panic字段
   3. 检查`mydefer`不是open-coded defer，调用`_defer.fn()`

> 这里暂停一下，我们需要明确此时`g`中`_defer`和`_panic`的状态，在调用`_defer.fn()`之前，`g`中有两个defer，分别为`mydefer`、`openDefer`，且`mydefer.link = openDefer`：有一个panic，为`panic1`。且，`mydefer._panic = panic1`.

继续，这里调用的`_defer.fn()`为`func(){panic("2")}`，在defer函数中再一次调用了panic，注意这里进行了栈帧的切换，当前的panic变成了`panic2`。这次调用panic的执行过程为：
1. 新建一个新建了一个`_panic`结构体，并将其插入到`g`的`_panic`链表头部，这里称为`panic2`
2. 这回不再增加新的open-coded defer
3. 当`g`上还有defer时，取出第一个defer，这里为`mydefer`
   1. `mydefer`在运行
      1. `mydefer._panic`不为空，将其标记为aborted，即把`panic1`标记为aborted
      2. `mydefer`不是open-coded defer，将`mydefer.fn`设为空，将`mydefer`从`g._defer`链表中取出
      3. 重新检查`g._defer`中是否还存在defer

> 再次暂停，此时`g`上只剩下一个`_defer`：`openDefer`

继续：
1. `g`上还有`openDefer`存在
2. `openDefer`不在运行，将其标记为运行，将`panic2`赋值到`openDefer._panic`上
3. 执行`openDefer`
4. 完成`openDefer`后，free it
5. 检查是否有recover调用
6. 调用fatalpanic使程序崩溃

分析完毕。

## 4. recover

编译器在将关键字`recover`转换成`runtime.gorecover`:
```go
func gorecover(argp uintptr) any {
	gp := getg()
	p := gp._panic
	if p != nil && !p.goexit && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
```

这个函数很简单，先是获取`g`，然后再获取`g._panic`的第一个元素，然后将其`recovered`标志设为true。

让我们先结合具体程序来简单看下recover流程：
```go
1 │ package main
2 │
3 │ func main() {
4 │     defer func() {
5 │         if r := recover(); r != nil {
6 │             println(r)
7 │         }
8 │     }()
9 │     panic("1")
10│ }
```

首先程序会执行到第9行，然后一个`panic1`将会被添加到`g._panic`链表上；然后在`runtime.gopanic`中会添加一个`openDefer`,  然后调用defer.fn，会执行到recover，根据`runtime.gorecover`，会将`g._panic`的第一个元素取出，然后将其设置为可recover。

现在，`g`中有了两个`_defer`（`mydefer.link = openDefer`），一个`_panic`（`panic1`），且`mydefer`被设置`recovered = true`。我们可以开始分析recover是怎么执行的了：

而对recover的处理，还要来看`runtime.gopanic`:
```go
1 │func gopanic(e any) {
2 │    ...
3 │    for {
4 │        d := gp._defer // panic退出程序前，要执行defer
5 │        if d == nil {
6 │            break
7 │        }
8 │
9 │        ...
10│
11│        if p.recovered { // 如果panic被recover，则继续执行下一个panic
12│            gp._panic = p.link
13│            if gp._panic != nil && gp._panic.goexit && gp._panic.aborted {
14│                // A normal recover would bypass/abort the Goexit.  Instead,
15│                // we return to the processing loop of the Goexit.
16│                gp.sigcode0 = uintptr(gp._panic.sp)
17│                gp.sigcode1 = uintptr(gp._panic.pc)
18│                mcall(recovery)
19│                throw("bypassed recovery failed") // mcall should not return
20│            }
21│            atomic.Xadd(&runningPanicDefers, -1)
22│
23│            // After a recover, remove any remaining non-started,
24│            // open-coded defer entries, since the corresponding defers
25│            // will be executed normally (inline). Any such entry will
26│            // become stale once we run the corresponding defers inline
27│            // and exit the associated stack frame. We only remove up to
28│            // the first started (in-progress) open defer entry, not
29│            // including the current frame, since any higher entries will
30│            // be from a higher panic in progress, and will still be
31│            // needed.
32│            d := gp._defer
33│            var prev *_defer
34│            if !done {
35│                // Skip our current frame, if not done. It is
36│                // needed to complete any remaining defers in
37│                // deferreturn()
38│                prev = d
39│                d = d.link
40│            }
41│            for d != nil { // 这里去除了已经开始的open defer
42│                // 暂时省略
43│            }
44│
45│            gp._panic = p.link
46│            // Aborted panics are marked but remain on the g.panic list.
47│            // Remove them from the list.
48│            for gp._panic != nil && gp._panic.aborted {
49│                gp._panic = gp._panic.link
50│            }
51│            if gp._panic == nil { // must be done with signal
52│                gp.sig = 0
53│            }
54│            // Pass information about recovering frame to recovery.
55│            gp.sigcode0 = uintptr(sp)
56│            gp.sigcode1 = pc
57│            mcall(recovery)
58│            throw("recovery failed") // mcall should not return
59│        }
60│    }
61│
62│    // ...
63│}
```

当程序开始进行recover时，首先在13行会做一个if判断。正常recover是会绕过Goexit的，所以为了解决这个，添加了这个判断，这样就可以保证Goexit也会被recover住，这里是通过从runtime._panic中取出了程序计数器pc和栈指针sp并且调用runtime.recovery函数触发goroutine的调度，调度之前会准备好 sp、pc 以及函数的返回值。

对于我们的程序来说，并未调用Goexit，因此这里会跳过，然后在32～43行，由于`done`为true，这里d将会被赋值为`mydefer`，然后来到45行，将`defer1`从`g._panic`链表中取出，然后将余下的被标记为aborted的`_panic`删除，这里没有。

55、56两行设置`g`的`sigcode0`、`sigcode1`指针，用于跳转，然后57行`mcall(recovery)`。

`mcall`是一个汇编实现的函数，其函数原型为：`func mcall(fn func(*g))`，其主要功能是切换到`g0`的栈，然后调用`fn(g)`，`fn(g)`将不会返回，并且触发`g`的重新调度。

这里的`fn`就是`recovery`，来看：
```go
func recovery(gp *g) {
	// Info about defer passed in G struct.
	sp := gp.sigcode0
	pc := gp.sigcode1

	// d's arguments need to be in the stack.
	if sp != 0 && (sp < gp.stack.lo || gp.stack.hi < sp) {
		print("recover: ", hex(sp), " not in [", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n")
		throw("bad recovery")
	}

	// Make the deferproc for this d return again,
	// this time returning 1. The calling function will
	// jump to the standard return epilogue.
	gp.sched.sp = sp
	gp.sched.pc = pc
	gp.sched.lr = 0
	gp.sched.ret = 1
	gogo(&gp.sched)
}
```

没什么特别的魔法，就是重新设置了`g`的一些指针，然后对其重新进行调度。

这样就完成了panic的恢复。

## 5. 场景分析

**Q1** 为什么recover必须放在defer里面
**A1** 不放到defer里面，没机会运行啊。。。


**Q2** 为什么如下使用方法不会恢复：
```go
func main() {
	defer recover()
	panic("1")
}
```
**A2** 在调用recover()函数时，会有如下if条件：
```go
if p != nil && !p.goexit && !p.recovered && argp == uintptr(p.argp) {
	p.recovered = true
	return p.arg
}
```
这里`p != nil && !p.goexit && !p.recovered`会满足，而`argp`和`uintptr(p.argp)`并不相等，`argp`是`runtime.gopinic`报告的参数指针，`p.argp`是最顶层 defer 函数调用的参数指针，二者并不相等。

**Q3** 下面这段代码将输出什么？为什么？
```go
func main() {
	defer func() { // topdefer
		fmt.Println(recover())
	}()

	defer panic(3) // defer3
	defer panic(2) // defer2
	defer panic(1) // defer1
	panic(0)
}
```
**A3** 将输出3.

分析：在`runtime.gopanic`中，有如下代码：
```go
if d.started {
	if d._panic != nil {
		d._panic.aborted = true
	}
	d._panic = nil
	if !d.openDefer {
		d.fn = nil
		gp._defer = d.link
		freedefer(d)
		continue
	}
}
```

当我们执行到`panic(0)`后，将返回执行`defer1`，这时`defer1`被设置为`started`，`panic(0)`被设置为`aborted`，然后`defer1`被释放；
紧接着执行`defer2`，`defer2`被设置为`started`，`panic(1)`被设置为`aborted`，然后`defer2`被释放；
紧接着执行`defer3`，`defer3`被设置为`started`，`panic(2)`被设置为`aborted`，然后`defer3`被释放；
最后执行到`topdefer`，又因如下代码：
```go
for gp._panic != nil && gp._panic.aborted {
	gp._panic = gp._panic.link
}
```
被标记为`aborted`的panic将被忽略，因此只剩下了`panic(3)`。

这样，最后输出的值就是3。

**Q4** 为什么recover不能捕获不同goroutine的panic
**A4** 查看`runtime.gorecover`源码：
```go
func gorecover(argp uintptr) any {
	gp := getg()
	p := gp._panic
	if p != nil && !p.goexit && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
```
这个函数获取了当前的`g`，并为其第一个`_panic`设置recover，跟其他`g`没有关系

**Q5** 为什么子goroutine的panic不被recover会造成整个程序的崩溃
**A5** 查看`runtime.fatalpanic`:
```go
func fatalpanic(msgs *_panic) {
	// ...

	systemstack(func() {
		exit(2)
	})

	*(*int)(nil) = 0 // not reached
}
```

其在执行`exit(2)`时，是在`systemstack`上执行的，因此整个程序都会退出。

END

## References

https://gfw.go101.org/article/panic-and-recover-more.html

https://golang.design/under-the-hood/zh-cn/part1basic/ch03lang/panic/

https://www.purewhite.io/2019/11/28/runtime-hacking-translate/

https://zhuanlan.zhihu.com/p/346514343

https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/

https://xiaomi-info.github.io/2020/01/20/go-trample-panic-recover/