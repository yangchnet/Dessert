---
author: "李昌"
title: "runtime篇五：slice"
date: "2022-08-14"
tags: ["golang", "runtime"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

> 本系列代码基于[golang1.19](https://github.com/golang/go/tree/1e5987635cc8bf99e8a20d240da80bd6f0f793f7)

- [runtime篇一：接口](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E4%B8%80%E6%8E%A5%E5%8F%A3/)
- [runtime篇二：通道](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E4%BA%8C%E9%80%9A%E9%81%93/)
- [runtime篇三：defer](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E4%B8%89defer/)
- [runtime篇四：panic](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E5%9B%9Bpanic/)
- [runtime篇五：slice](https://yangchnet.github.io/Dessert/posts/golang/runtime%E7%AF%87%E4%BA%94slice/)


> 推荐阅读：[slice和数组的区别](https://yangchnet.github.io/Dessert/posts/golang/slice%E5%92%8C%E6%95%B0%E7%BB%84%E7%9A%84%E5%8C%BA%E5%88%AB/)  [切片append规则](https://yangchnet.github.io/Dessert/posts/golang/%E5%88%87%E7%89%87append%E8%A7%84%E5%88%99/)

## 1. 切片的结构

切片的底层表示是`runtime.slice`:
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

非常简单，只有一个array指针指向底层数组结构，len标记切片长度，cap标记数组容量。

## 2. 新建一个切片
当我们使用如`make([]int, 0)`语法来新建一个切片的时候，调用的是`runtime.makeslice`函数：
```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```

这个函数也很简单，首先是使用`math.MulUintptr`计算slice需要的空间，`math.MulUintptr`会返回`_type`大小和len的乘积以及`overflow`标志来表明是否超出内存限制。如果超出内存限制或者len小于0，或len>cap，则分配失败，抛出panic。

否则直接调用`mallocgc`来为slice分配空间，并将返回的`unsafe.Pointer`赋值給slice.array

对于以下代码来说：
```go
a := make([]int, 10)
```

其slice构造过程为这样：
```s
0x0018 00024 (main.go:6)   LEAQ    type.int(SB), AX        # AX中存的是type.int，即int(_type)
0x001f 00031 (main.go:6)   MOVL    $10, BX                 # BX中存的是10，切片len
0x0024 00036 (main.go:6)   MOVQ    BX, CX                  # CX与BX相等，为cap = 10
0x0027 00039 (main.go:6)   PCDATA  $1, $0
0x0027 00039 (main.go:6)   CALL    runtime.makeslice(SB)   # runtime.makeslice(AX, BX, CX)
0x002c 00044 (main.go:6)   MOVQ    AX, "".a+56(SP)         # runtime.makeslice的返回值AX赋值给"".a+56(SP)，即slice.array
0x0031 00049 (main.go:6)   MOVQ    $10, "".a+64(SP)        # "".a+64(SP)，即slice.len 为10
0x003a 00058 (main.go:6)   MOVQ    $10, "".a+72(SP)        # "".a+72(SP)，即slice.cap 为10
```

## 3. 切片容量增长

> 大约2021年8月份，go社区对切片容量增长的方式进行了一次调整。具体讨论可见：https://groups.google.com/g/golang-nuts/c/UaVlMQ8Nz3o

### 3.1 之前的增长规则

先看源码
> runtime/slice.go
```go
func growslice(et *_type, old slice, cap int) slice {
	// 省略部分条件检查
    // ...

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.cap < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

    // 对切片进行内存对齐，可能会调整切片容量大小
    // ...

	return slice{p, old.len, newcap}
}
```

从代码中可以看出，当cap < 1024时，切片容量增长2倍，当cap > 1024时增长1.25倍，当然，在经过内存对齐等操作后，切片的长度可能不会严格等于原来的2倍或1.25倍。

存在的问题：
1. not monotonically increasing （不平滑）
```go
x1 := make([]int, 897)
x2 := make([]int, 1024)
y := make([]int, 100)
println(cap(append(x1, y...))) // 2048
println(cap(append(x2, y...))) // 1280
```

对于`append(x1, y...)`, 进行`growslice`，传入的`cap`为所需的最小容量，为：cap(x1) + cap(y) = 997。
此时`newcap`为897, `doublecap`为897×2 = 1794。由于`old.cap` = 897 < 1024, 所以`newcap` = `doublecap`，为1794,再经过内存对齐后，最终的容量为2048.

对于`append(x2, y...)`, 进行`growslice`，传入的`cap`为1124,此时`newcap`为1024, `doublecap`为1024×2 = 2048.
由于`cap` < `doublecap`且`newcap`< `cap`,因此`newcap` += `newcap` / 4 = 1280.经过内存对齐后，仍为1280.。

这样就产生了一个问题，在一定条件下，如果cap(A) < cap(B), 但二者都append一个相同长度的C后，反而cap(A) > cap(B)。

2. not symmetrical （不对称）
```go
x := make([]int, 98)
y := make([]int, 666)
println(cap(append(x, y...))) // 768
println(cap(append(y, x...))) // 1360
```

### 3.2 调整之后
源码：
```go
func growslice(et *_type, old slice, cap int) slice {

    // 省略部分条件检查
    // ...

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		const threshold = 256
		if old.cap < threshold {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				// Transition from growing 2x for small slices
				// to growing 1.25x for large slices. This formula
				// gives a smooth-ish transition between the two.
				newcap += (newcap + 3*threshold) / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	// 对切片进行内存对齐，可能会调整切片容量大小
    // ...

	return slice{p, old.len, newcap}
}
```

进行的调整：
- 增加了threshold = 256
- 增长策略不同的阈值从1024变为256
- 当切片容量大于阈值时，增长策略变化

原来的问题：
1. 增长不平滑
```go
x1 := make([]int, 897)
x2 := make([]int, 1024)
y := make([]int, 100)
println(cap(append(x1, y...))) // 1360
println(cap(append(x2, y...))) // 1536
```

2. 增长不对称
```go
x := make([]int, 98)
y := make([]int, 666)
println(cap(append(x, y...))) // 768
println(cap(append(y, x...))) // 1024
```

增长不平滑的问题得到了较好的解决，同时不对称的问题也得到了缓解。

### 3.3

这里要注意的是：
- 在经过append之后，slice.array发生了变化，可能不再是原来指向的地址了
- 在append中，最终要经过一次地址对齐，进行内存对齐之后，新 slice 的容量是要 大于等于 按照前半部分生成的newcap

## 4. Reference

https://golang.design/go-questions/slice/grow/

https://www.cnblogs.com/qcrao-2018/p/10631989.html

https://yangchnet.github.io/Dessert/posts/golang/slice%E5%92%8C%E6%95%B0%E7%BB%84%E7%9A%84%E5%8C%BA%E5%88%AB/

https://yangchnet.github.io/Dessert/posts/golang/%E5%88%87%E7%89%87append%E8%A7%84%E5%88%99/