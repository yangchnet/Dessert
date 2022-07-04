---
author: "李昌"
title: "inotify"
date: "2022-07-04"
tags: ["Linux"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

## 1. 什么是inotify

Inotify API提供了一种监视文件系统的机制事件。 Inotify可用于监视单个文件或监视目录。当监视目录时，Inotify将返回目录本身的事件，以及内部的文件目录。

简单的说，就是inotify可以为你监控文件系统的变化，在发生一些事件时通知你。

## 2. 实验

需要安装`inotify-tools`包

```sh
# arch系统
yay -S inotify-tools
```

`inotify-tools`提供了两个命令：`inotifywait`和`inotifywatch`

### 2.1 inotifywait

先来看`inotifywait`,它被用来"Wait for a particular event on a file or set of files.", 也就是说等待在文件上的某些事件发生。使用`inotifywait -h`查看其支持的参数：

```
-h show this help info

--exclude <pattern> 排除匹配所给正则的所有事件

--excludei <pattern> 类似前一个命令但非敏感

--include <pattern> 排除除匹配正则之外的所有事件

--includei <pattern> 类似前一个命令但非敏感

-m|--monitor 在timeout之前保持监听，若不设置此标志，inotifywait将在一个事件后退出。

-d|--daemon 类似前一个命令但在后台运行，将日志输出到`--outfile`所指定的文件

-P|--no-dereference 不跟踪符号链接

-r|--recursive  递归的监听、

--fromfile <file> Read files to watch from <file> or `-' for stdin.

-o|--outfile <file> 输出到<file>而不是标准输出

-s|--syslog 向syslog发送错误而不是Stderr

-q|--quiet 只输出事件

-qq 啥也不输出

--format <fmt>  以特定格式输出

--no-newline 在格式化输出后不打印换行符

--timefmt <fmt> strftime-compatible format string for use with
                        %T in --format string.

-c|--csv 以csv格式输出

-t|--timeout <seconds> 当监听一个事件时，在<seconds>秒后超时

-e|--event <event1> 监听特定事件，若不设置，则监听所有事件

```


支持的event有：
```sh
Events:
        access          file or directory contents were read
        modify          file or directory contents were written
        attrib          file or directory attributes changed
        close_write     file or directory closed, after being opened in
                        writable mode
        close_nowrite   file or directory closed, after being opened in
                        read-only mode
        close           file or directory closed, regardless of read/write mode
        open            file or directory opened
        moved_to        file or directory moved to watched directory
        moved_from      file or directory moved from watched directory
        move            file or directory moved to or from watched directory
        move_self               A watched file or directory was moved.
        create          file or directory created within watched directory
        delete          file or directory deleted within watched directory
        delete_self     file or directory was deleted
        unmount         file system containing file or directory unmounted
```

在终端1中：
```sh
mkdir ~/inotify-demo && cd inotify-demo # 建立测试文件夹

inotifywait -rmc -q .
```

在终端2中：
```sh
cd ~/inotify-demo

touch aa.txt

```

此时切换到终端1，可以看到事件输出：
```sh
...
./,CREATE,aa.txt
./,OPEN,aa.txt
./,ATTRIB,aa.txt
./,"CLOSE_WRITE,CLOSE",aa.txt
./,"OPEN,ISDIR",
./,"CLOSE_NOWRITE,CLOSE,ISDIR",
./,OPEN,aa.txt
./,"CLOSE_NOWRITE,CLOSE",aa.txt
```

打开文件`aa.txt`，写入一行
```sh
inotify-demo
```

可看到终端1中也已将事件打印出：
```sh
...
./,CREATE,4913
./,OPEN,4913
./,ATTRIB,4913
./,"CLOSE_WRITE,CLOSE",4913
./,DELETE,4913
./,MOVED_FROM,aa.txt
./,CREATE,aa.txt
./,OPEN,aa.txt
./,MODIFY,aa.txt
./,MODIFY,aa.txt
./,ATTRIB,aa.txt
./,"CLOSE_WRITE,CLOSE",aa.txt
./,ATTRIB,aa.txt
./,"OPEN,ISDIR",
./,"CLOSE_NOWRITE,CLOSE,ISDIR",
```

将所有的事件都打印出略显杂乱，这里我们只让其监听`modify`事件
```sh
inotifywait -rmc -q -e modify .
```

创建文件`bb.txt`
```sh
touch bb.txt
```

可观察到，未有任何事件被打印。

在`bb.txt`中写入一行`inotify-demo`

可观察到事件打印：
```sh
./,MODIFY,bb.txt
./,MODIFY,bb.txt
```

### 2.2 inotifywatch

inotifywatch被用于收集文件系统使用信息

其命令参数和`inotifywait`大致相同，可使用`inotifywatch`查看

实验：

```sh
cd ~/inotify-demo

# 在30秒内监听crate,modify,delete三个事件，在30秒结束后打印统计信息
inotifywatch -v -e create -e modify -e delete -t 30 -r .
```


接下来30秒，创建了4个文件，修改了2个文件，删除了1个文件

事件统计信息为:
```sh
Establishing watches...
Setting up watch(es) on .
OK, . is now being watched.
Total of 1 watches.
Finished establishing watches, now collecting statistics.
Will listen for events for 30 seconds.
total  modify  create  delete  filename
15     4       8       3       ./
```

TODO：统计信息与实际操作有所不符，目前不清楚为啥

## 3. 使用golang调用inotify

> https://github.com/fsnotify/fsnotify

```go
package main

import (
	"log"

	"github.com/fsnotify/fsnotify"
)

func main() {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		log.Fatal(err)
	}
	defer watcher.Close()

	done := make(chan bool)
	go func() {
		for {
			select {
			case event, ok := <-watcher.Events:
				if !ok {
					return
				}
				log.Println("event:", event)
				if event.Op&fsnotify.Write == fsnotify.Write {
					log.Println("modified file:", event.Name)
				}
			case err, ok := <-watcher.Errors:
				if !ok {
					return
				}
				log.Println("error:", err)
			}
		}
	}()

	err = watcher.Add("/tmp/foo")
	if err != nil {
		log.Fatal(err)
	}
	<-done
}
```

## References

https://juejin.cn/post/6988561056364757022

https://www.infoq.cn/article/inotify-linux-file-system-event-monitoring

https://www.cnblogs.com/centos2017/p/7896715.html