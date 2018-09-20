> 译自： [Go Channe Use cases](https://go101.org/article/channel-use-cases.html)。

# 通道使用案例

在阅读本文之前，请阅读文章[Channels In Go](https://go101.org/article/channel.html)，这篇文章解释了通道类型和值的一些细节。新Gophers可能需要反复阅读本文和之前那篇文章来理解在Go编程中如何使用通道。

本文余下的内容将列举各类通道使用案例。我希望这篇文章可以帮助你感觉到

- 通过使用Go通道进行异步和并发编程是方便且愉快的。
- 通道同步技术具有更加广泛的用途。并且具有比异步编程更多的变化。例如在其他语言中的[actor model](https://en.wikipedia.org/wiki/Actor_model)。

请注意，本文的目的旨在尽可能多的介绍通道的使用场景。我们应该需要知道的是， 通道并不是Go所支持的唯一的并发同步技术。并且在许多的场景下，通道并不是最佳的解决方案。请阅读[atomic operations](https://go101.org/article/concurrent-atomic-operation.html)和[some other synchornization techniques](https://go101.org/article/concurrent-synchronization-more.html)来更多Go中所支持的并发同步技术。

许多关于Go通道文章将通道用例分类为一些模式类别，例如请求 - 响应模式，基于事件的通知模式和数据流模式等。这篇文章并没有遵循这种方式，因为这些模式之间的障碍有些模糊。许多用例可能属于多个模式类别。这篇文章只展示了各种用例。

## 使用通道作为期刊/承诺（Futures/Promises）

通过将goroutine和通道的组合使用，我们可以完成在其他语言中的异步编程特性[Future and Promise](https://en.wikipedia.org/wiki/Futures_and_promises)相同的功能。

期刊和承诺经常和请求和响应关联在一起。通常来说，仅接收通道可以被看作是期刊（从请求端来看），仅发送通道可以被看作是承诺（从响应端来看）。

### 返回仅接收通道作为返回值

在下面的例子中，`sumSquares`函数调用的两个参数的值是同时被请求的。因为两个通道都是无缓冲的通道，每个通道的接收操作将会阻塞知道其相对应的通道发生了一个发送操作。返回最终结果大约需要三秒钟而不是六秒钟。

```go
package main

import (
	"time"
	"math/rand"
	"fmt"
)

func longTimeRequest() <-chan int32 {
	r := make(chan int32)

	// This goroutine treats the channel r as a promise.
	go func() {
		time.Sleep(time.Second * 3) // simulate a workload
		r <- rand.Int31n(100)
	}()

	return r // return r as a future
}

func sumSquares(a, b int32) int32 {
	return a*a + b*b
}

func main() {
	rand.Seed(time.Now().UnixNano())

	a, b := longTimeRequest(), longTimeRequest()
	fmt.Println(sumSquares(<-a, <-b))
}
```

### 将近发送通道作为参数传递

和上一个例子一样，在下面的例子中，`sumSquares`函数调用的两个参数的值也是同时被请求的。和上个例子不同的是，`longTimeRequest`函数接收了一个仅发送通道作为参数来代替返回一个仅接收通道的返回值。

```go
package main

import (
	"time"
	"math/rand"
	"fmt"
)

// Channel r is viewed as a promise by this function.
func longTimeRequest(r chan<- int32)  {
	time.Sleep(time.Second * 3)
	r <- rand.Int31n(100)
}

func sumSquares(a, b int32) int32 {
	return a*a + b*b
}

func main() {
	rand.Seed(time.Now().UnixNano())

	ra, rb := make(chan int32), make(chan int32)
	go longTimeRequest(ra)
	go longTimeRequest(rb)

	fmt.Println(sumSquares(<-ra, <-rb))
}
```

实际上，对于上面两个特定的例子，我们不不需要使用两个通道来传输结果，使用一个通道就可以了。

```go
...

	results := make(chan int32, 2) // can be buffered or not
	go longTimeRequest(results)
	go longTimeRequest(results)

	fmt.Println(sumSquares(<-results, <-results))
}
```

这是一种数据聚合，这个将在下面详细介绍。

### The First Response Wins

这是上面最后一个用例的增强版。

有时候，我们可能会从多个数据源来检索一段数据。因为一些不可控的因素，这些数据源的响应时间差异可能非常大。即使对于一个相同的数据源，其响应时间也不是恒定的。为了使得响应时间尽可能短，我们可以分别在不同的goroutine中向每个数据源发送同一个请求。但只使用第一个接收到的响应，其他比较慢的响应将会被丢弃。

注意，如果有`n`个数据源，那么用来通信的通道的容量大小必须至少是`n-1`，以避免goroutines对应的丢弃的响应永远阻塞。

```go
package main

import (
	"fmt"
	"time"
	"math/rand"
)

func source(c chan<- int32) {
	ra, rb := rand.Int32(), rand.Intn(3) + 1
	time.Sleep(time.Duration(rb) * time.Second) // sleep 1s, 2s or 3s
	c <- ra
}

func main() {
	rand.Seed(time.Now().UnixNano())

	startTime := time.Now()
	c := make(chan int32, 5) // need a buffered channel
	for i := 0; i < cap(c); i++ {
		go source(c)
	}
	rnd := <- c // only the first response is used
	fmt.Println(time.Since(startTime))
	fmt.Println(rnd)
}
```

也有其他的方式来实现**first-reponse-win**用例，比如通过使用`select`机制配合一个容量为1的缓冲通道。下面将会介绍这种方式。


### 更多请求-响应的变体
