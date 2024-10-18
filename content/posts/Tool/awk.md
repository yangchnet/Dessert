---
author: "李昌"
title: "awk基础"
date: "2024-10-18"
tags: ["awk", "linux"]
categories: ["tools"]
ShowToc: true
TocOpen: true
---

> https://github.com/wuzhouhui/awk/blob/master/The_AWK_Programming_Language_zh_CN.pdf

# 0. 实用一行手册
1. 输入行的总行数
```END { print NR }```
2. 打印第 10 行
`NR == 10`
3. 打印每一个输入行的最后一个字段
```{ print $NF }```
4. 打印最后一行的最后一个字段
```
{ field = $NF }
END { print field }
```
5. 打印字段数多于 4 个的输入行
`NF > 4`
6. 打印最后一个字段值大于 4 的输入行
`$NF > 4`
7. 打印所有输入行的字段数的总和
```
{ nf = nf + NF }
END { print nf }
```
8. 打印包含 Beth 的行的数量
```
/Beth/ { nlines = nlines + 1 }
END { print nlines }
```
9. 打印具有最大值的第一个字段, 以及包含它的行 (假设 $1 总是 正的) 18
```
$1 > max { max = $1; maxline = $0 }
END { print max, maxline }
```
10. 打印至少包含一个字段的行
`NF > 0`
11. 打印长度超过 80 个字符的行
`length($0) > 80`
12. 在每一行的前面加上它的字段数
`{ print NF, $0 }`
13. 打印每一行的第 1 与第 2 个字段, 但顺序相反
`{ print $2, $1 }`
14. 交换每一行的第 1 与第 2 个字段, 并打印该行
`{ temp = $1; $1 = $2; $2 = temp; print }`
15. 将每一行的第一个字段用行号代替
`{ $1 = NR; print }`
16. 打印删除了第 2 个字段后的行
`{ $2 = ""; print }`
17. 将每一行的字段按逆序打印
```{ for (i = NF; i > 0; i = i - 1) printf("%s ", $i)
printf("\n")
}```
18. 打印每一行的所有字段值之和
```
{ sum = 0
    for (i = 1; i <= NF; i = i + 1) sum = sum + $i
    print sum
}
```
19. 将所有行的所有字段值累加起来
```
{ for (i = 1; i <= NF; i = i + 1) sum = sum + $i }
END { print sum }
```
20. 将每一行的每一个字段用它的绝对值替换
```
{ for (i = 1; i <= NF; i = i + 1) if ($i < 0) $i = -$i
    print
}
```




----

# 1. 基本概念
awk是一种编程语言，用于处理文本文件。它是一种解释型语言，通常用于文本处理和数据提取。

awk程序的基本结构为：
```
pattern { action }
```

其中，`pattern`是一个条件，`action`是一个或多个命令，当`pattern`匹配时，`action`会被执行。

例如，对于如下文件内容：

```
Beth 4.00 0
Dan 3.75 0
Kathy 4.00 10
Mark 5.00 20
Mary 5.50 22
Susie 4.25 18
```
如果要打印每位雇员的名字以及他们的报酬 (每小时工资乘以工作时长), 而雇员的工作时长必须大于零：
```bash
awk '$3 > 0 { print $1, $2 * $3 }' file.txt
```
在这里，`$3 > 0`是一个条件，`{ print $1, $2 * $3 }`是一个命令，当条件满足时，命令会被执行。

运行一个awk程序有多种方式，上面我们直接在命令行中输入了awk程序，也可以将awk程序写入一个文件中，然后通过`-f`选项来执行：
```
awk -f progfile file.txt file1.txt
```

文本也可以被放在输出中：
```awk
{ print "total pay for", $1, "is", $2 * $3 }
```

# 2. 内置变量
1. `NF`, 表示当前行的字段数，例如：`print NF, $1, $NF`会打印出当前行的字段数、第一个字段和最后一个字段。
2. `NR`, 这个变量计算到当前为止，读取到的行的数量，例如：`awk '{ print NR, $0 }' file.txt`会为文件的每一行加上行号


# 3. 格式化输出
awk提供了`printf`函数来格式化输出，其语法与C语言中的`printf`函数类似：
```awk
printf("format", var1, var2, ...)
```
其中`format`是类似于C风格的printf函数的参数，var1、var2等是要输出的值。使用 printf 不会自动产生空格符或换行符; 用户必须自己创建它们, 不要忘了 `\n`.

# 4. 使用模式进行选择
- 通过比较操作符进行选择，例如：`$3 > 0`，`$1 == "Beth"`等
- 通过计算操作符进行选择，例如：`$2 * $3 > 10`，`$1 ~ /Beth/`等
- 可以使用正则表达式，`~`匹配，`!~`不匹配
- 可以使用逻辑运算，`&&`、`||`、`!`
- 可以使用括弧，`(condition)`

# 5. BEGIN 与 END
特殊的模式 BEGIN 在第一个输入文件的第一行之前被匹配, END 在最后一个输入文件的最后一行被处理
之后匹配. 这个程序使用 BEGIN 打印一个标题
```
BEGIN { print "NAME RATE HOURS"; print "" }
{ print }
```

# 6. 计算

在awk里，不仅可以使用内建变量，还可以自定义变量，用户创建的变量不需要声明就可使用。

- 计数，下面的程序用一个变量emp计算工作时长超过15小时的工作人员
```
$3 > 15 { emp = emp + 1 }
END { print emp, "employees worked more than 15 hours" }
```

- 操作文本
这个程序搜索每小时工资最高的雇员:
```
$2 > maxrate { maxrate = $2; maxemp = $1 }
END { print "highest hourly rate:", maxrate, "for", maxemp }
# highest hourly rate: 5.50 for Mary
```

程序
```
{ names = names $1 " " }
END { print names }


```
将所有雇员的名字都收集到一个单独的字符串中, 每一次拼接都是把名字与一个空格符添加到变量 names 的
值的末尾. names 在 END 动作中被打印出来:
Beth Dan Kathy Mark Mary Susie


- 内建函数length
length函数返回字符串的长度
```
{ print $1, length($1) }
```

# 7. 流程控制语句
- If-Else 语句
下面这个程序计算每小时工资多于 $6.00 的雇员的总报酬与平均报酬. 在计算平均数时, 它用到了 if 语
句, 避免用 0 作除数.
```
$2 > 6 { n = n + 1; pay = pay + $2 * $3 }
END     { if (n > 0)
            print n, "employees, total pay is", pay,
                "average pay is", pay/n
        else
            print "no employees are paid more than $6/hour"
}
```

- While 语句
一个 while 含有一个条件判断与一个循环体. 当条件为真时, 循环体执行. 下面这个程序展示了一笔钱在
一个特定的利率下, 其价值如何随着投资时间的增长而增加, 价值计算的公式是 value = amount(1+rate)
years
```
# interest1 - compute compound interest
# input: amount rate years
# output: compounded value at the end of each year
{ i = 1
    while (i <= $3) {
        printf("\t%.2f\n", $1 * (1 + $2) ^ i)
        i = i + 1
    }
}
```

- For 语句
大多数循环都包括初始化, 测试, 增值, 而 for 语句将这三者压缩成一行. 这里是前一个计算投资回报的
程序, 不过这次用 for 循环:
```
# interest2 - compute compound interest
# input: amount rate years
# output: compounded value at the end of each year
{ for (i = 1; i <= $3; i = i + 1)
    printf("\t%.2f\n", $1 * (1 + $2) ^ i)
}
```

# 8. 数组
Awk 提供了数组, 用来存储一组相关的值. 虽然数组给予了 awk 非常可观的力量, 但是我们在这里只展示
一个简单的例子. 下面这个程序按行逆序显示输入数据. 第一个动作将输入行放入数组 line 的下一个元素中;
也就是说, 第一行放入 line[1], 第二行放入 line[2], 依次类推. END 动作用一个 while 循环, 从数组的最
后一个元素开始打印, 一直打印到第一个元素为止:
```
# reverse - print input in reverse order by line
{ line[NR] = $0 } # remember each input line
    END { i = NR # print lines in reverse order
        while (i > 0) {
            print line[i]
            i = i - 1
    }
}
```

