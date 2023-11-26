---
author: "李昌"
title: "Linux命令之nc"
date: "2023-11-26"
tags: ["linux", "command"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---
`nc`： 任意的TCP/UDP连接和监听

It can open TCP connections, send UDP packets, listen on arbitrary TCP and UDP ports, do port scanning, and deal with both IPv4 and IPv6.

<a name="gJjG9"></a>
### 使用case

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
