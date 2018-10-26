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

## 在Goroutines的起的时候离开

Goroutine挂起是因为这些Goroutine进入了永久阻塞状态。有很多的原因可以导致Goroutines进入挂起状态。例如，

- 一个Goroutine尝试从一个`nil`通道上接收值，或者从一个没有其他Goroutines发送值的通道上接收值。
- 一个Goroutine尝试从一个`nil`通道上发送值，或者向一个没有其他Goroutines接收值的通道上发送值。
- 一个Goroutine因为自身的原因产生死锁。
- 一组Goroutines因为其他的Goroutines产生了死锁。
- 一个因为使用了`select{}`或者没有`default`分支的`select`代码块导致一个Goroutine进入阻塞状态，并且`select`代码块中的`case`关键字之后的所有通道操作都会永久阻塞。

除了有时候我们故意让主Goroutine挂起来避免程序退出，其他大多数Goroutine挂起都不是我们期望看到的。Go运行时很难判断阻塞状态下的Goroutine是挂起状态还是暂时处于阻塞状态。所以Go运行时永远不会释放由挂起的Goroutine所消耗的资源。
