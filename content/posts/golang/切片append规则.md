---
author: "李昌"
title: "切片append规则"
date: "2022-03-19"
tags: ["golang", "runtime"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

> 大约2021年8月份，go社区对切片容量增长的方式进行了一次调整。具体讨论可见：https://groups.google.com/g/golang-nuts/c/UaVlMQ8Nz3o

## 1. 之前的增长规则

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

## 2. 调整之后
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



