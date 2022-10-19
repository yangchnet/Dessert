---
author: "李昌"
title: "python高阶函数"
date: "2022-10-19"
tags: ["python"]
categories: [""]
ShowToc: true
TocOpen: true
---

> 如果一个函数能接收一个函数为参数或返回一个函数，那这个函数就被称为高阶函数。


### 1. sorted
sorted可以说是最常用的高阶函数，其用法如下：
```python
m = {"three": 3, "tow": 2, "one": 1

sorted(m.items(), key=lambda x: x[1]) # [('one', 1), ('tow', 2), ('three', 3)]
```

sorted可根据其参数key返回的值来进行排序。

### 2. map

map()函数接收两个参数，一个是函数，一个是Iterable，map将传入的函数依次作用到序列的每个元素，并把结果作为新的Iterator返回。

如：
```python
def add10(x):
    return x+10

print(map(add10, [1, 2, 3])) # <map object at 0x7f5824cd01c0>

list(map(add10, [1, 2, 3])) # [11, 12, 13]
```

```python
list(map(str, [1, 2, 3, 4, 5, 6, 7, 8, 9])) # ['1', '2', '3', '4', '5', '6', '7', '8', '9']
```

### functools.reduce

reduce把一个函数作用在一个序列[x1, x2, x3, ...]上，这个函数必须接收两个参数，reduce把结果继续和序列的下一个元素做累积计算，其效果就是：

reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)
例如：

```python
from functools import reduce

from operator import mul

reduce(mul, range(1, 5)) # 24 (1*2*3*4)
```

### filter

filter() 函数用于过滤序列，过滤掉不符合条件的元素，返回由符合条件元素组成的新列表。

该接收两个参数，第一个为函数，第二个为序列，序列的每个元素作为参数传递给函数进行判断，然后返回 True 或 False，最后将返回 True 的元素放到新列表中。

用法：
```python
def even(x:int) -> bool:
    """判断是否是奇数
    """
    if x % 2 == 0:
        return False
    else:
        return True

list(filter(even, range(1, 10))) # [1, 3, 5, 7, 9]
```

### zip

zip() 函数用于将可迭代的对象作为参数，将对象中对应的元素打包成一个个元组，然后返回由这些元组组成的列表。

如果各个迭代器的元素个数不一致，则返回列表长度与最短的对象相同，利用 * 号操作符，可以将元组解压为列表。

用法：
```python
a = ["one", "two", "three"]

b = [1, 2, 3, 4]

list(zip(a, b)) # [('one', 1), ('two', 2), ('three', 3)]
```

### operator.attrgetter

返回一个可从操作数中获取 attr 的可调用对象。 如果请求了一个以上的属性，则返回一个属性元组。 属性名称还可包含点号。 例如：

在 f = attrgetter('name') 之后，调用 f(b) 将返回 b.name。

在 f = attrgetter('name', 'date') 之后，调用 f(b) 将返回 (b.name, b.date)。

在 f = attrgetter('name.first', 'name.last') 之后，调用 f(b) 将返回 (b.name.first, b.name.last)。

用法：
```python
from collections import namedtuple

Name = namedtuple('Name', 'firstname lastname')

Person = namedtuple('Person', 'name age height')

myname = Name(firstname='li', lastname='chang')
me = Person(name=myname, age=18, height=180)

print(me) # Person(name=Name(firstname='li', lastname='chang'), age=18, height=180)

nameFn = attrgetter("name")
nameFn(me) # Name(firstname='li', lastname='chang')

firstNameFn = attrgetter("name.firstname")
firstNameFn(me) # 'li'
```

### operator.itemgetter

返回一个使用操作数的 __getitem__() 方法从操作数中获取 item 的可调用对象。 如果指定了多个条目，则返回一个查找值的元组。

例如：

在 f = itemgetter(2) 之后，调用 f(r) 将返回 r[2]。

在 g = itemgetter(2, 5, 3) 之后，调用 g(r) 将返回 (r[2], r[5], r[3])。

用法：
```python
from operator import itemgetter

index1 = itemgetter(1)

index1(range(1, 10)) # 2

index1(["one", "two", "three"]) # 'two'
```

### functools.partial

称为偏函数，作用是把一个函数的某些参数给固定住

用法：
```python
from functools import partial

def powN(a, n=2):
    return a**n

powN(2) # 4

pow10 = partial(powN, n=10)
pow10(2) # 1024
```

当partial没有指定参数的名字时，会把参数加到被调用函数的参数列表左边，如：
```python
from functools import partial
maxOr10 = partial(max, 10)

maxOr10(11) # 11
maxOr10(1) # 1
```