---
title: LVS persistent timeout和connection timeout解析
link: lvs-timeout-analyze
date: 2021-03-19 18:56:54
tags:
  - LVS
---

# 两个超时的注释

首先看一下一下`ipvsadm -h`对这两个参数的注释

## persistent timeout

```
--persistent  -p [timeout]     persistent service
Specify that a virtual service is persistent. If this option is specified, multiple requests from a client are redirected to the same real server selected for the first request. Optionally, the timeout of persistent sessions may be specified given in seconds, otherwise the default of 300 seconds will be used. This option may be used in conjunction with protocols such as SSL or FTP where it is important that clients consistently connect with the same real server.
```

说明这个VS是否是持久的。如果配置了这个选项，来自同一个客户端的链接（这里注意：这里的同一个客户端指的是同一个IP）会转发向相同的服务器。注释中特意提到了FTP协议。我查阅了一下资料，可能像FTP协议这种，客户端通过21端口打开控制连接，再通过20端口打开数据连接，这种协议，要求来自同一个客户端ip，不同端口的请求也送向同一个服务器，估计是这个参数存在的核心原因。如果是现在的系统，比如k8s使用ipvs，这个参数是完全没必要配置的

## connection timeout

```
--set tcp tcpfin udp
Change the timeout values used for IPVS connections. This command always takes 3 parameters, representing the timeout values (in seconds) for TCP sessions, TCP sessions after receiving a FIN packet, and UDP packets, respectively. A timeout value 0 means that the current timeout value of the corresponding entry is preserved.
```

更改用于ipvs连接的超时值。此命令始终使用3个参数，分别表示tcp会话，接收到FIN包的TCP会话和UDP包的超时值。单位为秒。设置为0并不代表将超时值设置为0，而是保持原有不变。顺便来说，`timeout`的默认值是900、120、300.

## 区别

一个以客户端ip为维度，一个以客户端ip+port为维度

## 联系：

- persistent值大于等于set时，persistent timeout以persistent的设置为准。
- persistent值小于set时，当set超时，但persistent超时后，会将persistent再次设置为60。只到set超时为止。所以这个时候，真实生效的persistent timeout是`(s/60)*60 + p%60 + 60`
