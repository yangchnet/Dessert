---
author: "李昌"
title: "标准库之unsafe"
date: "2022-03-04"
tags: ["golang", "unsafe"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

## 1. Go中对指针的限制
- Go 的指针不能进行数学运算。
- 不同类型的指针不能相互转换。
- 不同类型的指针不能使用 == 或 != 比较。只有在两个指针类型相同或者可以相互转换的情况下，才可以对两者进行比较。另外，指针可以通过 == 和 != 直接和 nil 作比较。
- 不同类型的指针变量不能相互赋值。
  
使用unsafe包，可以一定程度上打破这些限制，那么为什么要打破这些限制。请看下文。

## 2. unsafe.Pointer
unsafe.Pointer的定义
```golang
type ArbitraryType int

type Pointer *ArbitraryType
```

unsafe 包提供了 2 点重要的能力：
1. 任何类型的指针和 unsafe.Pointer 可以相互转换。
2. uintptr 类型和 unsafe.Pointer 可以相互转换。

![20220304164029](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220304164029.png)

pointer 不能直接进行数学运算，但可以把它转换成 uintptr，对 uintptr 类型进行数学运算，再转换成 pointer 类型。利用这两个对象的相互转换，就可以打破上述4个限制。

```go
// uintptr 是一个整数类型，它足够大，可以存储
type uintptr uintptr
```

还有一点要注意的是，uintptr 并没有指针的语义，意思就是 uintptr 所指向的对象会被 gc 无情地回收.而 unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收。

## 3. 利用unsafe获取slice和map的长度
slice和map的长度都存储在其内部变量中，因此我们先来看这两个结构体定义：
```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int // 长度 
    cap   int // 容量
}
```

调用 make 函数新建一个 slice，底层调用的是 makeslice 函数，返回的是 slice 结构体：
```go
func makeslice(et *_type, len, cap int) slice
```

因此我们可以通过 unsafe.Pointer 和 uintptr 进行转换，得到 slice 的字段值。

```go
func main() {
	s := make([]int, 9, 20)
    // Len: &s => pointer => uintptr => pointer => *int => int
	var Len = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(8)))
	fmt.Println(Len, len(s)) // 9 9

    // Cap: &s => pointer => uintptr => pointer => *int => int
	var Cap = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(16)))
	fmt.Println(Cap, cap(s)) // 20 20
}
```

总体思路是，先对结构体取指针，再转换成unsafe.Pointer，再转换成uintptr,然后做加法使指针偏移，然后重新转换为unsafe.Pointer,再转换成（*int）类型，最后取引用，得到值。可以看出，其中的重点在于计算偏移量。

对于map

```go
type hmap struct {
	count     int
	flags     uint8
	B         uint8
	noverflow uint16
	hash0     uint32

	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer
	nevacuate  uintptr

	extra *mapextra
}
```

和 slice 不同的是，makemap 函数返回的是 hmap 的指针，注意是指针：

```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap
```

我们依然能通过 unsafe.Pointer 和 uintptr 进行转换，得到 hamp 字段的值，只不过，现在 count 变成二级指针了：
```go
func main() {
	mp := make(map[string]int)
	mp["q"] = 100
	mp["s"] = 18

    // &mp => pointer => **int => int
	count := **(**int)(unsafe.Pointer(&mp))
	fmt.Println(count, len(mp)) // 2 2
}
```

## 4. 修改私有成员
总体思路是：通过 offset 函数获取结构体成员的偏移量，进而获取成员的地址，读写该地址的内存，就可以达到改变成员值的目的。

这里需要注意的是：结构体会被分配一块连续的内存，结构体的地址也代表了第一个成员的地址。

```go
package main

import (
	"fmt"
	"unsafe"
)

type Programmer struct {
	name string
	language string
}

func main() {
	p := Programmer{"a", "go"}
	fmt.Println(p)
	
	name := (*string)(unsafe.Pointer(&p))
	*name = "b"

	lang := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Offsetof(p.language)))
    // lang := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Sizeof(string(""))))
	*lang = "Golang"

	fmt.Println(p)
}
```

## 5. 字符串和byte切片的零拷贝转换
string和byte切片的内部结构如下：
```go
type StringHeader struct {
	Data uintptr
	Len  int
}

type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

转换：
```go
func string2bytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(&s))
}
func bytes2string(b []byte) string{
	return *(*string)(unsafe.Pointer(&b))
}
```

但这样转换会存在一个问题，字符串被转换为切片后，切片的cap无法准确得出。

## References

https://golang.design/go-questions/stdlib/unsafe/zero-conv/




