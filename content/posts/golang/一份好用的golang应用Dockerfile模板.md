---
author: "李昌"
title: "一份好用的golang应用Dockerfile模板"
date: "2022-04-02"
tags: ["golang", "Dockerfile"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

```dockerfile
# 编译环境
FROM golang:alpine as builder 

# 设置go环境变量
ENV GO111MODULE=on \
    GOPROXY=https://goproxy.cn,direct

# 工作目录
WORKDIR /app

# 将项目拷贝到docker中
COPY . .

# 拉取包，编译
RUN go mod tidy && CGO_ENABLED=0 GOOS=linux go build -a -ldflags '-extldflags "-static"' -o hello-app .

# 运行环境
FROM scratch

# 设置时区
COPY --from=builder /usr/share/zoneinfo/Asia/Shanghai /usr/share/zoneinfo/Asia/Shanghai
ENV TZ Asia/Shanghai

WORKDIR /app

# 将编译好的可执行文件从编译环境中拷贝到运行环境中
COPY --from=builder /app/hello-app .

# 启动
ENTRYPOINT ["./hello-app"]

# 端口
EXPOSE 10000
```