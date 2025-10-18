---
title: MySQL异步复制中的数据不一致问题
link: mysql-async-replica-inconsistency
date: 2024-07-09 20:51:03
tags:
  - MySQL
---
## 前提

Mysql8.0.X版本，且核心配置如下

```
gtid_mode=ON
binlog_format=row
slave_skip_errors=all
```

数据不一致的根本原因在于MySQL在设计上不具备分布式系统的完整语义，这导致主从复制在面对网络分区和延迟时无法保持数据一致性。（又不可能采取全同步的模式，那就变成一个CP系统了）。根据数据冲突的内容，如果是**“不同主键，不触发唯一键约束的数据冲突”**，那么后续很容易可以同步到一致。如果触发了主键或者唯一键的冲突，无法互相同步，场景会变得复杂一些，简而言之，只有当后续的操作可以同时在主/备两个数据库中抹平这个差距，数据才能恢复，并且约束越多，抹平也就变得愈困难。举例

- 仅存在主键约束，数据内容不同，通过下次操作主键(update/delete)，则可以恢复
- 数据库自增主键（两条数据主键不同），触发了唯一字段约束，后续的操作要同时抹平主键、唯一字段、其他内容才能恢复一致（比如根据相同的条件删除掉这条数据等）

下文将分别以插入为例讨论这几个场景，用红色叉号代表同步延迟或者断开。

注：由于Mysql主备同步时会将upsert类的sql转换为实际执行的insert、update语句，也就是说upsert的语义在主备同步不稳定/切换时，容易丢失。

## **不同主键，不触发唯一键约束的数据冲突**

设想表结构，仅有一个name字段，且name为主键。比如我们先在MysqlA中插入了数据name=tt，假设发生了切换，又向MysqlB插入了数据name=wtt。

![mysql-case1-insert-data](Images/mysql-case1-insert-data.png)

这就导致MysqlA与MysqlB里面的数据存在着不一致，但是一旦同步恢复，数据就会一致。

![mysql-case1-sync-success](Images/mysql-case1-sync-success.png)

## 仅主键约束，内容不一致冲突

表结构，拥有两个字段，name为主键，age为字段。

同样，插入了两条数据，导致冲突。

![mysql-case2-insert-data](Images/mysql-case2-insert-data.png)

即使MysqlA和MysqlB之间同步恢复，后续insert语句也会由于主键冲突同步失败。

![mysql-case2-sync-fail](Images/mysql-case2-sync-fail.png)

这种不一致要等到后续对主键进行update操作后，才能恢复一致

![mysql-case2-recovery](Images/mysql-case2-recovery.png)

## 包含主键、唯一约束在内的冲突场景

主键为数据库自增主键，其中一个库为奇数，另一个库为偶数。同时还有唯一约束name

![mysql-case3-insert-data](Images/mysql-case3-insert-data.png)

这时候插入数据，就会导致不一致，并且主键也不相同，由于业务不感知主键，使用不存在则更新的语法也会导致主键不一致。

![mysql-case3-upsert-data](Images/mysql-case3-upsert-data.png)

可以预想到即使恢复同步，MysqlA和MysqlB数据也无法一致。

![mysql-case3-sync-fail](Images/mysql-case3-sync-fail.png)

在这种场景下，任何针对id的SQL操作都无法在双方数据库中成功同步。例如，MysqlB数据库中不存在id为0的记录，而MysqlA中不存在id为1的记录，导致同步操作失败。

想要恢复一致，可以通过业务唯一约束来删除记录或者是根据业务约束把Mysql主键id也一并更新（不过这很困难，一般这种业务是不会直接操作id的）

那么可能会有人有疑问，为什么不像之前那样，用name作为唯一主键呢？

答：业务的需求多种多样，而且如果唯一约束由多个字段组成，使用Mysql自增主键是唯一的选择。

## 总结

本文探讨了Mysql异步复制模式下的数据不一致问题，容易在什么时候产生，什么时候恢复。总的来说，业务如果只有一个唯一主键，出现不一致的概率更小。如果业务用数据库自增作为主键，同时伴有唯一约束的插入操作（如upsert等），更容易出现长期的不一致。
