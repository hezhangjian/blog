---
title: BookKeeper 持久化文件解析
link: bookkeeper-persistent-file
date: 2021-02-16 12:19:19
tags:
  - BookKeeper
---

# Entry Log File

## 背景

测试环境上出现了一些entryLog解析异常的问题，想分析一下磁盘上.log文件的格式，分析分析我们的文件是否有问题

## 解析代码地址

https://github.com/protocol-laboratory/bookkeeper-codec-java/blob/main/src/main/java/com/github/protocol/EntryLogReader.java

## 正文

我们采用的配置是singleEntryLog模式，就是说很多ledger的信息都会放在一个log文件内部。

插一句话：这种log文件，其实和LSM相似，属于不可变的数据结构，这种数据结构，得益于不可变，所以内容可以安排的非常紧凑，不像B树结构，需要预留一定空间给原地更新，随机插入等。

![bookkeeper-entry-log-format](Images/bookkeeper-entry-log-format.png)

如上图所示，接下来，我们沿着解析的流程，解读每个部分的详细格式

### 解析头部

首先，我们解析文件的头部字段，bookkeeper的设计中，文件头部预留了1024字节，目前只使用了20个字节
前四个字节是**BKLO**的文件魔数
然后紧跟着的4个字节是bk文件的版本号，这里我们仅分析版本号1
然后8字节的long类型代表**ledgersMap**的开始位置，称为**ledgersMapOffset**。
然后4字节的int类型代表**ledgersMap**的总长度。

### 解析ledgerMap部分

最前面四个字节，代表这部分的大小

然后开始的ledgerId和entryId分别为-1，-2，随后是一个ledger的count大小，后面的ledgerId和size才是有效值

随后的部分非常紧凑，由一个个ledgerId，size组成

读取完ledgerMap，可以知道，这个文件包含了多少ledger，总大小是多少？

注：size代表这一段ledger占用的磁盘空间大小

### 解析body内容

body内容也非常紧凑.
最前面4个字节，代表这个entry的大小。
然后8个字节，ledgerId
然后8个字节，entryId
剩下的内容，就是pulsar写数据的编码，不再属于bookkeeper的格式范畴了

# Txn Log File

## 解析代码地址

https://github.com/protocol-laboratory/bookkeeper-codec-java/blob/main/src/main/java/com/github/protocol/TxnLogReader.java

## 简述

bookkeeper中的journal log，和大部分基于LSM的数据结构一样，是用来保证文件一定被写入的。会在数据写入的时候，写入journal log，崩溃恢复的时候从journal log里面恢复。

![bookkeeper-txn-log-format](Images/bookkeeper-txn-log-format.png)

### 解析头部
首先，我们解析文件的头部字段
前四个字节是BKLG的文件魔数
然后紧跟着的4个字节是bk文件的版本号

```java
private TxnHeader readHeader(FileChannel fileChannel) throws Exception {
    final ByteBuf headers = Unpooled.buffer(HEADER_SIZE);
    final int read = fileChannel.read(headers.internalNioBuffer( index: 0, HEADER_SIZE));
    headers.writerIndex(read);
    final byte[] bklgByte = new byte[4];
    headers.readBytes(bklgByte, dstIndex: 0, length: 4);
    final int headerVersion = headers.readInt();
    return new TxnHeader(headerVersion);
}
```

### 解析内容
内容非常紧凑，由ledgerId，entryId和内容组成。ledgerId一定大于0，entryId在小于0的情况下代表特殊的数据。如

- -0x1000即4096 代表ledger的masterKey
- -0x2000即8192 代表ledger是否被fence
- -0x4000即16384 代表ledger的force
- -0x8000即32768 代表ledger的显示LAC

## 回放流程
当bookkeeper启动的时候，他会从data路径下取得lastMark文件，该文件一定为16个字节，前八个字节代表落盘的最新journal log文件，后八个字节代表文件的位置。会从这个位置开始回放。
值得一提的是，lastId文件，代表下一个dataLog该使用什么文件名。
