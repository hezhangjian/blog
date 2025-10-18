---
title: java指标统计方案及代码
link: java-metrics-statistic
date: 2021-08-10 21:30:18
tags:
  - Java
---

# java 根据线程统计CPU

## 设计思路

java的ThreadMXBean可以获取每个线程CPU执行的nanoTime，那么可以以这个为基础，除以中间系统经过的纳秒数，就获得了该线程的`CPU`占比

## 编码

首先，我们定义一个结构体，用来存放一个线程上次统计时的纳秒数和当时的系统纳秒数

```java
import lombok.Data;

@Data
public class ThreadMetricsAux {

    private long usedNanoTime;

    private long lastNanoTime;

    public ThreadMetricsAux() {
    }

    public ThreadMetricsAux(long usedNanoTime, long lastNanoTime) {
        this.usedNanoTime = usedNanoTime;
        this.lastNanoTime = lastNanoTime;
    }
    
}
```

然后我们在SpringBoot中定义一个定时任务，它将定时地统计计算每个线程的CPU信息，并输出到`MeterRegistry`，当你调用`SpringActuator`的接口时，你将能获取到这个指标。

```java
import com.google.common.util.concurrent.AtomicDouble;
import io.micrometer.core.instrument.Meter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Tags;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;
import java.util.HashMap;

@Slf4j
@Service
public class ThreadMetricService {

    @Autowired
    private MeterRegistry meterRegistry;

    private final ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();

    private final HashMap<Long, ThreadMetricsAux> map = new HashMap<>();

    private final HashMap<Meter.Id, AtomicDouble> dynamicGauges = new HashMap<>();

    /**
     * one minutes
     */
    @Scheduled(cron = "0 * * * * ?")
    public void schedule() {
        final long[] allThreadIds = threadBean.getAllThreadIds();
        for (long threadId : allThreadIds) {
            final ThreadInfo threadInfo = threadBean.getThreadInfo(threadId);
            if (threadInfo == null) {
                continue;
            }
            final long threadNanoTime = getThreadCPUTime(threadId);
            if (threadNanoTime == 0) {
                // 如果threadNanoTime为0，则识别为异常数据，不处理，并清理历史数据
                map.remove(threadId);
            }
            final long nanoTime = System.nanoTime();
            ThreadMetricsAux oldMetrics = map.get(threadId);
            // 判断是否有历史的metrics信息
            if (oldMetrics != null) {
                // 如果有，则计算CPU信息并上报
                double percent = (double) (threadNanoTime - oldMetrics.getUsedNanoTime()) / (double) (nanoTime - oldMetrics.getLastNanoTime());
                handleDynamicGauge("jvm.threads.cpu", "threadName", threadInfo.getThreadName(), percent);
            }
            map.put(threadId, new ThreadMetricsAux(threadNanoTime, nanoTime));
        }
    }

    // meter Gauge相关代码
    private void handleDynamicGauge(String meterName, String labelKey, String labelValue, double snapshot) {
        Meter.Id id = new Meter.Id(meterName, Tags.of(labelKey, labelValue), null, null, Meter.Type.GAUGE);

        dynamicGauges.compute(id, (key, current) -> {
            if (current == null) {
                AtomicDouble initialValue = new AtomicDouble(snapshot);
                meterRegistry.gauge(key.getName(), key.getTags(), initialValue);
                return initialValue;
            } else {
                current.set(snapshot);
                return current;
            }
        });
    }

    long getThreadCPUTime(long threadId) {
        long time = threadBean.getThreadCpuTime(threadId);
        /* thread of the specified ID is not alive or does not exist */
        return time == -1 ? 0 : time;
    }

}

```

## 其他配置

### 依赖配置

`pom`文件中
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
```

### Prometheus接口配置

`application.yaml`中
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

## 效果

通过`curl`命令调用`curl localhost:20001/actuator/prometheus|grep cpu`

```
jvm_threads_cpu{threadName="RMI Scheduler(0)",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-10",} 0.0
jvm_threads_cpu{threadName="Signal Dispatcher",} 0.0
jvm_threads_cpu{threadName="Common-Cleaner",} 3.1664628758074733E-7
jvm_threads_cpu{threadName="http-nio-20001-Poller",} 7.772143763853949E-5
jvm_threads_cpu{threadName="http-nio-20001-Acceptor",} 8.586978352515361E-5
jvm_threads_cpu{threadName="DestroyJavaVM",} 0.0
jvm_threads_cpu{threadName="Monitor Ctrl-Break",} 0.0
jvm_threads_cpu{threadName="AsyncHttpClient-timer-8-1",} 2.524386571545477E-4
jvm_threads_cpu{threadName="Attach Listener",} 0.0
jvm_threads_cpu{threadName="scheduling-1",} 1.2269694160981585E-4
jvm_threads_cpu{threadName="container-0",} 1.999795692406262E-6
jvm_threads_cpu{threadName="http-nio-20001-exec-9",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-7",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-8",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-5",} 0.0
jvm_threads_cpu{threadName="Notification Thread",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-6",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-3",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-4",} 0.0
jvm_threads_cpu{threadName="Reference Handler",} 0.0
jvm_threads_cpu{threadName="http-nio-20001-exec-1",} 0.0012674719289349648
jvm_threads_cpu{threadName="http-nio-20001-exec-2",} 6.542541277148053E-5
jvm_threads_cpu{threadName="RMI TCP Connection(idle)",} 1.3998786340454562E-6
jvm_threads_cpu{threadName="Finalizer",} 0.0
jvm_threads_cpu{threadName="Catalina-utility-2",} 7.920883054498174E-5
jvm_threads_cpu{threadName="RMI TCP Accept-0",} 0.0
jvm_threads_cpu{threadName="Catalina-utility-1",} 6.80101662787773E-5
```

# Java计算磁盘使用率

https://support.huaweicloud.com/bestpractice-bms/bms_bp_2009.html

华为云文档上的材料值得学习。

翻阅资料

```
https://www.kernel.org/doc/Documentation/ABI/testing/procfs-diskstats

13 - time spent doing I/Os (ms)
```

这就意味着如果我想统计一个磁盘在一定周期内的利用率，只需要对这两个数字做差，除以统计的间隔，即就是这段时间内磁盘的利用率

```
cat /proc/diskstats
 253       0 vda 24046 771 2042174 180187 20689748 21411881 527517532 18028256 0 14610513 18201352
 253       1 vda1 23959 771 2038022 180153 20683957 21411881 527517532 18028066 0 14610312 18201129
```

样例代码

```
package com.github.hezhangjian.demo.metrics;

import com.github.hezhangjian.demo.base.module.ShellResult;
import com.github.hezhangjian.demo.base.util.LogUtil;
import com.github.hezhangjian.demo.base.util.ShellUtil;
import com.github.hezhangjian.demo.base.util.StringUtil;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * @author hezhangjian
 */
@Slf4j
public class DiskUtilizationMetrics {

    private static final ScheduledExecutorService scheduledExecutor = Executors.newSingleThreadScheduledExecutor();

    private static long lastTime = -1;

    public static void main(String[] args) {
        LogUtil.configureLog();
        String diskName = "vda1";
        scheduledExecutor.scheduleAtFixedRate(() -> metrics(diskName), 0, 10, TimeUnit.SECONDS);
    }

    private static void metrics(String diskName) {
        //假设统计vda磁盘
        String[] cmd = {
                "/bin/bash",
                "-c",
                "cat /proc/diskstats |grep " + diskName + "|awk '{print $13}'"
        };
        ShellResult shellResult = ShellUtil.executeCmd(cmd);
        String timeStr = shellResult.getInputContent().substring(0, shellResult.getInputContent().length() - 1);
        long time = Long.parseLong(timeStr);
        if (lastTime == -1) {
            log.info("first time cal, usage time is [{}]", time);
        } else {
            double usage = (time - lastTime) / (double) 10_000;
            log.info("usage time is [{}]", usage);
        }
        lastTime = time;
    }

}

```

# 打印CPU使用

```java
private static void printCpuUsage() {
        final com.sun.management.OperatingSystemMXBean platformMXBean = ManagementFactory.getPlatformMXBean(com.sun.management.OperatingSystemMXBean.class);
        double cpuLoad = platformMXBean.getProcessCpuLoad();
        System.out.println(cpuLoad);
    }
```

# 打印线程堆栈

```java
private static void printThreadDump() {
        final StringBuilder dump = new StringBuilder();
        final ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        // 100代表线程堆栈的层级
        final ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(threadMXBean.getAllThreadIds(), 100);
        for (ThreadInfo threadInfo : threadInfos) {
            dump.append('"');
            dump.append(threadInfo.getThreadName());
            dump.append("\" ");
            final Thread.State state = threadInfo.getThreadState();
            dump.append("\n   java.lang.Thread.State: ");
            dump.append(state);
            final StackTraceElement[] stackTraceElements = threadInfo.getStackTrace();
            for (final StackTraceElement stackTraceElement : stackTraceElements) {
                dump.append("\n        at ");
                dump.append(stackTraceElement);
            }
            dump.append("\n\n");
        }
        System.out.println(dump);
    }
```

# 打印内存统计信息

引入依赖

```xml
 <dependency>
            <groupId>com.jerolba</groupId>
            <artifactId>jmnemohistosyne</artifactId>
            <version>0.2.3</version>
        </dependency>
```

```java
 private static void printClassHisto() {
        Histogramer histogramer = new Histogramer();
        MemoryHistogram histogram = histogramer.createHistogram();

        HistogramEntry arrayList = histogram.get("java.util.ArrayList");
        System.out.println(arrayList.getInstances());
        System.out.println(arrayList.getSize());

        for (HistogramEntry entry : histogram) {
            System.out.println(entry);
        }
    }
```

# 打印死锁

javadoc中指出，这是一个开销较大的操作

```java
 private static void printDeadLock() {
        final ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        final long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        for (long deadlockedThread : deadlockedThreads) {
            final ThreadInfo threadInfo = threadMXBean.getThreadInfo(deadlockedThread);
            System.out.println(threadInfo + "deadLocked");
        }
    }
```
