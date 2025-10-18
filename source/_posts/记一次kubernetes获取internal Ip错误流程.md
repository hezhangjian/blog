---
title: 记一次kubernetes获取internal Ip错误流程
link: kubernetes-internal-ip-error
date: 2022-11-15 12:00:44
tags:
  - Kubernetes
---

偶尔也回首一下处理的棘手问题吧。问题的现象是，通过kubernetes get node输出的ip不是期望的ip地址。大概如下所示

```
ip addr

eth0 ip1
eth0:xxx ip2
```

最终输出的不是预期的ip1地址，而是ip2地址。

按藤摸瓜，**kubernetes**把节点信息保存在`/registry/minions/$node-name`中的**InternalIp** 字段。

**InternalIp**是如何确定的呢，这段代码位于`pkg/kubelet/nodestatus/setters.go`中

```
			// 1) Use nodeIP if set (and not "0.0.0.0"/"::")
			// 2) If the user has specified an IP to HostnameOverride, use it
			// 3) Lookup the IP from node name by DNS
			// 4) Try to get the IP from the network interface used as default gateway
			//
			// For steps 3 and 4, IPv4 addresses are preferred to IPv6 addresses
			// unless nodeIP is "::", in which case it is reversed.
```

我们的场景下没有手动设置nodeIp，如需设置通过kubelet命令行即可设置 **--node-ip=localhost**，最终通过如下的go函数获取ip地址

```go
addrs, _ = net.LookupIP(node.Name)
```

对这行go函数进行strace追溯，最终调用了c函数，**getaddrinfo**函数。**getaddrinfo**底层是发起了**netlink**请求，开启**netlink**的抓包

```bash
modprobe nlmon
ip link add nlmon0 type nlmon
ip link set dev nlmon0 up
tcpdump -i nlmon0 -w netlinik.pcap
# 使用nlmon 驱动模块，这个nlmon 驱动模块会注册一个 netlink tap 口，用户态向内核发送 netlink 消息、内核向用户态发送 netlink 消息，报文都会经过这个 tap 口。
```

通过抓包我看到通过**netlink**报文请求返回的ip地址顺序都是合乎预期的，只能是**getaddrinfo**函数修改了返回的顺序

Google了一下发现是**getaddrinfo**支持了**rfc3484**导致了ip的重新排序，代码地址`glibc/sysdeps/posix/getaddrinfo.c`

**RFC3484** 总共有十个规则，比较关键的有

### Rule9

```
   Rule 9:  Use longest matching prefix.
   When DA and DB belong to the same address family (both are IPv6 or
   both are IPv4): If CommonPrefixLen(DA, Source(DA)) >
   CommonPrefixLen(DB, Source(DB)), then prefer DA.  Similarly, if
   CommonPrefixLen(DA, Source(DA)) < CommonPrefixLen(DB, Source(DB)),
   then prefer DB.
```

举个例子，假如机器的ip地址是 `172.18.45.2/24`，它会更青睐于`172.18.45.6`而不是`172.31.80.8`。这个RFC存在较大的争议，它与dns轮询策略不兼容，如：dns服务器轮询返回多个ip地址，客户端总是选择第一个ip连接。与这个策略存在很大的冲突。并且社区内也有投票试图停止对**RFC3484** rule9的适配, 但是最终被拒绝了。

根据分析，认为是ip2的地址小于ip1的地址，最终glibc排序的时候把ip2放在了前面。最终我们给kubelet配置了eth0地址的--node-ip，解决了这个问题。
