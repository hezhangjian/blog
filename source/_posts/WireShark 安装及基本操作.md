---
title: Wireshark 安装及基本操作
link: wireshark-install-basic-op
date: 2018-09-07 21:41:22
tags:
  - Wireshark
---

# WireShark安装

wireshark在windows和mac上的安装方式都比较简单,下面是Linux下的安装方式

```
sudo apt-add-repository ppa:wireshark-dev/stable
sudo apt-get update
sudo apt-get install wireshark
#以root权限启动
sudo wireshark
```

# WireShark的名字解析

![wireshark-name-resolve](Images/wireshark-name-resolve.png)

- L2层的名字解析，对Mac地址进行解析，返回机器名
- L3层 ip解析为域名
- L4层 端口号解析为协议端口号

# Wireshark抓到的包更改时间格式

![wireshark-time-format](Images/wireshark-time-format.png)

# 查看EndPoint

点击Statistics->EndPoints，可以查看每一个捕获文件里的每个端点

![wireshark-endpoint](Images/wireshark-endpoint.png)

# 查看网络会话

Statistics->Conversations. 查看地址A和地址B，以及每个设备发送或收到的数据包和字节数

![wireshark-conversation](Images/wireshark-conversation.png)

# 基于协议分层结构的统计数据

Statistics->Protocol Hierarchy

![wireshark-protocol-hierarchy](Images/wireshark-protocol-hierarchy.png)

# 跟随流功能

右键选中一个数据包，然后右键，follow。比如我在这里跟随一个tcp流

![wireshark-tcp-stream](Images/wireshark-tcp-stream.png)

//这里也可以使用decode as解码功能，但是没有例子，暂不附图

# 查看IO图

Statistics->IO Graphs

![wireshark-io-graph](Images/wireshark-io-graph.png)

# 双向时间图

Statistics->TCP Stream Graph -> Round Trip Time Graph
![wireshark-rtt-graph](Images/wireshark-rtt-graph.png)

# 数据流图

Statistics->Flow Graph
![wireshark-flow-graph](Images/wireshark-flow-graph.png)

# 专家信息

Analyze->Expert Info Composite
![wireshark-expert-info](Images/wireshark-expert-info.png)

## 触发的专家信息

### 对话消息

### 窗口更新 由接收者发送，用来通知发送者TCP接收窗口的大小已被改变

## 注意消息

### TCP重传输 数据包丢失的结果，发生在收到重复的ACK，或者数据包的重传输计时器超时的时候

### 重复ACK 当一台主机没有收到下一个期望序列的数据包时，它会生成最近收到一次数据的重复ACK

### 零窗口探查ACK 用来响应零窗口探查数据包

### 窗口已满 用来通知传输主机及其接收者的TCP接收窗口已满

## 警告消息

### 上一段丢失 指明数据包丢失,发生在当数据流中一个期望的序列号被跳过时。

### 收到丢失数据包的ACK 发生在当一个数据包已经确认丢失但受到了其ACK数据包时

### 保活 当一个连接的保活数据包出现时触发

### 零窗口 当接收方已经达到TCP接收窗口大小时，发出一个零窗口通知，要求发送方停止传输数据

### 乱序 当数据包被乱序接收时，会利用序列号进行检测

### 快速重传输 一次重传会在收到一个重复ACK的20ms内进行

## WireShark性能

### Statistics -> Summary 查看平均速度

### Analyze -> Expert Infos

### Statistics -> TCP StreamGraph -> TCP Sequence Graph(Stenens)

### TCP Previous segment not captured

在TCP传输过程中,同一台主机发出的数据段应该是连续的,即后一个包的Seq号等于前一个包的Seq + Len. 如果在网络包中没有找到,就会出现这个错误

### TCP ACKed unseen segment

Wireshark发现被Ack的那个包没被wireshark捕获

### TCP Out-of-Order

在TCP传输过程中,同一台主机发出的数据段应该是连续的,即后一个包的Seq号等于前一个包的Seq +
Len.当Wireshark发现后一个包的Seq号小于前一个包的Seq+Len 就乱序le

### TCP Dup ACK

当乱序或者丢包的时候,接收方会收到Seq号比期望值大的包,每收到一个这种包就会Ack一次期望的Seq值

### TCP Fast Retransmission

当发送方收到3个或以上TCP Dup ACK,就意识到之前发的包可能丢了,触发快速重传

### TCP Retransmission

没有触发tcp超时重传,超时重传

### TCP zerowindow

缓存区已满,不能再接收数据了

### TCP window FUll

Wireshark检测到,发送方发送的数据会把接收方的接收窗口耗尽
