---
title: kubernetes容器获取IP地址
link: kubernetes-container-get-ip
date: 2023-01-06 19:05:07
tags:
  - Kubernetes
---

kubernetes中容器获取IP地址是一个常见的需求，常见的有两种获取IP地址的方式

## kubernetes环境变量注入

通过在部署时，**container**下的**env**中配置如下yaml

```yaml
            - name: POD_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
```

进入容器就可以根据环境变量获取到容器IP

```
# echo $POD_IP
172.17.0.2
```

## 通过shell脚本获取

### 通过ip命令（推荐）

```bash
# ip addr show eth0 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1
172.17.0.2
```

注意这里一定要用**inet\b**，不要用**inet**。使用**inet**的话，在Ipv6双栈场景下会因为匹配到**inet6**获取到错误的结果, Ipv6双栈场景下ip命令的部分输出结果如下图所示

```
inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
inet6 fe80::ffff prefixlen 64 scopeid 0x20<lin>
```

### 通过ifconfig命令（不推荐）

不推荐使用ifconfig命令的原因是，这个命令已经废弃，将会逐步删除

```bash
ifconfig eth0 | grep 'inet\b' | awk '{print $2}' | cut -d/ -f1
```

同样需要使用**inet\b**，不要使用**inet**

## TLDR

优先配置如下yaml进行环境变量注入，其次使用**ip addr show eth0 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1**命令获取

```yaml
            - name: POD_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
```
