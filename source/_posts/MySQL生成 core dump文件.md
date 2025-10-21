---
title: MySQL生成 core dump文件
link:
date: 2021-10-16 23:32:43
tags:
---

## 前提条件

## 添加配置

### [mysqld]下面添加`core-file`

![image-20211016232603862](Images/mysql-core-dump1.png)

### ulimit打开core file限制

```bash
ulimit -c unlimited
```

### 如需要，修改core file路径（如在容器内，需要特权容器权限）

```bash
echo "/opt/sh/mysql/core/core" > /proc/sys/kernel/core_pattern
```

### 使得core file携带pid信息

```bash
echo 1 >/proc/sys/kernel/core_uses_pid
```

## 通过kill命令获取core file

```bash
kill -11 $pid
```

![image-20211016233105059](Images/mysql-core-dump2.png)
