---
author: "李昌"
title: "channel的行为"
date: "2022-01-02"
tags: ["golang", "channel"]
categories: ["golang"]
ShowToc: true
TocOpen: true
---

## 1. nil channel
1. 接收
接收goroutine阻塞

2. 发送
发送个goroutine阻塞

## 2. 向无缓冲channel发送消息
1. 接受队列有goroutine
接收端将收到消息

2. 接收队列无goroutine
发送goroutine将阻塞

3. 已有发送goroutine阻塞
发送goroutine将阻塞

## 3. 从无缓冲channel接收消息
1. 无发送goroutine
接收端阻塞

2. 有发送goroutine
收到消息

## 4. 向有缓冲channel发送消息
1. 队列未满
正常发送

2. 队列已满
发送端阻塞

## 5. 从有缓冲channel接收消息
1. 队列中有消息
正常接收

2. 队列中无消息
接收端阻塞

## 6. 对close channel的操作
1. 向closed channel发送
panic

2. 从无缓冲closed channel接收
接收到对应类型0值

3. 从有缓冲closed channel接收
    - 若channel中仍有消息，则仍可接收到消息
    - channel已空, 接收到对应类型零值

## 7. 关闭channel
1. 关闭nll channel
panic

2. 关闭一个已经关闭的channel
panic