---
title: 打造可商用的Java程序之可维护性
link:
date: 2023-06-15 20:43:58
tags:
---

## 在主函数中捕获未处理的异常

在主函数中捕获未处理的异常，防止程序崩溃，同时记录日志，方便排查问题。

```java
public class UncaughtExceptionHandle {
    public static void main(String[] args) {
        Thread.setDefaultUncaughtExceptionHandler((t, e) -> log.error("Uncaught exception: ", e));
    }
}
```
