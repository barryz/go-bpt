> 译自： [Common Concurrent Programing Mistakes](https://go101.org/article/concurrent-common-mistakes.html)。

# 常见的并发编程错误

Go是一门内建并发编程的语言。通过使用`go`关键字创建Goroutines（轻量级线程），并且通过[使用通道](https://go101.org/article/channel-use-cases.html)和Go中提供的[其他并发同步技术](https://go101.org/article/concurrent-synchronization-more.html)，并发编程将变得简单，灵活，有趣。

另一方面，Go并不能防止开发者因为一些疏忽或缺乏经验而导致的一些并发编程的错误的情况发生。本文接下来的内容将会介绍Go中一些常见的并发编程错误，来帮助Go开发者避免犯这些错误。

## 在需要进行同步的时候不同步

代码行可能[并不会按照表面上的顺序执行](https://go101.org/article/memory-model.html)。

在下面的程序中有两个错误。

- 首先，主Goroutine关于`b`的读操作和新Goroutine关于`b`的写操作将产生数据竞争。
- 第二，条件`b == true`并不能确保主Goroutine中的`a != nil`成立。编译器和CPU可能会在新的Goroutine进行一些[指令重排](https://go101.org/article/memory-model.html)的优化，所以在运行时分配操作`b`可能会发生在分配操作`a`之前，这将会导致当在主Goroutine切片`a`的元素被修改时，`a`仍然是`nil`。

```go
package main

import (
	"time"
	"runtime"
)

func main() {
	var a []int // nil
	var b bool  // false

	// a new goroutine
	go func () {
		a = make([]int, 3)
		b = true // write b
	}()

	for !b { // read b
		time.Sleep(time.Second)
		runtime.Gosched()
	}
	a[0], a[1], a[2] = 0, 1, 2 // might panic
}
```

上面的程序可能在一个机器上运行的很正常，但在另一台机器上产生恐慌。也有可能运行N次都是ok的，但是第N+1次就出问题了。

我们应该使用通道或者其他同步技术来确保内存顺序。例如，

```go
package main

func main() {
	var a []int = nil
	c := make(chan struct{})

	// a new goroutine
	go func () {
		a = make([]int, 3)
		c <- struct{}{}
	}()

	<-c
	a[0], a[1], a[2] = 0, 1, 2
}
```

## 使用`time.Sleep`调用来进行一些同步操作

让我们来看一个简单的示例。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var x = 123

	go func() {
		x = 789 // write x
	}()

	time.Sleep(time.Second)
	fmt.Println(x) // read x
}
```

我们期待这个程序将会打印出`789`。如果我们运行这个程序，它确实会打印出`789`，在大多数情况下确实如此。但是这是一个足够好的同步程序吗？显然不是！原因在于Go运行时并不保证`x`的写入操作一定发生在`x`的读取操作之前。在一些特定的情况下，比如大部分的CPU都被同一个OS上其他进程占用时，`x`的写入将会发生在`x`的读取之后。这就是为什么我们绝不能使用`time.Sleep`调用在一些正式的项目中来进行同步操作。

让我们来看另一个例子。

```go
package main

import (
	"fmt"
	"time"
)

var x = 0

func main() {
	var num = 123
	var p = &num

	c := make(chan int)

	go func() {
		c <- *p + x
	}()

	time.Sleep(time.Second)
	num = 789
	fmt.Println(<-c)
}
```

这个程序将会打印出什么结果？`123`还是`789`？实际上，打印结果依赖于具体的编译器。对于目前标准的Go编译器1.11来说，它很有可能打印出`123`，但是理论上，它应该会打印出`789`，或者另一个不确定的值。

现在，让我们将`c <- *p + x`改成`c <- *p`，然后再重新运行该程序。你会发现打印结果就变成`789`了（对于标准Go编译器1.11来说）。再说一遍，这依赖于具体的编译器。

是的，上面的程序也有数据竞争的问题。表达式`*p`将有可能在分配操作`num = 789`处理之前，之后或者同时求值。`time.Sleep`调用并不能保证`*p`的求值发生在分配操作处理之前。

对于这个特定的例子来说，我们应该在创建一个新的Goroutine之前将这个需要发送的值存储到一个临时变量里，并且将这个临时变量发送给新的Goroutine来避免数据竞争。

```go
...
	tmp := *p + x
	go func() {
		c <- tmp
	}()
...
```

## 任由Goroutines挂起

Goroutine挂起是因为这些Goroutine进入了永久阻塞状态。有很多的原因可以导致Goroutines进入挂起状态。例如，

- 一个Goroutine尝试从一个`nil`通道上接收值，或者从一个没有其他Goroutines发送值的通道上接收值。
- 一个Goroutine尝试从一个`nil`通道上发送值，或者向一个没有其他Goroutines接收值的通道上发送值。
- 一个Goroutine因为自身的原因产生死锁。
- 一组Goroutines因为其他的Goroutines产生了死锁。
- 一个因为使用了`select{}`或者没有`default`分支的`select`代码块导致一个Goroutine进入阻塞状态，并且`select`代码块中的`case`关键字之后的所有通道操作都会永久阻塞。

除了有时候我们故意让主Goroutine挂起来避免程序退出，其他大多数Goroutine挂起都不是我们期望看到的。Go运行时很难判断阻塞状态下的Goroutine是挂起状态还是暂时处于阻塞状态。所以Go运行时永远不会释放由挂起的Goroutine所消耗的资源。

我们在[first-response-wins](https://go101.org/article/channel-use-cases.html#first-response-wins)的通道用例中提到过，如果一个在未来将要使用的通道的容量没有足够的大小，一些响应比较慢的Goroutines在尝试向这个通道发送结果时会挂起。例如，如果下面这个函数被调用了，那么将会有4个Goroutines进入永久阻塞状态。

```go
func request() int {
	c := make(chan int)
	for i := 0; i < 5; i++ {
		i := i
		go func() {
			c <- i // 4 goroutines will hang here.
		}()
	}
	return <-c
}
```

为了避免这4个Goroutines挂起，通道`c`的容量必须至少为`4`个。

我们在[the second way to implement the first-response-wins](https://go101.org/article/channel-use-cases.html#first-response-wins-2)的通道用例中提到，如果未来将要使用的通道是一个无缓冲的通道，那么很有可能这个通道的接收者将不会得到任何响应并处于挂起状态。例如，如果下面的函数在一个Goroutine中被调用，这个Goroutine可能就会挂起。原因在于，如果5个尝试发送的操作都发生在接收操作`<-c`就绪之前，那么这5个尝试发送的操作将会失败，所以调用者Goroutine将不会接收到一个值。

```go
func request() int {
	c := make(chan int)
	for i := 0; i < 5; i++ {
		i := i
		go func() {
			select {
			case c <- i:
			default:
			}
		}()
	}
	return <-c
}
```

上述例子中，如果我们改变下通道`c`的缓冲值大小，就可以保证至少有一个尝试发送的操作能够完成，所以调用者Goroutine将不会挂起。

## 拷贝`sync`标准库内类型的值

实践中，`sync`标准库中类型的值（除了`Locker`接口类型外）都不应该被拷贝。我们应该拷贝这些值的指针。

下面的例子是一个糟糕的并发编程例子。在这个例子中，当`Counter.Value`方法被调用时，一个`Counter`接收者值将会被拷贝。作为该接收者值的一个字段，还将拷贝`Counter`接收者中相应的`Mutex`字段。这种拷贝并不是同步的，所以拷贝的`Mutex`值可能已经损坏。即使它没有损坏，它所保护的是拷贝过的`Counter`接收者的访问，这通常来说是没有意义的。

```go
import "sync"

type Counter struct {
	sync.Mutex
	n int64
}

// This method is okay.
func (c *Counter) Increase(d int64) (r int64) {
	c.Lock()
	c.n += d
	r = c.n
	c.Unlock()
	return
}

// The method is bad. When it is called, a Counter
// receiver value will be copied.
func (c Counter) Value() (r int64) {
	c.Lock()
	r = c.n
	c.Unlock()
	return
}
```

我们应该将`Value`方法的接收者改成`*Counter`类型来避免拷贝`Mutex`值。

Go SDK中的`go vet`命令将会检查出代码中潜在的错误值拷贝。

## 在错误的地方调用`sync.WaitGroup`类型的方法

每个`sync.WaitGroup`在内部都维护了一个计数器，这个计数器的初始值是0。如果一个`WaitGroup`的计数器值为0，那么调用`sync.WaitGroup`中的`Wait`方法将不会阻塞，反之，该调用将会阻塞至该值的计数器归零为止。

为了使`sync.WaitGroup`的使用有意义，当一个`WaitGroup`的计数器的值为0时，`Add`方法的调用必须发生在相应`WaitGroup`值的`Wait`方法之前。

例如，在下面的程序中，`Add`方法在不恰当的为止进行了调用，这就导致了最终的输出结果不总是`100`。实际上，该程序最终输出的值可能会是在`[0, 100]`之间的任何值。原因就在于该程序并没有保证`Add`方法是在`Wait`方法之前调用的。

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var wg sync.WaitGroup
	var x int32 = 0
	for i := 0; i < 100; i++ {
		go func() {
			wg.Add(1)
			atomic.AddInt32(&x, 1)
			wg.Done()
		}()
	}

	fmt.Println("To wait ...")
	wg.Wait()
	fmt.Println(atomic.LoadInt32(&x))
}
```

想要保证这个程序运行正常，我们应该将`Add`方法移动到`for`循环内部的Goroutine创建之前，就像下面的代码一样。

```go
...
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			atomic.AddInt32(&x, 1)
			wg.Done()
		}()
	}
...
```

## 错误地将通道作为Future使用

通过文章[channel use cases](https://go101.org/article/channel-use-cases.html)，我们知道有些函数会返回[channels as futures](https://go101.org/article/channel-use-cases.html#future-promise)。 假设`fa`和`fb`就是这样的两个函数，那么下面代码中的使用方法就是不当的。

```go
doSomethingWithFutureArguments(<-fa(), <-fb())
```

在上面的代码行中，两个通道的接收操作是串行处理的，而不是并行处理的。我们应该像下面这样修改，以保证它们能够并行处理。

```go
ca, cb := fa(), fb()
doSomethingWithFutureArguments(<-ca, <-cb)
```

## 不从最后一个活跃的发送Goroutine去关闭一个通道

Go开发者有一个常犯的错误就是当一个通道还有一些潜在的发送者Goroutines时就关闭了通道。当这些潜在的发送者确实发送（到一个已关闭的通道）消息之后，就将会导致恐慌。

这类的错误曾在一些有名的Go项目中出现过，比如Kubernetes项目的两个bug： [bug1](https://github.com/kubernetes/kubernetes/pull/45291/files?diff=split)，[bug2](https://github.com/kubernetes/kubernetes/pull/39479/files?diff=split)。

请阅读[这篇文章](https://go101.org/article/channel-closing.html)来学习如何优雅的关闭通道。

## 对不能保证64位对齐的值执行64位的原子操作

直到Go1.11为止，就标准Go编译器来说，64位原子操作涉及的值的地址需要64位对齐。如果不这么做可能会导致当前Goroutine恐慌。对标准Go编译器来说，这样的错误只会发生在32位架构平台上。请阅读[内存布局](https://go101.org/article/memory-layout.html)来学习如何保证在32位的操作系统上得到64位对齐的64位的字(word)。

## 滥用`time.After`函数导致消耗太多资源

`time`标准库中的`time.After`函数会返回一个[延迟通知的通道](https://go101.org/article/channel-use-cases.html#timer)。这个函数很方便。然而每个这样的函数调用都会产生一个`time.Timer`类型的值。新创建的`Timer`值将在`After`函数参数指定的持续时间内一直存在。如果这个函数在这个持续时间内被多次调用，那么将会产生很多个这样的`Timer`值，这些值将会消耗大量的内存和计算资源。

例如，如果下面的`longRunning`函数调用时有大量的消息在一分钟内进入，那么在一定的周期内将会有百万个`Timer`值同时存在，尽管它们中大多数都是无用的。

```go
import (
	"fmt"
	"time"
)

// The function will return if a message arrival interval
// is larger than one minute.
func longRunning(messages <-chan string) {
	for {
		select {
		case <-time.After(time.Minute):
			return
		case msg := <-messages:
			fmt.Println(msg)
		}
	}
}
```

为避免上述例子中产生大量无用的`Timer`值，我们应该仅使用一个`Timer`值来完成此次工作。

```go
func longRunning(messages <-chan string) {
	timer := time.NewTimer(time.Minute)
	defer timer.Stop()

	for {
		select {
		case <-timer.C:
			return
		case msg := <-messages:
			fmt.Println(msg)
			if !timer.Stop() {
				<-timer.C
			}
		}

		// The above "if" block can also be put here.

		timer.Reset(time.Minute)
	}
}
```

## 不正确的使用`time.Timer`值

最后一小节展示了`time.Timer`值的惯用方法。其中我们应该需要注意的一个细节就是，应该始终在停止或者已经过期后调用`Reset`方法。

在`select`代码块的第一个`case`分支的结尾，`time.Timer`值已经过期了，因此我们不需要在停止它。但是我们必须在第二个分支内停止这个定时器。如果第二个分支中的`if`代码块缺失，则有可能一个发送至通道`time.C`的操作和`Reset`方法调用产生竞争，并且有可能导致`longRunning`函数过早返回，因为`Reset`方法只会将内部的定时器置零，它不会清除（排出）已经发送道`time.C`通道的值。

例如，下面程序非常有可能在1秒内退出，而不是10秒内。而且最重要的一点，这个程序是有数据竞争的。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	start := time.Now()
	timer := time.NewTimer(time.Second/2)
	select {
	case <-timer.C:
	default:
		time.Sleep(time.Second) // go here
	}
	timer.Reset(time.Second * 10)
	<-timer.C
	fmt.Println(time.Since(start)) // 1.000188181s
}
```

一个`time.Timer`的值可以以非停止状态离开，当它不在被使用时，但是我们推荐在不需要使用它时显式地停止它。

它很容易出错，不建议在多个Goroutines中同时使用time.Timer值。

我们不应该依赖`Reset`方法调用的返回值。 `Reset`方法的返回结果仅用于兼容性目的。
