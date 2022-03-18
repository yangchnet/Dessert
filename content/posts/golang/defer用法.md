---
author: "李昌"
title: "defer用法"
date: "2021-2-25"
tags: ["GoLang"]
categories: ["GoLang"]
ShowToc: true
TocOpen: true
---

> Revise in 2022-3-18

# defer用法

defer用来延迟对某个语句的调用，常用于处理成对的操作，如打开、关闭、连接、断开连接，加锁、释放锁。通过defer语句，无论函数逻辑多复杂，都能保证在任何代码执行路径下，资源被释放。defer应该直接跟在请求资源的语句后。

defer语句将函数的调用push到一个列表中，当外层函数返回时，会执行保存的函数列表

举个例子，这个程序打开两个文件并将一个文件的内容复制到另一个文件的函数
```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}
```

这个函数似乎可以正常工作，但其实存在一个bug，如果对```os.Create```的调用失败，该函数将返回但却不关闭源文件，通过在第二个return语句中调用```src.Close```可以解决这个问题。但是如果函数更加复杂，问题可能不会那么容易被发现和解决。通过使用defer语句，可以确保始终关闭文件。
```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```
Defer语句使我们可以考虑在打开每个文件后立即将其关闭，从而确保无论函数中有return语句多少，文件都将被关闭。


## defer 的特性
### 1. defer函数的参数是在defer函数声明时的参数

被defer调用的函数是在其调用被推入调用堆栈之前被估值的。
```go
func main() {
    var f = func() {
        fmt.Println(false)
    }
    defer f()
    f = func() {
        fmt.Println(true)
    }
}
```

```
    false
```

一个延迟调用的实参是在此调用被推入延迟调用堆栈之前估值的。
```go
import "fmt"
func a() {
    i := 0
    defer fmt.Println(i) // 这里的i，值为0,那么defer调用时，i也为0
    i++
    return
}
a()
```

```
    0
```

如果传入函数的实参为指针，那么对指针指向的值进行的更改会被defer显示出来。例如：
```go
type User struct {
        Name   string
        Passwd string
}

func main() {
        u := &User{
                Name: "tom",
        }
        f(u)

        u.Name = "Alice"
}

func f(u *User) {
        fmt.Println(u.Name)
}
```

```
    tom
```

### 2. defer函数的执行顺序与声明顺序相反，类似于栈


```go
func b() {
    for i := 0; i < 4; i++ {
        defer fmt.Println(i)
    }
}
b()
```

```
    3
    2
    1
    0
```

### 3. defer函数可以读取函数的返回值

```go
func c() (i int) {
    defer func() { i++ }()
    return 1
}
c()
```

```
    2
```

defer语句中的函数会在return语句更新返回值变量后再执行，又因为在函数中定义的匿名函数可以访问该函数包括返回值变量在内的所有变量，所以，对匿名函数采用defer机制，可以使其访问函数的返回值。

被延迟执行的匿名函数甚至可以修改函数返回给调用者的返回值


```go
func triple(x int) ( result int){
    defer func() {result += x}()
    return x*2
}
fmt.Println(triple(4))
```

```
    12
```

### 4. defer调用的属主实参

方法的属主实参是在被调用时被推入延迟调用堆栈的
```go
type T int

func (t T) M(n int) T {
    print(t)
    return t
}

func main() {
    var t T
    defer t.M(1).M(2)
    t.M(3).M(4)
}
```

```
    1342
```


### 5. 循环体中的defer语句

在循环语句中的defer语句要特别注意，因为只有在函数执行完毕后，这些被延迟的函数才会执行。
下面的代码将会导致系统的文件描述符耗尽，因为在所有文件都被处理之前， 没有文件会被关闭。
```go
fo _,filename := range filenames{
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.close()
    descriptors
    //...
}
```

一种解决办法是将循环体中的defer语句移至另外一个函数，在每次循环时，调用这个函数。
```go
for _, filename := {
    if err := doFile(filename); err != nil {
        return err
    }
}
func doFile(filename string) error{
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    //...process
}
```
