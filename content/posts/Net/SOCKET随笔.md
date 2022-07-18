---
author: "李昌"
title: "SOCKET随笔"
date: "2022-07-18"
tags: ["socket", "net"]
categories: ["Net"]
ShowToc: true
TocOpen: true
---

一个UDP客户可以创建一个套接口并发送一个数据报給一个服务器，然后立即用同一个套接口发送另一个数据报給另一个服务器。同样，一个UDP服务器可以用同一个UDP套接口从5个不同的客户一连串接收5个数据报。

UDP可以是全双工的

----

TCP连接断开时，主动要求断开的一方会存在TIME_WAIT状态，此状态需要持续60s时间（Linux），在此状态期间，连接未完全断开，依然占用一个socket连接。

![20220718100334](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220718100334.png)

存在TIME_WAIT状态的理由：
1. 实现终止TCP全双工连接的可靠性
    如果最后一个ACK丢失，服务器将重发最终的FIN，因此客户必须维护状态信息以允许它重发最终的ACK。

2. 允许老的重复分节在网络中消逝。
    如果某个连接被关闭后，在以后的某个时刻又重新建立起相同的IP地址可端口之间的TCP连接。后一个连接称为前一个连接的化身（incarnation），因为它们的IP地址和端口号都相同，TCP必须防止来自某个连接的老重复分组在连接终止后再现，从而被误解成属于同一连接的化身。
    也就是说，某一个Socket对失效后，在网络中还有针对这个Socket对的分组的情况下又建立了一个一样的Socket对，为了防止网络中针对上一个socket对的分组被误认为是当前连接的分组，必须存在一个时间间隔，使得网络中针对上一个socket对的分组失效。

---


TCP连接耗尽：
对于客户端来说，其端口耗尽后，就不能再建立连接。
对于服务端来说，其使用socket对`server_ip.server_port:client_ip.client_port`来标识一个socket连接，对于某个特定服务来说，其server_ip和server_port是不变的，每次有一个连接进来，服务端都会fork出一个自身的子进程来对连接进行服务。这样，可服务的连接数就由client_ip和client_port两个共同决定，client_ip最多有$2^32$个，client_port最多有$2^16$个，因此理论上最多可服务连接数为$2^48$个。但每一个socket连接都需要消耗一个文件描述符，因此最大连接数还收到文件描述符数目的限制。除此之外，socket连接还会占用内存等资源，这些资源也限制了最大连接数。

---

Unix系统有保留端口的概念，它是小于1024的任何端口。这些端口只能分配給超级用户进程的套接口，所有众所周知的端口（0-1023）都为保留端口，因此分配这些端口的服务器启动时必须具有超级用户的特权。

---

### TCP套接口编程

![20220718165228](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220718165228.png)


1. socket函数

> 为了执行网络I/O，一个进程必须做的第一件事就是调用socket函数，指定期望的通信协议类型。

```c
#include <sys/socket.h>

int socket(int family, int type, int protocol);
```
其中family指明协议族，type是某个常值，参数protocol一般设为0，除非用在原始套接口上。

对于family，其是以下常值之一：

| 族       | 解释       |
| -------- | ---------- |
| AF_INET  | IPv4协议   |
| AF_INET6 | IPv6协议   |
| AF_LOCAL | Unix域协议 |
| AF_ROUTE | 路由套接口 |
| AF_KEY   | 密钥套接口 |


对于type，其是以下常值之一：
| 类型        | 解释         |
| ----------- | ------------ |
| SOCK_STREAM | 字节流套接口 |
| SOCK_DGRAM  | 数据报套接口 |
| SOCKE_RAW   | 原始套接口   |

socket函数在成功时返回一个小的非负整数值，它与文件描述字类似，我们将其称为套接口描述字(socket descriptor)，简称套接字(socketfd)。注意，套接字并没有指定本地协议地址或远程协议地址。

2. bind函数

> 函数bind給套接字分配一个本地协议地址，对于网际协议，协议地址是32位IPv4地址或128位IPv6与16位的TCP或UDP端口号的组合。

```c
#include<sys/socket.h>

int bind(int sockefd, const struct sockeaddr *myaddr, sockelen_t addrlen);
```

第一个参数是套接口描述字，第二个参数是一个指向特定于协议的地址结构的指针，第三个参数是该地址结构的长度。

对于TCP，调用函数bind可以指定一个端口号，指定一个IP地址，可以两者都指定，也可以一个也不指定。

3. listen函数

函数listen仅被TCP服务器调用，它做两件事情：
- 当函数socket创建一个套接口时，它被假设为一个主动接口，也就是说，它是一个将调用connect发起连接的客户套接口，函数listen将未连接的套接口转换成被动套接口，指示内核应接受指向此套接口的连接请求。调用函数listen导致套接口从CLOSED状态转换到LISTEN状态
- 函数的第二个参数规定了内核为此套接口排队的最大连接个数

```c
#include<sys/socket.h>

int listen(int sockefd, int backlog);

// return: 0,success; -1,failed
```

4. accept函数

accept由TCP服务器调用，从已完成连接队列头返回下一个已完成连接。若已完成连接队列为空，则进程睡眠。

```c
#include<sys/socket.h>

int accept(int sockefd, struct sockaddr *cliaddr, socklen_t *addrlen);

// return: 非负描述字，OK；-1，出错
```

参数cliaddr和addrlen用来返回连接对方进程（客户）的协议地址。

如果函数accept执行成功，则返回值是由内核自动生成的一个全新描述字，代表与客户的TCP连接。

当我们讨论函数accept时，常把它的第一个参数称为监听套接口(linstening socket)描述字(由函数socket生成的描述字，用作函数bind和函数listen的第一个参数)，把它的返回值称为已连接套接口(connected socket)描述字。

将这两个套接口区分开是很重要的，一个给定的服务常常是只生成一个监听套接口且一直存在，直到该服务关闭。内核为每个被接受的客户连接创建了一个已连接套接口(也就是说内核已为它完成TCP的三路握手过程)。当服务器完成某客户的服务时，关闭已连接套接口。

5. connect函数

TCP客户用connect函数来建立一个与TCP服务器的连接。

```c
#include<sys/socket.h>

int connect(int sockefd, const struct sockeaddr *servaddr, socklen_t addrlen);

// return: 0,success; -1,failed
```

sockefd是由socket函数返回的套接口描述字，第二，第三个函数分别是一个指向套接口地址结构的指针和该结构的大小。

如果是TCP套接口的话，函数connect激发TCP的三路握手过程，且仅在连接建立成功或出错时才返回。

6. close函数

close用来关闭套接口，终止TCP连接

```c
#include<unistd.h>

int close(int sockefd);

// return: 0,ok; -1,failed
```

