---
author: "李昌"
title: "Go汇编之定义基本数据类型"
date: "2022-06-10"
tags: ["golang", "汇编"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

## 1. Go汇编基础

> 这里只介绍本文会用到的语法

#### 4个虚拟寄存器

- FP: Frame pointer：伪FP寄存器对应函数的栈帧指针，一般用来访问函数的参数和返回值；golang语言中，函数的参数和返回值，函数中的局部变量，函数中调用子函数的参数和返回值都是存储在栈中的，我们把这一段栈内存称为栈帧（frame），伪FP寄存器对应栈帧的底部，但是伪FP只包括函数的参数和返回值这部分内存，其他部分由伪SP寄存器表示；注意golang中函数的返回值也是通过栈帧返回的，这也是golang函数可以有多个返回值的原因；

- PC: Program counter：指令计数器，用于分支和跳转，它是汇编的IP寄存器的别名；

- SB: Static base pointer：一般用于声明函数或者全局变量，对应代码区（text）内存段底部；可认为是内存的起源，所以符号`foo(SB)`就是名称`foo`作为内存中的一个地址。这种形式被用于命名全局函数和数据，如果将`<>`添加到名称中，如`foo<>(SB)`，则代表此标识符只在当前源文件中可见。可对名称添加偏移量，如`foo+4(SB)`指foo开头之后的四个字节。

- SP: Stack pointer：指向当前栈帧的局部变量的开始位置，一般用来引用函数的局部变量，这里需要注意汇编中也有一个SP寄存器，它们的区别是：1.伪SP寄存器指向栈帧（不包括函数参数和返回值部分）的底部，真SP寄存器对应栈的顶部；所以伪SP寄存器一般用于寻址函数局部变量，真SP寄存器一般用于调用子函数时，寻址子函数的参数和返回值（后面会有具体示例演示）；2.当需要区分伪寄存器和真寄存器的时候只需要记住一点：伪寄存器一般需要一个标识符和偏移量为前缀，如果没有标识符前缀则是真寄存器。比如(SP)、+8(SP)没有标识符前缀为真SP寄存器，而a(SP)、b+8(SP)有标识符为前缀表示伪寄存器；

所有用户定义的符号都作为偏移量写入伪寄存器 FP（参数和局部变量）和 SB（全局变量）

#### 常量

Go汇编语言中常量以$美元符号为前缀。常量的类型有整数常量、浮点数常量、字符常量和字符串常量等几种类型。

```
$1           // 十进制
$0xf4f8fcff  // 十六进制
$1.5         // 浮点数
$'a'         // 字符
$"abcd"
```

#### DATA指令
DATA命令用于初始化包变量，DATA命令的语法如下：
```
DATA symbol+offset(SB)/width, value
```
其中symbol为变量在汇编语言中对应的标识符，offset是符号开始地址的偏移量，width是要初始化内存的宽度大小，value是要初始化的值。其中当前包中Go语言定义的符号symbol，在汇编代码中对应·symbol，其中·中点符号为一个特殊的unicode符号；DATA命令示例如下

```
DATA ·Id+0(SB)/1,$0x37
DATA ·Id+1(SB)/1,$0x25
```

这两条指令的含义是将全局变量Id赋值为16进制数0x2537，也就是十进制的9527；
我们也可以合并成一条指令

#### GLOBL

用于将符号导出，例如将全局变量导出（所谓导出就是把汇编中的全局变量导出到go代码中声明的相同变量上，否则go代码中声明的变量感知不到汇编中变量的值的变化），其语法如下：
```
GLOBL symbol(SB), width
```
其中symbol对应汇编中符号的名字，width为符号对应内存的大小；GLOBL命令示例如下：
GLOBL ·Id, $8这条指令的含义是导出一个全局变量Id，其大小是8字节（byte）；
结合DATA和GLOBL指令，我们就可以初始化并导出一个全局变量.例如：
```
GLOBL ·Id, $8
DATA ·Id+0(SB)/8,$0x12345
```

## 2. 基本数据类型
建立如下文件结构：
```
├── main.go
├── go.mod
└── pkg
    ├── pkg_amd64.s
    └── pkg.go
```

#### int
文件内容如下:

```
// pkg/pkg_amd64.s

#include "textflag.h" // 使用NOPTR标志时必须导入此文件，此文件位置在：$GOROOT/src/runtime/textflag.h
// var MyInt int = 1234
GLOBL ·MyInt(SB),NOPTR,$8 // 导出MyInt,NOPTR表明不包含指针
DATA ·MyInt+0(SB)/8,$1234 // 定义了一个8字节的数据，值为1234
```

```go
// pkg/pkg.go

package pkg

var MyInt int
```

```go
// main.go

package main

import (
    "simple-go/pkg"
)


func main() {
    println(MyInt)
}
```

运行main.go可得到MyInt的值为1234

#### float

```
// pkg/pkg_amd64.s

#include "textflag.h"
GLOBL ·MyFloat64(SB),NOPTR,$8
DATA ·MyFloat64+0(SB)/8,$0.01
```

```go
// pkg.go

var MyFloat64 float64
```

#### string

首先需要了解一下字符串的内部标识
```go
// $GOROOT/src/runtime/string.go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```
因此，我们需要首先定义字符数据，再将字符串的str指向这个字符数据

```
// pkg/pkg_amd64.s

GLOBL NameData<>(SB),NOPTR,$8
DATA  NameData<>(SB)/8,$"abc" // 定义一个非导出的数据NameData<>(SB),size为8字节

GLOBL ·MyStr0(SB),NOPTR,$16
DATA  ·MyStr0+0(SB)/8,$NameData<>(SB) // MyStr的前8个字节内容为NameData<>(SB)
DATA  ·MyStr0+8(SB)/8,$3 // 后8个字节为3,代表字符串长度
```

```go
// pkg/pkg.go

var MyString string
```

#### bool

```
// pkg/pkg_amd64.s

GLOBL ·MyBool(SB),NOPTR,$1
DATA ·MyBool+0(SB)/1,$1
```

```go
// pkg/pkg.go

var MyBool bool
```

#### *int

```
// 首先定义一个int变量
GLOBL IntData<>(SB),NOPTR,$8
DATA IntData<>(SB)/8,$9876

// 将指针指向int变量
GLOBL ·MyIntPtr(SB),NOPTR,$8
DATA ·MyIntPtr+0(SB)/8,$IntData<>(SB)
```

```go
var MyIntPtr *int
```

## 3. 复合数据类型

#### 数组
```
// pkg/pkg_amd64.s

// array [2]int = {12, 34}
GLOBL ·MyArray(SB),NOPTR,$16
DATA  ·MyArray+0(SB)/8,$12
DATA ·MyArray+8(SB)/8,$34
```

```go
// pkg/pkg.go

var MyArray [2]int
```

#### 切片
首先需要了解切片的内部结构
```go
// $GOROOT/src/runtime/slice.go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

切片是截取数组的一部分得来的，因此要先定义一个数组，然后将切片的`array`指针指向这个数组的某个偏移

```
// pkg/pkg_amd64.s

// 定义三个string临时变量，作为切片元素
GLOBL str0<>(SB),NOPTR,$40
DATA  str0<>(SB)/40,$"Thoughts in the Still of the Night"

GLOBL str1<>(SB),NOPTR,$40
DATA  str1<>(SB)/40,$"A pool of moonlight before the bed"

GLOBL str2<>(SB),NOPTR,$8
DATA  str2<>(SB)/8,$"libai"

// 定义一个[3]string的数组，元素就是上面的三个string变量
GLOBL strarray<>(SB),NOPTR,$48
DATA  strarray<>+0(SB)/8,$str0<>(SB)
DATA  strarray<>+8(SB)/8,$34
DATA  strarray<>+16(SB)/8,$str1<>(SB)
DATA  strarray<>+24(SB)/8,$34
DATA  strarray<>+32(SB)/8,$str2<>(SB)
DATA  strarray<>+40(SB)/8,$5

// var MySlice []string
GLOBL ·MySlice(SB),NOPTR,$24
// 上面[3]string数组的首地址用来初始化切片的Data字段
DATA  ·MySlice+0(SB)/8,$strarray<>(SB)
DATA  ·MySlice+8(SB)/8,$3
DATA  ·MySlice+16(SB)/8,$4
```

上面的切片是截取了全部的数组元素，如果想要从第二个开始截取，可增加偏移：
```
DATA  ·MySlice+0(SB)/8,$strarray<>+16(SB)
```

#### map/chan

map/channel等类型并没有公开的内部结构，它们只是一种未知类型的指针，无法直接初始化。在汇编代码中我们只能为类似变量定义并进行0值初始化：

```go
var m map[string]int

var ch chan int
```

```
GLOBL ·m(SB),$8  // var m map[string]int
DATA  ·m+0(SB)/8,$0

GLOBL ·ch(SB),$8 // var ch chan int
DATA  ·ch+0(SB)/8,$0
```

## References

[Golang学习笔记-汇编](https://segmentfault.com/a/1190000041159550)

[map的实现原理](https://golang.design/go-questions/map/principal/)

[chan](https://golang.design/go-questions/channel/csp/)

[A Quick Guide to Go's Assembler](https://go.dev/doc/asm)

[Go语言高级编程](https://chai2010.cn/advanced-go-programming-book)