---
title: Go Unsafe包
link:
date: 2021-07-09 22:46:32
tags:
---

## 翻译自

https://medium.com/a-journey-with-go/go-what-is-the-unsafe-package-d2443da36350

## 正文

包的名称指引着我们的使用。了解此包可能不安全的原因，让我们首先查看如下文档

```
unsafe包包含可以绕过Go程序类型安全的操作
引用了unsafe的包，可能会无法跨平台、跨设备，并且Go1 兼容性指南不保证这些包的兼容
```

因此，该名称被用作对为 Go 提供类型的安全性的反面。 现在让我们深入研究文档中提到的这两点。

## 类型安全

在Go中，每个变量都拥有一个类型，在赋值给另外一个变量之前，可以转换为另外的类型。在此转换期间，Go 执行此数据的转换以适应所请求的类型。 这是一个例子

```go
package unsafe_test

import "testing"

func TestConvert(t *testing.T) {
	// -1 二进制表达 11111111
	var i int8 = -1
	// -1 二进制表达 11111111 11111111
	var j = int16(i)
	println(i, j)
}
```

输出

```
-1 -1
```

`unsafe`包让我们直接操作这个变量的内存地址，直接获取里面的二进制值。在绕过类型约束时，我们可以随意使用它。

```go
	var k uint8 = *(*uint8)(unsafe.Pointer(&i))
	println(k)
	// 255 is the uint8 value for the binary 11111111
```

## Go1 兼容性指南

unsafe包可能会依赖于Go的底层实现。Go可能会做出破坏性的改动。

## Go 反射包中使用Unsafe

`reflect`包是使用最多的一个包。反射基于空interface包含的数据。为了读取数据，Go只是将我们的变量转化为一个空接口，并通过映射一个结构体来读取它们，该结构体匹配空接口的内部表示与指针地址处的内存。

```go
func ValueOf(i interface{}) Value {
   [...]
   return unpackEface(i)
}
// unpackEface converts the empty interface i to a Value.
func unpackEface(i interface{}) Value {
   e := (*emptyInterface)(unsafe.Pointer(&i))
   [...]
}
```

变量 e 现在包含有关该值的所有信息，例如类型或该值是否已导出。 反射还使用 unsafe 包通过直接在内存中更新值来修改反射变量的值。

## Go Sync包使用Unsafe

`sync.Pool` 这些池通过一段go协程都能访问的内存片段来共享数据。这访问模式和C语言数组的访问方式很像

```go
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
   lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
   return (*poolLocal)(lp)
}
```

`l`是内存片段，`i`是数字。函数 `indexLocal` 只是读取这个内存段——它包含 X个poolLocal 结构——与它读取的索引相关的偏移量。 存储指向完整内存段的单个指针是实现共享池的一种非常轻量的方式。

## 在Go runtime包中的使用

Go 在`runtime`包也大量使用 unsafe 包，因为它必须处理内存操作，如栈分配或释放栈内存。 栈由其结构中的两个边界表示：

```go
type stack struct {
    lo uintptr
    hi uintptr
}
```

`unsafe`包可以完成这个操作

```go
func stackfree(stk stack) {
    [...]
    v := unsafe.Pointer(stk.lo)
    n := stk.hi - stk.lo
    // 内存基于栈上的指针释放
    [...]
}
```

## 开发平时使用

`unsafe`包的一个很好的用法是转换具有相同底层数据结构的不同结构体，这是转换器无法实现的

```go
package unsafe

import (
	"testing"
	"unsafe"
)

type A struct {
	A int8
	B string
	C float32
}

type B struct {
	D int8
	E string
	F float32
}

func TestStructConvert(t *testing.T) {
	a := A{A: 1, B: `foo`, C: 1.23}
	b := *(*B)(unsafe.Pointer(&a))
	println(b.D, b.E, b.F) // 1 foo 1.23
}
```
