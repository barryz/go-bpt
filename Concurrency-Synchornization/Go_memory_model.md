> 译自：[Memory Order Guarantees In Go](https://go101.org/article/memory-model.html)。

# Go中的内存顺序保证

## 关于内存顺序

现代编译器（在编译期间）和CPU处理器（在运行期间）通常针对指令顺序都做了一些优化，所以指令的执行顺序一般和代码中的顺序不一致。

_（指令重排也常被称作[内存排序](https://en.wikipedia.org/wiki/Memory_ordering)。）_

当然， 指令重排不是随便就可以的。在指定的Goroutine内重排的最基本的要求是：如果Goroutine不与其他的Goroutine共享数据，那么Goroutine本身必须不能检测到重排。换句话说，从这个Goroutine角度来看，它可以认为它的指令执行顺序总是和其代码指定的顺序是一致的，即使它内部确实发生了一些指令重排。

但是，如果一些Goroutines共享了某些数据，发生在这些Goroutines内部的指令重排可能就会被其他的Goroutines观察到，并且会影响到所有这些Goroutines。共享数据在并发编程中非常常见，如果我们忽略了指令重排导致的结果，那么我们的并发程序的行为可能依赖于编译器和CPU，会经常出现异常。

这里有一个不专业的Go程序，该程序没有考虑到指令重排。该程序从[Go1 memory model](https://golang.org/ref/mem)中的一个示例扩展而来。

```go
package main

import "runtime"

var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()

	for !done {
		runtime.Gosched()
	}
	println(a) // expect to print: hello, world
}
```

上面这个程序的行为非常有可能如我们期待的那样，最终打印出`hello, world`。但是，实际上这个程序的行为依赖于具体的编译器和CPU。如果使用不同的编译器来编译这个程序，或者（这个程序）运行在不同架构的平台上，它有可能什么也不打印，或者是一个和`hello, world`不同的文本。原因是不同的编译器或CPU会交换函数`setup`中两行代码的执行顺序，所以对函数`setup`造成的影响可能就是：

```go
func setup() {
	done = true
	a = "hello, world"
}
```

上面程序中的`setup`的Goroutine并不能观察到自身的指令重排，但是，主Goroutine可以观察到这一现象。

除了重排问题之外，该程序中也存在数据竞争的问题。变量`a`和`done`上并没有使用数据同步技术。所以，上面的程序是一个充斥着各种错误的。专业的Go程序员不应该犯这些错误。

## Go的内存模型

有时候，我们需要确保在一个Goroutine中某些代码必须在另一个Goroutine中的某些代码执行之前（或之后）执行，以此保证程序的正确性。在这种情况下，指令重排可能会导致一些问题。那么我们应该如何避免一些可能的指令重排呢？

不同的CPU架构提供了不同的栅栏指令以防止不同类型的指令重新排序。一些编程语言提供了相应的函数以在代码中插入这些栅栏指令。但是，理解并正确使用这些栅栏指令会提高并发编程的难度。

Go的设计哲学就是用尽可能少的特性来支持尽可能多的用例，同时确保整体代码有足够好的运行效率。所以Go的内置和标准库并没有提供直接的方式来使用栅栏指令。实际上，在Go中，我们可以使用一些同步方法来确保代码以我们期待的顺序执行。

下面我们将要列出一些在[Go1 memory model](https://golang.org/ref/mem)中和其他官方Go文档中提到（或未提及）的一些方法来保证（或不保证）代码的执行顺序。

在接下来的描述中，如果我们说事件`A`将保证在事件`B`之前发生，那么它就意味着这两个事件中涉及的任何Goroutine都将观察到，源代码中的事件`A`之前出现的任何语句都将在源代码中事件`B`之后出现的任何语句之前执行。对于其他不相关的Goroutines，它们观察到的顺序可能和刚才我们描述的顺序不同。

## `init`函数相关的顺序保证

Go内存模型提到了两个关于`init`函数的顺序保证：

- 一个包中的相关依赖包中的所有`init`函数的完成发生在此包中任何`init`函数执行之前。
- 一个程序中所有`init`函数的完成发生在程序主入口函数开始之前。

实际上，还有其他两种顺序保证：

- 一个包中所有包级别变量的初始化的完成发生在该包中任何`init`函数执行之前。
- 一个包中的依赖包中的所有`init`函数的完成发生在该包中的任何包级别变量初始化之前。

### Goroutine的创建在Goroutine执行之前发生

在下面的函数中，分配语句`x, y = 123, 789`将会在`fmt.Println(x)`调用之前被执行，并且`fmt.Println(x)`调用将在`fmt.Println(y)`调用之前执行。

```go
var x, y int
func f1() {
	x, y = 123, 789
	go func() {
		fmt.Println(x)
		go func() {
			fmt.Println(y)
		}()
	}()
}
```

但是，下面函数中三者的执行顺序是不确定的。此函数中存在数据竞争。

```go
var x, y int
func f2() {
	go func() {
		fmt.Println(x) // might print 0, 123, or some others.
	}()
	go func() {
		fmt.Println(y) // might print 0, 789, or some others.
	}()
	x, y = 123, 789
}
```

### 通道相关的顺序保证

下面我们将列举一些在Go1内存模型未提及的顺序保证，但是我认为它们是显而易见的。

- 一个通道的发送操作在发送操作完成之前完成。
- 一个通道的接收操作在接收操作完成之前完成。
- 同一个通道上两个发送操作的完成不能同时发生。
- 同一个通道上两个接收操作的完成不能同时发生。

Go1内存模型列举了下面三种通道相关的顺序保证。

1. 一个通道上第**n**个成功发送操作在该通道上第**n**个成功接收操作完成之前完成，无论这个通道是缓冲的还是无缓冲的。
2. 一个容量为**m**的通道上第**n**个成功接收操作在该通道上第**(n+m)** 个成功发送操作完成之前完成。特别的，如果这个通道是无缓冲的（ **m** == 0），该通道上第**n**个成功接收操作在该通道上第**n**个成功发送操作完成之前完成。
3. 如果接收操作返回零值，则会在接收完成之前完成关闭通道操作，因为该通道已经关闭了。

请注意，所有上述的顺序保证都是针对所涉及的Goroutines。对于其它不相关的Goroutines，它们观察到的顺序可能和我们在上面描述的有所不同。

下面是一个使用无缓冲通道来展示一些受保证的代码执行顺序的示例。

```go
func f() {
	var a, b int
	var c = make(chan bool)

	go func() {
		a = 1
		c <- true
		if b != 1 { // impossible
			panic("b != 1") // will never happen
		}
	}()

	go func() {
		b = 1
		<-c
		if a != 1  { // impossible
			panic("a != 1") // will never happen
		}
	}()
}
```

这里，对于两个新建的Goroutines，下列的顺序将会被保证：

- 分配语句`b = 1`绝对发生在条件求值语句`b != 1`之前。
- 分配语句`a = 1`绝对发生在条件求值语句`a != 1`之前。

因此，上面例子中两个调用`panic`的代码永远不会被执行到。但是，下面例子中`panic`将会被执行到。

```go
func f() {
	var a, b, x, y int
	c := make(chan bool)

	go func() {
		a = 1
		c <- true
		x = 1
	}()

	go func() {
		b = 1
		<-c
		y = 1
	}()

	// Many data races are in this goroutine.
	// Don't write code as such.
	go func() {
		if x == 1 {
			if a != 1 { // possible
				panic("a != 1") // may happen
			}
			if b != 1 { // possible
				panic("b != 1") // may happen
			}
		}

		if y == 1 {
			if a != 1 { // possible
				panic("a != 1") // may happen
			}
			if b != 1 { // possible
				panic("b != 1") // may happen
			}
		}
	}()
}
```

这里，对于第三个Goroutine，是和通道`c`的操作不相关的。我们无法保证它遵守前两个新创建的goroutines观察到的顺序。所以，这里面四个`panic`中任何一个都有可能被执行到。

实际上，大多数编译器的实现确实保证了上面例子中四个`panic`调用永远不会被执行，但是，没有一个Go的官方文档会做这样的保证。所以上述例子中的代码并不是跨编译器或跨编译器版本兼容的。我们应该坚持使用Go官方文档来编写专业的Go代码。
