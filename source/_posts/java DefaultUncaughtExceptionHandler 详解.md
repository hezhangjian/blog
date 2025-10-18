---
title: Java DefaultUncaughtExceptionHandler 详解
link: java-default-uncaught-exception-handler
date: 2023-06-15 21:41:49
tags:
  - Java
---
在Java程序运行时，一些非受检异常可能会导致程序崩溃，比如NullPointerException、ArrayIndexOutOfBoundsException等等，这些异常都是由JVM抛出的，如果不对这些异常进行处理，小则线程运行中突然退出，大则整个程序崩溃。理想的场景下，每一个非受检异常都应该被捕获并进行处理，但是在实际开发中，我们往往会忽略一些异常，这些异常可能是由于程序员的疏忽导致的，也可能是由于程序员无法预知的原因导致的，比如第三方库抛出的异常。

为了避免这些异常导致程序崩溃，Java提供了一个全局的异常处理器，即DefaultUncaughtExceptionHandler，它可以捕获所有未被捕获的异常，从而避免程序崩溃。

DefaultUncaught的使用示例如下：

```java
public class UncaughtExceptionHandle {
    public static void main(String[] args) {
        Thread.setDefaultUncaughtExceptionHandler((t, e) -> log.error("Uncaught exception: ", e));
    }
}
```

上述的代码会将未捕获的异常打印到日志中，如果你希望打印至标准输出或标准输出，可以将log替换为：

```java
// 标准输出
System.out.println("Uncaught exception: " + e);
// 错误输出
System.err.println("Uncaught exception: " + e);
```
