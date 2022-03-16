---
author: "李昌"
title: "理解反射之：一个reflect。Type可以做什么"
date: "2022-03-15"
tags: ["reflect"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

反射是一个接口，其定义如下：

```go
type Type interface {
	// 返回具体类型在内存分配时的字节分配方式
	Align() int

	// 返回具体类型在结构体中作为一个字段是内存对齐方式
	FieldAlign() int

	// 返回具体类型的第x个方法
	Method(int) Method

	// 根据函数名返回具体类型的方法
	MethodByName(string) (Method, bool)

	// 返回类型的方法个数
	NumMethod() int

	// 返回类型的名字
	Name() string

	// 返回类型的包名
	PkgPath() string

	// 返回类型所占内存字节大小
	Size() uintptr

	// 返回类型的简单描述，如：main.User
	String() string

	// 返回这个类型的Kind
	Kind() Kind

	// 检查类型是否实现了某个接口
	//stringer := reflect.TypeOf((*fmt.Stringer)(nil)).Elem()
	//fmt.Println(reflect.ValueOf(u).Type().Implements(stringer))
	Implements(u Type) bool

	// 检查类型是否可以被赋值给某个类型
	AssignableTo(u Type) bool

	// 检查类型是否可以转换到类型u
	ConvertibleTo(u Type) bool

	// 检查类型是否可比较
	Comparable() bool

	// 返回Int, Uint, Float, or Complex kinds.的字节大小
	Bits() int

	// 返回一个通道类型的方向
	ChanDir() ChanDir

	// 判断一个函数类型的最后一个参数是否是"..."参数
	IsVariadic() bool

	// 返回Array, Chan, Map, Ptr, or Slice类型的元素的类型
	Elem() Type

	// 返回结构体的第i个字段的详细信息
	Field(i int) StructField

	// 返回结构体内的嵌套结构体
	FieldByIndex(index []int) StructField

	// 根据名字返回结构体内的某个字段
	FieldByName(name string) (StructField, bool)

	// 根据match函数的返回值返回结构体内某个字段
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// 返回函数的第i个参数类型
	In(i int) Type

	// 返回map的键的类型
	Key() Type

	// 返回数组的长度
	Len() int

	// 返回结构体的字段个数
	NumField() int

	// 返回参数的入参个数
	NumIn() int

	// 返回函数的返回值个数
	NumOut() int

	// 返回函数第i个返回值的类型
	Out(i int) Type

	common() *rtype
	uncommon() *uncommonType
}
```

从接口定义中我们可以知道:

对于所有的类型：
1. 字节对齐方式
   - 具体类型在内存分配时的字节分配方式
   - 返回具体类型在结构体中作为一个字段是内存对齐方式
2. 方法
   - 具体类型的第x个方法
   - 根据函数名获取具体类型的方法
   - 检查类型是否实现了某个接口
3. 基本信息
   - 方法个数
   - 类型名
   - 类型包名
   - 类型所占字节大小
   - 类型的简单描述（如main.User）
4. 获取类型的Kind

关于类型转换：
1. 检查类型是否实现了某个接口
2. 检查类型是否可以被赋值给某个类型
3. 检查类型是否可以转换到类型u

对于通道:
1. 获取通道的方向
2. 获取通道内元素的类型

对于函数：
1. 检查其最后一个参数是否是`...Type`类型
2. 获取函数的入参、返回函数的返回值个数
3. 根据下标获取函数入参、返回值的类型

对于结构体：
1. 获取结构体的第i个字段的详细信息
2. 获取结构体内的嵌套结构体
3. 根据名字获取结构体内的某个字段
4. 根据match函数的返回值获取结构体内某个字段

对于map:
1. 获取map的键的类型

对于Array:
1. 获取Array的长度

其他:
1. 检查类型是否可比较