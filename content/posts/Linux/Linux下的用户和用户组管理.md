---
author: "李昌"
title: "Linux下的用户和用户组管理"
date: "2022-03-26"
tags: ["Linux"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

## 1.概述
Linux 是多用户多任务操作系统，换句话说，Linux 系统支持多个用户在同一时间内登陆，不同用户可以执行不同的任务，并且互不影响。

例如，某台 Linux 服务器上有 4 个用户，分别是 root、www、ftp 和 mysql，在同一时间内，root 用户可能在查看系统日志、管理维护系统；www 用户可能在修改自己的网页程序；ftp 用户可能在上传软件到服务器；mysql 用户可能在执行自己的 SQL 查询，每个用户互不干扰，有条不紊地进行着自己的工作。与此同时，每个用户之间不能越权访问，比如 www 用户不能执行 mysql 用户的 SQL 查询操作，ftp 用户也不能修改 www 用户的网页程序。

不同用户具有不问的权限，毎个用户在权限允许的范围内完成不间的任务，Linux 正是通过这种权限的划分与管理，实现了多用户多任务的运行机制。

因此，如果要使用 Linux 系统的资源，就必须向系统管理员申请一个账户，然后通过这个账户进入系统（账户和用户是一个概念）。通过建立不同属性的用户，一方面可以合理地利用和控制系统资源，另一方面也可以帮助用户组织文件，提供对用户文件的安全性保护。

每个用户都有唯一的用户名和密码。在登录系统时，只有正确输入用户名和密码，才能进入系统和自己的主目录。

用户组是具有相同特征用户的逻辑集合。简单的理解，有时我们需要让多个用户具有相同的权限，比如查看、修改某一个文件的权限，一种方法是分别对多个用户进行文件访问授权，如果有 10 个用户的话，就需要授权 10 次，那如果有 100、1000 甚至更多的用户呢？

显然，这种方法不太合理。最好的方式是建立一个组，让这个组具有查看、修改此文件的权限，然后将所有需要访问此文件的用户放入这个组中。那么，所有用户就具有了和组一样的权限，这就是用户组。

将用户分组是 Linux 系统中对用户进行管理及控制访问权限的一种手段，通过定义用户组，很多程序上简化了对用户的管理工作。

## 2.用户和用户组的关系
用户和用户组的对应关系有以下 4 种：

一对一：一个用户可以存在一个组中，是组中的唯一成员；
一对多：一个用户可以存在多个用户组中，此用户具有这多个组的共同权限；
多对一：多个用户可以存在一个组中，这些用户具有和组相同的权限；
多对多：多个用户可以存在多个组中，也就是以上 3 种关系的扩展。 用户和组之间的关系可用下图表示：图 1 Linux用户和用户组
![20220326152629](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220326152629.png)


## 3. UID和GID(用户ID和组ID)
登陆 Linux 系统时，虽然输入的是自己的用户名和密码，但其实 Linux 并不认识你的用户名称，它只认识用户名对应的 ID 号（也就是一串数字）。Linux 系统将所有用户的名称与 ID 的对应关系都存储在 /etc/passwd 文件中。

说白了，用户名并无实际作用，仅是为了方便用户的记忆而已。

要论证 "Linux系统不认识用户名" 也很简单，在前面章节，我们曾经在网络上下载过 ".tar.gz" 或 ".tar.bz2" 格式的文件，在解压缩之后的文件中，你会发现文件拥有者的属性显示的是一串数字，这很正常，就是因为系统只认识代表你身份的 ID，这串数字就是用户的 ID（UID）号。

Linux 系统中，每个用户的 ID 细分为 2 种，分别是用户 ID（User ID，简称 UID）和组 ID（Group ID，简称 GID），这与文件有拥有者和拥有群组两种属性相对应（如下图所示）。

![20220326152705](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220326152705.png)

每个文件都有自己的拥有者 ID 和群组 ID，当显示文件属性时，系统会根据 /etc/passwd 和 /etc/group 文件中的内容，分别找到 UID 和 GID 对应的用户名和群组名，然后显示出来。

> 使用cat命令，查看/etc/passwd文件，查看当前用户和root用户的UID和GID

## 4. /etc/passwd内容解释

Linux 系统中的 /etc/passwd 文件，是系统用户配置文件，存储了系统中所有用户的基本信息，并且所有用户都可以对此文件执行读操作。 可以看到，/etc/passwd 文件中的内容非常规律，每行记录对应一个用户。

为什么Linux 系统中默认怎么会有这么多的用户？ 这些用户中的绝大多数是系统或服务正常运行所必需的用户，这种用户通常称为系统用户或伪用户。系统用户无法用来登录系统，但也不能删除，因为一旦删除，依赖这些用户运行的服务或程序就不能正常执行，会导致系统问题。

不仅如此，每行用户信息都以 "：" 作为分隔符，划分为 7 个字段，每个字段所表示的含义如下： 用户名：密码：UID（用户ID）：GID（组ID）：描述性信息：主目录：默认Shell

接下来，给大家逐个介绍这些字段。

**1). 用户名**

用户名，就是一串代表用户身份的字符串。

前面讲过，用户名仅是为了方便用户记忆，Linux 系统是通过 UID 来识别用户身份，分配用户权限的。/etc/passwd 文件中就定义了用户名和 UID 之间的对应关系。

**2). 密码**
"x" 表示此用户设有密码，但不是真正的密码，真正的密码保存在 /etc/shadow 文件中（下一节做详细介绍）。

在早期的 UNIX 中，这里保存的就是真正的加密密码串，但由于所有程序都能读取此文件，非常容易造成用户数据被窃取。 虽然密码是加密的，但是采用暴力破解的方式也是能够进行破解的。

因此，现在 Linux 系统把真正的加密密码串放置在 /etc/shadow 文件中，此文件只有 root 用户可以浏览和操作，这样就最大限度地保证了密码的安全。

需要注意的是，虽然 "x" 并不表示真正的密码，但也不能删除，如果删除了 "x"，那么系统会认为这个用户没有密码，从而导致只输入用户名而不用输入密码就可以登陆（只能在使用无密码登录，远程是不可以的），除非特殊情况（如破解用户密码），这当然是不可行的。

**3). UID**
UID，也就是用户 ID。每个用户都有唯一的一个 UID，Linux 系统通过 UID 来识别不同的用户。

实际上，UID 就是一个 0~65535 之间的数，不同范围的数字表示不同的用户身份，具体如表 1 所示。

|UID范围|用户身份|
|--- |--- |
|0|超级用户。UID 为 0 就代表这个账号是管理员账号。在 Linux 中，如何把普通用户升级成管理员呢？只需把其他用户的 UID 修改为 0 就可以了，这一点和 Windows 是不同的。不过不建议建立多个管理员账号。|
|1-499|系统用户（伪用户）。也就是说，此范围的 UID 保留给系统使用。其中，1~99 用于系统自行创建的账号；100~499 分配给有系统账号需求的用户。其实，除了 0 之外，其他的 UID 并无不同，这里只是默认 500 以下的数字给系统作为保留账户，只是一个公认的习惯而已。|
|500-65535|普通用户。通常这些 UID 已经足够用户使用了。但不够用也没关系，2.6.x 内核之后的 Linux 系统已经可以支持 232 个 UID 了|

**4). GID**
全称“Group ID”，简称“组ID”，表示用户初始组的组 ID 号。这里需要解释一下初始组和附加组的概念。

初始组，指用户登陆时就拥有这个用户组的相关权限。每个用户的初始组只能有一个，通常就是将和此用户的用户名相同的组名作为该用户的初始组。比如说，我们手工添加用户 lamp，在建立用户 lamp 的同时，就会建立 lamp 组作为 lamp 用户的初始组。

附加组，指用户可以加入多个其他的用户组，并拥有这些组的权限。每个用户只能有一个初始组，除初始组外，用户再加入其他的用户组，这些用户组就是这个用户的附加组。附加组可以有多个，而且用户可以有这些附加组的权限。

举例来说，刚刚的 lamp 用户除属于初始组 lamp 外，我又把它加入了 users 组，那么 lamp 用户同时属于 lamp 组和 users 组，其中 lamp 是初始组，users 是附加组。

当然，初始组和附加组的身份是可以修改的，但是我们在工作中不修改初始组，只修改附加组，因为修改了初始组有时会让管理员逻辑混乱。

需要注意的是，在 /etc/passwd 文件的第四个字段中看到的 ID 是这个用户的初始组。

**5). 描述性信息**
这个字段并没有什么重要的用途，只是用来解释这个用户的意义而已。

**6). 主目录**
也就是用户登录后有操作权限的访问目录，通常称为用户的主目录。

例如，root 超级管理员账户的主目录为 /root，普通用户的主目录为 /home/yourIDname，即在 /home/ 目录下建立和用户名相同的目录作为主目录，如 lamp 用户的主目录就是 /home/lamp/ 目录。

**7). 默认的Shell**
Shell 就是 Linux 的命令解释器，是用户和 Linux 内核之间沟通的桥梁。

我们知道，用户登陆 Linux 系统后，通过使用 Linux 命令完成操作任务，但系统只认识类似 0101 的机器语言，这里就需要使用命令解释器。也就是说，Shell 命令解释器的功能就是将用户输入的命令转换成系统可以识别的机器语言。

通常情况下，Linux 系统默认使用的命令解释器是 bash（/bin/bash），当然还有其他命令解释器，例如 sh、csh 等。

## 5. /etc/shadow内容解释
/etc/shadow 文件，用于存储 Linux 系统中用户的密码信息，又称为“影子文件”。

前面介绍了 /etc/passwd 文件，由于该文件允许所有用户读取，易导致用户密码泄露，因此 Linux 系统将用户的密码信息从 /etc/passwd 文件中分离出来，并单独放到了此文件中。

/etc/shadow 文件只有 root 用户拥有读权限，其他用户没有任何权限，这样就保证了用户密码的安全性。 注意，如果这个文件的权限发生了改变，则需要注意是否是恶意攻击。

介绍此文件之前，我们先打开看看，执行如下命令：
```sh
[root@localhost ~]#vim /etc/shadow
root: $6$9w5Td6lg
$bgpsy3olsq9WwWvS5Sst2W3ZiJpuCGDY.4w4MRk3ob/i85fl38RH15wzVoom ff9isV1 PzdcXmixzhnMVhMxbvO:15775:0:99999:7:::
bin:*:15513:0:99999:7:::
daemon:*:15513:0:99999:7:::
…省略部分输出…
```

同 /etc/passwd 文件一样，文件中每行代表一个用户，同样使用 ":" 作为分隔符，不同之处在于，每行用户信息被划分为 9 个字段。每个字段的含义如下： 用户名：加密密码：最后一次修改时间：最小修改时间间隔：密码有效期：密码需要变更前的警告天数：密码过期后的宽限时间：账号失效时间：保留字段

接下来，给大家分别介绍这 9 个字段。

**1). 用户名**
同 /etc/passwd 文件的用户名有相同的含义。

**2). 加密密码**
这里保存的是真正加密的密码。目前 Linux 的密码采用的是 SHA512 散列加密算法，原来采用的是 MD5 或 DES 加密算法。SHA512 散列加密算法的加密等级更高，也更加安全。

注意，这串密码产生的乱码不能手工修改，如果手工修改，系统将无法识别密码，导致密码失效。很多软件透过这个功能，在密码串前加上 "!"、"*" 或 "x" 使密码暂时失效。

所有伪用户的密码都是 "!!" 或 "*"，代表没有密码是不能登录的。当然，新创建的用户如果不设定密码，那么它的密码项也是 "!!"，代表这个用户没有密码，不能登录。

**3). 最后一次修改时间**
此字段表示最后一次修改密码的时间，可是，为什么 root 用户显示的是 15775 呢？

这是因为，Linux 计算日期的时间是以 1970 年 1 月 1 日作为 1 不断累加得到的时间，到 1971 年 1 月 1 日，则为 366 天。这里显示 15775 天，也就是说，此 root 账号在 1970 年 1 月 1 日之后的第 15775 天修改的 root 用户密码。

那么，到底 15775 代表的是哪一天呢？可以使用如下命令进行换算：
```sh
[root@localhost ~]# date -d "1970-01-01 15775 days"
2013年03月11日 星期一 00:00:00 CST
```

可以看到，通过以上命令，即可将其换算为我们习惯的系统日期。

**4). 最小修改时间间隔**
最小修改间隔时间，也就是说，该字段规定了从第 3 字段（最后一次修改密码的日期）起，多长时间之内不能修改密码。如果是 0，则密码可以随时修改；如果是 10，则代表密码修改后 10 天之内不能再次修改密码。

此字段是为了针对某些人频繁更改账户密码而设计的。

**5). 密码有效期**
经常变更密码是个好习惯，为了强制要求用户变更密码，这个字段可以指定距离第 3 字段（最后一次更改密码）多长时间内需要再次变更密码，否则该账户密码进行过期阶段。 该字段的默认值为 99999，也就是 273 年，可认为是永久生效。如果改为 90，则表示密码被修改 90 天之后必须再次修改，否则该用户即将过期。管理服务器时，通过这个字段强制用户定期修改密码。

**6). 密码需要变更前的警告天数**
与第 5 字段相比较，当账户密码有效期快到时，系统会发出警告信息给此账户，提醒用户 "再过 n 天你的密码就要过期了，请尽快重新设置你的密码！"。

该字段的默认值是 7，也就是说，距离密码有效期的第 7 天开始，每次登录系统都会向该账户发出 "修改密码" 的警告信息。

**7). 密码过期后的宽限天数**
也称为“口令失效日”，简单理解就是，在密码过期后，用户如果还是没有修改密码，则在此字段规定的宽限天数内，用户还是可以登录系统的；如果过了宽限天数，系统将不再让此账户登陆，也不会提示账户过期，是完全禁用。

比如说，此字段规定的宽限天数是 10，则代表密码过期 10 天后失效；如果是 0，则代表密码过期后立即失效；如果是 -1，则代表密码永远不会失效。

**8). 账号失效时间**
同第 3 个字段一样，使用自 1970 年 1 月 1 日以来的总天数作为账户的失效时间。该字段表示，账号在此字段规定的时间之外，不论你的密码是否过期，都将无法使用！

该字段通常被使用在具有收费服务的系统中。

**9). 保留**
这个字段目前没有使用，等待新功能的加入。

## 6. /etc/group解释
/ect/group 文件是用户组配置文件，即用户组的所有信息都存放在此文件中。

此文件是记录组 ID（GID）和组名相对应的文件。前面讲过，etc/passwd 文件中每行用户信息的第四个字段记录的是用户的初始组 ID，那么，此 GID 的组名到底是什么呢？就要从 /etc/group 文件中查找。

/etc/group 文件的内容可以通过 Vim 看到：
```sh
[root@localhost ~]#vim /etc/group
root:x:0:
bin:x:1:bin,daemon
daemon:x:2:bin,daemon
…省略部分输出…
lamp:x:502:
```

可以看到，此文件中每一行各代表一个用户组。在前面章节中，我们曾创建 lamp 用户，系统默认生成一个 lamp 用户组，在此可以看到，此用户组的 GID 为 502，目前它仅作为 lamp 用户的初始组。

各用户组中，还是以 "：" 作为字段之间的分隔符，分为 4 个字段，每个字段对应的含义为： 组名：密码：GID：该用户组中的用户列表

接下来，分别介绍各个字段具体的含义。

**1). 组名**
也就是是用户组的名称，有字母或数字构成。同 /etc/passwd 中的用户名一样，组名也不能重复。

**2). 组密码**
和 /etc/passwd 文件一样，这里的 "x" 仅仅是密码标识，真正加密后的组密码默认保存在 /etc/gshadow 文件中。

不过，用户设置密码是为了验证用户的身份，那用户组设置密码是用来做什么的呢？用户组密码主要是用来指定组管理员的，由于系统中的账号可能会非常多，root 用户可能没有时间进行用户的组调整，这时可以给用户组指定组管理员，如果有用户需要加入或退出某用户组，可以由该组的组管理员替代 root 进行管理。但是这项功能目前很少使用，我们也很少设置组密码。如果需要赋予某用户调整某个用户组的权限，则可以使用 sudo 命令代替。

**3). 组ID (GID)**
就是群组的 ID 号，Linux 系统就是通过 GID 来区分用户组的，同用户名一样，组名也只是为了便于管理员记忆。

这里的组 GID 与 /etc/passwd 文件中第 4 个字段的 GID 相对应，实际上，/etc/passwd 文件中使用 GID 对应的群组名，就是通过此文件对应得到的。

**4). 组中的用户**
此字段列出每个群组包含的所有用户。需要注意的是，如果该用户组是这个用户的初始组，则该用户不会写入这个字段，可以这么理解，该字段显示的用户都是这个用户组的附加用户。

举个例子，lamp 组的组信息为 "lamp:x:502:"，可以看到，第四个字段没有写入 lamp 用户，因为 lamp 组是 lamp 用户的初始组。如果要查询这些用户的初始组，则需要先到 /etc/passwd 文件中查看 GID（第四个字段），然后到 /etc/group 文件中比对组名。

每个用户都可以加入多个附加组，但是只能属于一个初始组。所以我们在实际工作中，如果需要把用户加入其他组，则需要以附加组的形式添加。例如，我们想让 lamp 也加入 root 这个群组，那么只需要在第一行的最后一个字段加入 lamp，即 root:x:0:lamp 就可以了。

一般情况下，用户的初始组就是在建立用户的同时建立的和用户名相同的组。

到此，我们已经学习了/etc/passwd、/etc/shadow、/etc/group，它们之间的关系可以这样理解，即先在 /etc/group 文件中查询用户组的 GID 和组名；然后在 /etc/passwd 文件中查找该 GID 是哪个用户的初始组，同时提取这个用户的用户名和 UID；最后通过 UID 到 /etc/shadow 文件中提取和这个用户相匹配的密码。

## 7. /etc/gshadow解释
前面讲过，/etc/passwd 文件存储用户基本信息，同时考虑到账户的安全性，将用户的密码信息存放另一个文件 /etc/shadow 中。本节要将的 /etc/gshadow 文件也是如此，组用户信息存储在 /etc/group 文件中，而将组用户的密码信息存储在 /etc/gshadow 文件中。

首先，我们借助 Vim 命令查看一下此文件中的内容：

```sh
[root@localhost ~]#vim /etc/gshadow
root:::
bin:::bin, daemon
daemon:::bin, daemon
...省略部分输出...
lamp:!::
```

文件中，每行代表一个组用户的密码信息，各行信息用 ":" 作为分隔符分为 4 个字段，每个字段的含义如下： 组名：加密密码：组管理员：组附加用户列表

**1). 组名**
同 /etc/group 文件中的组名相对应。

**2). 组密码**
对于大多数用户来说，通常不设置组密码，因此该字段常为空，但有时为 "!"，指的是该群组没有组密码，也不设有群组管理员。

**3). 组管理员**
从系统管理员的角度来说，该文件最大的功能就是创建群组管理员。那么，什么是群组管理员呢？

考虑到 Linux 系统中账号太多，而超级管理员 root 可能比较忙碌，因此当有用户想要加入某群组时，root 或许不能及时作出回应。这种情况下，如果有群组管理员，那么他就能将用户加入自己管理的群组中，也就免去麻烦 root 了。

不过，由于目前有 sudo 之类的工具，因此群组管理员的这个功能已经很少使用了。

**4). 组中的附加用户**
该字段显示这个用户组中有哪些附加用户，和 /etc/group 文件中附加组显示内容相同。

## 8. 初始组和附加组

通过学习用户和群组我们知道，群组可以让多个用户具有相同的权限，同时也可以这样理解，一个用户可以所属多个群组，并同时拥有这些群组的权限，这就引出了初始组（有时也称主组）和附加组。

/etc/passwd 文件中每个用户信息分为 7 个字段，其中第 4 字段（GID）指的就是每个用户所属的初始组，也就是说，当用户一登陆系统，立刻就会拥有这个群组的相关权限。

举个例子，我们新建一个用户 lamp，并将其加入 users 群组中，执行命令如下：

```sh
[root@localhost ~]# useradd lamp  <--添加新用户
[root@localhost ~]# groupadd users  <--添加新群组
[root@localhost ~]# usermod -G users lamp  <--将用户lamp加入 users群组
[root@localhost ~]# grep "lamp" /etc/passwd /etc/group /etc/gshadow
/etc/passwd:lamp:x:501:501::/home/lamp:/bin/bash
/etc/group:users:x:100:lamp
/etc/group:lamp:x:501:
/etc/gshadow:users:::lamp
/etc/gshadow:lamp:!::
```

useradd 和 groupadd 分别是添加用户和群组的命令，后续章节会做详细讲解。

可以看到，在 etc/passwd 文件中，lamp 用户所属的 GID（群组 ID）为 501，通过搜索 /etc/group 文件得知，对应此 GID 的是 lamp 群组，也就是说，lamp 群组是 lamp 用户的初始组。

lamp 群组是添加 lamp 用户时默认创建的群组，在 root 管理员使用 useradd 命令创建新用户时，若未明确指定该命令所属的初始组，useradd 命令会默认创建一个同用户名相同的群组，作为该用户的初始组。

正因为 lamp 群组是 lamp 用户的初始组，该用户一登陆就会自动获取相应权限，因此不需要在 /etc/group 的第 4 个字段额外标注。

但是，附加组就不一样了，从例子中可以看到，我们将 lamp 用户加入 users 群组中，由于 users 这个群组并不是 lamp 的初始组，因此必须要在 /etc/group 这个文件中找到 users 那一行，将 lamp 这个用户加入第 4 段中（群组包含的所有用户），这样 lamp 用户才算是真正加入到 users 这个群组中。

在这个例子中，因为 lamp 用户同时属于 lamp 和users 两个群组，所在，在读取\写入\运行文件时，只要是 user 和 lamp 群组拥有的功能，lamp 用户都拥有。

一个用户可以所属多个附加组，但只能有一个初始组。那么，如何知道某用户所属哪些群组呢？使用 groups 命令即可。

例如，我们现在以 lamp 用户的身份登录系统，通过执行如下命令即可知晓当前用户所属的全部群组：

```sh
[root@localhost ~]# groups
lamp users
```

通过以上输出信息可以得知，lamp 用户同时属于 lamp 群组和 users 群组，而且，第一个出现的为用户的初始组，后面的都是附加组，所以 lamp 用户的初始组为 lamp 群组，附加组为 users 群组。

## 9. useradd命令
Linux 系统中，可以使用 useradd 命令新建用户，此命令的基本格式如下：

```sh
[root@localhost ~]#useradd [选项] 用户名
```
|选项|含义|
|--- |--- |
|-u UID|手工指定用户的 UID，注意 UID 的范围（不要小于 500）。|
|-d 主目录|手工指定用户的主目录。主目录必须写绝对路径，而且如果需要手工指定主目录，则一定要注意权限；|
|-c 用户说明|手工指定/etc/passwd文件中各用户信息中第 5 个字段的描述性内容，可随意配置；|
|-g 组名|手工指定用户的初始组。一般以和用户名相同的组作为用户的初始组，在创建用户时会默认建立初始组。一旦手动指定，则系统将不会在创建此默认的初始组目录。|
|-G 组名|指定用户的附加组。我们把用户加入其他组，一般都使用附加组；|
|-s shell|手工指定用户的登录 Shell，默认是 /bin/bash；|
|-e 曰期|指定用户的失效曰期，格式为 "YYYY-MM-DD"。也就是 /etc/shadow 文件的第八个字段；|
|-o|允许创建的用户的 UID 相同。例如，执行 "useradd -u 0 -o usertest" 命令建立用户 usertest，它的 UID 和 root 用户的 UID 相同，都是 0；|
|-m|建立用户时强制建立用户的家目录。在建立系统用户时，该选项是默认的；|
|-r|创建系统用户，也就是 UID 在 1~499 之间，供系统程序使用的用户。由于系统用户主要用于运行系统所需服务的权限配置，因此系统用户的创建默认不会创建主目录。|


其实，系统已经帮我们规定了非常多的默认值，在没有特殊要求下，无需使用任何选项即可成功创建用户。例如：
```sh
[root@localhost ~]# useradd lamp
```

此行命令就表示创建 lamp 普通用户。

## 10. passwd命令：修改用户密码

学习 useradd 命令我们知道，使用此命令创建新用户时，并没有设定用户密码，因此还无法用来登陆系统，本节就来学习 passwd 密码配置命令 。

passwd 命令的基本格式如下：

```sh

[root@localhost ~]#passwd [选项] 用户名
```

选项：
- -S：查询用户密码的状态，也就是 /etc/shadow 文件中此用户密码的内容。仅 root 用户可用；
- -l：暂时锁定用户，该选项会在 /etc/shadow 文件中指定用户的加密密码串前添加 "!"，使密码失效。仅 root 用户可用；
- -u：解锁用户，和 -l 选项相对应，也是只能 root 用户使用；
- --stdin：可以将通过管道符输出的数据作为用户的密码。主要在批量添加用户时使用；
- -n 天数：设置该用户修改密码后，多长时间不能再次修改密码，也就是修改 /etc/shadow 文件中各行密码的第 4 个字段；
- -x 天数：设置该用户的密码有效期，对应 /etc/shadow 文件中各行密码的第 5 个字段；
- -w 天数：设置用户密码过期前的警告天数，对于 /etc/shadow 文件中各行密码的第 6 个字段；
- -i 日期：设置用户密码失效日期，对应 /etc/shadow 文件中各行密码的第 7 个字段。
例如，我们使用 root 账户修改 lamp 普通用户的密码，可以使用如下命令：
```sh
[root@localhost ~]#passwd lamp
Changing password for user lamp.
New password: <==直接输入新的口令，但屏幕不会有任何反应
BAD PASSWORD: it is WAY too short <==口令太简单或过短的错误！这里只是警告信息，输入的密码依旧能用
Retype new password:  <==再次验证输入的密码，再输入一次即可
passwd: all authentication tokens updated successfully.  <==提示修改密码成功
```

当然，也可以使用 passwd 命令修改当前系统已登录用户的密码，但要注意的是，需省略掉 "选项" 和 "用户名"。例如，我们登陆 lamp 用户，并使用 passwd 命令修改 lamp 的登陆密码，执行过程如下：
```sh
[root@localhost ~]#passwd
# passwd直接回车代表修改当前用户的密码
Changing password for user vbird2.
Changing password for vbird2
(current) UNIX password: <==这里输入『原有的旧口令』
New password: <==这里输入新口令
BAD PASSWORD: it is WAY too short <==口令检验不通过，请再想个新口令
New password: <==这里再想个来输入吧
Retype new password: <==通过口令验证！所以重复这个口令的输入
passwd: all authentication tokens updated successfully. <==成功修改用户密码
```

注意，普通用户只能使用 passwd 命令修改自己的密码，而不能修改其他用户的密码。

可以看到，与使用 root 账户修改普通用户的密码不同，普通用户修改自己的密码需要先输入自己的旧密码，只有旧密码输入正确才能输入新密码。不仅如此，此种修改方式对密码的复杂度有严格的要求，新密码太短、太简单，都会被系统检测出来并禁止用户使用。 很多Linux 发行版为了系统安装，都使用了 PAM 模块进行密码的检验，设置密码太短、与用户名相同、是常见字符串等，都会被 PAM 模块检查出来，从而禁止用户使用此类密码。

而使用 root 用户，无论是修改普通用户的密码，还是修改自己的密码，都可以不遵守 PAM 模块设定的规则，就比如我刚刚给 lamp 用户设定的密码是 "123"，系统虽然会提示密码过短和过于简单，但依然可以设置成功。当然，在实际应用中，就算是 root 身份，在设定密码时也要严格遵守密码规范，因为只有好的密码规范才是服务器安全的基础。

实例
【例 1】
```sh
# 查看用户密码的状态
[root@localhost ~]# passwd -S lamp
lamp PS 2013-01-06 0 99999 7 -1 (Password set, SHA512 crypt.)
# 上面这行代码的意思依次是：用户名 密码 设定时间(2013*01-06) 密码修改间隔时间(0) 密码有效期(99999) 警告时间(7) 密码不失效(-1)，密码已使用
```

"-S"选项会显示出密码状态，这里的密码修改间隔时间、密码有效期、警告时间、密码宽限时间其实分别是 /etc/shadow 文件的第四、五、六、七个字段的内容。 当然，passwd 命令是可以通过命令选项修改这几个字段的值的，例如：
```sh
# 修改 lamp的密码，使其具有 60 天变更、10 天密码失效
[root@localhost ~]# passwd -x 60 -i 10 lamp
[root@localhost ~]# passwd -S lamp
lamp PS 2013-01-06 0 60 7 10 (Password set, SHA512 crypt.)
```

【例 2】
```sh
# 锁定 lamp 用户
[root@localhost ~]# passwd -l lamp
Locking password for user lamp.
passwd:Successg
# 用"-S"选项査看状态，很清楚地提示密码已被锁定
[root@localhost ~]# passwd -S lamp
lamp LK 2013-01-06 0 99999 7 -1 (Password locked.)
[root@localhost ~]# grep "lamp" /etc/shadow
lamp:!! $6$ZTq7o/9o $lj07iZ0bzW.D1zBa9CsY43d04onskUCzjwiFMNt8PX4GXJoHX9zA1S C9.i Yzh9LZA4fEM2lg92hM9w/p6NS50.:15711:0:99999:7:::
# 可以看到，锁定其实就是在加密密码之前加入了"!!"，让密码失效而已
```
暂时锁定 lamp 用户后，此用户就不能登录系统了。那么，怎么解锁呢？也一样简单，使用如下命令即可：

```sh
# 解锁 lamp 用户
[root@localhost ~]# passwd -u lamp
Unlocking password for user lamp.
passwd:Success
[root@localhost ~]# passwd -S lamp
lamp PS 2013-01-06 0 99999 7 -1 (Password set, SHA512 crypt.)
# 可以看到，锁定状态消失
[root@localhost ~]# grep "lamp" /etc/shadow
lamp: $6$ZTq7cV9o $lj07iZ0bzW.D1zBa9CsY43d04onskUCzjwiFMNt8PX4GXJoHX9zA1S C9.iYz h9LZA4fEM2lg92hM9w/p6NS50.:15711:0:99999:7:::
# 密码前面的 "!!" 删除了
```

## 11. usermod命令：修改用户信息
前面章节介绍了如何利用 useradd 命令添加用户，但如果不小心添错用户信息，后期如何修改呢？

办法有两个，一个是使用 Vim 文本编辑器手动修改涉及用户信息的相关文件（/etc/passwd、/etc/shadow、/etc/group、/etc/gshadow），另一个方法就是使用本节介绍了 usermod 命令，该命令专门用于修改用户信息。 这里一定要分清 useradd 命令和 usermod 命令的区别，前者用于添加用户，当然，添加用户时可以对用户信息进行定制；后者针对与已存在的用户，使用该命令可以修改它们的信息。

usermod 命令的基本格式如下：

```sh
[root@localhost ~]#usermod [选项] 用户名
```

选项：

- -c 用户说明：修改用户的说明信息，即修改 /etc/passwd 文件目标用户信息的第 5 个字段；
- -d 主目录：修改用户的主目录，即修改 /etc/passwd 文件中目标用户信息的第 6 个字段，需要注意的是，主目录必须写绝对路径；
- -e 日期：修改用户的失效曰期，格式为 "YYYY-MM-DD"，即修改 /etc/shadow 文件目标用户密码信息的第 8 个字段；
- -g 组名：修改用户的初始组，即修改 /etc/passwd 文件目标用户信息的第 4 个字段（GID）；
- -u UID：修改用户的UID，即修改 /etc/passwd 文件目标用户信息的第 3 个字段（UID）；
- -G 组名：修改用户的附加组，其实就是把用户加入其他用户组，即修改 /etc/group 文件；
- -l 用户名：修改用户名称；
- -L：临时锁定用户（Lock）；
- -U：解锁用户（Unlock），和 -L 对应；
- -s shell：修改用户的登录 Shell，默认是 /bin/bash。

如果你仔细观察会发现，其实 usermod 命令提供的选项和 useradd 命令的选项相似，因为 usermod 命令就是用来调整使用 useradd 命令添加的用户信息的。

不过，相比 useradd 命令，usermod 命令还多出了几个选项，即 -L 和 -U，作用分别与 passwd 命令的 -l 和-u 相同。需要注意的是，并不是所有的 Linux 发行版都包含这个命令，因此，使用前可以使用 man usermod 命令确定系统是否支持。

> 此命令对用户的临时锁定，同 passwd 命令一样，都是在 /etc/passwd 文件目标用户的加密密码字段前添加 "!"，使密码失效；反之，解锁用户就是将添加的 "!" 去掉。

实例
【例 1】
```sh
# 锁定用户
[root@localhost ~]# usermod -L lamp
[root@localhost ~]# grep "lamp" /etc/shadow
lamp:!$6$YrPj8g0w$ChRVASybEncU24hkYFqxREH3NnzhAVDJSQLwRwTSbcA2N8UbPD9bBKVQSky xlaMGs/Eg5AQwO.UokOnKqaHFa/:15711:0:99999:7:::
# 其实锁定就是在密码字段前加入"!"，这时lamp用户就暂时不能登录了

# 解锁用户
[root@localhost ~]# usermod -U lamp
[root@localhost ~]# grep "lamp" /etc/shadow
lamp:$6$YrPj8g0w$ChRVASybEncU24hkYFqxREH3NnzhAVDJSQLwRwTSbcA2N8UbPD9bBKVQSkyx laMGs/Eg5AQwO.UokOnKqaHFa/:15711:0:99999:7:::
# 取消了密码字段前的 "!"
```

【例 2】
```sh
# 把lamp用户加入root组
[root@localhost ~]# usermod -G root lamp
[root@localhost ~]# grep "lamp" /etc/group
root:x:0:lamp
# lamp用户已经加入了root组
lamp:x:501:
```

【例 3】
```sh
# 修改用户说明
[root@localhost ~]# usermod -c "test user" lamp
[root@localhost ~]# grep "lamp" /etc/passwd
lamp:x:501:501:test user:/home/lamp:/bin/bash
# 查看一下，用户说明已经被修改了
```

## 12. chage命令：修改用户密码状态
除了 passwd -S 命令可以查看用户的密码信息外，还可以利用 chage 命令，它可以显示更加详细的用户密码信息，并且和 passwd 命令一样，提供了修改用户密码信息的功能。 如果你要修改用户的密码信息，我个人建议，还是直接修改 /etc/shadow 文件更加方便。

首先，我们来看 chage 命令的基本格式：
```sh
[root@localhost ~]#chage [选项] 用户名
```
选项：

- -l：列出用户的详细密码状态;
- -d 日期：修改 /etc/shadow 文件中指定用户密码信息的第 3 个字段，也就是最后一次修改密码的日期，格式为 YYYY-MM-DD；
- -m 天数：修改密码最短保留的天数，也就是 /etc/shadow 文件中的第 4 个字段；
- -M 天数：修改密码的有效期，也就是 /etc/shadow 文件中的第 5 个字段；
- -W 天数：修改密码到期前的警告天数，也就是 /etc/shadow 文件中的第 6 个字段；
- -i 天数：修改密码过期后的宽限天数，也就是 /etc/shadow 文件中的第 7 个字段；
- -E 日期：修改账号失效日期，格式为 YYYY-MM-DD，也就是 /etc/shadow 文件中的第 8 个字段。

【例 1】
```sh
# 查看一下用户密码状态
[root@localhost ~]# chage -l lamp
Last password change:Jan 06, 2013
Password expires:never
Password inactive :never
Account expires :never
Minimum number of days between password change :0
Maximum number of days between password change :99999
Number of days of warning before password expires :7
```

## 13. userdel命令：删除用户
userdel 命令功能很简单，就是删除用户的相关数据。此命令只有 root 用户才能使用。

通过前面的学习我们知道，用户的相关数据包含如下几项：

用户基本信息：存储在 /etc/passwd 文件中；
用户密码信息：存储在 /etc/shadow 文件中；
用户群组基本信息：存储在 /etc/group 文件中；
用户群组信息信息：存储在 /etc/gshadow 文件中；
用户个人文件：主目录默认位于 /home/用户名，邮箱位于 /var/spool/mail/用户名。
其实，userdel 命令的作用就是从以上文件中，删除与指定用户有关的数据信息。
userdel 命令的语法很简单，基本格式如下：
```sh

[root@localhost ~]# userdel -r 用户名
```
- -r 选项表示在删除用户的同时删除用户的家目录。

注意，在删除用户的同时如果不删除用户的家目录，那么家目录就会变成没有属主和属组的目录，也就是垃圾文件。

例如，删除前面章节中创建的 lamp 用户，只需执行如下命令：

```sh
[root@localhost ~]# userdel -r lamp
```
手动删除指定用户的具体操作如下
```sh
# 建立新 lamp 用户
[root@localhost ~]# useradd lamp
[root@localhost ~]# passwd lamp
# 为 lamp 用户设置密码，由此 lamp 用户才算是创建成功
# 下面开始手动删除 lamp
[root@localhost ~]# vi /etc/passwd
lamp:x:501:501::/home/lamp:/bin/bash   <--删除此行
# 修改用户信息文件，删除lamp用户行
[root@localhost ~]#vi /etc/shadow
lamp:$6$KoOYtcOJ $56Xk9vp3D2vMRBxibNOn.21cVJ9onbW8IHx4WrOx6qBqfGa9U3mjMsGjqYnj L/4t3zt3YxElce2X8rbb12x4a0:15716:0:99999:7:::   <--删除此行
# 修改影子文件，删除lamp用户密码行，注意，这个文件的权限是000，所以要强制保存
[root@localhost ~]#vi /etc/group
lamp:x:501:  <--删除此行
# 修改组信息文件，删除lamp群组信息
[root@localhost ~]#vi /etc/gshadow
lamp:!::  <--删除此行
# 修改组影子文件，删除lamp群组密码信息。同样注意需要强制保存
[root@localhost ~]# rm -rf /var/spod/mail/lamp  #删除用户邮箱
[root@localhost ~]# rm -rf/home/lamp/  #删除用户的家目录
# 至此，用户彻底删除，再新建用户lamp。如果可以正常建立，则说明我们手工删除干净了
[root@localhost ~]# useradd lamp
[root@localhost ~]# passwd lamp
# 重新建立同名用户，没有报错，说明前面的手工删除是可以完全删除用户的
```

## 14. id命令：查看用户的UID和GID
id 命令可以查询用户的UID、GID 和附加组的信息。命令比较简单，格式如下：
```sh
[root@localhost ~]# id 用户名
```

【例 1】
```sh
[root@localhost ~]# id lamp
uid=501(lamp) gid=501(lamp) groups=501(lamp)
# 能看到uid(用户ID)、gid(初始组ID), groups是用户所在组，这里既可以看到初始组，如果有附加组，则也能看到附加组
【例 2】

[root@localhost ~]# usermod -G root lamp
# 把用户加入root组
[root@localhost ~]# id lamp
uid=501(lamp) gid=501(lamp) groups=501(lamp),0(root)
# 大家发现root组中加入了lamp用户的附加组信息
```

## 15. su命令：用户间切换
su 是最简单的用户切换命令，通过该命令可以实现任何身份的切换，包括从普通用户切换为 root 用户、从 root 用户切换为普通用户以及普通用户之间的切换。 普通用户之间切换以及普通用户切换至 root 用户，都需要知晓对方的密码，只有正确输入密码，才能实现切换；从 root 用户切换至其他用户，无需知晓对方密码，直接可切换成功。

su 命令的基本格式如下：
```sh
[root@localhost ~]# su [选项] 用户名
```
选项：

- -：当前用户不仅切换为指定用户的身份，同时所用的工作环境也切换为此用户的环境（包括 PATH 变量、MAIL 变量等），使用 - 选项可省略用户名，默认会切换为 root 用户。
- -l：同 - 的使用类似，也就是在切换用户身份的同时，完整切换工作环境，但后面需要添加欲切换的使用者账号。
- -p：表示切换为指定用户的身份，但不改变当前的工作环境（不使用切换用户的配置文件）。
- -m：和 -p 一样；
- -c 命令：仅切换用户执行一次命令，执行后自动切换回来，该选项后通常会带有要执行的命令。

【例 1】
```sh
[lamp@localhost ~]$ su -root
密码： <-- 输入 root 用户的密码
# "-"代表连带环境变量一起切换，不能省略
```

## 16. groupadd命令：添加用户组
添加用户组的命令是 groupadd，命令格式如下:

```sh
[root@localhost ~]# groupadd [选项] 组名
```
选项：

- -g GID：指定组 ID；
- -r：创建系统群组。
使用 groupadd 命令创建新群组非常简单，例如：

```sh
[root@localhost ~]# groupadd group1
# 添加group1组
[root@localhost ~]# grep "group1" /etc/group
/etc/group:group1:x:502:
/etc/gshadow:group1:!::
```

## 17. groupmod命令：修改用户组
groupmod 命令用于修改用户组的相关信息，命令格式如下：
```sh
[root@localhost ~]# groupmod [选现] 组名
```

选项：
- -g GID：修改组 ID；
- -n 新组名：修改组名；

例子：
```sh
[root@localhost ~]# groupmod -n testgrp group1
# 把组名group1修改为testgrp
[root@localhost ~]# grep "testgrp" /etc/group
testgrp:x:502:
# 注意GID还是502，但是组名已经改变
```

不过大家还是要注意，用户名不要随意修改，组名和 GID 也不要随意修改，因为非常容易导致管理员逻辑混乱。如果非要修改用户名或组名，则建议大家先删除旧的，再建立新的。

## 18. groupdel命令：刪除用户组
groupdel 命令用于删除用户组（群组），此命令基本格式为：
```sh
[root@localhost ~]#groupdel 组名
```

通过前面的学习不难猜测出，使用 groupdel 命令删除群组，其实就是删除 /etc/gourp 文件和 /etc/gshadow 文件中有关目标群组的数据信息。

例如，删除前面章节中用 groupadd 命令创建的群组 group1，执行命令如下：
```sh
[root@localhost ~]#grep "testgrp" /etc/group /etc/gshadow
/etc/group:testgrp:x:1002:
/etc/gshadow:testgrp:!::
[root@localhost ~]#groupdel testgrp
[root@localhost ~]#grep "testgrp" /etc/group /etc/gshadow
[root@localhost ~]#
```

注意，不能使用 groupdel 命令随意删除群组。此命令仅适用于删除那些 "不是任何用户初始组" 的群组，换句话说，如果有群组还是某用户的初始群组，则无法使用 groupdel 命令成功删除。例如：

```sh
[root@localhost ~]# useradd temp
# 运行如下命令，可以看到 temp 用户建立的同时，还创建了 temp 群组，且将其作为 temp用户的初始组（组ID都是 505）
[root@localhost ~]# grep "temp" /etc/passwd /etc/group /etc/gshadow
/etc/passwd:temp:x:505:505::/home/temp:/bin/bash
/etc/group:temp:x:505:
/etc/gshadow:temp:!::
# 下面尝试删除 temp 群组
[root@localhost ~]# groupdel temp
groupdel:cannot remove the primary group of user 'temp'
```

可以看到，groupdel 命令删除 temp 群组失败，且提示“不能删除 temp 用户的初始组”。如果一定要删除 temp 群组，要么修改 temp 用户的 GID，也就是将其初始组改为其他群组，要么先删除 temp 用户。

切记，虽然我们已经学了如何手动删除群组数据，但胡乱地删除群组可能会给其他用户造成不小的麻烦，因此更改文件数据要格外慎重。

## 19. gpasswd命令：把用户添加进组或从组中删除
为了避免系统管理员（root）太忙碌，无法及时管理群组，我们可以使用 gpasswd 命令给群组设置一个群组管理员，代替 root 完成将用户加入 或移出群组的操作。

gpasswd 命令的基本格式如下：
```sh
[root@localhost ~]# gpasswd 选项 组名
```

选项：
- 选项为空时，表示给群组设置密码，仅 root 用户可用。
- -A user1,... 将群组的控制权交给 user1,... 等用户管理，也就是说，设置 user1,... 等用户为群组的管理员，仅 root 用户可用。
- -M user1,... 将 user1,... 加入到此群组中，仅 root 用户可用。
- -r 移除群组的密码，仅 root 用户可用。
- -R 让群组的密码失效，仅 root 用户可用。
- -a user 将 user 用户加入到群组中。
- -d user 将 user 用户从群组中移除。
【例 1】
```sh
# 创建新群组 group1，并将群组交给 lamp 管理
[root@localhost ~]# groupadd group1  <-- 创建群组
[root@localhost ~]# gpasswd group1   <-- 设置密码吧！
Changing the password for group group1
New Password:
Re-enter new password:
[root@localhost ~]# gpasswd -A lamp group1  <==加入群组管理员为 lamp
[root@localhost ~]# grep "group1" /etc/group /etc/gshadow
/etc/group:group1:x:506:
/etc/gshadow:group1:$1$I5ukIY1.$o5fmW.cOsc8.K.FHAFLWg0:lamp:
```

可以看到，此时 lamp 用户即为 group1 群组的管理员。

【例 2】
```sh
# 以lamp用户登陆系统，并将用户 lamp 和 lamp1 加入group1群组。
[lamp@localhost ~]#gpasswd -a lamp group1
[lamp@localhost ~]#gpasswd -a lamp1 group1
[lamp@localhost ~]#grep "group1" /etc/group
group1:x:506:lamp,lamp1
```

## 20. newgrp命令：切换用户的有效组

我们知道，每个用户可以属于一个初始组（用户是这个组的初始用户），也可以属于多个附加组（用户是这个组的附加用户）。既然用户可以属于这么多用户组，那么用户在创建文件后，默认生效的组身份是哪个呢？

当然是初始用户组的组身份生效，因为初始组是用户一旦登陆就获得的组身份。也就是说，用户的有效组默认是初始组，因此所创建文件的属组是用户的初始组。那么，既然用户属于多个用户组，能不能改变用户的初始组呢？使用命令 newgrp 就可以。

newgrp 命令可以从用户的附加组中选择一个群组，作为用户新的初始组。此命令的基本格式如下：
```sh
[root@localhost ~]# newgrp 组名
```
下面，我们通过实例，讲解此命令的具体用法和功能：

首先，建立 3 个用户组 group1、group2 和 group3，命令如下：
```sh
[root@localhost ~]# groupadd group1
[root@localhost ~]# groupadd group2
[root@localhost ~]# groupadd group3
```

创建一个用户 user1，同时指定 user1 的初始组为 group1，附加组为 group2 和 group3，执行命令如下：
```sh
[root@localhost ~]# useradd -g group1 -G group2,group3 user1
# 由于指定了初始组，因此不会在创建 user1 默认群组
[root@localhost ~]# more /etc/group | grep user1
group2:x:501:user1
group3:x:502:user1
```

对用户 user1 设置密码，执行命令如下：
```sh
[root@localhost ~]# passwd user1
Changing password for user user1.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

切换至 user1 用户，通过 newgrp 切换用户组进行下列操作，读者可从中体会出 newgrp 命令的作用。
```sh
# 切换至 user1 用户
[root@localhost ~]# su - user1
[root@localhost ~]# whoami
user1
# 使用 newgrp 命令一边切换 user1 的初始组，一边创建文件
[root@localhost ~]# mkdir user1_doc
[root@localhost ~]# newgrp group2
[root@localhost ~]# mkdir user2_doc
[root@localhost ~]# newgrp group3
[root@localhost ~]# mkdir user3_doc
# 查看各文件的详细信息
[root@localhost ~]# ll
total 12
drwxr-xr-x 2 user1 group1 4096 Oct 24 01:18 user1_doc
drwxr-xr-x 2 user1 group2 4096 Oct 24 01:18 user2_doc
drwxr-xr-x 2 user1 group3 4096 Oct 24 01:19 user3_doc
```

可以看到，通过使用 newgrp 命令切换用户的初始组，所创建的文件各自属于不同的群组，这就是 newgrp 所发挥的作用，即通过切换附加组成为新的初始组，从而让用户获得使用各个附加组的权限。
