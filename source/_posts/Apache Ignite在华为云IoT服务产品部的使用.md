---
title: Apache Ignite在华为云IoT服务产品部的使用
link: ignite-huaweicloud-usage
date: 2023-12-07 14:01:05
tags:
  - Ignite
---

# Apache Ignite简介

Apache Ignite是一个开源分布式的数据库、缓存和计算平台。它的核心是一个内存数据网格，它可以将内存作为分布式的持久化存储，以提供高性能和可扩展性。它还提供了一个分布式的键值存储、SQL数据库、流式数据处理和复杂的事件处理等功能。

Ignite的核心竞争力包括：
- 兼容Mysql、Oracle语法
- 性能强大，可以水平扩展
- 缓存与数据库同源，可通过KV、SQL、JDBC、ODBC等方式访问

同时，为了便于开发，除了jdbc、odbc、restful方式外，Ignite还官方提供了Java、C++、.Net、Python、Node.js、PHP等语言的客户端，可以方便的与Ignite进行交互。

![ignite-storage-access](Images/ignite-storage-access.png)

# Apache Ignite的问题

## 频繁创建删除表，导致IGNITE_DISCOVERY_HISTORY_SIZE超过限制

根据Ignite2的拓扑模型，集群的拓扑版本会在创建表/删除表的时候发生变化，该变化版本号递增，且仅会保留最近$IgniteDiscoveryHistorySize条记录，程序某处会写死读取版本为0的数据，读取不到时，ignite集群会重启。默认值为500。
社区issue: https://github.com/apache/ignite/issues/10894
笔者暂时没有时间来修复这个issue，可以通过将IGNITE_DISCOVERY_HISTORY_SIZE设置地比较大，来规避这个问题。

## Ignite2客户端易用性问题

### Ignite2客户端超时默认值不合理

Ignite2客户端的连接超时、执行sql超时默认都是0，没有精心研究过配置的用户在异常场景下，应用程序可能会hang住。从易用性的角度来说，网络通信的任何操作，默认都应该有超时时间。

### Ignite2客户端不支持永远的重试

Ignite通过预先计算出所有需要重连的时间点来实现重连，如果想配置成永远的重连，会因为时间点的计算导致内存溢出。从易用性的角度来说，应该支持永远的重连。

## Ignite2客户端在某些异常下无法自愈

当client执行sql的时候，碰到如下异常的时候，无法自愈。可以通过执行SQL对client进行定期检查并重建。

```
Caused by: org.apache.ignite.internal.client.thin.ClientServerError: Ignite failed to process request [47]: 50000: Can not perform the operation because the cluster is inactive. Note, that the cluster is considered inactive by default if Ignite Persistent Store is used to let all the nodes join the cluster. To activate the cluster call Ignite.cluster.state(ClusterState.ACTIVE)
```

## Ignite2 SocketChannel泄露问题

Ignite客户端在连接时，如果对应的Server端没有启动，会导致SocketChannel泄露，已由笔者提交代码修复：https://github.com/apache/ignite/pull/11016/files
