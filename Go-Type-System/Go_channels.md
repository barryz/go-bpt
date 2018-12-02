> 译自[Go channels](https://go101.org/article/channel.html).

# Go中的通道

通道是Go中一个非常重要的内建特性。它是使Go特别的众多特性之一。和另一个独特的功能goroutine一起，使得在Go中并发编程变得非常方便和有趣。同时这两个特性也降低了并发编程的难度。

这篇文章将会列举一些关于通道的概念，语法和规则。为了更好地理解通道，我们还简单地描述了通道的内部结构和标准Go编译器/运行时的一些关于通道的实现细节。

对于新的gophers而言，本文的一些内容可能比较晦涩。有些内容可能需要反复阅读多次才能消化。

## 并发编程和并发同步

现代CPU往往支持多核，并且一些CPU核心也支持超线程。换句话说，现代CPU可以同时处理多条指令流水线（pipeline）。想要完全使用到多核CPU的性能，我们需要在编写程序时使用并发。

并发计算是在重叠的时间周期（时间片）内进行多次计算。下图描绘了两个并发计算的场景。图中，A和B代表了两个单独的计算。第二个例子被称作是并行计算，是一种特殊的并发计算。第一个例子中，A和B只有在很小的时间片内才可能并行。

![](https://go101.org/article/res/concurrent-vs-parallel.png)

并发计算可以在程序、计算机或者网络中发生。在Go101教程里，我们只讨论编程范畴的并发计算。正如前面介绍的[Gorotines](https://go101.org/article/control-flows-more.html)，是Go支持的一种提供并发计算的方式。

并发计算可能会共享某些资源，一般来说特指内存资源。在并发计算期间可能会发生某些情况。

- 在某次计算的同一周期内，将数据写入到内存段时，另一个计算从这个内存段中读取数据。所以有可能另一个计算读取到数据会不完整、正确。

- 在某次计算的同一周期内，将数据写入到内存段时，另一个计算也将数据写入到这个内存段。所以有可能另一个计算写入的数据会不完整、正确。

这两种情形被称为数据竞争。并发编程的职责之一就是控制资源在多个并发计算之间共享，这样数据竞争就不会发生。实现此任务的方法我们称之为并发同步，或数据同步。Go支持多种数据同步技术。下面的文章将会介绍到其中的一种，通道。

并发编程的其他任务包括：

- 确定需要多少计算。

- 确定一个计算的起始终止时间。

- 确定如何在多个并发计算中分摊负载。

Go中大多数的操作是非同步的，换句话说，它们不是线程安全的。这些操作包括值分配，参数传递和容器类型的元素操作，等等。Go中只有少数的操作是同步的，包括下文要引入的通道操作。

在Go里，通常来说，每个计算都是一个goroutine。所以后面我们将使用goroutine来代替计算这个术语。

## 通道介绍

Go语言创始人之一的*Rob Pike*给出的关于并发编程的一个建议就是**不要让goroutine通过共享内存通信，而是让它们通过通道通信来共享内存**。通道机制就是这种哲学的体现。

通过共享内存通信和通过通信来共享内存是并发编程的两种模式。当goroutine通过共享内存来通信时，我们需要用到一些传统的并发同步技术，例如互斥锁，用来保护共享资源从而防止数据竞争。

Go提供了一种独一无二的并发同步技术，通道。通道使得goroutines可以通过通信来共享内存。我们可以把通道看作是一个程序内的一个**FIFO**队列。一些goroutine往这个队列（通道）发送值，另外一些goroutine从这个队列接收值。

除了通过通道传递值，一些值的所有权也可以通过通道在goroutines之间传递。当一个goroutine向一个通道发送一些值，我们可以将其看作是goroutine释放了某些值得所有权。当一个goroutine从一个通道接收一些值，我们可将其看作是goroutine获取了某些值得所有权。值(其所有权被转移)通常是被转移的值的引用(但不需要被引用)。

当然，也可能没有任何所有权通过通道通信被转移。

需要注意的是，在这里，当我们讨论所有权的时候，我们所指的是逻辑概念上的所有权。不同于`Rust`语言的所有权，Go并不在语法层面支持值所有权。Go通道可以帮助程序员轻松地写出无数据竞争的代码，但是也不能预防程序员写出糟糕的并发代码。

尽管Go支持传统的数据同步技术。但只有通道是Go中的**一等公民**。通道是Go中的一个类型，所以我们可以直接使用通道而无需导入任何包。另一方面，标准库`sync`和`sync/atomic`包将提供[一些传统的并发同步技术](https://go101.org/article/concurrent-synchronization-overview.html)。

老实说，每种并发同步技术都有其自身的应用场景。但是通道的[应用范围更广、使用范围也更广](https://go101.org/article/channel-use-cases.html)。使用通道的一个问题就是，使用其编程的体验是如此的愉快和有趣，以至于程序员在某些不是通道最佳使用场景下仍然使用通道。

## 通道类型和值

通道类型是一个综合（组合）类型。和数组、切片和map一样，每个通道类型都有一个元素类型。所有发送到通道的数据都必须是该通道元素类型的值。

通道类型可以是双向的，也可以是单向的，假设`T`是一个任意类型，

- `chan T`表示的是一个双向通道类型。编译器允许双向通道既接收值，又可以发送值。

- `chan<- T`表示的是一个仅发送通道类型，编译器不允许在发送通道类型上接收值。

- `<-chan T`表示的是一个仅接收通道类型，编译器不允许在接收通道类型上发送值。

`T`被称作是这些通道类型的元素类型。

双向通道`chan T`的值可以被转换成发送通道`chan<- T`和接收通道`<-chan T`的值。但是反之不亦然。发送通道`chan<- T`的值也不能转换成接收通道`<-chan T`的值，反之亦然。请注意，通道类型字面值中的`<-`是修饰符。

每个通道有一个容量，这个将在下一节解释。一个零容量的通道我们称之为无缓冲通道，非零容量的通道我们称之为缓冲通道。

通道类型的零值使用预先声明的标识符`nil`表示。一个非`nil`的通道值必须通过内建函数`make`函数来创建。例如，`make(chan int, 10)`将会创建一个类型为`int`容量为0的通道。第二个参数（容量）是可选的，默认值是0。

## 通道的分配和比较

所有的通道类型都是可比较的类型。

通过文章[值部分](https://go101.org/article/value-part.html)，我们知道非`nil`通道类型是一个**多部分值**。当将一个通道值分配给另一个通道时，这两个通道共享相同的底层部分。换句话说，这两个通道表示了相同的内部通道对象。比较它们的结果将是`true`。

## 通道操作

我们可以在一个通道上执行五种操作。假设通道是`ch`，下面将列出这些操作的语法和函数调用。

1. 使用以下函数调用关闭通道

  ```go
  close(ch)
  ```

  这里的`close`是一个内建函数。`close`函数的参数必须是一个通道值，且通道值不能是一个只接收类型的通道。

2. 发送一个值`v`，使用下列语法

  ```go
  ch <- v
  ```

  这里的`v`必须是可赋值给通道`ch`元素类型的值。这里要注意的是符号`<-`是通道发送符号。

3. 从通道中接收一个值，使用下列语法

  ```go
  <- ch
  ```

  一个通道接收操作总是至少返回一个结果，这个结果就是通道元素类型的一个值。注意这里的符号`<-`是一个通道接收符号。（它的表示与通道发送操作符相同。）

  在大多数场景中，一个通道接收操作可以被视为一个单值表达式。然而，当通道操作用作赋值中唯一的源值表达式时，它却可以被视为一个多值表达式且会产生另一个可选的无类型的布尔值。（第二个结果是bool值），这个布尔值表示在通道关闭之前是否发送了第一个值（结果）。（下面我们将了解到我们可以从一个关闭的通道中接收无限个值。） 例如，

  ```go
  v = <-ch
  v, sentBeforeClosed = <-ch
  ```

4. 查询一个缓冲通道的容量，可以使用下列函数调用

  ```go
  cap(ch)
  ```

在之前的文章[containers in Go](https://go101.org/article/container.html#cap-len)中，我们已经介绍过内建函数`cap`。`cap`函数的返回值是一个`int`类型。

5. 查询通道的值缓冲区（或长度）中的当前值的个数，使用下列函数调用

  ```go
  len(ch)
  ```

`len`内建函数我们之前也介绍过。`len`函数的返回值是一个`int`类型。 上述列子返回的结果是当前被查询的通道已经被成功发送且还未被接收的值的个数。

上述所有的操作都是同步的，因此执行这些操作不需要进一步的同步。然而，和Go中其他大多数操作一样，通道的值分配并不是同步的。类似地，虽然任何通道接收操作是同步的，但是将接收的值分配给另一个值却是不同步的。

如果被查询的通道是一个`nil`通道，内建函数`cap`和`len`都会返回0。事实上，这两个查询操作非常简单，所以我们以后不再进一步解释。其实，这两种操作在实践中很少使用。

下面的章节将会解释通道发送，接收和关闭操作相关的规则。

## 通道操作的简单汇总

为了使通道操作规则的解释简单明了，在下文中，通道将会被分为三类：

1. `nil`通道。
2. 非`nil`但是已关闭的通道。
3. 非`nil`且未关闭的通道。

下面的表格简单地描述了应用在`nil`，非`nil`，已关闭和未关闭通道上所有种类的操作的相关规则。下节中将会详细解释更多的细节。

|操作|`nil`通道|已关闭通道|未关闭非`nil`通道|
|-----|-----|------|------|
|关闭|panic|panic|成功关闭 (C)|
|发送值|永久阻塞|panic|阻塞或成功发送 (B)|
|接收值|永久阻塞|永不阻塞(D)|阻塞或成功接收 (A)|

对于上述表格中五个未标记的案例，规则非常明确。

- 关闭一个已经关闭的或`nil`通道将会导致当前goroutine panic。

- 向一个已经关闭的或`nil`通道发送值也将会导致当前goroutine panic。

- 向一个`nil`通道发送值或接收值将会导致当前的goroutine进入永久阻塞状态。

下面我们将会解释上述表格中四种已经标记的案列。

 为了更好地理解通道类型和值，使得一些解释更加容易理解，了解通道内部对象的大致内部结构会非常有帮助。我们可以认为每个通道在内部都维护了三个队列：

1. 接收goroutines队列。这个队列是没有大小限制的链表。此队列中的goroutines被称为此通道上的被阻塞接收goroutine。

2. 发送goroutines队列。这个队列也是一个没有大小限制的链表。此队列中的goroutines被称为此通道上的被阻塞发送goroutine。这些goroutines试图发送的值的地址也与每个goroutine一起存储在这个队列里。

3. 值缓冲队列。这是一个环形队列。它的大小等于通道的容量大小。如果存储在通道的值缓冲队列中的当前值的数量达到通道的容量上限，则这个通道被称为满（full）状态。如果没有值存储在通道的值缓冲队列里，则这个通道被称为空（empty）状态。对一个0容量的通道来说，它总是满足满状态和空状态。

[*通道规则A*]：当一个goroutine尝试从一个未关闭或非`nil`通道中接收值时，这个goroutine首先会尝试获取跟这个通道相关的锁，然后执行以下步骤，直到满足一个条件。

1. 当一个通道的值缓冲队列不为空时，在这种情况下，正在接收的goroutine队列必须为空，这些goroutine将会从值缓冲队列中接收一个值。如果通道的发送goroutine队列也不为空，则发送goroutine将不会从发送goroutine队列中移出，且会重新恢复运行状态。正在尝试发送的未移位的发送goroutine的值将被推入通道的值缓冲区队列。而接收goroutine则继续运行。在这种场景下，通道发送操作被称作**非阻塞操作**。

2. 否则，当一个通道的值缓冲队列为空但通道的发送gouroutines队列为空，在这种情况下，该通道必须是一个无缓冲通道，接收goroutine将从通道的发送goroutines队列中删除发送goroutine，并接收刚刚未移动的发送goroutine试图发送的值。刚未移位的发送goroutine将被解除阻塞，重新恢复至运行状态。接收goroutine继续运行。像这种情况，通道发送操作称为**非阻塞操作**。

3. 如果值缓冲区队列和通道的发送goroutines队列都为空，则goroutine将被推入通道的接收goroutines队列，进入(并保持)阻塞状态。当另一个goroutine稍后向该通道发送一个值时，可能会恢复到运行状态。对于这种情况，通道发送操作称为**阻塞操作**。

[*通道规则B*]：当一个goroutine尝试下一个未关闭非`nil`的通道发送一个值时，这个goroutine首先会尝试获取跟这个通道相关的锁，然后执行以下步骤，直到满足一个条件。

1. 如果通道的接收goroutine队列不为空，那么通道的值缓冲队列必须为空，发送goroutine将把接收goroutine从通道的接收goroutines队列中移出，并将该值发送到刚刚未移动的接收goroutine。刚未移位的接收goroutine将被解除阻塞，并恢复至运行状态。发送的goroutine继续运行。对于这种情况，通道发送操作称为**非阻塞操作**。

2. 如果接收goroutines队列为空值且缓冲区队列的通道不是满的，在这种情况下，发送goroutines队列必须也是空的，发送goroutine试图发送的值将推入值缓冲区队列，发送goroutine继续运行。对于这种情况，通道发送操作称为**非阻塞操作**。

3. 如果接收goroutines队列为空，且通道的值缓冲队列为满，发送goroutine将被推入通道的发送goroutines队列，进入(并保持)阻塞状态。当另一个goroutine稍后从通道接收到值时，可能会恢复到运行状态。对于这种情况，通道发送操作称为**阻塞操作**。

一旦非`nil`通道被关闭，向该通道发送一个值将在当前goroutine中产生运行时panic，发送操作被视为**非阻塞操作**。

[*通道规则C*]：当goroutine试图关闭一个未关闭的非`nil`通道时，将按照以下两个步骤顺序执行。

1. 如果接收goroutines队列的通道并不是空的,在这种情况下，缓冲区的通道的值必须是空的，通道的接收goroutines队列中的所有goroutines将逐个未移位，并且它们将会接收一个通道元素类型的零值。

2. 如果通道的发送goroutines队列不是空的，那么该通道的发送goroutines队列中的所有goroutines都将保持不变，并且每个goroutine在向已关闭的通道发送值时会产生panic。已经被推送到通道值缓冲区中的值仍然存在。

[*通道规则D*]：当通道关闭后，通道上的接收操作永远不会阻塞。通道的值缓冲区中的值仍然可以接收。一旦将值缓冲区中的所有的值取出后，通道上的任何后续接收操作都将无限地接收该通道元素类型的零值。如上所述，通道接收操作的第二返回结果是无类型布尔值，其表示在通道关闭之前是否发送了第一个结果（接收值）。

了解什么是**阻塞**和**非阻塞**通道发送或接收操作对于理解`select`控制流的机制很重要。这部分内容将在后面的章节中介绍。

如上文解释，如果一个goroutine未能从通道的队列（接收goroutine或发送goroutine）中移出，而goroutine由于在[`select`控制流代码块](https://go101.org/article/channel.html#select)中因为需要推入到通道队列中而被阻塞，然后，该goroutine将在[`select`控制流代码块执行](https://go101.org/article/channel.html#select-implementation)的第9步中恢复到运行状态。它可以从`select`控制流代码块中从对应的goroutine队列中清出队列。

根据上面列出的规则，我们可以得到关于通道内部队列的一些事实。

- 如果通道关闭，其发送goroutines队列和接收goroutines队列都必须为空，但其值缓冲区队列可能不为空。
- 在任何时候，如果值缓冲区不为空，则其接收goroutines队列必须为空。
- 在任何时候，如果值缓冲区不是满状态，则其发送goroutines队列必须为空。
- 如果是缓冲通道，那么在任何时候，它的发送goroutines队列和接收goroutines队列中的一个必须为空。
- 如果通道是无缓冲的，那么在任何时候，通常它的发送goroutines队列和接收goroutines队列中的一个必须为空，但有例外情况，在执行`select`控制流代码块时，goroutine可能被推入两个队列。

## 一些通道的用法示例

让我们来使用一些例子来消化下上述的各种规则。

一个简单的请求/应答例子，这个例子中的两个goroutines通过一个无缓冲的通道互相通信。

```go
package main

import "fmt"

func main() {
	c := make(chan int) // an unbuffered channel
	go func() {
		x := <- c // blocking here until a value is received.
		c <- x*x  // blocking here until the result is received.
	}()
	c <- 3   // blocking here until the value is received.
	y := <-c // blocking here until the result is received.
	fmt.Println(y) // 9
}
```

使用缓冲通道的示例。该程序不是并发的。通道关闭后，可以从中接收无限个值。

```go
package main

import "fmt"

func main() {
	c := make(chan int, 2) // a buffered channel
	c <- 3
	c <- 5
	close(c)
	fmt.Println(len(c), cap(c)) // 2 2
	x, ok := <-c
	fmt.Println(x, ok) // 3 true
	fmt.Println(len(c), cap(c)) // 1 2
	x, ok = <-c
	fmt.Println(x, ok) // 5 true
	fmt.Println(len(c), cap(c)) // 0 2
	x, ok = <-c
	fmt.Println(x, ok) // 0 false
	x, ok = <-c
	fmt.Println(x, ok) // 0 false
	fmt.Println(len(c), cap(c)) // 0 2
	close(c) // panic!
	c <- 7   // also panic if the above close call is removed.
}

```

一个永不会结束的足球游戏：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var ball = make(chan string)
	kickBall := func(playerName string) {
		for {
			fmt.Println(<-ball, "kicked the ball.")
			time.Sleep(time.Second)
			ball <- playerName
		}
	}
	go kickBall("John")
	go kickBall("Alice")
	go kickBall("Bob")
	go kickBall("Emily")
	ball <- "referee" // kick off
	var c chan bool   // nil
	<-c               // blocking here for ever
}
```
可以通过阅读[channel use cases](https://go101.org/article/channel-use-cases.html)了解更多关于使用通道的例子。

## 通道元素值是通过拷贝传输的

当一个值从通道中接收或者发送，将发生拷贝操作。就像值分配和函数参数传递一样，当一个值被拷贝时，[只有它的直接部分被拷贝](https://go101.org/article/value-part.html#about-value-copy)。

对于标准的Go编译器来说，通道元素类型的大小必须小于`65535`。然而，通常，我们不应该创建超大元素类型的通道，因为所有通道内的发送和接收操作都需要拷贝传输的值。当一个元素值从一个goroutine拷贝到另一个goroutine时，将发生两次值拷贝。所以如果传递的值大小过大，最好是使用元素类型指针代替，以避免大量的拷贝开销。

## 关于通道和Goroutine垃圾回收

注意，一个通道会被该通道内发送或接收goroutine队列中的所有goroutines引用，一次如果通道的两个队列都不为空，那么通道将不会被垃圾回收。另一方面，如果一个goroutine被阻塞并仍然停留在通道发送或接收goroutine队列中，那么这个goroutine也不会被垃圾回收，尽管这个通道只被这个goroutine引用。实际上，一个goroutine只会在其退出时被垃圾回收掉。

## 通道发送和接收操作只是一条简单的语句

注意，通道的发送操作和接收操作只是[简单语句](https://go101.org/article/expressions-and-statements.html#simple-statements)。一个通道接收操作同样可以被当作一个单值表达式。可以在[基本控制流代码块](https://go101.org/article/control-flows.html)的某些部分使用简单语句和表达式。

例子：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fibonacci := func() chan uint64 {
		c := make(chan uint64)
		go func() {
			var x, y uint64 = 0, 1
			for ; y < (1 << 63); c <- y { // here
				x, y = y, x+y
			}
			close(c)
		}()
		return c
	}
	c := fibonacci()
	for x, ok := <-c; ok; x, ok = <-c { // here
		time.Sleep(time.Second)
		fmt.Println(x)
	}
}
```

## 通道的For-Range操作

`for-range`控制流语句可以在通道上使用。此循环会尝试迭代接收发送到一个通道上的值，直到这个通道被关闭而且该通道值缓冲队列没有值为止。不同于数组，切片和map中的`for-range`语句，大部分单一迭代变量允许出现在通道的`for-range`语法当中。

```go
for v = range aChannel {
	// use v
}
```

等价于

```go
for {
	v, ok = <-aChannel
	if !ok {
		break
	}
	// use v
}
```

当然，此处的`aChannel`值不能是仅发送通道。如果它是一个`nil`通道，那么这个循环将永久阻塞。

例如，上面例子中的第二个`for`循环可以简化成下例中的代码：

```go
for x := range c {
  time.Sleep(time.Second)
  fmt.Println(x)
}
```

## `select-case`控制流代码块

`select-case`控制流代码块是Go特意为通道而设计的语句。此语句在语法上和`switch`代码块很相似，例如，它们都可以有多个`case`分支，而且在`select-case`语句中至多只有一个`default`分支。但是它们之间也有一些明显的区别。

- 关键字`select`之后（`{`之前）不允许有语句和表达式。

- `case`分支里不允许有`fallthrough`语句。

- 在`select-case`代码块中每个关键字`case`后面跟随的语句必须是通道接收或者发送语句。

- 如果在`select-case`代码块中有一些非阻塞的通道操作，那么Go运行时将随机选择一个非阻塞通道操作来执行，然后继续执行相应的`case`分支。

- 如果在`select-case`代码块中在`case`关键字之后所有的通道操作都是阻塞操作，那么默认的分支将被选中来执行，如果默认的分支存在的话。如果没有默认分支，当前的goroutine将被推入相应的通道发送goroutine队列中， `case`关键字后面的通道操作涉及的每个通道的接收goroutine队列将会进入阻塞状态。

通过这些规则，如果一个没有任何分支的`select-case`的代码块：`select {}`，将会使当前的goroutine进入永久阻塞状态。

下面例子中的程序会进入默认分支执行：

```go
package main

import "fmt"

func main() {
	var c chan struct{} // nil
	select {
	case <-c:             // blocking operation
	case c <- struct{}{}: // blocking operation
	default:
		fmt.Println("Go here.")
	}
}
```

一个展示如何尝试发送和尝试接收的例子：

```go
package main

import "fmt"

func main() {
	c := make(chan string, 2)
	trySend := func(v string) {
		select {
		case c <- v:
		default: // go here if c is full.
		}
	}
	tryReceive := func() string {
		select {
		case v := <-c: return v
		default: return "-" // go here if c is empty.
		}
	}
	trySend("Hello!")
	trySend("Hi!")
	trySend("Bye!") // fail to send, but will not blocked.
	fmt.Println(tryReceive()) // Hello!
	fmt.Println(tryReceive()) // Hi!
	fmt.Println(tryReceive()) // -
}
```

当下例中的两个`case`操作都是非阻塞的情况下面，将会有50%的可能导致panic。

```go
package main

func main() {
	c := make(chan struct{})
	close(c)
	select {
	case c <- struct{}{}: // panic if this case is selected.
	case <-c:
	}
}
```

## Select机制的实现

Go中的select机制是一个重要且独一无二的特性。这里列出了[官方Go运行时select机制实现](https://github.com/golang/go/blob/master/src/runtime/select.go)的步骤。

执行一个select代码块有以下几个步骤：

1. 对所有涉及到的通道和可能发送的值进行求值，从上到下，从左到右。

2. 随机轮询`case`语句，默认的分支可以被看作是一个特殊的`case`。该顺序下相应的通道可能是重复的。默认的分支应该始终放在最后的位置。

3. 对所有涉及到的通道进行排序，以避免在下一步骤中出现死锁。排序结果的前`N`个通道没有重复的通道。这里的`N`是`select`代码块中涉及到的通道数。下面步骤中的**已排序的锁定顺序(sorted lock order)** 表示的就是前`N`个非重复的通道。

4. 按照最后一步的**已排序的锁定顺序**来锁定相关的通道。

5. 按照随机的`case`顺序来轮询`select`代码块中的每个`case`：

    1. 如果相关的通道操作是一个**发送值到已关闭的通道**操作，解锁所有的通道并panic，直接进入第12步。
    2. 如果相关的通道操作是非阻塞的，运行这个通道操作并解锁所有的通道，然后执行相关的`case`分支体。此通道的操作可能会唤醒其他阻塞的goroutine。 直接进入第12步。
    3. 如果`case`是默认分支，那么将解锁所有通道且执行默认分支体。直接进入第12步。

6. 将当前的goroutine（和`case`通道的操作信息）推入（入队）到每个`case`操作中相关通道的接收或发送goroutine队列中。 当前的gooutine可能会入队多次，在多种情况下，相关的通道可能是相同的。

7. 阻塞当前的goroutine并通过**已排序额锁定顺序**来解锁其他通道。

8. ...，处在阻塞状态下，等待其他的通道操作来唤醒当前的goroutine。...

9. 当前的goroutine被另一个goroutine中的通道操作唤醒。这里其他的操作可能是一个通道关闭操作或是一个通道接收/发送操作。如果它是一个通道接收/发送操作，则必须有与其配合的`case`通道的接收/发送操作。在此次配合中，当前的goroutine将会从通道的接收/发送goroutine队列中出队。

10. 通过**已排序的锁定顺序**来锁定所有涉及的通道。

11. 在每个`case`操作中将当前的goroutine从涉及的通道中的发送/接收goroutine队列中移出。

    1. 如果当前的goroutine被一个通道关闭操作唤醒，则进入第5步。
    2. 如果当前的goroutine被一个通道接收/发送操作唤醒，那么在其出队过程中已经找到了与之配合接收/发送操作的相关`case`, 所以只需要按照**已排序的锁定顺序**解锁所有通道并执行相应的`case`体。

12. 完成。

通过上述实现步骤，我们知道

- 一个goroutine可能会在同一时间处在多个通道的发送/接收goroutine队列里。 它甚至可以同时驻留在同一通道的发送和接收goroutine队列中。

- 当一个goroutine在一个`select-case`代码块被阻塞时，它将会被重新恢复，它将从`select-case`代码块中跟随`case`关键字后的通道操作涉及到的每个通道的发送和接收goroutine队列中删除。

## 更多

我们可以在[这篇文章](https://go101.org/article/channel-use-cases.html)中找到更多通道的用法。尽管通道可以帮助我们[轻易写出正确的并发代码](https://go101.org/article/channel-closing.html)，像其他数据同步技术一样，通道也不能阻止你[写出不正确的并发代码](https://go101.org/article/concurrent-common-mistakes.html)。而且通道并不一定是所有数据同步问题的最佳解决方案。请阅读[这篇文章](https://go101.org/article/concurrent-synchronization-more.html)和[这篇文章](https://go101.org/article/concurrent-atomic-operation.html)以获得更多Go中的数据同步技术。
