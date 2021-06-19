---
author: "李昌"
title: "zsh的基本配置"
date: "2021-05-20"
tags: ["zsh", "terminal", "美化", "tools", "效率工具"]
categories: ["Tools"]
ShowToc: true
TocOpen: true
---

## 1. 按照Oh my zsh
```bash
$ sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## 2. 配置Oh my zsh
1. 将zsh设置为默认Shell (脚本的最后一般会问你是否切换)
    ```bash
    chsh -s /bin/zsh # 不需要使用root权限
    ```
2. 更换主题
   ```bash
   vim ~/.zshrc
   ```
   找到`ZSH_THEME='robbyrussell'`, 更换为你想要使用的主题，可以在[这里](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)找到你想要的主题
3. 安装插件
   ```bash
   vim ~/.zshrc
   ```
   找到`plugins=()`, 添加插件名称，我这里添加的插件有：
   ```
   plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
   ```

   ```bash
   git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

   git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
   ```
4. 完成
   ```bash
   source ~/.zshrc # 启动zsh
   ```


## 3. 使用主题`powerlevel10k`
下载主题
```bash
git clone --depth=1 https://gitee.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

打开你的`~/.zshrc`,将主题换为：`powerlevel10k/powerlevel10k`

更改保存并使用主题
```bash
source ~/.zshrc
```

这时`powerlevel10k`会自动启动，询问你想要的配置
![20210619122024](https://raw.githubusercontent.com/lich-Img/blogImg/master/img20210619122024.png)

按照提示配置你想要的风格即可
