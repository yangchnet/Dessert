---
author: "李昌"
title: "croc: 跨网络、跨系统的空投"
date: "2021-12-21"
tags: ["croc", "空投"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

## 1. install
```sh
curl https://getcroc.schollz.com | bash
```
或  
```sh
go install github.com/schollz/croc/v9@latest
```

## 2. basic usage
sender:  
```sh
$ croc send [file(s)-or-folder]
Sending 'file-or-folder' (X MB)
Code is: code-phrase
```

receiver:  
```sh
croc code-phrase
```

## 3. comment
可用于替代ftp上传文件，用于不同主机之间文件共享，类似“空投”的效果

