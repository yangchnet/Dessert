---
author: "李昌"
title: "WSL2-不输Mac的开发体验（三）：WSL2中使用docker&k8s"
date: "2021-08-12"
tags: ["wsl", "docker", "k8s", "windows"]
categories: ["Windows"]
ShowToc: true
TocOpen: true
---


## 1. docker for wsl2

在wsl2中使用docker的最佳实践不是在wsl2中安装docker，而是安装`docker desktop`：  
![20210813164719](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813164719.png)

从docker官网下载并安装完成后，打开`docker desktop`，选择`setting->General`，确保`Use the WSL 2 based engine`选项被勾选，然后选择右下角`Apply&Restart`。

![20210813165210](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813165210.png)

重启`docker desktop`后，再次打开设置，确保`setting->Resources->WSL INTEGRATION`选项页中你的WSL发行版被勾选。

![20210813165525](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813165525.png)

完成以上步骤之后，打开你的wsl, 输入docker： 
![20210813165705](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813165705.png)

出现这一堆说明安装成功。

使用`docker run helloworld`验证你的docker可以正常启动容器。

![20210813170109](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813170109.png)

> 如果输入`docker`命令后无法启动，可以尝试`sudo docker`

## 2. k8s for wsl2

安装了`docker desktop`后，可以通过`setting->Kubernetes`，勾选`Enable Kubernetes`来为你的wsl提供k8s服务，但由于网络问题，通常不可能成功。

所以我们要"换源"。

打开`setting->Docker Engine`，将右侧配置文件改为：

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://registry.docker-cn.com"
  ],
  "insecure-registries": [],
  "debug": false,
  "experimental": false,
  "features": {
    "buildkit": true
  }
}
```

![20210813170651](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813170651.png)

`Apply&Restart`，重启`docker desktop`。

现在我们还需要一些额外的镜像。
clone [AliyunContainerService/k8s-for-docker-desktop](https://github.com/AliyunContainerService/k8s-for-docker-desktop) 这个项目。
```bash
git clone https://github.com/AliyunContainerService/k8s-for-docker-desktop.git
```

查看自己的`docker desktop`上`Kubernetes`的版本。

![20210813171034](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813171034.png)

可以看到我们这里是`v1.21.2`。相应的，我们进入刚才clone的文件夹下，切换到`v1.21.2`分支
```bash
git checkout v1.21.2
```

切换分支后，在当前目录下执行：
```powershell
.\load_images.ps1
```

> 如果因为安全策略无法执行 PowerShell 脚本，请在 “以管理员身份运行” 的 PowerShell 中执行 `Set-ExecutionPolicy RemoteSigned` 命令

最后一步，`setting->Kubernetes` 确保`Enable Kubernetes`被勾选，然后`Apply&Restart`，这时候你的`docker desktop`左下角会出现k8s的图标，并逐渐从黄色变成绿色，代表你的k8s环境启动成功。

![20210813171606](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210813171606.png)

## References

[Get started with Docker remote containers on WSL 2](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-containers)
[Docker Desktop WSL 2 backend](https://docs.docker.com/docker-for-windows/wsl/)
[AliyunContainerService/k8s-for-docker-desktop](https://github.com/AliyunContainerService/k8s-for-docker-desktop)
[在 Docker Desktop 中启用 K8s 服务](https://www.cnblogs.com/danvic712/p/enable-k8s-in-docker-desktop.html)


