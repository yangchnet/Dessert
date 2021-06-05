---
author: "李昌"
title: "WSL2开发环境配置"
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


## 2. WSLg

1. 设置环境变量
```bash
export DISPLAY=$(awk '/nameserver / {print $2; exit}' /etc/resolv.conf 2>/dev/null):0
export LIBGL_ALWAYS_INDIRECT=1
```


2. 安装Xserver，这里选择的软件是`vcxsrv`， 可在[sourceforge](https://sourceforge.net/projects/vcxsrv/)中下载安装。

3. 安装完成后直接启动即可   
   
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


## 3. WSL中文

### 3.1 WSL中文显示
1. 安装语言包
```bash
sudo apt install language-pack-zh-hans
```

2. 设置local
```bash
sudo vim /etc/locale.gen
```
找到 `zh_CN.UTF-8 UTF-8` , 取消注释，保存并退出
```
# zh_CN.GBK GBK
zh_CN.UTF-8 UTF-8
# zh_HK BIG5-HKSCS
```

3. 编译语言
```bash
sudo locale-gen
```
4. 设置默认语言为中文
```bash
sudo vim /etc/default/locale
```
将内容修改为：
```
LANG=zh_CN.UTF-8
```

5. 注销重新登录，即可显示中文。

> 依然不能显示中文怎么办: 
> 找到自己的VcXsrv安装位置，找到`vsxsrc.exe`和`xlaunch.exe`两个应用程序文件，右键属性>兼容性>更改高DPI设置>勾选替代高DPI缩放行为（应用程序）> 确定。



### 3.2 WSL中文输入法配置

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


## Reference


[在WSL上配置输入法](https://patrickwu.space/2019/10/28/wsl-fcitx-setup-cn/)   
[wsl下Ubuntu中文显示方法](https://www.apull.net/html/20200604102131.html)   
[WSL 下 Ubuntu 20.04 中文显示设置](https://www.gelomen.com/optimize/wsl-ubuntu-20-04-zh-cn)
