---
author: "李昌"
title: "使用kubeadm安装单节点Kubernetes"
date: "2021-09-04"
tags: ["kubernetes", "install"]
categories: ["Kubernetes"]
ShowToc: true
TocOpen: true
---

> 环境：ubuntu-20.04, kubernetes:v1.22.1

## 1. 安装docker
> 安装时有可能会遇到网络问题，你可以选择换源或是为apt设置代理，设置代理的方法见[这里](https://yangchnet.github.io/Dessert/posts/linux/%E4%B8%BAapt%E8%AE%BE%E7%BD%AE%E4%BB%A3%E7%90%86/)

1. 更新源镜像并安装依赖
```sh
sudo apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

```

2. 安装docker 官方GPG密钥
```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

3. 设置稳定版本
```sh
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. 安装docker
```sh
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io
```

5. 安装docker-compose(可选)

```sh
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

## 2. 安装kubectl, kubeadm, kubelet

1. 更新源镜像并安装依赖
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

2. 下载谷歌公共签名密钥
> 对于curl, 可使用 -x , --proxy <[protocol://][user:password@]proxyhost[:port]> 来设置代理
```sh
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

3. 将kubernetes增加到apt仓库
```sh
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4. 更新软件源并安装
```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 3. 使用kubeadm安装kubernetes

```sh
kubeadm init
```

> 这一步回到Google的镜像仓库拉取一些镜像，如果拉取不到或者速度过慢，可使用阿里的镜像源
> kubeadm init --image-repository=registry.aliyuncs.com/google_containers 

若这一步失败，调整设置重新启动之前，需要先
```sh
kubeadm reset
```

## 4. 遇到的坑

### 4.1 swap
要确保swap已被关闭
使用`free -m`查看swap大小
```sh
free -m
```
确保swap那一行为0

关闭swap方法
```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 4.2 cgourp-driver

docker的`cgroup driver`默认是`cgroupfs`，需要更改为`systemed`

使用`docker info`命令查看：
![20210904092149](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210904092149.png)

如果显示的是`cgroupfs`，那么需要进行更改

改动方法：
创建`/etc/docker/daemon.json`，编辑内容为：
```json
{
    "exec-opts": ["native.cgroupdriver=systemd"]
}
```

然后重启docker服务：
```sh
systemctl restart docker
```

再次重试：
```sh
kubeadm reset

kubeadm init
```

### 4.3 阿里云镜像tag不对
解决方法是拉取最新版本的镜像，然后更改其tag
```sh
docker pull registry.aliyuncs.com/google_container/coredns:latest
docker tag registry.aliyuncs.com/google_container/coredns:latest registry.aliyuncs.com/google_container/coredns:v1.8.4
docker rmi registry.aliyuncs.com/google_container/coredns:latest
```

### 4.4 node NotReady
安装完成后，使用`kubectl get nodes`查看发现状态为`NotReady`

使用`kubectl describe ndoes`查看报错信息，有如下输出，可以看到问题是network plugin is not ready: cni config uninitialized

```
...
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sat, 04 Sep 2021 10:16:25 +0800   Sat, 04 Sep 2021 09:15:45 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sat, 04 Sep 2021 10:16:25 +0800   Sat, 04 Sep 2021 09:15:45 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sat, 04 Sep 2021 10:16:25 +0800   Sat, 04 Sep 2021 09:15:45 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Sat, 04 Sep 2021 10:16:25 +0800   Sat, 04 Sep 2021 09:15:45 +0800   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
...
```

解决方法： 安装weave  
根据https://www.weave.works/docs/net/latest/kubernetes/kube-addon/，只需运行如下命令:
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

再次查看nodes
```sh
kubectl get nodes
```
状态为Ready

### 4.5 1 node(s) had taints that the pod didn't tolerate:
这个问题是在创建pod时出现的

这是因为kubernetes默认不许往master安装，强制允许
```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
```


