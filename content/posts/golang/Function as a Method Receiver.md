---
author: "李昌"
title: "Function as a Method Receiver"
date: "2023-11-26"
tags: ["golang"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---
通常来说，我们会使用某个具体的对象（`struct`）来作为`receiver`实现接口，但有时候，使用函数作为`receiver`可以起到不一样的效果。

使用函数作为`receiver`一个最常见的例子是`HandleFunc`:
```bash
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
这里将`HandleFunc`作为接收器，实现了`ServeHTTP`方法，同时也实现了`http.Handler`接口
```bash
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
代码注释中对与`HandleFunc`的解释如下：
```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
```
`HandleFunc`的定位是：**适配器。**<br />为什么这么说？<br />有了这个适配器，我们就可以这样完成一个http server：
```go
func main() {
    http.HandleFunc("/echo", func(w http.ResponseWriter, r *http.Request) {
        // do something
    })

    http.ListenAndServe(":8888", nil)
}
```
这里我们简单的将一个函数作为实参传入了`http.HandleFunc`，`http.HandleFunc`底层调用了`mux.Handle`进行注册，`mux.Handle`函数签名如下：
```go
func (mux *ServeMux) Handle(pattern string, handler Handler)
```
这里的`Handler`，就是上面提到的`http.Handler`接口。<br />中间发生了什么，为什么一个函数突然变成了一个接口对象，从一个被定义的行为，变成了一个执行某种动作的对象。<br />魔法发生在`mux.HandleFunc`：
```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler)) // 这里进行了类型的转化
}
```
经过一个类型转换，一个不具名函数类型，变成了一个具名函数类型`HandlerFunc`。然后在代码的调用链中，最终又调用了这个具名函数类型的`ServerHTTP`方法，这个`ServerHTTP`方法内部又是调用的这个函数本身。
```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	// ...

	handler.ServeHTTP(rw, req) // 这里的handle就是我们传入的不具名函数对象转化而来的具名函数对象
}
```
那么，如果没有`HandleFunc`适配器，会发生什么？
```go
func (mux *ServeMux) Handle(pattern string, handler Handler)
```
这个函数的第二个形参是一个接口，那么我们需要定义一个接口的具体实现，接口的具体实现需要一个`receiver`，我们需要定义一个receiver，，一般来说，我们可能会定义一个空struct，并让他来实现`ServeHTTP(ResponseWriter, *Request)`方法（成为`Handler`接口对象）
```go
type HandlerReceiver struct {
}

func (h HandlerReceiver) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// ...
}
```
这个我们实现一个接口handler的过程，如果有多个handler呢，一个应用程序，上百个接口总是有的，难道我们需要定义上百个甚至更多空的、无意义的struct来完成这个适配？

Function As Method Receiver还能做什么？<br />TODO
