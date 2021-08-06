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

wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash

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