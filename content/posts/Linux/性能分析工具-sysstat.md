---
author: "李昌"
title: "stat系列性能分析工具"
date: "2022-09-17"
tags: ["Linux", "perf"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

## 1. sysstat简介

sysstat是一个软件包，包含监测系统性能及效率的一组工具，这些工具对于我们收集系统性能数据，比如：CPU 使用率、硬盘和网络吞吐数据，这些数据的收集和分析，有利于我们判断系统是否正常运行，是提高系统运行效率、安全运行服务器的得力助手。

sysstat包含多个工具套件：
首先是stat系列
1. iostat
输出CPU的统计信息和所有I/O设备的输入输出（I/O）统计信息
2. mpstat
关于CPU的详细信息（单独输出或者分组输出）
3. pidstat
关于运行中的进程/任务、CPU、内存等的统计信息
4. sysstat
sysstat 工具包的 man 帮助页面。
5. nfsiostat
NFS（Network File System）的I/O统计信息
6. cifsiostat
CIFS(Common Internet File System)的统计信息

还有sa系列
7. sar
保存并输出不同系统资源（CPU、内存、IO、网络、内核等）的详细信息
8. sadc
系统活动数据收集器，用于收集sar工具的后端数据
9. sa1
系统收集并存储sadc数据文件的二进制数据，与sadc工具配合使用
10. sa2
配合sar工具使用，产生每日的摘要报告
11. sadf
用于以不同的数据格式（CVS或者XML）来格式化sar工具的输出


## 2. 简单使用

1. [iostat](http://sebastien.godard.pagesperso-orange.fr/man_iostat.html)

- `iostat`: 显示从系统启动至今的io历史数据

- `iostat -d 2`: 每隔两秒显示一个连续的设备报告。

- `iostat -d 2 6`: 为所有设备显示6个报告，每隔两秒显示一个

- `iostat -x sda sdb 2 6`: 为sda和sdb两个设备显示6个报告，每隔2秒显示一个

- `iostat -p sda 2 6`: 为sda设备和其所有部分显示6个报告，每隔两秒


2. [mpstat](http://sebastien.godard.pagesperso-orange.fr/man_mpstat.html)


- `mpstat -o JSON 2 2`: 以json格式输出cpu信息， 每隔两秒输出一次，输出两个

- `mpstat -P ALL 2 5`: 以2秒为间隔输出5个统计信息

3. [pidstat](http://sebastien.godard.pagesperso-orange.fr/man_pidstat.html)


- `pidstat 2 5`: 为所有活动任务以2秒为间隔输出5个统计信息

- `pidstat -r -p 1643 2 5`: 输出pid为1643的进程的缺页信息，2/5

- `pidstat -T CHILD -r 2 5`: 为所有任务的子进程输出缺页信息，只有非0的缺页才会被统计

- `pidstat -C "fox|bird" -r -p ALL-`: 为命令名包含“box”或“bird”的进程输出全局缺页信息

4. [sar](http://sebastien.godard.pagesperso-orange.fr/man_sar.html)

- `sar -u 2 5`: 每 2 秒报告一次 CPU 利用率, 显示 5 行。

- `sar -I 14 -o int14.file 2 10`: 每 2 秒报告一次 IRQ 14 的统计信息, 显示 10 行, 数据存储在名为 int14.file 的文件中。

- `sar -r -n DEV -f /var/log/sa/sa16`: 显示保存在每日数据文件“sa16”中的内存和网络统计信息。

- `sar -A`: 显示当前每日数据文件中保存的所有统计数据。





