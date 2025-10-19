---
title: Linux根据源地址ip路由
link:
date: 2020-09-10 20:18:27
tags:
  - Linux
---

## Hash-based multipath routing
该特性在Linux4.4版本引入，一个难以被大家发现的好处是，基于源地址路由，没有做地址转换，并不会在nf_conntrack中添加记录
它是对源IP和目标IP进行哈希处理(端口不参与哈希的计算)计算选路。配置的命令如下:
weight代表权重


### 通过网关负载均衡
```
ip route add default  proto static scope global \
nexthop  via <gw_1> weight 1 \
nexthop  via <gw_2> weight 1
```
### 通过网卡负载均衡
```
ip route add default  proto static scope global \
 nexthop  dev <if_1> weight 1 \
 nexthop  dev <if_2> weight 1
```
