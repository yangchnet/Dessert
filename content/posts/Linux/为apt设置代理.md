---
author: "李昌"
title: "为apt设置代理"
date: "2021-09-03"
tags: ["apt", "proxy"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

## 1. 临时设置

```sh
sudo apt-get -o Acquire::http::proxy="http://127.0.0.1:8000/" update
```

## 2. 永久设置

创建`/etc/apt/apt.conf`

```sh
touch /etc/apt/apt.conf
```

写入如下内容：
```conf
Acquire::http::Proxy "http://yourproxyaddress:proxyport";
```

如果proxy需要密码，则格式如下：
```sh
Acquire::http::Proxy "http://username:password@yourproxyaddress:proxyport";
```

Reference:
https://www.jianshu.com/p/fdae9cb5181b

https://askubuntu.com/questions/257290/configure-proxy-for-apt