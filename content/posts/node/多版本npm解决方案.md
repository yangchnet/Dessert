---
author: "李昌"
title: "多版本npm解决方案"
date: "2021-08-06"
tags: ["npm"]
categories: ["node"]
ShowToc: true
TocOpen: true
---

## 1. nvm install

```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
```

安装完毕后会提示你让你将以下命令加入你的配置文件中
```sh
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```
默认情况下，已经加入你的`~/.bashrc`文件中，如果你使用的是zsh，那么就需要手动将其添加到`~/.zshrc`中

## 2. 常用操作

- 列出本地所有npm版本
  ```bash
  nvm ls
  ```

- 列出可获取的所有版本
  ```bash
  nvm ls-remote
  ```

- 安装指定版本
  ```bash
  nvm install 14 # 14是版本号
  ```
- 指定使用某个版本
  ```bash
  nvm use 14
  ```