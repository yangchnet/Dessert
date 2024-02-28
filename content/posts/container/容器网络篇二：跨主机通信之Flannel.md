---
author: "李昌"
title: "容器网络篇二：跨主机通信之Flannel"
date: "2024-02-28"
tags: ["container", "network"]
categories: ["container"]
ShowToc: true
TocOpen: true
---

[flannel项目](https://github.com/flannel-io/flannel)本身只是一个框架，真正实现容器网络容器功能的，是[Flannel的后端实现](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)，目前，Flannel推荐的后端实现有四种，分别是：

- VXLAN
- host-gw
- WireGuard
- UDP

此外还有一些[实验性的后端实现](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#experimental-backends)

# UDP模式
> UDP 模式，是最直接、也是最容易理解的容器跨主网络实现。

假设有两台宿主机：

- 宿主机 Node 1 上有一个容器 container-1，它的 IP 地址是 100.96.1.2，对应的 docker0 网桥的地址是：100.96.1.1/24。
- 宿主机 Node 2 上有一个容器 container-2，它的 IP 地址是 100.96.2.3，对应的 docker0 网桥的地址是：100.96.2.1/24。

现在的任务是：让container-1访问container-2。<br />此时访问container-2的这个IP包的源地址为100.96.1.2，目的地址为100.96.2.1。<br />按照上篇所述，首先这个IP包会通过veth对来到docker0网桥，而此时docker0网桥上并没有插入IP为100.96.1.2的设备，因此会走默认路由规则，出现在宿主机上。<br />此时如果机器上已经安装了flannel，那么flannel会为宿主机创建出一系列路由规则，如：
```bash
# 在Node 1上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0
100.96.1.0/24 dev docker0  proto kernel  scope link  src 100.96.1.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.2
```
根据路由表，目的地址为100.96.2.1的IP包应匹配到第一条路由规则，于是IP包被交给flannel0设备。<br />flannel0是一个TUN设备（Tunnel设备），在 Linux 中，TUN 设备是一种工作在三层（Network Layer）的虚拟网络设备。TUN 设备的功能非常简单，即：在操作系统内核和用户应用程序之间传递 IP 包。<br />传递到flannel0设备的包，会被交给创建这个设备的应用程序，即flannel进程。<br />flannel进程会从ETCD中获取各个宿主机包含的子网地址，并按照子网与宿主机的对应关系，将IP包发往对应的宿主机。<br />例如：在我们的例子中，Node 1 的子网是 100.96.1.0/24，container-1 的 IP 地址是 100.96.1.2。Node 2 的子网是 100.96.2.0/24，container-2 的 IP 地址是 100.96.2.3。<br />而这些子网与宿主机的对应关系，正是保存在 Etcd 当中，如下所示：
```bash
$ etcdctl ls /coreos.com/network/subnets
/coreos.com/network/subnets/100.96.1.0-24
/coreos.com/network/subnets/100.96.2.0-24
/coreos.com/network/subnets/100.96.3.0-24
```
所以，flanneld 进程在处理由 flannel0 传入的 IP 包时，就可以根据目的 IP 的地址（比如 100.96.2.3），匹配到对应的子网（比如 100.96.2.0/24），从 Etcd 中找到这个子网对应的宿主机的 IP 地址是 10.168.0.3，如下所示：
```bash
$ etcdctl get /coreos.com/network/subnets/100.96.2.0-24
{"PublicIP":"10.168.0.3"}
```
这样，fannel就会把这个IP报文封装在一个UDP包里，然后发送给node2的8285端口，这是flannel程序监听的端口。<br />UDP报文到达node2后，node2上的flannel程序会将它解包后交给node2上的flannel0设备，接下来又是查表，node2上的flannel0设备维护的路由表与node1上的类似：
```bash
# 在Node 2上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.2.0
100.96.2.0/24 dev docker0  proto kernel  scope link  src 100.96.2.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.3
```
目的地址100,96.2.1，会匹配到第二条路由，于是IP报文被交给了docker0设备，而此时container-2就在docker0上，IP报文直接到达目的地。
![20240228203944](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20240228203944.png)
可以看到，Flannel UDP 模式提供的其实是一个三层的 Overlay 网络，即：它首先对发出端的 IP 包进行 UDP 封装，然后在接收端进行解封装拿到原始的 IP 包，进而把这个 IP 包转发给目标容器。这就好比，Flannel 在不同宿主机上的两个容器之间打通了一条“隧道”，使得这两个容器可以直接使用 IP 地址进行通信，而无需关心容器和宿主机的分布情况。



Flannel UDP模式有较严重的性能问题，相比于两台宿主机之间的直接通信，基于 Flannel UDP 模式的容器通信多了一个额外的步骤，即 flanneld 的处理过程。而这个过程，由于使用到了 flannel0 这个 TUN 设备，仅在发出 IP 包的过程中，就需要经过三次用户态与内核态之间的数据拷贝，如下所示：

![20240228204003](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20240228204003.png)

第一次，用户态的容器进程发出的 IP 包经过 docker0 网桥进入内核态；第二次，IP 包根据路由表进入 TUN（flannel0）设备，从而回到用户态的 flanneld 进程；第三次，flanneld 进行 UDP 封包之后重新进入内核态，将 UDP 包通过宿主机的 eth0 发出去。<br />此外，Flannel进行UDP封装和解封时，也都是在用户态完成，这些操作也较为耗时。

# VXLAN模式
VXLAN的原理：https://yangchnet.github.io/Dessert/posts/net/vxlan/

VXLAN，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。所以说，VXLAN 可以完全在内核态实现上述封装和解封装的工作，从而通过与前面相似的“隧道”机制，构建出覆盖网络（Overlay Network）。

VXLAN 的覆盖网络的设计思想是：在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信。当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里。

而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。

TEP 设备的作用，其实跟前面的 flanneld 进程非常相似。只不过，它进行封装和解封装的对象，是二层数据帧（Ethernet frame）；而且这个工作的执行流程，全部是在内核里完成的（因为 VXLAN 本身就是 Linux 内核中的一个模块）。

下图是flannel VXLAN模式通信的过程：

![20240228204029](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20240228204029.png)


假设当前container-1的IP地址为10.1.15.2，要访问的container-2的IP地址为10.1.16.3。其通信过程如下：

1. container-1发出请求，该请求的目的地址为10.1.16.3。首先，这个IP包会出现在docker0网桥上，然后被路由到本机flannel.1设备。我们称这个IP包为：原始IP包
2. flannel.1设备是一个VTEP设备，现在它要找到另一个宿主机（node2）上对应的VTEP设备，node2上的VTEP设备信息，来自于flanneld进程。当node2启动并加入Flannel网络后，在除node2以外所有节点上，flanneld就会添加一条路由规则：
```bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
...
10.1.16.0       10.1.16.0       255.255.255.0   UG    0      0        0 flannel.1
```
这条路由规则规定，凡是发往10.1.16.0的包，都要经flannel.1发出，并且其最后被发往的网关地址为10.1.16.0。而10.1.16.0正是node2上VTEP设备flannel.1的IP地址。
> 现在node1上的flannel.1设备知道了它要发往的对端VTEP设备的IP地址为10.1.16.0，但是具体如何将原始IP包发送到node2上呢？

3. 知道了三层IP地址以后，按照网络协议栈的通信逻辑，应该通过ARP协议去寻找对应的二层MAC地址，这里flanneld进程在node2启动时，也在node1上添加了ARP记录：
```bash
# 在Node 1上
$ ip neigh show dev flannel.1
10.1.16.0 lladdr 5e:f8:4f:00:e3:37 PERMANENT
```
这条记录的意思非常明确，即：IP 地址 10.1.16.0，对应的 MAC 地址是 5e:f8:4f:00:e3:37

4. 拿到了MAC地址以后，Linux内核即可开始二层封包工作，封装的二层数据帧格式如下：
![20240228204054](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20240228204054.png)


Linux 内核会把“目的 VTEP 设备”的 MAC 地址，填写在图中的 Inner Ethernet Header 字段，得到一个二层数据帧。这里的封包过程只是加了一个二层头，内层的目的容器的IP地址仍然是container-2的IP地址
> 这里VTEP设备的MAC地址，对于实际通信来说实际上没什么用（因为要使用eth0设备来通信），因此还需要再次进行封装。到这里，我们把目的地址为目的VTEP设备的MAC地址的二层数据帧称为“内部数据帧”

1. 内部数据帧将被封装成宿主机网络里的一个普通数据帧，通过宿主机的eth0网卡设备进行传输：Linux内核会在内部数据帧的前面加上一个VXLAN头，表示这是一个VXLAN帧，VXLAN头中的VNI字段表明了该VXLAN数据帧应该被哪个VTEP设备处理，Flannel使用的VNI为1。然后这个数据帧会被装入一个UDP包里等待eth0发出。
2. 此时数据帧已经准备完成，但还不知道这个UDP包应该发往哪个IP地址。在这种场景下，flannel.1 设备实际上要扮演一个“网桥”的角色，在二层网络进行 UDP 包的转发。而在 Linux 内核里面，“网桥”设备进行转发的依据，来自于一个叫作 FDB（Forwarding Database）的转发数据库。这个 flannel.1“网桥”对应的 FDB 信息，也是 flanneld 进程负责维护的。它的内容可以通过 bridge fdb 命令查看到，如下所示：
```bash
# 在Node 1上，使用“目的VTEP设备”的MAC地址进行查询
$ bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
```
在上面这条 FDB 记录里，指定了这样一条规则，即：发往我们前面提到的“目的 VTEP 设备”（MAC 地址是 5e:f8:4f:00:e3:37）的二层数据帧，应该通过 flannel.1 设备，发往 IP 地址为 10.168.0.3 的主机。显然，这台主机正是 Node 2，UDP 包要发往的目的地就找到了。

1. UDP是一个四层数据包，Linux内核会将刚才找到的目的地址与本机源地址添加到封装了该UDP包的IP包头中，然后通过内核协议栈获取到node2的MAC地址，最终封装为二层数据帧。这个二层数据帧我们称为：外部数据帧
2. 外部数据帧通过eth0设备从node1发往node2。node2的内核网络栈经过解包后拿到内部数据帧，然后发现这是一个VXLAN帧，通过VNI得知应该交给flannel.1设备。flannel.1设备拿到数据包后进一步解包，取出“原始IP包”，就可以通过docker0网桥将数据发送到目的容器container-2。

最终发往node2的外部数据帧，其格式大致为：

![20240228204117](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20240228204117.png)

总结一下：通过路由规则知道了该将数据包发送到flanneld.1这个vtep设备处理，并且知道了对端vtep的ip地址；通过vtep IP地址找到了对端的vtep mac地址；通过对端vtep mac地址查询 bridge fdb list 查到了这个mac地址对应的宿主机地址；通过宿主机地址，走正常的网络协议栈将数据发往目的宿主机；目的宿主机通过VNI得知应交给flannel.1设备处理。

