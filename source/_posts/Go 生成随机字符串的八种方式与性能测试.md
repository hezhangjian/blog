---
title: Go 生成随机字符串的八种方式与性能测试
link:
date: 2021-06-20 13:57:42
tags:
---

## 前言

这是**[icza](https://stackoverflow.com/users/1705598/icza)**
在StackOverflow上的一篇[回答](https://stackoverflow.com/questions/22892120/how-to-generate-a-random-string-of-a-fixed-length-in-go/22892986#22892986)
，质量很高，翻译一下，大家一起学习

问题是：在go，有没有什么最快最简单的方法，用来生成只包含英文字母的随机字符串

icza给出了8个方案，最简单的方法并不是最快的方法，它们各有优劣，末尾附上性能测试结果：

### 1. Runes

比较简单的答案，声明一个rune数组，通过随机数选取rune字符，拼接成结果

```go
package approach1

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")

func randStr(n int) string {
	b := make([]rune, n)
	for i := range b {
		b[i] = letters[rand.Intn(len(letters))]
	}
	return string(b)
}

func TestApproach1(t *testing.T) {
	rand.Seed(time.Now().UnixNano())
	fmt.Println(randStr(10))
}

func BenchmarkApproach1(b *testing.B) {
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < b.N; i++ {
		_ = randStr(10)
	}
}
```

### 2. Bytes

如果随机挑选的字符只包含英文字母，我们可以直接使用bytes，因为在UTF-8编码模式下，英文字符和Bytes是一对一的（Go正是使用UTF-8模式编码）

所以可以把

```go
var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")
```

用这个替代

```go
var letters = []byte("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")
```

或者更好

```go
const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
```

现在我们有很大的进展了，我们把它变为了一个常数，在go里面，只有string常数，可并没有slice常数。额外的收获，表达式`len(letters)`
也变为了一个常数（如果`s`为常数，那么`len(s)`也将是常数)

我们没有付出什么代码，现在`letters`可以通过下标访问其中的bytes了，这正是我们需要的。

```go
package approach2

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func randStr(n int) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = letters[rand.Intn(len(letters))]
	}
	return string(b)
}

func TestApproach2(t *testing.T) {
	rand.Seed(time.Now().UnixNano())

	fmt.Println(randStr(10))
}

func BenchmarkApproach2(b *testing.B) {
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < b.N; i++ {
		_ = randStr(10)
	}
}
```

### 3. Remainder

上面的解决方法通过`rand.Intn()`来获得一个随机字母，这个方法底层调用了`Rand.Intn()`，然后调用了`Rand.Int31n()`

相比于生成63个随机bits的函数`rand.Int63()`来说，`Rand.Int31n()`很慢。

我们可以简单地调用`rand.Int63()`然后除以`len(letterBytes)`，使用它的余数来生成字母

```go
package approach3

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func randStr(n int) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = letters[rand.Int63() % int64(len(letters))]
	}
	return string(b)
}

func TestApproach3(t *testing.T) {
	rand.Seed(time.Now().UnixNano())

	fmt.Println(randStr(10))
}

func BenchmarkApproach3(b *testing.B) {
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < b.N; i++ {
		_ = randStr(10)
	}
}

```

这个算法能正常工作并且非常快，不过它牺牲了部分精确性，字母出现的概率并不是精确一样的（假设`rand.Int63()`
生成63比特的数字是等概率的）。由于字母总共才52个，远小于 1<<63 - 1，因此失真非常小，因此实际上这完全没问题。

解释: 假设你想要0~5的随机数，如果使用3位的bit，3位的bit等概率出现0~7，所以出现0和1的概率是出现2、3、4概率的两倍。使用5位的
bit，0和1出现的概率是`6/32`，2、3、4出现的概率是`5/32`。现在接近了一些了，是吧？不断地增加比特位，这个差距就会变得越小，当你有63位地时候，这差别已经可忽略不计。

### 4. Masking

在上一个方案的基础上，我们通过仅使用随机数的最低n位保持均匀分布，n为表示所有字符的数量。比如我们有52个字母，我们需要6位（52 =
110100b）。所以我们仅仅使用了`rand.Int63()`的最后6位。并且，为了保持所有字符的均匀分布，我们决定只接受在
`0..len(letterBytes)-1`的数字即0~51。（译者注：这里已经没有第三个方案的不准确问题了）

最低几位大于等于`len(letterBytes)`的概率一般小于`0.5`
（平均值为0.25），这意味着出现这种情况，只要重试就好。重试n次之后，我们仍然需要丢弃这个数字的概率远小于0.5的n次方（这是上界了，实际会低于这个值）。以本文的52个字母为例，最低6位需要丢弃的概率只有
`(64-52)/64=0.19`。这意味着，重复10次，仍然没有数字的概率是1*10^-8。

```go
package approach4

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

const (
	// 6 bits to represent a letters index
	letterIdBits = 6
	// All 1-bits as many as letterIdBits
	letterIdMask = 1 <<letterIdBits - 1
)

func randStr(n int) string {
	b := make([]byte, n)
	for i := range b {
		if idx := int(rand.Int63() & letterIdMask); idx < len(letters) {
			b[i] = letters[idx]
			i++
		}
	}
	return string(b)
}

func TestApproach4(t *testing.T) {
	rand.Seed(time.Now().UnixNano())

	fmt.Println(randStr(10))
}

func BenchmarkApproach4(b *testing.B) {
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < b.N; i++ {
		_ = randStr(10)
	}
}
```

### 5. Masking Improved

第4节的方案只使用了`rand.Int63()`方法返回的64个随机字节的后6位。这实在是太浪费了，因为`rand.Int63()`是我们算法中最耗时的部分了。

如果我们有52个字母，6位就能生成一个随机字符串。所以63个随机字节，可以利用`63/6=10`次。

译者注：使用了缓存，缓存了`rand.Int63()`方法返回的内容，使用10次，不过已经并不是协程安全的了。

```go
package approach5

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

const (
	// 6 bits to represent a letter index
	letterIdBits = 6
	// All 1-bits as many as letterIdBits
	letterIdMask = 1<<letterIdBits - 1
	letterIdMax  = 63 / letterIdBits
)

func randStr(n int) string {
	b := make([]byte, n)
	// A rand.Int63() generates 63 random bits, enough for letterIdMax letters!
	for i, cache, remain := n-1, rand.Int63(), letterIdMax; i >= 0; {
		if remain == 0 {
			cache, remain = rand.Int63(), letterIdMax
		}
		if idx := int(cache & letterIdMask); idx < len(letters) {
			b[i] = letters[idx]
			i--
		}
		cache >>= letterIdBits
		remain--
	}
	return string(b)
}

func TestApproach5(t *testing.T) {
	rand.Seed(time.Now().UnixNano())

	fmt.Println(randStr(10))
}

func BenchmarkApproach5(b *testing.B) {
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < b.N; i++ {
		_ = randStr(10)
	}
}

```

### 6. Source

第5个方案非常好，能改进的点并不多。我们可以但不值得搞得很复杂。

让我们来找可以改进的点：**随机数的生成源**

`crypto/rand`的包提供了`Read(b []byte)`的函数，可以通过这个函数获得需要的随机比特数，只需要一次调用。不过并不能提升性能，因为
`crypto/rand`实现了一个密码学上的安全伪随机数，所以速度比较慢。

所以让我们坚持使用`math/rand`包，`rand.Rand`使用`rand.Source`作为随机位的来源，`rand.Source`是一个声明了`Int63() int64`
的接口：正是我们在最新解决方案中需要和使用的唯一方法。

所以我们不是真的需要`rand.Rand`，`rand.Source`包对于我们来说已经足够了

```go
package approach6

import (
	"fmt"
	"math/rand"
	"testing"
	"time"
)

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

var src = rand.NewSource(time.Now().UnixNano())

const (
	// 6 bits to represent a letter index
	letterIdBits = 6
	// All 1-bits as many as letterIdBits
	letterIdMask = 1<<letterIdBits - 1
	letterIdMax  = 63 / letterIdBits
)

func randStr(n int) string {
	b := make([]byte, n)
	// A rand.Int63() generates 63 random bits, enough for letterIdMax letters!
	for i, cache, remain := n-1, src.Int63(), letterIdMax; i >= 0; {
		if remain == 0 {
			cache, remain = src.Int63(), letterIdMax
		}
		if idx := int(cache & letterIdMask); idx < len(letters) {
			b[i] = letters[idx]
			i--
		}
		cache >>= letterIdBits
		remain--
	}
	return string(b)
}

func TestApproach6(t *testing.T) {
	fmt.Println(randStr(10))
}

func BenchmarkApproach6(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = randStr(10)
	}
}
```

注意到这里我们没有使用种子初始化rand了，取而代之的是初始化了`rand.Source`

还有一件需要注意的事，`math/rand`的文档指出

```
默认的Source是协程安全的
```

所以默认的Source比通过`rand.NewSource()`创建出来的`Source`要慢。不用处理协程并发场景，当然慢啦。

### 7. 使用 strings.Builder

之前的解决方案都返回了通过slice构造的字符串。最后的一次转换进行了一次拷贝，因为字符串是不可变的，如果转换的时候不进行拷贝，就无法保证转换完成之后，byte
slice再被修改后，字符串仍能保持不变。

Go1.10引入了**strings.Builder**，这是一个新的类型，和bytes.Buffer类似，用来构造字符串。底层使用`[]byte`
来构造内容，正是我们现在在做的，最后可以通过`Builder.String()`方法来获得最终的字符串值。但它很酷的地方在于，它无需执行刚才谈到的复制即可完成此操作。它敢这么做是因为它底层构造的
`[]byte`从未暴露出来，所以仍然可以保证没有人可以无意地、恶意地来修改已经生成的不可变字符串。

所以我们的下一个想法不是在slice中构建随机字符串，而是在 strings.Builder 的帮助下，一旦我们完成，我们就可以获取并返回结果，而无需复制。
这可能在速度方面有所帮助，并且在内存使用和分配方面肯定会有所帮助（译者注：等会在benchmark中会清晰地看到）。
