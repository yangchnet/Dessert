---
author: "李昌"
title: "vxlan"
date: "2023-08-21"
tags: ["net", "vxlan"]
categories: ["Net"]
ShowToc: true
TocOpen: true
---

## 基本概念
`VXLAN`(Virtual Extensible LAN, 虚拟局域网扩展)是一种网络虚拟化技术，它试图改善大云计算部署相关的可扩展性问题。它采用类似 VLAN 的封装技术来封装基于 MAC 含括第4层的 UDP 数据包的 OSI 第2层 以太网帧，使用 4789 作为默认分配的 IANA 目的地 UDP 端口号。

`VTEP`是参与VXLAN网络中的主机、虚拟交换机或物理交换机设备，用于在VXLAN隧道和基础网络之间进行数据包的封装和解封装。
具体而言，VTEP设备负责将本地主机或虚拟机的数据包封装为VXLAN数据包，以便在底层IP网络中进行传输。它添加了VXLAN头部，其中包括VXLAN标识符（VNI）和源/目的VTEP IP地址。VTEP还负责在接收到VXLAN数据包时解封装数据包，并将其传递给目标主机或虚拟机。

VTEP (VXLAN Tunnel End Point（虚拟隧道端点）)：vxlan 网络的边缘设备，用来进行 vxlan 报文的处理（封包和解包）。vtep 可以是网络设备（比如交换机），也可以是一台机器（比如虚拟化集群中的宿主机）
VNI（VXLAN Network Identifier）：VNI 是每个 vxlan 的标识，是个 24 位整数，一共有 2^24 = 16,777,216（一千多万），一般每个 VNI 对应一个租户，也就是说使用 vxlan 搭建的公有云可以理论上可以支撑千万级别的租户

在VXLAN（Virtual Extensible LAN）中，`VNI`（VXLAN Network Identifier）用于标识不同的虚拟网络。
VNI是一个32位的标识符，它在VXLAN隧道中的头部中承载，以便在底层IP网络中传递。VNI允许不同的虚拟网络共享同样的物理网络基础设施，每个虚拟网络都可以具有不同的VNI

具体而言，VNI的作用如下：

1. 虚拟网络隔离：VNI用于将不同的虚拟网络隔离开来。当VXLAN网络中的主机或虚拟机发送数据包时，VNI用于将数据包与特定的虚拟网络关联起来。这样，VXLAN网络可以在相同的物理基础设施上同时支持多个虚拟网络，而不会相互干扰。
2. 数据包识别：当接收到VXLAN数据包时，目的VTEP（VXLAN Tunnel Endpoint）设备会根据VNI解析数据包，并根据VNI将数据包交付到正确的虚拟网络。VNI充当了在底层IP网络上传递数据包，并识别数据包的虚拟网络归属的关键标识符。

`FDB(Forwarding Database)`
二层网桥的FDB表项格式可表达为:
```bash
<MAC> <VLAN> <DEV PORT>
```
VXLAN设备的表项与之类似，可以表达为:
```bash
<MAC> <VNI> <REMOTE IP>
```
VXLAN设备根据MAC地址来查找相应的VTEP IP地址，继而将二层数据帧封装发送至相应VTEP。
可以使用如下命令查看fdb表项：
```bash
$ bridge fdb
```
## 点对点vxlan
```bash
$ ip link add vxlan0 type vxlan \
    id 1 \
    remote 192.168.56.3 \
    local 192.168.56.2 \
    dstport 4789 \
    dev eth1

$ ip addr add 10.0.0.2/24 dev vxlan0 # 设置ip

$ ip link set vxlan0 up # 启动设备

$ ip -d link show dev vxlan0 # 查看设备
```
```bash
ip link add vxlan0 type vxlan \
    id 1 \
    remote 192.168.56.2 \
    local 192.168.56.3 \
    dstport 4789 \
    dev eth1

$ ip addr add 10.0.0.3/24 dev vxlan0 # 设置ip

$ ip link set vxlan0 up #  启动设备

$ ip -d link show dev vxlan0 # 查看设备
```
## 多播vxlan
如果 vxlan 要使用多播模式，那么底层的网络结构需要支持多播的功能， virtualbox 网络支持多播。
要组成同一个 vxlan 网络，vtep 必须能感知到彼此的存在。
```bash
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    group 239.1.1.1 \
    dev eth1

$ ip addr add 10.20.1.2/24 dev vxlan0 # 设置ip

$ ip link set vxlan0 up # 启动vxlan0
```
```bash
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    group 239.1.1.1 \
    dev eth1

$ ip addr add 10.20.1.3/24 dev vxlan0 # 设置ip

$ ip link set vxlan0 up # 启动vxlan0
```

## References
[linux 上实现 vxlan 网络 | Cizixs Write Here](https://cizixs.com/2017/09/28/linux-vxlan/)
