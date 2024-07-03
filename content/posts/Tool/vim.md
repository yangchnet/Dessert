---
author: "李昌"
title: "vim"
date: "2024-07-03"
tags: ["vim", "tools"]
categories: ["Tool"]
ShowToc: true
TocOpen: true
---

# 基本操作

- `%`跳转到匹配的括号
- `f<character>` 找到当前行中，在光标之后的字符，在找到字符后，使用`;`跳转到后一个字符，`,`跳转到前一个字符
- `F<character>` 找到当前行中，在光标之前的字符
- `<line_number>G` 跳转到某行
- `CTRL-e` 向下翻滚页面
- `CTRL-u` 向上翻滚半页
- `CTRL-d` 向下翻滚半页
- `d$`删除至行末，`dgg`删除从光标到文件开头，`ggdG`删除整个文件，`diw`删除当前单词，`dip`删除当前段，`ciw`替换当前单词，`yiw`复制当前单词

# vim的抽象层

## 缓冲区
缓冲区有三种不同状态：
- `active` 缓冲区显示在窗口
- `hidden` 缓冲区不显示，但存在且文件处于打开状态
- `inactive` 缓冲区不显示且为空，没有链接任何文件

- `:buffers` 查看所有打开的缓冲区
- `:buffer <ID_or_name>` 移动到该缓冲区 
- `:bnext`或`:bn` 移动到下一个缓冲区
- `:bprevious`或`:bp`移动到上一个缓冲区
- `:bfirst`或`:bf`移动到第一个
- `:blast`或`:bl`移动到最后一个缓冲区
- `CTRL-^`切换到备用缓冲区，在列表中用`#`显示
- `<ID>CTRL-^`切换到ID的特定缓冲区，例如`75CTRL-^`切换到ID为75的缓冲区
- `:bufdo <command>` 将命令应用到所有缓冲区
- 并非所有缓冲区但在列表中，使用`:buffers!`或`:ls!`列出所有缓冲区
- `:badd <filename>` 将某文件添加到缓冲区
- `:bdelete <ID_or_name>` 或`:bd 1`删除缓冲区，`1,10bdelete`删除id从1到10的缓冲区

## 窗口 windows
vim中的窗口是一个用来显示缓冲区内容的空间，当关闭窗口时，缓冲区仍保持打开状态

- `:new`创建新的窗口
- `CTRL-W s` 水平拆分当前窗口
- `CTRL-W v` 垂直拆分当前窗口
- `CTRL-W n` 水平拆分当前窗口并编辑新文件
- `CTRL-W ^` 使用备用文件拆分
- `<buffer_ID>CTRL-W ^` 使用某ID的缓冲区拆分窗口
- `CTRL-W 上下左右` 将光标移动到另一个窗口
- `CTRL-W r` 旋转窗口，`CTRL-W x`与下一个窗口互换
- `CTRL-W =` 调整窗口大小，使其适合相同大小的屏幕
- `CTRL-W -` 降低窗口高度，`CTRL-W +` 增加窗口高度, `CTRL-W <` 减小窗口宽度， `CTRL-W >` 增加窗口宽度
- `:q` 退出当前窗口

## Tabs

> `:help tab-page`

缓冲区是一个打开的文件，窗口是活动缓冲区的容器。可以将选项卡视为一堆窗口的容器。这样一来，它与标准 IDE 中的选项卡概念非常不同

- `:tabnew` 或 `:tabe` 打开新的选项卡
- `:tabclose`或 `:tabc` 关闭当前选项卡
- `tabonly` 或 `tabo` 关闭除当前选项卡外的所有传其他选项卡
- `gt` go 到下一个tab
- `gT` go 到上远程tab
- `1gT` go到第一个选项卡


## 参数列表 arglist

> `:help arglist`

参数列表用于组织打开的文件，可将其视为缓冲区列表的稳定子集。它遵循以下两个原则:
- arglist 中的每个文件都将位于缓冲区列表中。
- 缓冲区列表中的某些缓冲区不会出现在 arglist 中。

运行 Vim 时要打开的文件（例如执行 `vim file1 file2 file3` ）将自动添加到 arglist 中
- `:args` 显示arglist
- `:argadd` 将文件添加到arglist
- `argdo` 对arglist中的每个文件执行命令
- `:next` 移动到arglist中的下一个文件
- `:prev` 移动到arglist中的上一个文件
- `:first` 移动到arglist中的第一个文件


# 跳跃

> `:jumps`

- `CTRL-o` 转到上一个光标位置
- `CTRL-i` 转到下一个光标位置


> :help jump-motions
> :help jumplist
> :help changelist
- `[m` 移动到方法的开头
- `]m` 移动到方法的末尾

# 更改列表

每次插入内容（使用 INSERT 模式）时，光标的位置都会保存在更改列表中。

- `g;` 跳转到下一个更改
- `g,` 跳转到上一个更改

# 重复

> :help single-repeat

- `.` 重复上一次更改
- `@:` 重复执行上次的命令

# 录制宏

  - `q<lowecase_letter>`` - 开始在寄存器中记录击键。您可以将寄存器视为内存中的一个位置，也可以将其视为剪贴板。
  - 您接下来要执行的每次击键都将被保存。
  - `q` - 停止录制。
  - `@<lowercase_letter>` - 执行您录制的击键。

# 命令行窗口

> :help cmdline-window

- `q:` 打开命令行历史记录
- `q/` 或 `q?` 打开搜索历史记录
- `CTRL+f` 在命令行模式下打开命令行历史记录
- `:history :` 命令行历史记录
- `:history /` 或 `:history ?`搜索记录
