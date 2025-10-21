---
title: Go并发范式：超时
link:
date: 2021-06-27 10:40:47
tags:
---

## 翻译自

https://blog.golang.org/concurrency-timeouts

## Go并发范式：超时，继续执行

并发编程有自己的习惯用法。 超时是一个很好的例子。在商用软件开发时，所有操作都需要有超时。

虽然 Go 的**channel**不直接支持超时，但很容易实现。假设我们想从通道 ch 接收，但希望实现一秒钟超时。 我们可以创建一个信号**channel**并启动一个在通道上发送之前休眠的 goroutine

```go
timeout := make(chan bool, 1)
go func() {
    time.Sleep(1 * time.Second)
    timeout <- true
}()
```

我们可以使用**select**语句从`ch`或者`timeout`中接收。如果过了1秒还没有数据返回，超时的case会被选中，尝试从`ch`读取的操作被放弃

```go
select {
case <-ch:
    // a read from ch has occurred
case <-timeout:
    // the read from ch has timed out
}
```

timeout channel有1个buffer，允许超时 goroutine 发送到通道然后退出。 goroutine 不关心这个值是否被接收了。这意味着如果 ch 接收发生在超时之前，goroutine 不会永远挂起。timeout channel 最终会被gc释放。

（在这个例子中，我们使用 time.Sleep 来演示 goroutines 和通道的机制。在实际程序中，你应该使用 [time.After](https://golang.org/pkg/time/#After)来完成这个延迟发送）

让我们看看这种模式的另一种变体。在这个例子中，我们有一个程序可以同时从多个数据库的副本中读取数据。程序只需要一个结果，它应该接受最先返回的结果。

函数 Query 接受多个数据库连接和一个查询字符串。它并行查询每个数据库并返回它收到的第一个响应：

```go
func Query(conns []Conn, query string) Result {
    ch := make(chan Result)
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <-ch
}
```

在这个例子中，闭包执行**非阻塞发送**，它通过在把`send`放在带有`default`的`select`中实现。 如果`send`不能立即完成，则将选择`default`情况。 **非阻塞发送**保证循环中启动的任何 goroutine 都不会挂起。 

但是，这个例子中有一个问题，如果结果在主函数执行到11行接收的时候之前到达，发送都会失败，最终函数无法取得结果。

这个问题是所谓的竞争条件的教科书示例，修复非常简单。 我们只要保证channel有着缓冲通道（通过添加缓冲区长度作为 make 的第二个参数）来保证第一次发送有一个放置值的地方。 这确保发送总是成功，并且无论执行顺序如何，第一个值都会被获取。

这两个例子展示了 Go 可以简单地表达 goroutines 之间复杂的交互。
