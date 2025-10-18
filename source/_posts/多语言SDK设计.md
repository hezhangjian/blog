---
title: 多语言SDK设计
link: multi-lang-client-sdk-design
date: 2023-11-12 21:09:32
tags:
  - Code
  - SDK
---
# 多语言SDK设计的常见问题

## 日志打印的设计策略

在SDK的关键节点，比如初始化完成、连接建立或者连接断开，都可以打印日志。如果是PerRequest的日志，一般默认不会打印INFO级别的日志。

SDK应该避免仅仅打印错误日志然后忽略异常；相反，它应该提供机制让调用者能够捕获并处理异常信息。这种做法有助于保持错误处理的透明性，并允许调用者根据需要采取适当的响应措施。正如**David J. Wheeler**所说"Put the control in the hands of those who know how to handle the information, not those who know how to manage the computers, because encapsulated details will eventually leak out."把控制权放到那些知道如何处理信息的人手中，而不是放在那些知道如何管理计算机的人手中，因为封装的细节最终都会暴露。

## 是否需要使用显式的`start`/`connect`方法？

像go这样的语言，一般来说不太在意特定的时间内，某个协程是否处于阻塞等待连接的状态。而在java这样的语言，特别是在采用响应式编程模型的场景下，通常需要通过异步操作来管理连接的建立。这可以通过显式的start/connect方法来或者是异步的工厂方法来实现。
