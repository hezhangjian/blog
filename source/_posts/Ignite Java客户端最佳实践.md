---
title: Ignite Java客户端最佳实践
link: ignite-java-client-best-practice
date: 2023-10-10 22:08:55
tags:
  - Ignite
  - Java
---
# Ignite Java 客户端最佳实践

## 背景

本文总结了在使用Apache Ignite（Ignite2.0）的Java客户端时，需要注意的一些问题，以及一些最佳实践。值得一提的是 Ignite的Java客户端有一些跟直觉上不太一样的地方，需要注意下。

## 客户端相关

Ignite客户端有两处跟直觉上相差较大：

- Ignite客户端连接没有默认超时时间，如果连接不上，有概率会导致创建客户端一直阻塞，所以一定要设置timeout参数
- Ignite客户端默认不会重连，更不用说无限重连了。并且Ignite客户端重连的实现方式是预先计算出所有重连的时间戳，然后在这些时间戳到达时重连，由于要预先计算出重连的时间戳存入数组，这也就意味着不能无限重连。如果您的应用程序需要无限重连（在云原生环境下，这是非常常见的场景），那么您需要自己实现重连逻辑。

ClientConfiguration里的重要参数

### ClientConfiguration timeout

控制连接超时的参数，单位是毫秒。必须设置！如果不设置，有概率会导致创建客户端一直阻塞。

## SQL相关

SQL查询典型用法

```java
SqlFieldsQuery query = new SqlFieldsQuery("SELECT 42").setTimeout(5, TimeUnit.SECONDS);
FieldsQueryCursor<List<?>> cursor = igniteClient.query(query))
List<List<?>> result = cursor.getAll();
```

注意：Ignite query出来的cursor如果自己通过iterator遍历则必须要close，否则会导致内存泄漏。

Query相关参数

### SqlFieldsQuery timeout

SqlQuery的超时时间，必须设置。默认是0，表示永不超时。如果不设置，有概率会导致查询一直阻塞。
