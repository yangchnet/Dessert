---
author: "李昌"
title: "runtime篇一：接口"
date: "2022-08-04"
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

## 1. 接口的内部结构
```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```

一个接口是一个`iface`结构体，其中包含一个`itab`指针和一个`unsafe.Pointer`。

概念上讲一个接口的值,接口值,由两个部分组成,一个具体的类型和那个类型的值。它们被称为接口的动态类型和动态值。

一个`itab`可以表示一个接口的类型和赋给这个接口的实体类型，即为接口的动态类型，而`data`所指向的`unsafe.Pointer`则指向接口的动态值。

近距离来看`itab`：

```go
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

其中`inter`字段描述了接口的类型，`_type`字段描述了实体类型，`hash`字段是类型哈希，用于类型匹配，`fun`字段放置和接口方法对应的具体数据类型的方法地址。

再来看`interfacetype`
```go
type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}
```
其中`typ`和`itab`中的`_type`为同一个值，`pkgpath`则存储了接口的包名，`mhdr`则表示接口所定义的函数列表。

这样来看，一个接口主要有两个部分构成：第一是对于接口本身的描述，包括接口的包名`iface.itab.inter.pkgpath`、接口的函数列表`iface.itab.inter.mhdr`，接口的hash值`iface.itab.hash`。第二部分是对于实现接口的实体的描述，包括实体的类型`iface.itab._type`，实体的值`iface.data`。

可以将itab的值输出看看：
```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter  *interfacetype
	_type  uintptr
	hash   uint32
	_      [4]byte
	myfunc [1]uintptr
}

type interfacetype struct {
	mytype  uintptr
	pkgpath string
	mhdr    []uintptr
}

type Person interface {
	Walk()
	Say(words []string) string
}

type Student struct {
	name string
	age  int
}

func (s Student) Walk() {
	return
}

func (s Student) Say(words []string) string {
	return strings.Join(words, " ")
}

func main() {
	var p = Person(Student{
		name: "lichang",
		age:  18,
	})

	// 查看iface的结构
	p_iface := *(*iface)(unsafe.Pointer(&p))
	fmt.Println(p_iface.tab)

	// 查看接口的动态值
	s := (*Student)(unsafe.Pointer(p_iface.data))
	fmt.Println(*s)
}
```

输出：
```
&{0x489a80 4773888 3558907866 [0 0 0 0] [4234176]}
{lichang 18}
```

而对于一个空接口来说，其并不是一个`iface`，而是一个`eface`:
```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

一个空接口，其没有函数列表，用于描述空接口的结构体只有两个字段，一是`_type`，表示实体类型，二是`data`，指向动态值。

## 2. 接口的类型转换

### 2.1 结构体到接口类型的转换


对于如下代码：
```go
type Person interface {
	Walk()
	Say(words []string) string
}

type Student struct {
	name string
	age  int
}

func (s Student) Walk() {
	return
}

func (s Student) Say(words []string) string {
	return strings.Join(words, " ")
}

func main() {
	var p = Person(Student{
		name: "lichang",
		age:  18,
	})

	fmt.Println(p)
}
```

通过查看汇编代码可知，其在进行`Student`->`Person`的转换时，调用了`runtime.convT`，来看一下这个函数：
```go
func convT(t *_type, v unsafe.Pointer) unsafe.Pointer {
	// ... 部分条件检查

	x := mallocgc(t.size, t, true)
	typedmemmove(t, x, v)
	return x
}

func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer

func typedmemmove(typ *_type, dst, src unsafe.Pointer)
```

`convT`函数接收一个`*_type`：t, 一个`unsafe.Pointer`：v，将类型为t的v指向的值转换为一个可以作为iface结构体第二个字段的值。

真正工作的只有两行，第一行分配一个类型为t的新的内存空间，并为其赋零值，第二行将v指向的值复制到x，最后返回。

对于如下代码：
```go
var _ Person = (*Student)(nil)
```
可以检查Student是否实现了Person接口，其底层同样是调用的`runtime.convT()`，如果Student未实现Person接口，则在typedmemmove会发生panic

主要汇编代码如下：
```s
0x0027 00039 (main.go:28)       LEAQ    go.string."lichang"(SB), CX		# 将字符串从内存装载到CX中
0x002e 00046 (main.go:28)       MOVQ    CX, ""..autotmp_6+56(SP)		# CX中的值被mov到""..autotmp_6+56(SP)
0x0033 00051 (main.go:28)       MOVQ    $7, ""..autotmp_6+64(SP)		# 字面量7被mov到""..autotmp_6+64(SP)，这里可能是要做内存对齐
0x003c 00060 (main.go:29)       MOVQ    $18, ""..autotmp_6+72(SP)		# 字面量18被mov到""..autotmp_6+72(SP)
0x0045 00069 (main.go:27)       LEAQ    type."".Student(SB), AX			# 组装好的Student被装载到AX
0x004c 00076 (main.go:27)       LEAQ    ""..autotmp_6+56(SP), BX		# ""..autotmp_6+56(SP)被装载到BX
0x0051 00081 (main.go:27)       PCDATA  $1, $0
0x0051 00081 (main.go:27)       CALL    runtime.convT(SB)				# 调用runtime.convT构造itab
0x0056 00086 (main.go:32)       MOVUPS  X15, ""..autotmp_11+40(SP)
0x005c 00092 (main.go:32)       MOVQ    go.itab."".Student,"".Person+8(SB), CX # 构造好的itab被装载到CX
0x0063 00099 (main.go:32)       MOVQ    CX, ""..autotmp_11+40(SP)		# CX中的值，即itab被mov到""..autotmp_11+40(SP)
0x0068 00104 (main.go:32)       MOVQ    AX, ""..autotmp_11+48(SP)		# AX中的值，即Student字面量被装载到""..autotmp_11+48(SP)
```

### 2.2 接口类型之间的转换

```go
type Person interface {
	Walk()
	Say(words []string) string
}

type Walker interface {
	Walk()
}

type Student struct {
	name string
	age  int
}

func (s Student) Walk() {
	return
}

func (s Student) Say(words []string) string {
	return strings.Join(words, " ")
}

func main() {
	var p = Person(Student{
		name: "lichang",
		age:  18,
	})

	var w Walker = p

	fmt.Println(w)
}
```

编译后查看汇编代码可知，在进行`Person`接口到`Student`接口的转换时调用了`runtime.convI2I`:
```go
func convI2I(dst *interfacetype, src *itab) *itab {
	if src == nil {
		return nil
	}
	if src.inter == dst {
		return src
	}
	return getitab(dst, src._type, false)
}
```

`runtime.convI2I`函数将src itab中的inter转换到dst类型，并返回一个新的itab，首先检查src不为空，然后判断src的inter是否与dst相等，最后调用了`runtime.getitab`

来看`runtime.getitab`
```go
func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
	// ...

	var m *itab

    // 首先会从已经存在的表中查找，如果找到了可以直接结束，否则进行下一步。
    // 这里使用原子操作保证在这之前对itabTable的写操作结束。
	t := (*itabTableType)(atomic.Loadp(unsafe.Pointer(&itabTable)))
	if m = t.find(inter, typ); m != nil {
		goto finish
	}

	// 如果没找到，加锁继续找
	lock(&itabLock)
	if m = itabTable.find(inter, typ); m != nil {
		unlock(&itabLock)
		goto finish
	}

    // 还没找到，搞个新的
	m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*goarch.PtrSize, 0, &memstats.other_sys))
	m.inter = inter
	m._type = typ
	m.hash = 0
	m.init()
	itabAdd(m)
	unlock(&itabLock)
finish:
	if m.fun[0] != 0 {
		return m
	}
	if canfail {
		return nil
	}

	panic(&TypeAssertionError{concrete: typ, asserted: &inter.typ, missingMethod: m.init()})
}
```

这里一开始在构造`itab`时并没有直接构造，而是去一个`runtime.itabTableType`结构体中去查找这个`itab`是否存在，`runtime.itabTableType`的定义如下：
```go
type itabTableType struct {
	size    uintptr             // length of entries array. Always a power of 2.
	count   uintptr             // current number of filled entries.
	entries [itabInitSize]*itab // really [size] large
}
```

用于查找`itab`的函数如下：
```go
func (t *itabTableType) find(inter *interfacetype, typ *_type) *itab {
	// Implemented using quadratic probing.
	// Probe sequence is h(i) = h0 + i*(i+1)/2 mod 2^k.
	// We're guaranteed to hit all table entries using this probe sequence.
	mask := t.size - 1
	h := itabHashFunc(inter, typ) & mask
	for i := uintptr(1); ; i++ {
		p := (**itab)(add(unsafe.Pointer(&t.entries), h*goarch.PtrSize))
		// Use atomic read here so if we see m != nil, we also see
		// the initializations of the fields of m.
		// m := *p
		m := (*itab)(atomic.Loadp(unsafe.Pointer(p)))
		if m == nil {
			return nil
		}
		if m.inter == inter && m._type == typ {
			return m
		}
		h += i
		h &= mask
	}
}
```

在完成这些步骤之后，一个新的`itab`就构造完成了，而由于在进行转换时实现接口的实体并没有变化，只是接口类型发生了变化，因此我们只需要将`iface.itab`重新赋值为我们需要的`itab`即可。

### 2.3 空接口的转换
```go
func main() {
        var x int = 1

        var ix interface{} = x

        fmt.Println(ix)
}
```

这里调用的是`runtime.convT64(SB)`函数：
```go
func convT64(val uint64) (x unsafe.Pointer) {
	if val < uint64(len(staticuint64s)) {
		x = unsafe.Pointer(&staticuint64s[val])
	} else {
		x = mallocgc(8, uint64Type, false)
		*(*uint64)(x) = val
	}
	return
}
```

这里首先检查其值是否小于`len(staticuint64s)`，如果是的话，就不需要再去进行内存分配，而是直接到数组中取，算是进行了一步优化。否则调用`mallocgc`为其分配一个新的内存空间，然后将其底层值赋值为val。

这里如果是将一个字符串转换为空接口类型，则调用的是`runtime.convTstring`:
```go
func convTstring(val string) (x unsafe.Pointer) {
	if val == "" {
		x = unsafe.Pointer(&zeroVal[0])
	} else {
		x = mallocgc(unsafe.Sizeof(val), stringType, true)
		*(*string)(x) = val
	}
	return
}
```
逻辑相差不大。

在runtime包中，对于某些特殊的类型做了优化，可直接调用相应的函数进行实体类型到空接口类型的转化，这些被调用的函数有：
```go
func convT16(val uint16) unsafe.Pointer
func convT32(val uint32) unsafe.Pointer
func convT64(val uint64) unsafe.Pointer
func convTstring(val string) unsafe.Pointer
func convTslice(val []uint8) unsafe.Pointer
```

## 3. 类型断言

### 3.1 空接口断言

对如下golang代码进行编译：
```go
type User struct {
	name string
}

func main() {

	var x interface{} = &User{name: "lichang"}

	i, ok := x.(int)

	if !ok {
		return
	} else {
		fmt.Println(i)
	}
}
```

使用如下命令编译
```sh
go tool compile -S -N -l  main.go # -N禁止优化，-l禁止内联
```

```s
# 构造接口
0x0026 00038 (main.go:11)       MOVUPS  X15, ""..autotmp_8+96(SP)
0x002c 00044 (main.go:11)       LEAQ    ""..autotmp_8+96(SP), CX
0x0031 00049 (main.go:11)       MOVQ    CX, ""..autotmp_7+48(SP)
0x0036 00054 (main.go:11)       TESTB   AL, (CX)
0x0038 00056 (main.go:11)       MOVQ    $7, ""..autotmp_8+104(SP)
0x0041 00065 (main.go:11)       LEAQ    go.string."lichang"(SB), DX
0x0048 00072 (main.go:11)       MOVQ    DX, ""..autotmp_8+96(SP)
0x004d 00077 (main.go:11)       MOVQ    CX, ""..autotmp_3+56(SP)
0x0052 00082 (main.go:11)       LEAQ    type.*"".User(SB), DX
0x0059 00089 (main.go:11)       MOVQ    DX, "".x+80(SP) 	# "".x+80(SP)是eface._type
0x005e 00094 (main.go:11)       MOVQ    CX, "".x+88(SP)		# "".x+88(SP)是eface.data

# 接口断言
0x0063 00099 (main.go:13)       MOVQ    "".x+80(SP), CX		# 将eface._type移动到CX
0x0068 00104 (main.go:13)       MOVQ    "".x+88(SP), DX		# 将eface._data移动到DX
0x006d 00109 (main.go:13)       LEAQ    type.int(SB), BX	# 加载int的_type到BX
0x0074 00116 (main.go:13)       CMPQ    CX, BX				# 对比CX与BX，即eface._type与int._type
0x0077 00119 (main.go:13)       JEQ     123					# 如果_type对比成功，则跳转到123
0x0079 00121 (main.go:13)       JMP     133					# 对比不成功，跳转133
0x007b 00123 (main.go:13)       MOVQ    (DX), CX			# 将eface.data暂存到CX
0x007e 00126 (main.go:13)       MOVL    $1, AX
0x0083 00131 (main.go:13)       JMP     139
0x0085 00133 (main.go:13)       XORL    CX, CX				# 类型断言失败，清空寄存器
0x0087 00135 (main.go:13)       XORL    AX, AX				# 清空AX寄存器
0x0089 00137 (main.go:13)       JMP     139
0x008b 00139 (main.go:13)       MOVQ    CX, ""..autotmp_4+40(SP)
0x0090 00144 (main.go:13)       MOVB    AL, ""..autotmp_5+31(SP)
0x0094 00148 (main.go:13)       MOVQ    ""..autotmp_4+40(SP), CX
0x0099 00153 (main.go:13)       MOVQ    CX, "".i+32(SP)		# 断言结果保存在"".i+32(SP)
0x009e 00158 (main.go:13)       MOVBLZX ""..autotmp_5+31(SP), CX
0x00a3 00163 (main.go:13)       MOVB    CL, "".ok+30(SP)	# ok保存在"".ok+30(SP)
```

从以上汇编代码可以看出，空接口在进行类型断言时，会将`eface._type`取出与断言类型的`_type`进行对比，如果对比成功，则将其`eface.data`取出使用，否则断言失败。

### 3.2 非空接口断言

#### 3.2.1 断言为接口

对如下代码进行编译：
```go
type Person interface {
	Walk()
	Say(words []string) string
}

type Walker interface {
	Walk()
}

type Student struct {
	name string
	age  int
}

func (s Student) Walk() {
	return
}

func (s Student) Say(words []string) string {
	return strings.Join(words, " ")
}

func main() {
	var p = Person(Student{
		name: "lichang",
		age:  18,
	})

	w, ok := p.(Walker)

	if !ok {
		return
	} else {
		fmt.Println(w)
	}
}
```

```sh
go tool compile -S -N -l  main.go
```

可得部分汇编代码如下：
```s
# iface构建
0x0026 00038 (main.go:32)       MOVQ    $0, ""..autotmp_3+176(SP)
0x0032 00050 (main.go:31)       MOVUPS  X15, ""..autotmp_3+184(SP)
0x003b 00059 (main.go:32)       LEAQ    go.string."lichang"(SB), CX
0x0042 00066 (main.go:32)       MOVQ    CX, ""..autotmp_3+176(SP)
0x004a 00074 (main.go:32)       MOVQ    $7, ""..autotmp_3+184(SP)
0x0056 00086 (main.go:33)       MOVQ    $18, ""..autotmp_3+192(SP)
0x0062 00098 (main.go:31)       LEAQ    type."".Student(SB), AX
0x0069 00105 (main.go:31)       LEAQ    ""..autotmp_3+176(SP), BX
0x0071 00113 (main.go:31)       PCDATA  $1, $0
0x0071 00113 (main.go:31)       CALL    runtime.convT(SB)
0x0076 00118 (main.go:31)       MOVQ    AX, ""..autotmp_7+56(SP)
0x007b 00123 (main.go:31)       LEAQ    go.itab."".Student,"".Person(SB), CX
0x0082 00130 (main.go:31)       MOVQ    CX, "".p+88(SP)			# iface.itab在"".p+88(SP)处
0x0087 00135 (main.go:31)       MOVQ    AX, "".p+96(SP)			# iface.data在"".p+96(SP)处

# 类型断言
0x008c 00140 (main.go:36)       MOVUPS  X15, ""..autotmp_4+120(SP)
0x0092 00146 (main.go:36)       MOVQ    "".p+88(SP), BX			# iface.itab被转移到BX
0x0097 00151 (main.go:36)       MOVQ    "".p+96(SP), CX			# iface.data被转移到CX
0x009c 00156 (main.go:36)       LEAQ    type."".Walker(SB), AX	# Walker._type被加载到AX
0x00a3 00163 (main.go:36)       CALL    runtime.assertI2I2(SB)	# 调用runtime.assertI2I2
0x00a8 00168 (main.go:36)       MOVQ    AX, ""..autotmp_4+120(SP)	# runtime.assertI2I2的返回值存储在""..autotmp_4+120(SP)
0x00ad 00173 (main.go:36)       MOVQ    BX, ""..autotmp_4+128(SP)
0x00b5 00181 (main.go:36)       TESTQ   AX, AX
0x00b8 00184 (main.go:36)       SETNE   ""..autotmp_5+47(SP)
0x00bd 00189 (main.go:36)       MOVQ    ""..autotmp_4+128(SP), CX
0x00c5 00197 (main.go:36)       MOVQ    ""..autotmp_4+120(SP), DX
0x00ca 00202 (main.go:36)       MOVQ    DX, "".w+72(SP)			# 断言接口的itab在"".w+72(SP)
0x00cf 00207 (main.go:36)       MOVQ    CX, "".w+80(SP)			# 断言接口的data在"".w+80(SP)
0x00d4 00212 (main.go:36)       MOVBLZX ""..autotmp_5+47(SP), CX
0x00d9 00217 (main.go:36)       MOVB    CL, "".ok+46(SP)		# ok被存储在"".ok+46(SP)
```

`runtime.assertI2I2`函数如下：
```go
func assertI2I2(inter *interfacetype, i iface) (r iface) {
	tab := i.tab
	if tab == nil {
		return
	}
	if tab.inter != inter {
		tab = getitab(inter, tab._type, true)
		if tab == nil {
			return
		}
	}
	r.tab = tab
	r.data = i.data
	return
}
```

该函数接受一个一个`interfacetype`和一个`iface`，检查`iface.itab.inter`是否与传入的`interfacetype`相同，若相同，则直接将原来的`iface`复制一个新的返回，否则构建一个具有传入的`interfacetype`的新的`itab`，并赋值给新的`iface`;`iface.data`直接复制到新的`iface`。

#### 3.2.2 断言为实体对象

对如下代码进行编译：
```go

type Person interface {
	Walk()
	Say(words []string) string
}

type Student struct {
	name string
	age  int
}

func (s Student) Walk() {
	return
}

func (s Student) Say(words []string) string {
	return strings.Join(words, " ")
}

func main() {
	var p = Person(Student{
		name: "lichang",
		age:  18,
	})

	w, ok := p.(Student)

	if !ok {
		return
	} else {
		fmt.Println(w)
	}
}
```

```sh
go tool compile -S -N -l  main.go
```

有关类型断言的汇编代码如下：
```s
# 接口构造
0x0035 00053 (main.go:27)       MOVUPS  X15, ""..autotmp_8+224(SP)
0x003e 00062 (main.go:28)       LEAQ    go.string."lichang"(SB), CX
0x0045 00069 (main.go:28)       MOVQ    CX, ""..autotmp_8+216(SP)
0x004d 00077 (main.go:28)       MOVQ    $7, ""..autotmp_8+224(SP)
0x0059 00089 (main.go:29)       MOVQ    $18, ""..autotmp_8+232(SP)
0x0065 00101 (main.go:27)       MOVQ    CX, ""..autotmp_12+240(SP)
0x006d 00109 (main.go:27)       MOVQ    $7, ""..autotmp_12+248(SP)
0x0079 00121 (main.go:27)       MOVQ    $18, ""..autotmp_12+256(SP)
0x0085 00133 (main.go:27)       LEAQ    go.itab."".Student,"".Person(SB), CX
0x008c 00140 (main.go:27)       MOVQ    CX, "".p+88(SP)						# iface.itab
0x0091 00145 (main.go:27)       LEAQ    ""..autotmp_12+240(SP), CX
0x0099 00153 (main.go:27)       MOVQ    CX, "".p+96(SP)						# iface.data

# 类型断言
0x009e 00158 (main.go:32)       MOVQ    $0, ""..autotmp_8+216(SP)
0x00aa 00170 (main.go:32)       MOVUPS  X15, ""..autotmp_8+224(SP)
0x00b3 00179 (main.go:32)       MOVQ    "".p+96(SP), CX						# iface.data被装载到CX
0x00b8 00184 (main.go:32)       MOVQ    "".p+88(SP), DX						# iface.itab被装载到DX
0x00bd 00189 (main.go:32)       LEAQ    go.itab."".Student,"".Person(SB), SI # 重新构造一个itab，存储在SI
0x00c4 00196 (main.go:32)       CMPQ    DX, SI 								# 对比新构造的itab和老的itab
0x00c7 00199 (main.go:32)       JEQ     203
0x00c9 00201 (main.go:32)       JMP     221
0x00cb 00203 (main.go:32)       MOVQ    (CX), DX							# CX(iface.data)指向的值0-7个字节mov到DX
0x00ce 00206 (main.go:32)       MOVQ    8(CX), SI							# 第8-15个字节mov到SI
0x00d2 00210 (main.go:32)       MOVQ    16(CX), CX							# 第16-24个字节mov到CX
0x00d6 00214 (main.go:32)       MOVL    $1, AX
0x00db 00219 (main.go:32)       JMP     231
0x00dd 00221 (main.go:32)       XORL    SI, SI								# 断言失败执行
0x00df 00223 (main.go:32)       XORL    CX, CX
0x00e1 00225 (main.go:32)       XORL    AX, AX
0x00e3 00227 (main.go:32)       XORL    DX, DX
0x00e5 00229 (main.go:32)       JMP     231
0x00e7 00231 (main.go:32)       MOVQ    DX, ""..autotmp_8+216(SP)
0x00ef 00239 (main.go:32)       MOVQ    SI, ""..autotmp_8+224(SP)
0x00f7 00247 (main.go:32)       MOVQ    CX, ""..autotmp_8+232(SP)
0x00ff 00255 (main.go:32)       MOVB    AL, ""..autotmp_9+47(SP)
0x0103 00259 (main.go:32)       MOVQ    ""..autotmp_8+216(SP), CX
0x010b 00267 (main.go:32)       MOVQ    ""..autotmp_8+224(SP), DX
0x0113 00275 (main.go:32)       MOVQ    ""..autotmp_8+232(SP), SI
0x011b 00283 (main.go:32)       MOVQ    CX, "".s+168(SP)					# 构造Student结构体
0x0123 00291 (main.go:32)       MOVQ    DX, "".s+176(SP)
0x012b 00299 (main.go:32)       MOVQ    SI, "".s+184(SP)
0x0133 00307 (main.go:32)       MOVBLZX ""..autotmp_9+47(SP), CX
0x0138 00312 (main.go:32)       MOVB    CL, "".ok+46(SP)					# ok存储在"".ok+46(SP)
```

在进行接口到实体对象的断言时，编译器会尝试构建一个断言对象对应的itab，然后和原接口的itab进行对比，如果相同，则断言可成功，否则失败。

断言成功后，将`iface.data`解析，重新组装成断言对象。断言失败则清空寄存器，返回一个断言对象的零值

## References

[Go Questions](https://golang.design/go-questions/interface/assert/)

[interface的类型断言是如何实现](https://segmentfault.com/a/1190000039894161)
