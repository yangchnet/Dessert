---
author: "李昌"
title: "slice和数组的区别"
date: "2021-03-30"
tags: ["slice", "数组"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

## 1. 长度
> 数组  

对于数组来说，它的长度是固定的，并且**数组的长度是其类型的一部分**，即对于以下两个数组来说，他们是不同的类型。  
```go
var a [5]int
var b [6]int

fmt.Printf("%v", reflect.TypeOf(a) == reflect.TypeOf(b))

// 输出： false
```
数组的长度必须是常量表达式，因为数组的长度需要在编译阶段确定。  
对于数组来说，由于其长度是固定的，因此不能添加或删除元素。

> 切片

而对于切片，其长度是不固定的，不同长度的切片，只要其元素类型相同，则它们就是相同的切片类型。  
```go
a := make([]int, 5)
b := make([]int, 6)

fmt.Printf("%v\n", reflect.TypeOf(a) == reflect.TypeOf(b))
// 输出： true
```
如果切片操作超出cap(s)的上限将导致一个panic异常，但是超出len(s)则是意味着扩展了 slice，因为新slice的长度会变大：
```go
months := [...]string{1: "January", /* ... */, 12: "December"}
summer := months[6:9]

fmt.Println(summer[:20]) // panic: out of range
endlessSummer := summer[:5] // extend a slice (within capacity) 
fmt.Println(endlessSummer) // "[June July August September October]"
```

## 2. 比较
> 数组  

如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的，这时候我们可以直接通过==比较运算符来比较两个数组。只有当两个数组的所有元素都是相等的时候数组才是相等的。不想等比较运算符!=遵循同样的规则.
```go
a := [2]int{1, 2}
b := [...]int{1, 2}
c := [2]int{1, 3}

fmt.Println(a==b, a==c, b==c) // "true, false, false"
d := [3]int{1, 2}
fmt.Println(a==d) // compile error: cannot compare [2]int == [3]int
```

> 切片

切片之间不能比较，因此我们不能使用==操作符来判断两个slice是否含有全部相等元素。一般来说，要想判断两个slice是否相等，我们必须自己展开每个元素进行比较。

为什么slice不直接支持比较运算符呢？1. 一个slice的元素是间接引用的，一个slice甚至可以包含自身。虽然有很多办法可以处理这种情形，但没有一个是简单有效的。2. 因为slice是间接引用的，一个固定值的slice在不同的时间可能包含不同的元素，因为底层数组的元素可能会被修改.  
slice唯一合法的比较操作是和nil进行比较，例如：
```go
if summer == nil { /* ... */ }
```

## 3. 作为函数参数
> 数组  

当数组作为函数的参数传递时，实际上传递的是一个数组的复制。因此，任何在函数内部对于数组的修改都是在数组的复制上修改的，并不能直接修改调用时原始的数组变量。  
```go
//先定义一个数组
var a = [5]int{1, 2, 3, 4, 5}

//定义一个函数，将数组中的第一个值设为0
func change(a [5]int){
    a[0] = 0
    fmt.Println(a)
}
change(a)
fmt.Println(a)
/*
输出：
[0 2 3 4 5]
[1 2 3 4 5]
*/
// 只修改了复制的数组，原始数组没有得到修改。
```

可以显式的传入一个数组指针，这样的话函数通过指针对数组的任何修改都可以直接反馈到调用者。
```go
func main() {
	var a = [5]int{1, 2, 3, 4, 5}
	fmt.Println(a)
	change(&a)
	fmt.Println(a)
}

func change(ptr *[5]int) {
	ptr[0] = 0
	fmt.Println(*ptr)
}
/* 输出:
[1 2 3 4 5]
[0 2 3 4 5]
[0 2 3 4 5]
*/
```

> slice

slice是一个引用类型，slice值包含指向第一个slice元素的指针，因此向函数传递slice将允许在函数内部修改底 层数组的元素。换句话说，复制一个slice只是对底层的数组创建了一个新的slice别名。
```go
func reverse(s []int) { 
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 { 
        s[i], s[j] = s[j], s[i]
    } 
}

a := [...]int{0, 1, 2, 3, 4, 5} 
reverse(a[:]) 
fmt.Println(a) // "[5 4 3 2 1 0]"
```

这里存在一个坑，就是我们会误以为对于传入的切片作出的任何更改都会反映到原来的切片上，其实不然，请看下面程序：
```go
package main

import "fmt"

func myAppend(s []int) []int {
        // 这里 s 虽然改变了，但并不会影响外层函数的 s
        s = append(s, 100)
        return s
}

func myAppendPtr(s *[]int) {
        // 会改变外层 s 本身
        *s = append(*s, 100)
        return
}

func main() {
        s := []int{1, 1, 1}
        newS := myAppend(s)

        fmt.Println(s)
        fmt.Println(newS)

        s = newS

        myAppendPtr(&s)
        fmt.Println(s)
}
```

运行结果为：
```
[1 1 1]
[1 1 1 100]
[1 1 1 100 100]
```

很明显，对于切片作出的改变未反映到原切片上，这是为什么呢？  
仔细观察，可以看出，对于反映到原切片上的改动，都是直接访问了切片元素，而没有反映到原切片上的改动，则没有直接访问元素。这是因为对于切片元素的访问，实际上访问的是切片内部array指针，因此才能反映到原数组中。  
而对于切片的直接修改，没有访问array指针(可能是改变了array或len、cap)，因此无法反映到原切片上.  
但当我们传入切片指针时，作出的任何改变就都可以反映到原切片上了。
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

示意图如下，传入函数内部的slice是对外部slice的一个复制，因此二者具有相同的array、len、cap等。对于函数内部的slice.array进行访问修改，实际上就是在对外部slice的array进行修改。但如果是直接对array修改（不访问指针），那么无论作出任何修改，都仅仅是在外部slice的一个复制上作出的改动。
![slice.drawio](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/slice.drawio.png)