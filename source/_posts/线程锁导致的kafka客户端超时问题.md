---
title: 线程锁导致的kafka客户端超时问题
link:
date: 2023-07-08 14:58:52
tags:
  - Kafka
---

## 问题背景

有一个环境的kafka client发送数据有部分超时，拓扑图也非常简单

![Untitled](Images/20230708p1.png)

## 定位历程

我们先对客户端的环境及JVM情况进行了排查，从JVM所在的虚拟机到kafka server的网络正常，垃圾回收（GC）时间也在预期范围内，没有出现异常。

紧接着，我们把目光转向了kafka 服务器，进行了一些基础的检查，同时也查看了kafka处理请求的超时日志，其中我们关心的metadata和produce请求都没有超时。

问题就此陷入了僵局，虽然也搜到了一些kafka server会对连上来的client反解导致超时的问题（ [https://github.com/apache/kafka/pull/10059](https://github.com/apache/kafka/pull/10059)），但通过一些简单的分析，我们确定这并非是问题所在。

同时，我们在环境上也发现一些异常情况，当时觉得不是核心问题/解释不通，没有深入去看

- 问题JVM线程数较高，已经超过10000，这个线程数量虽然确实较高，但并不会对1个4U的容器产生什么实质性的影响。
- 负责指标上报的线程CPU较高，大约占用了1/4 ~ 1/2 的CPU核，这个对于4U的容器来看问题也不大

当排查陷入僵局，我们开始考虑其他可能的调查手段。我们尝试抓包来找线索，这里的抓包是SASL鉴权+SSL加密的，非常难读，只能靠长度和响应时间勉强来推断报文的内容。

在这个过程中，我们发现了一个非常重要的线索，客户端竟然发起了超时断链，并且超时的那条消息，实际服务端是有响应回复的。

随后我们将kafka client的trace级别日志打开，这里不禁感叹kafka client日志打的相对较少，发现的确有**log.debug("Disconnecting from node {} due to request timeout.", nodeId)**;的日志打印。

与网络相关的流程：

```java
try {
    // 这里发出了请求
    client.send(request, time.milliseconds());
    while (client.active()) {
        List<ClientResponse> responses = client.poll(Long.MAX_VALUE, time.milliseconds());
        for (ClientResponse response : responses) {
            if (response.requestHeader().correlationId() == request.correlationId()) {
                if (response.wasDisconnected()) {
                    throw new IOException("Connection to " + response.destination() + " was disconnected before the response was read");
                }
                if (response.versionMismatch() != null) {
                    throw response.versionMismatch();
                }
                return response;
            }
        }
    }
    throw new IOException("Client was shutdown before response was read");
} catch (DisconnectException e) {
    if (client.active())
        throw e;
    else
        throw new IOException("Client was shutdown before response was read");

}
```

这个poll方法，不是简单的poll方法，而在poll方法中会进行超时判断，查看poll方法中调用的handleTimedOutRequests方法

```java
@Override
public List<ClientResponse> poll(long timeout, long now) {
    ensureActive();

    if (!abortedSends.isEmpty()) {
        // If there are aborted sends because of unsupported version exceptions or disconnects,
        // handle them immediately without waiting for Selector#poll.
        List<ClientResponse> responses = new ArrayList<>();
        handleAbortedSends(responses);
        completeResponses(responses);
        return responses;
    }

    long metadataTimeout = metadataUpdater.maybeUpdate(now);
    try {
        this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));
    } catch (IOException e) {
        log.error("Unexpected error during I/O", e);
    }

    // process completed actions
    long updatedNow = this.time.milliseconds();
    List<ClientResponse> responses = new ArrayList<>();
    handleCompletedSends(responses, updatedNow);
    handleCompletedReceives(responses, updatedNow);
    handleDisconnections(responses, updatedNow);
    handleConnections();
    handleInitiateApiVersionRequests(updatedNow);
    // 关键的超时判断
    handleTimedOutRequests(responses, updatedNow);
    completeResponses(responses);

    return responses;
}
```

由此我们推断，问题可能在于客户端hang住了一段时间，从而导致超时断链。我们通过工具Arthas深入跟踪了Kafka的相关代码，甚至发现一些简单的操作（如A.field）也需要数秒的时间。这进一步确认了我们的猜想：问题可能出在JVM。JVM可能在某个时刻出现问题，导致系统hang住，但这并非由GC引起。

![Untitled](Images/20230708p2.png)

为了解决这个问题，我们又检查了监控线程CPU较高的问题。我们发现线程的执行热点是从"sun.management.ThreadImpl"中的"getThreadInfo"方法。

```
"metrics-1@746" prio=5 tid=0xf nid=NA runnable
  java.lang.Thread.State: RUNNABLE
    at sun.management.ThreadImpl.getThreadInfo(Native Method)
	  at sun.management.ThreadImpl.getThreadInfo(ThreadImpl.java:185)
	  at sun.management.ThreadImpl.getThreadInfo(ThreadImpl.java:149)
```

进一步发现，在某些版本的JDK8中，读取线程信息是需要加锁的。

至此，问题的根源已经清晰明了：过高的线程数以及线程监控时JVM全局锁的存在导致了这个问题。您可以使用如下的demo来复现这个问题

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ThreadLockSimple {
    public static void main(String[] args) {
        for (int i = 0; i < 15_000; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(200_000);
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            }).start();
        }
        ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor();
        executorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                System.out.println("take " + " " + System.currentTimeMillis());
            }
        }, 1, 1, TimeUnit.SECONDS);
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ScheduledExecutorService metricsService = Executors.newSingleThreadScheduledExecutor();
        metricsService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                ThreadInfo[] threadInfoList = threadMXBean.getThreadInfo(threadMXBean.getAllThreadIds());
                System.out.println("threads count " + threadInfoList.length + " cost :" + (System.currentTimeMillis() - start));
            }
        }, 1, 1, TimeUnit.SECONDS);
    }
}
```

为了解决这个问题，我们有以下几个可能的方案：

- 将不合理的线程数往下降，可能存在线程泄露的场景
- 升级jdk到jdk11或者jdk17（推荐）
- 将Thread相关的监控临时关闭

这个问题的解决方案应根据实际情况进行选择，希望对你有所帮助。
