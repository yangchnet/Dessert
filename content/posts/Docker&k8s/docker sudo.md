---
author: "李昌"
title: "docker sudo"
date: "2021-06-20"
tags: ["docker"]
categories: ["Docker&k8s"]
ShowToc: false
TocOpen: false
---

```bash
sudo usermod -aG docker $USER && newgrp docker # 将当前用户添加到docker用户组
```

退出重新登陆即可aaa