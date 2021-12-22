---
author: "李昌"
title: "Google搜索指令"
date: "2021-12-22"
tags: ["google", "tips"]
categories: ["杂记"]
ShowToc: true
TocOpen: true
---

## 1. site
site: 搜索指定站点

用法:
```
site:[example.com]
```

示例:
```
golang site:github.com
```

## 2. source
source: 在谷歌新闻中指定来源

用法:
```
source:[sourcesite]
```

示例:
```
COVID source:yahoo
```

## 3. intext
intext: 查询的内容必须出现在正文中

用法:
```
intext:[somewords]
```

示例:
```
intext:xiaomi
```

## 4. allintext
allintext: 查询的每个单词都必须包含在页面中

用法:
```
allintext:[somewords]
```

示例:
```
allintext:Quantum Network Coding
```

## 5. intitle
intitle: 标题中包含要查询的内容

用法:
```
intitle:[somewords]
```

示例:
```
intitle:Quantum
```

## 6. allintitle
allintitle: 类似`allintext`


## 7. url
url:结果的url中必须包含某些内容

用法:
```
url:[somewords]
```

示例:
```
url:airpods
```

## 8. allinurl
allinurl: 结果的url必须包含所有查询内容


## 9. filetype
filetype: 查询的结果满足某种文件类型

用法:
```
filetype:[filetype]
```

示例:
```
golang filetype:pdf
```

## 10. related
related: 查找有关内容

用法:
```
related:[somewords]
```

示例:
```
related:[airpods]
```

## 11. AROUND(X)
AROUND(X): 将结果限制为:包含彼此相差X个单词的搜索词的页面。

用法:
```
AROUND(X) [somewords]
```

示例:
```
golang AROUND(10) best
```

## 12. 引号""
"": 将搜索词包含在引号中将启动对该短语的完全匹配搜索，确保引号中的内容必须在页面上

用法:
```
"[somewords]"
```

示例:
```
"Quantum Network Coding"
```

## 13. AND
AND: 搜索并集

用法:
```
[x] AND [y]
```

示例:
```
golang AND tutorial
```

## 14. -(横线)
-: 差集操作

用法:
```
[x] -[y]
```

示例:
```
golang tutorial -site:youtube.com
golang -tutorial
```

## 15. *(星号)
*:通配

用法:
```
[somewords] * [somewords]
```

示例:
```
wsl * tutorial
```

## 16. () 括号
(): 将术语或搜索运算符组合在一起帮助构建搜索指令

用法:
```
[somewords] ([somewords])
```

示例:
```
golang site:github.com (pr OR commit)
```

## 17. OR(或|竖线)
OR/|: 或

用法:
```
[x] OR [y]
[x] | [y]
```

示例:
```
golang (user OR org)
golang (user | org)
```

## 18. $/€ (美元符或英镑符)
$/€: 搜索商品的价格

用法:
```
[merchandise] [price]$
```

示例:
```
mac pro 2000$
```

## 19. Year..Year
Year..Year: 在两年之间放置两个点会为该年份范围内的结果创建一个 Google 搜索命令

用法:
```
[somewords] [Year]..[Year]
```

示例:
```
mac 2019..2021
```

## 20. info
info: 查询网站的相关信息

用法:
```
info:[site]
```

示例:
```
info:google.com
```

## 21. link
link: 查询显示指向那个的页面

用法:
```
link:[site]
```

示例:
```
link:golang.dev
```

## 22. movie
movie: 查询电影

用法:
```
movie:[movie name or other information about it]
```

示例:
```
movie:Titanic
```

## 23. insubject
insubject: 查询主题

用法:
```
insubject:[subject]
```

示例:
```
site:github.com insubject:Linux
```

## 24. define
define: 查询术语定义

用法:
```
define:[somewords]
```

示例:
```
define:(Quantum Network)
```

## 25. cache
cache: 查询Google缓存的网页版本，而不是网页的当前版本

用法:
```
cache:[site]
```

示例:
```
cache:juejin.cn
```

## 26. author
author: 如果在查询中包含`author:`，Google将网上论坛结果限制为包含指定的作者的新闻组文章。用于网上论坛查询

用法:
```
author:[author]
```

示例:
```
 children author:doe@someaddress.com
```

## References  
http://www.googleguide.com/advanced_operators_reference.html