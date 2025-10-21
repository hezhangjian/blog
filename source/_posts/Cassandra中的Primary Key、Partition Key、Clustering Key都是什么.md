---
title: Cassandra中的Primary Key、Partition Key、Clustering Key都是什么？
link:
date: 2021-10-20 20:36:27
tags:
  - Cassandra
---

Cassandra中的Key有如下三种类型

- Primary Key
- Partitioning Key
- Clustering Key

## Primary Key 主键

每张表都需要有主键。主键可以是一个字段或者多个字段的组合。每条记录的主键必须唯一。举个例子

```cql
CREATE TABLE player (
   name text,
   club text,
   league text,
   nationality text,
	 kit_number text,
   position text,
	 goals int,
   assists int,
   appearences int,
   PRIMARY KEY (club, league, name, kit_number, position, goals)
)
```

这个数据表的主键有多个字段，称做复合主键。

## 分区键

`Cassandra`根据分区键，使用一致性哈希算法，把数据分配到集群的各个机器上。一个机器可以包含多个分区。`Cassandra`保证同一分区键的数据都在一台机器上。通过合理的设置分区键，可以让你的查询让尽量少的机器处理，提升查询的效率

对于单主键字段来说，分区键和主键是同一个字段。

对于复合主键字段来说，默认情况下，分区键是复合主键的第一个字段。如上例中，分区键是`club`字段

可以通过括号来将分区键指定为多个字段，如将上面`CQL`的11行修改为

```CQL
PRIMARY KEY ((name, club), league, kit_number, position, goals)
```

## Clustering Key

Clustering Keys决定了分区内数据的排序。让我们再看一下最初的例子

```cql
CREATE TABLE player (
   name text,
   club text,
   league text,
   nationality text,
	 kit_number text,
   position text,
	 goals int,
   assists int,
   appearences int,
   PRIMARY KEY (club, league, name, kit_number, position, goals)
)
```

在主键中的字段，除了分区键外都是**clustering key**。既然`club`是主键，那么`league name kit_number position goals`是Clustering key。你可以定义**clustering key**中每个字段的升降序。可以将`kit_number`降序、`goals`升序

排序顺序与主键中字段的顺序相同。因此，在上面的例子中，数据是按照如下布局的

- 所有相同`club`的运动员都将分在同一个分区
- 在分区内，按照`leauge`排序
- 然后按照`name`排序
- 然后按照`kit_number`排序
- ...

定义不同字段升降序的语法如下(默认为升序)

```cql
CREATE TABLE player (
   name text,
   club text,
   league text,
   nationality text,
	 kit_number text,
   position text,
	 goals int,
   assists int,
   appearances int,
   PRIMARY KEY (club, league, name, kit_number, position, goals)
)
WITH CLUSTERING ORDER BY (league ASC, name DESC, kit_number ASC, position DESC );
```
