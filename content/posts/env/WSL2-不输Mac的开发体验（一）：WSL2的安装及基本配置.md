---
author: "李昌"
title: "WSL2-不输Mac的开发体验（一）：WSL2的安装及基本配置"
date: "2021-06-01"
tags: ["windows", "wsl", "env"]
categories: ["Windows"]
ShowToc: true
TocOpen: true
---

## 1. WSL的安装

### 1.1 升级Windows
WSL需要高版本的windows，可使用微软官方的`易升`工具或直接从设置中升级，升级需要一定的时间。

### 1.2 安装WSL
1. 使用管理员模式打开`power shell`， 使用如下命令开启WSL功能
```shell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```
**重启你的机器**


2. 启用虚拟机功能
以管理员身份打开`powershell`，使用如下命令：
```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```
**重新启动电脑**

3. 下载Linux内核更新包
[适用于 x64 计算机的 WSL2 Linux 内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)

运行你下载的更新包。

4. 将WSL2设置为默认版本
以管理员身份打开`powershell`，使用如下命令：
```powershell
wsl --set-default-version 2
```

5. 选择你要安装的发行版

![20210601184148](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601184148.png)
这里我选择了Ubuntu18.04，获取后安装


6. 启动安装的发行版即可


## 2. 使用WSL图形界面(可选)

1. 设置环境变量
```bash
export DISPLAY=$(awk '/nameserver / {print $2; exit}' /etc/resolv.conf 2>/dev/null):0
export LIBGL_ALWAYS_INDIRECT=1
```


2. 安装Xserver，这里选择的软件是`vcxsrv`， 可在[sourceforge](https://sourceforge.net/projects/vcxsrv/)中下载安装。

3. 安装完成后直接启动即可  
> 每次重新启动电脑都要重新打开一下，略烦。微软官方的WSLg已经发布，不过需要Windows Insider,先等等吧
   
![20210601184622](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601184622.png)

![20210601184640](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601184640.png)

![20210601184659](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601184659.png)

4. 在wsl中安装`gedit`
```bash
sudo apt install gedit
```

尝试启动gedit：
```bash
gedit
```

![20210601184847](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601184847.png)

完成！！！


> 可以从wsl中启动图形界面意味着你可以将vscode、idea、goland等开发工具直接装在wsl中。但我强烈建议你使用vscode的`Remote-WSL`插件来进行开发，而只在一些必要的时候才从wsl中直接启动图形界面。


## 3. WSL中文(如未安装GUI，则不需要配置)

### 3.1 WSL中文显示
1. 安装语言包
```bash
sudo apt-get -y install locales xfonts-intl-chinese fonts-wqy-microhei  
```

2. 语言环境设置
```bash
sudo dpkg-reconfigure locales
```
使用空格键选中en_US.UTF-8 UTF-8、zh_CN.UTF-8 UTF-8，使用Tab键切换至OK，再将en_US.UTF-8选为默认。
![20210615144553](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210615144553.png)

3. 重启系统
```bash
exit
...
```
```shell
wsl --shutdown

wsl
```

### 3.2 WSL中文输入法配置(如未安装GUI，则不需要配置)

1. 安装fcitx及输入法   
安装fcitx核心及CJK字体  
```bash
sudo apt install fcitx fonts-noto-cjk fonts-noto-color-emoji dbus-x11
```

安装搜狗输入法，直接到搜狗输入法官网网站下载并按照说明安装即可

2. 配置输入环境
   
首先使用root账号生成dbus机器码
```bash
dbus-uuidgen > /var/lib/dbus/machine-id
```

用**root账号**创建`/etc/profile.d/fcitx.sh`文件，内容如下：
```bash
#!/bin/bash
export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export DefaultIMModule=fcitx

#可选，fcitx 自启
fcitx-autostart &>/dev/null
```
3. 其他配置
将以下内容添加到你的`bashrc`配置文件中
```bash
vim ~/.bashrc

export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export DefaultIMModule=fcitx
fcitx-autostart &>/dev/null
```

```bash
source ~/.bashrc
```
运行`fcitx-config-gtk3`,会出现如图的界面：  
![20210601182023](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601182023.png)
按照提示将键盘布局放在第一位，输入法放在第二位。  

为防止和Windows上的输入法切换快捷键冲突，这里将快捷键切换更改为`ctrl+shift`(Windows上为`shift`)  
![20210601182148](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601182148.png)

到这里，配置就完成了。

4. 开始输入  
![20210601182727](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210601182727.png)

## 4. WSL 开发环境备份
可以使用`wsl --export <DistributionName> <fileName>`， 例如，我们使用`Ubuntu18.04`的Linux版本，那么使用如下命令导出：
```shell
# --export <分发版> <文件名>
wsl --export Ubuntu18.04 ubuntuLinux
```
那么就会在当前文件夹下生成一个名为ubuntuLinux的文件，这就是我们的WSL开发环境了。

如何导入？
```shell
# --import <分发版> <安装位置> <文件名> 
wsl --import Ubuntu18.04 . ubuntuLinux
```
> 导入的WSL默认用户会被设置为root，但是有时候使用root会比较麻烦，比如我们使用普通账户开发的项目，如果用root打开的话，可能有些文件的所有人和所有组会变成root，这时候如果再使用普通账户读写就会产生问题。因此要设置默认用户为普通用户。
```bash
sudo vim /etc/wsl.conf
```
文件内容如下：
```
[user]
default=lc
```
![20210709150131](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210709150131.png)

## FAQ
1. 打开的图形界面字体很模糊    
找到自己的VcXsrv安装位置，找到`vsxsrc.exe`和`xlaunch.exe`两个应用程序文件，右键属性>兼容性>更改高DPI设置>勾选替代高DPI缩放行为（应用程序）> 确定。

2. 如何从windows访问wsl2的网络   
在命令行中使用如下命令显示wsl2的ip地址
```
wsl -- ifconfig eth0
```

3. 启动vsxsrc后，依然不能打开图形界面
首先检查vsxsrc的配置是否正确，环境变量是否配置，若检查无误，则可能是防火墙阻止了连接。
尝试关闭全部防火墙（事后再重新打开），再次尝试启动，若可以启动成功，则证明为防火墙问题。
修复方法：确保防火墙入站规则中私有和公有规则都允许连接
![20210924183744](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210924183744.png)
![20210924183850](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20210924183850.png)

## Reference

[在WSL上配置输入法](https://patrickwu.space/2019/10/28/wsl-fcitx-setup-cn/)   
[wsl下Ubuntu中文显示方法](https://www.apull.net/html/20200604102131.html)   
[WSL 下 Ubuntu 20.04 中文显示设置](https://www.gelomen.com/optimize/wsl-ubuntu-20-04-zh-cn)   
[Windows Subsystem for Linux Installation Guide for Windows 10]()   
