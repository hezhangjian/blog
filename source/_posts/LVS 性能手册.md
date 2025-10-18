---
title: lvs 性能手册
link: lvs-performance-manual
date: 2022-08-01 21:37:08
tags:
  - LVS
---

# lvs是做什么的

lvs通常用做tcp/udp协议的四层负载均衡

![lvs-brief](Images/lvs-brief.png)

相比也可以用于四层负载的**Nginx**组件，**Lvs**因为运行在内核态，性能高是它的主要优势，同样，因为运行在内核态中，无法像Nginx那样，对四层的tls做卸载等动作。

# lvs性能相关指标(用户视角)

## 客户端的连接数

- UDP模式下，按连接超时时间计算（根据业务需求决定）。可通过`ipvsadm -l --timeout`来查看udp超时时间
- TCP模式下，即为tcp连接数

## 客户端请求流量

即client与lvs、lvs与RS之间交互的流量

## 客户端请求平均包大小

即client与lvs、lvs与RS之间的平均包大小

# lvs性能相关参数

## 会话超时时间

### 查看

```bash
ipvsadm -l --timeout
```

### 修改

```bash
ipvsadm --set ${tcptimeout} ${tcpfintimeout} ${udptimeout}
```

## vm conntrack最大个数

### 查看

```bash
sysctl -a |grep net.netfilter.nf_conntrack_max
```

### 查看当前nf_conntrack个数

```bash
# 方式一
conntrack -C
# 方式二
cat /proc/net/nf_conntrack | wc -l
```

### 修改

```bash
sysctl -w net.netfilter.nf_conntrack_max=1024
```

## hashsize

### 什么是hashsize

**hashsize**也就是**nf_conntrack_buckets**，如果不手动指定。linux会根据机器的内存计算。如果要支持海量的**nf_conntrack**，则可以适当调大。

```c
    // nf_conntrack_core.c
    nf_conntrack_htable_size
			= (((nr_pages << PAGE_SHIFT) / 16384)
			   / sizeof(struct hlist_head));
		if (BITS_PER_LONG >= 64 &&
		    nr_pages > (4 * (1024 * 1024 * 1024 / PAGE_SIZE)))
			nf_conntrack_htable_size = 262144;
		else if (nr_pages > (1024 * 1024 * 1024 / PAGE_SIZE))
			nf_conntrack_htable_size = 65536;

		if (nf_conntrack_htable_size < 1024)
			nf_conntrack_htable_size = 1024;
```

**hlist_head**的大小在64位的机器下大小为16

### 查看

```bash
cat /sys/module/nf_conntrack/parameters/hashsize
```

### 修改 （方式一）

```bash
echo 65536 > /sys/module/nf_conntrack/parameters/hashsize
```

### 修改（方式二）永久生效

```bash
# exmaple file, you can modify this config if exists. File name doesn't matter.
# 样例文件，你可以修改已存在的这个文件。文件名称并不重要。
touch /etc/modprobe.d/lvs.conf
echo "options nf_conntrack hashsize=65536" >> /etc/modprobe.d/lvs.conf
# then you need reboot
# 需要重试来使配置生效
```

## 文件句柄数

### 查看

```bash
ulimit -n
```

### 修改

不同的linux发行版，修改方式不太一样，以RedHat为例

```bash
num=`ulimit -n`
sed -i "s|$num|65536|g" /etc/security/limits.d/*-nofile.conf
```

# lvs性能瓶颈

## 虚拟机内存

contnrack使用**slab**分配内存，可以通过**slabtop**命令查看nf_conntrack模块占用的内存。当连接数较高时，Lvs的内存瓶颈在于会话管理。

**conntrack**最大理论内存占用为

```
max_mem_used = conntrack * max * sizeof (struct nf_conntrack) + conntrack_buckets * sizeof (struct list_head)
```

使用如下python代码计算

```python
import ctypes

# 这个是nf_conntrack的动态库所在路径
# libnetfilter git地址 git://git.netfilter.org/libnetfilter_conntrack
LIBNETFILTER_CONNTRACK = '/usr/lib/aarch64-linux-gnu/libnetfilter_conntrack.so.3.7.0'
nfct = ctypes.CDLL(LIBNETFILTER_CONNTRACK)
print("max size of struct nf_conntrack:")
print(nfct.nfct_maxsize())
print("sizeof(struct list_head):")
print(ctypes.sizeof(ctypes.c_void_p) * 2)
```

其中`nfct_maxsize`出自于`git://git.netfilter.org/libnetfilter_conntrack`中的`src/conntrack/api.c`

```c
/**
 * nfct_maxsize - return the maximum size in bytes of a conntrack object
 */
```

在如下操作系统下

```
uname -a
> Linux primary 5.4.0-122-generic #138-Ubuntu SMP Wed Jun 22 15:05:39 UTC 2022 aarch64 aarch64 aarch64 GNU/Linux
```

以100万conntrack_max，65536buckets为例，占用的内存为

1_000_000 * 392 + 65536 * 16 约等于 373.84 + 1 为374M内存

## 网卡流量

最大进出带宽。在云上，通常由云厂商限制。如果你将lvs上面的浮动Ip通过EIP的方式暴露出去（这很常见），还需要考虑EIP自身的带宽

## 网卡进出包个数（PPS)

最大进出包个数

## 虚拟机能支持的最大网络连接数

ECS上可以支持的最大网络连接数。在云上，通常由云厂商限制

# Lvs监控&扩容

## cpu使用率

可在超过百分之80的时候告警。处理方式：

- 如果内存还没有到达瓶颈，可以通过扩大hashsize的方式，降低hash链上元素的个数，减少匹配消耗的cpu
- 如果内存水位也较高。对CPU进行扩容

## 内存使用率

可在超过内存容量百分之80的时候告警。处理方式：扩容内存

## conntrack个数

通过`conntrack -C`或`cat /proc/net/nf_conntrack | wc -l`, 定期进行统计，使用`sysctl -w net.netfilter.nf_conntrack_max`进行扩容

## 网卡流量、网卡进出包个数

可以利用云厂商的监控或`nicstat`命令查看。处理方式：扩容网卡

## 最大网络连接数

可以利用云厂商的监控或`netstat -an|egrep "tcp|udp"|grep -v "LISTEN"|wc -l`或`ss -tun state all | grep -v LISTEN | wc -l`查看。处理方式：扩容ECS规格

## EIP带宽

通过云厂商的指标来监控。处理方式，扩容EIP的BGP带宽
