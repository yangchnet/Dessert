---
author: "李昌"
title: "解决github每次push都要输入密码"
date: "2021-02-25"
tags: ["Git"]
categories: ["Git"]
ShowToc: true
TocOpen: true
---

## 记住账号密码
```bash
git config --global credential.helper store
```
然后再运行一遍`git pull`或`git push`就可以了

## 使用密钥验证(推荐)

参考[主机上设置两个git账号](https://yangchnet.github.io/Dessert/posts/git/%E4%B8%BB%E6%9C%BA%E4%B8%8A%E8%AE%BE%E7%BD%AE%E4%B8%A4%E4%B8%AAgit%E8%B4%A6%E5%8F%B7/)
