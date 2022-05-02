---
author: "李昌"
title: "使用sealyun安装k8s"
date: "2022-05-01"
tags: ["k8s"]
categories: ["k8s"]
ShowToc: true
TocOpen: true
---

## 1. 准备节点

可以使用云服务器，这里我们选择vagrant搭建本地虚拟机节点。

安装vagrant(需要提前安装virtualbox)
```sh
yay -S vagrant
```

创建虚拟机
```sh
mkdir ~/vagrant

cd ~/vagrant && mkdir 0.2 0.3 0.4

cd ~/vagrant/0.2

vagrant init bento/centos-7.7
```

这会在`~/vagrant/0.2`下创建`Vagrantfile`，其内容大致为：（省略注释）
```vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
  config.vm.box = "bento/centos-7.7"
end
```

修改`vagrantfile`:
```vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "bento/centos-7.7"

  # 设置虚拟机ip地址
  config.vm.network "private_network", ip: "192.168.0.2"
  # 设置虚拟机host
  config.vm.hostname = "node.02"

   config.vm.provider "virtualbox" do |vb|
     # Display the VirtualBox GUI when booting the machine
     vb.gui = false

     # Customize the amount of memory on the VM:
     vb.memory = "2048"
     vb.cpus = 2
   end

end
```

复制`~/vagrant/0.2`中的Vagrantfile到`~/vagrant/0.3`, `~/vagrant/0.4`
```sh
cp ~/vagrant/0.2/Vagrantfile ~/vagrant/0.3/Vagrantfile
cp ~/vagrant/0.2/Vagrantfile ~/vagrant/0.4/Vagrantfile
```

修改`~/vagrant/0.3/Vagrantfile`中的虚拟机ip地址为：`192.168.0.3`

修改`~/vagrant/0.4/Vagrantfile`中的虚拟机ip地址为：`192.168.0.4`

分别在3个文件夹中启动虚拟机。
```sh
cd ~/vagrant/0.2 && vagrant up
cd ~/vagrant/0.3 && vagrant up
cd ~/vagrant/0.4 && vagrant up
```

## 2. 使用sealyun安装k8s

安装`sealyun`
```sh
# 下载并安装sealos, sealos是个golang的二进制工具，直接下载拷贝到bin目录即可, release页面也可下载
wget -c https://sealyun-home.oss-cn-beijing.aliyuncs.com/sealos/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin 
```

下载k8s资源包
```sh
wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/05a3db657821277f5f3b92d834bbaf98-v1.22.0/kube1.22.0.tar.gz
```

安装K8s
```sh
# 安装一个三master的kubernetes集群
sealos init --passwd 'vagrant' \
	--master 192.168.0.2 \
	--node 192.168.0.3 --node 192.168.0.4 \
	--pkg-url /path/to/kube1.22.0.tar.gz \
	--version v1.22.0
```
这里注意虚拟机密码和k8s资源包地址,同时需要在宿主机上使用root用户

注意事项：
- 必须同步所有服务器时间
- 所有服务器主机名不能重复
- 系统支持：centos7.6以上（其中centos8不支持） ubuntu16.04以上
- 内核推荐4.14以上， 系统推荐：centos7.7

更多细节参考：https://www.sealyun.com/instructions/1

