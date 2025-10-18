---
title: Wireshark 显示过滤器
link: wireshark-display-filter
date: 2018-09-09 09:55:09
tags:
  - Wireshark
---

# 如何使用显示过滤器

![wireshark-display-filter1](Images/wireshark-display-filter1.png)
或者按住 CTRL + F，输入显示过滤器
![wireshark-display-filter2](Images/wireshark-display-filter2.png)

## 二层显示过滤器举例

### 长度小于128字节的数据包

frame.len<=128

### 排除ARP流量

!arp

## 三层显示过滤器举例

### 只显示192.168.0.1 IP相关数据包

ip.addr==192.168.0.1

## 四层显示过滤器举例

### 排除RDP流量

!tcp.port==3389

### 具有SYN标志的TCP数据包

tcp.flags.syn==1

### 具有RST标志的TCP数据包

tcp.flags.rst==1

### TCP确认时间较久

tcp.analysis.ack_rtt > 0.2 and tcp.len == 0
###启用TCP Relative Sequence Number的情况
如何启用?
Edit -> Preferences -> Protocols -> TCP Relative Sequence Numbers

### 握手被对方拒绝的包

tcp.flags.reset == 1 && tcp.seq == 1

### 客户端重传

tcp.flags.syn == 1 && tcp.analysis.retransmission

### Tcp包含

tcp contains {str}

## 应用层显示过滤器举例

### 所有http流量

http

### 文本管理流量

tcp.port == 23 || tcp.port == 21

### 文本email流量

email || pop || imap

### 只显示访问某指定主机名的HTTP协议数据包

http.host == <"hostname">

### 只显示包含HTTP GET方法的HTTP协议数据包

http.request.method == 'GET'

### 只显示HTTP 客户端发起的包含指定URI请求的HTTP协议数据包

http.request.uri == <"Full request URI">

### 只显示包含ZIP文件的数据包

http matches "\.zip" && http.request.method == 'GET'
