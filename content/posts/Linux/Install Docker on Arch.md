---
author: "李昌"
title: "Install Docker on Arch"
date: "2023-02-21"
tags: ["Linux", "Docker"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

> 官方给出的教程是使用docker desktop, 但内存占用太大。


## 1. 安装docker二进制文件

从这里下载平台对应的二进制文件：https://download.docker.com/linux/static/stable/

然后解压安装：
```bash
$ tar xzvf /path/to/<FILE>.tar.gz

$ sudo cp docker/* /usr/bin/
```

> 这时可以手动执行`sudo dockerd &`以启动守护进程，但我们不这样做

## 配置守护进程

从https://github.com/moby/moby/tree/master/contrib/init/systemd拷贝`docker.service`, `docker.socket`两个文件到`/etc/systemd/system/docker.service`, `/etc/systemd/system/docker.socket`


然后运行：
```bash
$ sudo systemctl daemon-reload # 重新加载配置

$ sudo systemctl start docker #启动dockerd

$ sudo systemctl enable docker # 设置开机启动
```

> 其实也可以直接`yay -S docker` :-)

## docker-compose

```bash
$ yay -S docker-compose
```

## References

https://docs.docker.com/desktop/install/archlinux/

https://docs.docker.com/engine/install/binaries/#install-daemon-and-client-binaries-on-linux

https://docs.docker.com/config/daemon/systemd/

https://wiki.archlinux.org/title/docker

https://wiki.archlinux.org/title/Systemd#Drop-in_files


