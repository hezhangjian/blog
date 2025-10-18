---
title: 多语言编程 各大Http库配置指南
link: multi-lang-http-client-config
date: 2023-11-18 17:24:58
tags:
  - Code
---

# Go

## Go标准库

### timeout

```go
client := http.Client{
    Timeout: timeout,
}
```

### connection timeout

```go
client := http.Client{
    Transport: &http.Transport{
        Dial: (&net.Dialer{
            Timeout: timeout,
        }).Dial,
    },
}
```

# Java

## 标准库(jdk17+)

### timeout

```java
    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("http://example.com"))
        .timeout(Duration.ofSeconds(10))
        .build();
```

### connectionTimeout

```java
        HttpClient.Builder builder = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(10))
        .version(HttpClient.Version.HTTP_1_1);
```

## Reactor Netty

### timeout

```java
        HttpClient client = HttpClient.create().responseTimeout(Duration.ofSeconds(10));
```

### connectionTimeout

```java
        HttpClient client = HttpClient.create().option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);
```
