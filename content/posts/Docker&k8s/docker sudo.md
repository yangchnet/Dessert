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
sudo usermod -aG docker $USER && newgrp docker
```

退出重新登陆即可