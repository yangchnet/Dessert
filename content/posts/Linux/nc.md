---
author: "李昌"
title: "nc-任意的TCP/UDP连接和监听"
date: "2023-08-21"
tags: [""]
categories: [""]
ShowToc: true
TocOpen: true
---

在进行一些网络测试时，可能会需要一些网络监听，这时如果再手动写一个网络程序，未免有些麻烦，此时可以使用nc命令直接创建tcp连接或监听某端口。

It can open TCP connections, send UDP packets, listen on arbitrary TCP and UDP ports, do port scanning, and deal with both IPv4 and IPv6.

## 使用case

1. 快速创建client/server对
```bash
nc -l 1234
```
```bash
nc 127.0.0.1 1234
```
任何在client端的输出都将在server端显示

2. 端口扫描
```bash
nc -zv 127.0.0.1 8885-8889
```

3. 其他例子
```bash
nc -p 31337 -w 5 abc.com 42
```
```bash
nc -u abc.com 53
```
```bash
nc -s 10.1.2.3 abc.com 42
```
```bash
nc -lU /var/tmp/dsocket
```
