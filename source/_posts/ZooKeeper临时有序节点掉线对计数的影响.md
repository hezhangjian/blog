---
title: ZooKeeper临时有序节点掉线对计数的影响
link:
date: 2020-11-10 23:57:09
tags:
---
# 准备工作
## 运行zookeeper
```
docker run -p 2181:2181 -d ttbb/zookeeper:stand-alone
```
## 代码参考

```
https://github.com/hezhangjian/maven-demo/tree/master/demo-zookeeper/src/main/java/com/github/hezhangjian/demo/zookeeper
```

# 第一次运行代码
创建一个临时有序Znode，程序维持一个小时
```
package com.github.hezhangjian.demo.zookeeper;

import com.github.hezhangjian.javatool.util.CommonUtil;
import com.github.hezhangjian.javatool.util.LogUtil;
import lombok.extern.slf4j.Slf4j;
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.RetryNTimes;
import org.apache.zookeeper.CreateMode;

import java.util.concurrent.TimeUnit;

/**
 * @author hezhangjian
 */
@Slf4j
public class TempOrderTest {

    public static void main(String[] args) throws Exception {
        LogUtil.configureLog();
        RetryPolicy retryPolicy = new RetryNTimes(3, 5000);
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy).build();
        client.start();
        String path = client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath("/seq", "World".getBytes());
        log.info("path is [{}]", path);
        CommonUtil.sleep(TimeUnit.HOURS, 1);
    }

}
```
## 运行第一次
![first-run](Images/zookeeper-temp-seq1.png)
## 运行完之后zk
![after-first-run](Images/zookeeper-temp-seq2.png)


## 第二次运行代码
创建一个临时有序Znode，创建完成后立刻关闭
```
package com.github.hezhangjian.demo.zookeeper;

import com.github.hezhangjian.javatool.util.CommonUtil;
import com.github.hezhangjian.javatool.util.LogUtil;
import lombok.extern.slf4j.Slf4j;
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.RetryNTimes;
import org.apache.zookeeper.CreateMode;

import java.util.concurrent.TimeUnit;

/**
 * @author hezhangjian
 */
@Slf4j
public class TempOrderTest2 {

    public static void main(String[] args) throws Exception {
        LogUtil.configureLog();
        RetryPolicy retryPolicy = new RetryNTimes(3, 5000);
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString("127.0.0.1:2181")
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy).build();
        client.start();
        String path = client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath("/temp/seq", "World".getBytes());
        log.info("path is [{}]", path);
        client.close();
    }

}
```
## 运行第二次结果
![second-run](Images/zookeeper-temp-seq3.png)
## 运行后zk
![after-second-run](Images/zookeeper-temp-seq4.png)
因为是临时节点，所以存在一会后删除

## 第三次仍然运行第一次的代码
## 运行第三次结果
![third-run](Images/zookeeper-temp-seq5.png)
## zk
![after-third-run](Images/zookeeper-temp-seq6.png)


## 第四次，修改了zkPath的值。值得一提的是，同一个ZkPath下似乎共享临时有序节点的最大值。如果修改zkPath从/temp/seq到/temp/seqX,出现的并不是预估的0000,而是0003
![fourth-run](Images/zookeeper-temp-seq7.png)
也就是意味着，zk的临时node序号添加是根据父目录下一个标志计数的
## zk
![after-fourth-run](Images/zookeeper-temp-seq8.png)
