---
author: "李昌"
title: "manjaro初始化配置"
date: "2022-04-11"
tags: ["manjaro", "install"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

##  proxy

见：https://yangchnet.github.io/Dessert/posts/env/%E5%AE%89%E8%A3%85%E4%B8%8E%E9%85%8D%E7%BD%AEclash/

## 系统更新

首先要换源

```sh
sudo pacman-mirrors -i -c China -m rank
```

在弹出的窗口中选择你要切换的源。

然后
```sh
sudo pacman -Syyu
```

安装yay包管理
```sh
sudo pacman -S yay
```

## vim 配置
见：https://yangchnet.github.io/Dessert/posts/linux/vim%E9%85%8D%E7%BD%AE/
## 输入法配置

### 安装fcitx5（输入法框架）

```sh
yay -S fcitx5-im
```

配置fcitx5的环境变量：
```sh
vim ~/.pam_environment
```

内容为：
```
GTK_IM_MODULE DEFAULT=fcitx
QT_IM_MODULE  DEFAULT=fcitx
XMODIFIERS    DEFAULT=\@im=fcitx
SDL_IM_MODULE DEFAULT=fcitx
```

安装fcitx5-rime（输入法引擎）
```sh
yay -S fcitx5-rime
```

安装fcitx5-gtk
```sh
yay -S fcitx5-gtk # 不装的话，部分软件可能会出现不能输入中文的情况
```

安装rime-cloverpinyin（输入方案）

```sh
yay -S rime-cloverpinyin
```

如果出现问题可能还需要做下面这步：
```sh
yay -S base-devel
```

创建并写入rime-cloverpinyin的输入方案：
```sh
vim ~/.local/share/fcitx5/rime/default.custom.yaml
```

内容为：
```sh
patch:
  "menu/page_size": 5
  schema_list:
    - schema: clover
```

> 可参考：https://github.com/fkxxyz/rime-cloverpinyin/wiki/linux

### 配置主题

参考：https://github.com/hosxy/Fcitx5-Material-Color

### 安裝双拼(optional)
```bash
cd $HOME/.local/share/fcitx5 # rime_dir
```

安装plum
```bash
curl -fsSL https://git.io/rime-install | bash
```

安装双拼输入法
```bash
rime_dir="/home/lc/.local/share/fcitx5/rime" bash ./rime-install emoji
rime_dir="/home/lc/.local/share/fcitx5/rime" bash ./rime-install double-pinyin
```

配置双拼：
```
patch:
  "menu/page_size": 5
  schema_list:
    - schema: double_pinyin_flypy
      #- schema: clover
```

参考：

- https://help.flypy.com/#/up
- https://www.bilibili.com/read/cv14690644
- https://github.com/rime/home/wiki/CustomizationGuide#%E4%B8%80%E4%BE%8B%E5%AE%9A%E8%A3%BD%E7%B0%A1%E5%8C%96%E5%AD%97%E8%BC%B8%E5%87%BA
- https://github.com/rime/plum


## 配置命令行工具

见：https://yangchnet.github.io/Dessert/posts/env/zsh%E7%9A%84%E5%9F%BA%E6%9C%AC%E9%85%8D%E7%BD%AE/


## 社交工具

微信

```sh
yay -S wechat-uos
```


QQ

```sh
yay -S deepin-wine-tim
```

或
```sh
yay -S linuxqq
```

## Virtualbox

```sh
sudo pacman -S virtualbox

sudo pacman -S linux-headers  # 安装对应自己内核版本的一个
# linux61-virtualbox-host-modules

sudo pacman -S virtualbox-host-dkms

sudo modprobe vboxdrv
```


## Reference

https://zhuanlan.zhihu.com/p/114296129?ivk_sa=1024320u