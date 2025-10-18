---
title: WebFlux最佳实践
link: webflux-best-practice
date: 2024-06-08 15:50:57
tags:
  - Java
  - Spring
  - WebFlux
---

WebFlux是Spring 5引入的新的响应式编程框架，它提供了一种基于反应式流的编程模型，可以用于构建高性能、高吞吐量的Web应用程序。

# 防止大量请求堆积

## 限制同一时间的并发处理个数

由于WebFlux可以处理大量的请求，如果后端处理较慢（如写db较慢等），可能会导致大量的请求堆积，可以通过限制同一时间的并发处理个数来防止请求堆积。

```java
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

import java.util.concurrent.Semaphore;

@Component
public class ConcurrencyLimitingFilter implements WebFilter {
    private final Semaphore semaphore;

    public ConcurrencyLimitingFilter() {
        this.semaphore = new Semaphore(10);
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        if (semaphore.tryAcquire()) {
            return chain.filter(exchange)
                    .doFinally(sig -> semaphore.release());
        } else {
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }
    }
}

```

## 配置超时时间

网络编程中，任何操作都应该有超时时间。WebFlux允许大量的请求进入，如果不设置超时时间，可能会导致大量的请求排队处理（可能客户端早已放弃），可以通过统一Filter来设置最大超时时间。

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ResponseStatusException;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.concurrent.TimeoutException;

@Slf4j
@Component
public class WebRequestTimeoutFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        return chain.filter(exchange)
                .timeout(Duration.ofSeconds(10))
                .onErrorResume(TimeoutException.class, e -> {
                    log.error("Request timeout", e);
                    return Mono.error(new ResponseStatusException(HttpStatus.GATEWAY_TIMEOUT, "Request timeout"));
                });
    }
}
```
