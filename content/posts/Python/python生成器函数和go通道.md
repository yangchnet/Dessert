---
author: "李昌"
title: "python生成器函数和go通道"
date: "2022-10-28"
tags: ["python", "go"]
categories: ["Python"]
ShowToc: true
TocOpen: true
---

## python生成器函数

```python
def gen_char():
    """
    定义了一个生成器函数，将会生成a~z
    """
    for i in range(0, 26):
        yield chr(ord('a') +i)

for char in gen_char():
    print(char, end=' ')

# 输出
# a b c d e f g h i j k l m n o p q r s t u v w x y z
```

## golang通道

```go
func main() {

	ch := make(chan string) // 一个通道
	wg := &sync.WaitGroup{} // 用于goroutine同步

	wg.Add(1)
	go func() { // 定义了一个“生成器函数“
		defer wg.Done()
		var r rune = 'a'
		for i := 0; i < 26; i++ {
			ch <- string(r + rune(i)) // 功能类似于yield chr(ord('a') + i) 这一句
		}
		close(ch)
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		for { // 遍历“生成器函数”
			select {
			case r, ok := <-ch:
				if !ok {
					return
				}
				fmt.Printf("%v ", r)
			}
		}
	}()

	wg.Wait()

}
```

## 二者的相似性

在阅读《流畅的Python》第16.2节时，有这样一段代码：
```python
def simple_coroutine():
    print('-> coroutine started')
    x = yield
    print('-> coroutine received:', x)

my_coro = simple_coroutine()
next(my_coro)
# -> coroutine started

my_coro.send(42)
# -> coroutine received: 42
# Traceback (most recent call last):
# ...
# StopIteration
```

这一段代码展示了yield的一种用法，当一个生成器函数内部存在`x = yield`, 并对其使用`.send()`方法时，`.send()`发送的内容就会传送到生成器函数内部。

而相应的，`yield`的另一种用法是，将函数内部的值不断“发送出去”，例如我们上文的python程序。

如此看来，`yield`实际上提供了一种通道，一种联通函数内部和外部的通道，通过`yield`关键字，我们可以从函数内部获得值，也可以在函数外部像函数内部发送值（如果内部有接收）。

上面的`simple_coroutine`函数片段用go可以这样写：
```go
func main() {
	wg := &sync.WaitGroup{}
	var ch chan int = nil
	sig := make(chan struct{})

	wg.Add(1)
	simple_coroutine := func() {
		defer wg.Done()
		ch = make(chan int, 1)
		fmt.Println("-> coroutine started")
		sig <- struct{}{} // 用于同步
		i := <-ch         // 对应x = yield
		fmt.Printf("-> coroutine received: %d", i)
	}

	go simple_coroutine() // 对应next(my_coro)

	<-sig

	ch <- 42

	wg.Wait()
}

// -> coroutine started
// -> coroutine received: 42
```

总结一下

当在python函数中使用`yield x` 时，这个python函数就成为一个生成器函数，此时，这个生成器函数的行为类似于一个通道，它会不断的往外发送值(通道写入)，此时这个python函数生成器的行为类似于golang的通道。而通过`yield`，我们也可以往函数内部发送值。

而根据生成器函数内部是惰性生成值还是非惰性生成值，又可细化的对应于golang中的无缓冲通道和有缓冲通道。

