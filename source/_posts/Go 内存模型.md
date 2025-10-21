---
title: Go 内存模型
link:
date: 2021-07-02 14:28:45
tags:
---

## 参考资料

https://golang.org/ref/mem

## TLDR

- 协程之间的数据可见性满足HappensBefore法则，并具有传递性
- 如果包 p 导入包 q，则 q 的 init 函数的完成发生在任何 p 的操作开始之前
- main.main 函数的启动发生在所有 init 函数完成之后
- `go`语句启动新的协程发生在新协程启动开始之前
- `go`协程的退出并不保证发生在任何事件之前
- `channel`上的发送发生在对应`channel`接收之前
- 无buffer`channel`的接收发生在发送操作完成之前
- 对于容量为C的buffer channel来说，第k次从channel中接收，发生在第`k + C`次发送完成之前。
- 对于任何的`sync.Mutex`或者`sync.RWMutex`变量`，且有`n<m`，第`n`个调用`UnLock`一定发生在`m`个`Lock`之前。
- 从 once.Do(f) 对 f() 的单个调用返回在任何一个 once.Do(f) 返回之前。
- 如果两个动作不满足HappensBefore，则顺序无法预测

## 介绍

Go内存模型指定了在何种条件下可以保证在一个 goroutine 中读取变量时观察到不同 goroutine 中写入该变量的值。

## 建议

通过多个协程并发修改数据的程序必须将操作序列化。为了序列化访问，通过channel操作或者其他同步原语（`sync`、`sync/atomic`）来保护数据。

如果你必须要阅读本文的其他部分才能理解你程序的行为，请尽量不要这样...

## Happens Before

在单个 `goroutine` 中，读取和写入的行为必须像按照程序指定的顺序执行一样。 也就是说，只有当重新排序不会改变语言规范定义的 goroutine 中的行为时，编译器和处理器才可以重新排序在单个 goroutine 中执行的读取和写入。 由于这种重新排序，一个 goroutine 观察到的执行顺序可能与另一个 goroutine 感知的顺序不同。 例如，如果一个 goroutine 执行 a = 1; b = 2;，另一个可能会在 a 的更新值之前观察到 b 的更新值。

为了满足读写的需求，我们定义了`happens before`，Go程序中内存操作的局部顺序。如果事件`e1`在`e2`之前发生，我们说`e2`在`e1`之后发生。还有，如果`e1`不在`e2`之前发生、`e2`也不在`e1`之前发生，那么我们说`e1`和`e2`并发happen。

在单个`goroutine`中，`happens-before`顺序由程序指定。

当下面两个条件满足时，变量`v`的阅读操作`r`就**可能**观察到写入操作`w`

- `r`不在`w`之前发生
- 没有其他的请求`w2`发生在`w`之后，`r`之前

为了保证`r`一定能阅读到`v`，保证`w`是`r`能观测到的唯一的写操作。当下面两个条件满足时，`r`保证可以读取到`w`

- `w`在`r`之前发生
- 任何其他对共享变量`v`的操作，要么在`w`之前发生，要么在`r`之后发生

这一对条件比上一对条件更强；这要求无论是`w`还是`r`，都没有相应的并发操作。

在单个`goroutine`中，没有并发。所以这两个定义等价：读操作`r`能读到最近一次`w`写入`v`的值。但是当多个`goroutine`访问共享变量时，它们必须使用同步事件来建立`happens-before`关系。

使用变量`v`类型的0值初始化变量`v`的行为类似于内存模型中的写入。

对于大于单个机器字长的值的读取和写入表现为未指定顺序的对多个机器字长的操作。

## 同步

### 初始化

程序初始化在单个 goroutine 中运行，但该 goroutine 可能会创建其他并发运行的 goroutine。

**如果包 p 导入包 q，则 q 的 init 函数的完成发生在任何 p 的操作开始之前。**

**main.main 函数的启动发生在所有 init 函数完成之后。**

### Go协程的创建

**`go`语句启动新的协程发生在新协程启动开始之前。**

举个例子

```go
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

调用`hello`将会打印`hello, world`。当然，这个时候`hello`可能已经返回了。

### Go协程的销毁

**`go`协程的退出并不保证发生在任何事件之前**

```go
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

对 a 的赋值之后没有任何同步事件，因此不能保证任何其他 goroutine 都会观察到它。 事实上，激进的编译器可能会删除整个 go 语句。

如果一个 goroutine 的效果必须被另一个 goroutine 观察到，请使用同步机制，例如锁或通道通信来建立相对顺序。

### 通道通信

通道通信是在go协程之间传输数据的主要手段。在特定通道上的发送总有一个对应的channel的接收，通常是在另外一个协程。

**`channel`上的发送发生在对应`channel`接收之前**

```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

程序能保证输出`hello, world`。对a的写入发生在往`c`发送数据之前，往`c`发送数据又发生在从`c`接收数据之前，它又发生在`print`之前。

**`channel`的关闭发生在从`channel`中获取到0值之前**

在之前的例子中，将`c<-0`替换为`close(c)`，程序还是能保证输出`hello, world`

**无buffer`channel`的接收发生在发送操作完成之前**
这个程序，和之前一样，但是调换发送和接收操作，并且使用无buffer的channel

```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}

func main() {
	go f()
	c <- 0
	print(a)
}
```

也保证能够输出`hello, world`。对a的写入发生在c的接收之前，继而发生在c的写入操作完成之前，继而发生在print之前。

如果该`channel`是buffer`channel`（例如：`c=make(chan int, 1)`），那么程序就不能保证输出`hello, world`。可能会打印空字符串、崩溃等等。从而，我们得到一个相对通用的推论：

**对于容量为C的buffer channel来说，第k次从channel中接收，发生在第`k + C`次发送完成之前。**

此规则将先前的规则推广到缓冲通道。 它允许通过buffer channel 来模拟信号量：通道中的条数对应活跃的数量，通道的容量对应于最大并发数。向channel发送数据相当于获取信号量，从channel中接收数据相当于释放信号量。 这是限制并发的常用习惯用法。

该程序为工作列表中的每个条目启动一个 goroutine，但是 goroutine 使用`limit`channel进行协调，以确保一次最多三个work函数正在运行。

```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

### 锁

`sync`包中实现了两种锁类型：`sync.Mutex`和`sync.RWMutex`

**对于任何的`sync.Mutex`或者`sync.RWMutex`变量`，且有`n<m`，第`n`个调用`UnLock`一定发生在`m`个`Lock`之前。**

```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

这个程序也保证输出`hello,world`。第一次调用`unLock`一定发生在第二次`Lock`调用之前

**对于任何`sync.RWMutex`的`RLock`方法调用，存在变量n，满足`RLock`方法发生在第`n`个`UnLock`调用之后，并且对应的`RUnlock`发生在第`n+1`个`Lock`方法之前。**

### Once

在存在多个 goroutine 时，`sync`包通过`once`提供了一种安全的初始化机制。对于特定的`f`，多个线程可以执行`once.Do(f)`，但是只有一个会运行`f()`，另一个调用会阻塞，直到`f()`返回

**从 once.Do(f) 对 f() 的单个调用返回在任何一个 once.Do(f) 返回之前。**

```go
var a string
var once sync.Once

func setup() {
	a = "hello, world"
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

调用 twoprint 将只调用一次 setup。 `setup`函数将在任一打印调用之前完成。 结果将是`hello, world`打印两次。

## 不正确的同步

注意，读取`r`有可能观察到了由写入`w`并发写入的值。尽管观察到了这个值，也并不意味着`r`后续的读取可以读取到`w`之前的写入。

```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

有可能`g`会接连打印2和0两个值。

双检查锁是为了降低同步造成的开销。举个例子，`twoprint`方法可能会被误写成

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

因为没有任何机制保证，协程观察到done为true的同时可以观测到a为`hello, world`,其中有一个`doprint`可能会输出空字符。

另外一个例子

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

和以前一样，不能保证在 main 中，观察对 done 的写入意味着观察对 a 的写入，因此该程序也可以打印一个空字符串。 更糟糕的情况下，由于两个线程之间没有同步事件，因此无法保证 main 会观察到对 done 的写入。 main 中的循环会一直死循环。

下面是该例子的一个更微妙的变体

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

尽管`main`观测到g不为nil，但是也没有任何机制保证可以读取到t.msg。

在上述例子中，解决方案都是相同的：请使用显式的同步机制。
