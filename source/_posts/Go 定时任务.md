---
title: Go 定时任务
link:
date: 2021-07-06 08:28:50
tags:
---

## 不使用三方库

### 协程Sleep方式

```go
go func() {
    for true {
        fmt.Println("Hello !!")
        time.Sleep(1 * time.Second)
    }
}()
```

### 使用ticker方式1

```go
ticker := time.NewTicker(1 * time.Second)
go func() {
    for range ticker.C {
        fmt.Println("Hello !!")
    }
}()

// wait for 10 seconds
time.Sleep(10 *time.Second)
ticker.Stop()
```

### 使用ticker方式2

```go
done := make(chan bool)
ticker := time.NewTicker(1 * time.Second)

go func() {
    for {
        select {
        case <-done:
            ticker.Stop()
            return
        case <-ticker.C:
            fmt.Println("Hello !!")
        }
    }
}()

// wait for 10 seconds
time.Sleep(10 *time.Second)
done <- true
```

## 参考

https://stackoverflow.com/questions/53057237/how-to-schedule-a-task-at-go-periodically
