---
author: "李昌"
title: "CgroupV2"
date: "2022-05-19"
tags: ["Cgroup"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

## 1. Cgroup概览

cgroup是Linux内核提供的一种按层次组织进程，并对进程资源按层次进行分配和限制的机制。

cgroup 主要由两部分组成——core和controller。core主要负责分层组织进程。 controller负责为属于当前cgroup的进程分配和限制资源.

多个cgroup以树形结构组织，系统中每个进程都属于一个cgroup，一个进程中的所有线程都属于同一个cgroup。

controller可以在cgroup上有选择的开启，开启后的controller将影响这个cgroup内的所有进程。

使用如下命令挂载cgroupv2

```sh
mount -t cgroup2 none $MOUNT_POINT # MOUNT_POINT 是任意你想要挂载到的位置
```

直接在$MOUNT_POINT创建一个文件夹即可创建一个cgroup

```sh
mkdir $MOUNT_POINT/$GROUP_NAME
```

每个cgroup内都有一个`cgroup.procs`接口文件，其中逐行列出了属于当前cgroup的所有进程的PID。需要注意的是，PID可能重复出现且无序。

若想将某个进程移动到一个cgroup中，只需将其PID写入cgroup.procs文件,进程中的所有线程也会迁移到该cgroup中。fork出的子进程依然属于这个cgroup。

若要删除一个cgroup，需要注意一点：这个cgroup内需要**没有任何子进程且仅与僵尸进程相关联**，且没有子cgroup.满足了上述条件后，将其作为一个**空目录**删除即可,使用`rm -rf`无法对cgroup目录进行删除。

```sh
rmdir $MOUNT_POINT/$GROUP_NAME
```

`/proc/$PID/cgroup`中包含一个进程所属的cgroup。

```sh
$ cat /proc/self/cgroup # self表示当前shell进程
0::/user.slice/user-1000.slice/session-2.scope
```

如果进程成为僵尸进程并且随后删除了与之关联的 cgroup，则将“（已删除）”附加到路径中：

```sh
$ cat /proc/842/cgroup
0::/test-cgroup/test-cgroup-nested (deleted)
```

CgroupV2还支持[线程模式](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#threads)

每个非根 cgroup 都有一个`cgroup.events`文件，其中包含`populated`字段，指示 cgroup 的子层次结构中是否有实时进程。 如果 cgroup 及其后代中没有实时进程，则其值为 0； 否则为1.

例如：考虑如下cgroup结构，括号内数字代表cgroup内进程数：
```
A(4) - B(0) - C(1)
            \ D(0)
```

则A、B 和 C 的`populated`字段将为 1，而 D 为 0。在 C 中的一个进程退出后，B 和 C 的`populated`字段将翻转为“0”，文件修改事件将在两个cgroup的`cgroup.events`中出现。

每个 cgroup 都有一个“cgroup.controllers”文件，其中列出了 cgroup 可启用的所有控制器：
```sh
$ cat cgroup.controllers
cpu io memory
```

默认情况下不启用任何控制器。可以通过写入`cgroup.subtree_control`文件来启用和禁用控制器：
```sh
$ echo "+cpu +memory -io" > cgroup.subtree_control # 开启cpu、memory 控制器， 关闭io控制器
```

资源是自上而下分布的，只有当资源从父级分发给它时，cgroup 才能进一步分发资源。父cgroup可以限制子cgroup中所包含的controller。只有包含在父cgroup的`cgroup.subtree_control`中的controller才能分配給子cgroup。

![20220519155708](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220519155708.png)

进程只能分配到根cgroup和叶子cgroup。

## 2. core接口文件

> **cgroup.type**

存在于非根cgroup中的可读写单值文件。指示当前cgroup的类型。
其可能是以下值：
`domain`: 当前cgroup是一个正常的domain cgroup

`domain threaded`: 一个线程域 cgroup，用作线程子树的根。

`domain invalid`: 处在非法状态的cgroup，不能包含实时进程或controller，但允许成为一个`threaded`cgroup.

`threaded`: 一个线程化的 cgroup，它是线程化子树的成员。

可直接向此文件写入`threaded`使当前cgroup成为一个线程cgroup

> **cgroup.procs**

按行列出当前cgroup中包含的进程PID. PID无序，且可能重复。可直接向此文件写入某个PID以实现将进程迁移进当前cgroup。

> **cgroup.threads**

同`cgroup.procs`,但其中列出的数字为线程TID

> **cgroup.controllers**

列出当前cgroup中启用的controller

> **cgroup.subtree_control**

指出子cgroup可启用的controller。可通过以下命令启用或关闭某个controller
```sh
$ echo "+cpu +memory -io" > cgroup.subtree_control # 开启cpu、memory 控制器， 关闭io控制器
```

> **cgroup.events**

只读文件.populated指示cgroup中是否存在存活进程，frozen指示 cgroup是否被冻结

> **cgroup.max.descendants**

允许的最大后代cgroup数量。

> **cgroup.max.depth**

最大cgroup树深度。

> **cgroup.stat**

只读文件。

nr_descendants： 可见后代cgroup的总数
nr_dying_descendants: 待死亡的后代 cgroup 总数。 一个 cgroup 在被用户删除后会死掉。 在完全销毁之前，cgroup 将保持在死亡状态一段时间（可能取决于系统负载）。

进程不能加入待死亡的cgroup. 待死亡的 cgroup 可以消耗不超过限制的系统资源，这些资源在 cgroup 删除时处于活动状态。

> **cgroup.freeze**

存在于非根cgroup。允许的值为1或0，默认为0.

当值为1时，代表冻结这个cgroup。冻结一个cgroup，其中的所有进程将不再运行，直到这个cgroup解冻。对于冻结的cgroup，其子cgroup也将冻结。

> **cgroup.kill**

只写文件，唯一允许值为1.向这个文件写入1将导致这个cgroup以及所有子cgroup被kill。这将导致当前cgroup中的所有进程都被`SIGKILL`信号杀死。


## 3. cpu controller

> **cpu.stat**

只读键值文件，在cpu controller开启时才会出现。

usage_usec：占用cpu总时间。

user_usec：用户态占用时间。

system_usec：内核态占用时间。

nr_periods：周期计数。

nr_throttled：周期内的限制计数。

throttled_usec：限制执行的时间。

> **cpu.weight**

只存在于非根cgroup，默认值100. 值域为[1, 10000]

> **cpu.weight.nice**

只存在于非根cgroup，默认值0, 值域为[-20, 19]

> **cpu.max**

其中包含空格分隔的两个值，$MAX $PERIOD，表示在每个$PERIOD时期内，可消耗的cpu为$MAX，当$MAX为max时，代表无限制。

> **cpu.max.burst**

默认值为0，burst值域为[0, $MAX]

> **cpu.pressure**

显示当前cgroup的cpu使用压力状态。详情参见：Documentation/accounting/psi.rst。psi是内核新加入的一种负载状态检测机制，可以目前可以针对cpu、memory、io的负载状态进行检测。通过设置，我们可以让psi在相关资源负载达到一定阈值的情况下给我们发送一个事件。用户态可以通过对文件事件的监控，实现针对相关负载作出相关相应行为的目的。

> **cpu.uclamp.min**

uclamp提供了一种用户空间对于task util进行限制的机制，通过该机制用户空间可以将task util钳制在[util_min, util_max]范围内，而cpu util则由处于其运行队列上的task的uclamp值决定.

这个文件表明了对cpu利用率的最小限制。其数字为百分比数字。

> **cpu.uclamp.max**

设置了cpu的最大利用率

## 4. memory controller



## References

1. [浅谈 Cgroups V2](https://www.infoq.cn/article/hbqqfeyqxzhnes5jipqt)

2. [详解Cgroup V2](https://zorrozou.github.io/docs/%E8%AF%A6%E8%A7%A3Cgroup%20V2.html)

3. [The Linux Kernel · Control Group v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)