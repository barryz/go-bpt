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

参数或者返回值通道可以是缓冲的，以便响应方不需要等待请求方取出被传输的值。

有时候，请求方不能保证回复正常的值。由于各种原因，可能会返回一个错误。像这样的场景，我们可以使用类似于`type{v T; err error}`这样的结构体或者空接口类型作为通道的元素类型。

有时候，由于一些原因，响应可能会比期待的时间等待更久才能返回，或者永远不可达。我们可以使用下面将要介绍的超时机制来处理这种情况。

有时候，响应端可能会返回一连串的值，这种数据流机制也会将下面介绍到。

## 使用通道作为通知

通知也可以被看作是一种特殊的请求/响应，由通道接收或者发送的值可能不太重要。通常来说，我们使用空结构体类型`struct{}`作为通道的元素类型，因为类型`struct{}`的大小是0，因此`struct{}`的值不占用任何内存。

### 通过发送一个值到通道实现一对一的通知

如果从一个通道没有接收到任何值，那么在该通道上的下一次接收操作将会阻塞，直到另一个goroutine发送一个值到该通道。因此我们可以发送一个值到一个通道以此来通知另一个在该通道上等待接收值的goroutine。

在下面的例子中，通道`done`被用作一个信号通道来通知一些事件。

```go
package main

import (
	"crypto/rand"
	"fmt"
	"os"
	"sort"
)

func main() {
	values := make([]byte, 32 * 1024 * 1024)
	if _, err := rand.Read(values); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	done := make(chan struct{})
	go func() { // the sorting goroutine
		sort.Slice(values, func(i, j int) bool {
			return values[i] < values[j]
		})
		done <- struct{}{} // notify sorting is done
	}()

	// do some other things ...

	<- done // waiting here for notification
	fmt.Println(values[0], values[len(values)-1])
}
```

### 通过从一个通道接收一个值来实现一对一通知

如果一个可缓冲通道的值是满的，那么该通道上一个发送操作将会阻塞，直到另一个goroutine从该通道上接收一个值。因此我们可以通过在一个通道上接收一个值以此来通知在该通道上等待发送值的goroutine。通常，这种通道应该是一个无缓冲的通道。

这种通知方式的使用方式比上一个示例中介绍的方式要少得多。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	done := make(chan struct{}, 1) // the signal channel
	done <- struct{}{}             // fill the channel
	// Now the channel done is full. A new send will block.

	go func() {
		fmt.Print("Hello")
		time.Sleep(time.Second * 2) // simulate a workload
		<- done // receive a value from the done channel, to
		        // unblock the second send in main goroutine.
	}()

	// do some other things ...

	done <- struct{}{} // block here until a receive is made.
	fmt.Println(" world!")
}
```

实际上，上面例子中的通道充当了一个一次性的二进制信号量，这个我们将在下面介绍。

### 多对一通知和一对多通知

通过扩展上面的两个例子，很容易实现一对多和多对一的通知。

```go
package main

import "log"
import "time"

func worker(id int, ready <-chan struct{}, done chan<- struct{}) {
	<-ready // wait until ready is closed
	log.Print("Worker#", id, " started to process.")
	time.Sleep(time.Second) // simulate a workload
	log.Print("Worker#", id, " finished its job.")
	done <- struct{}{} // notify the main goroutine (N-to-1)
}

func main() {
	log.SetFlags(0)

	ready, done := make(chan struct{}), make(chan struct{})
	go worker(0, ready, done)
	go worker(1, ready, done)
	go worker(2, ready, done)

	time.Sleep(time.Second * 2) // simulate an initialzation phase
	// 1-to-N notifications.
	ready <- struct{}{}; ready <- struct{}{}; ready <- struct{}{}
	// Being N-to-1 notified.
	<-done; <-done; <-done
}
```

事实上，上面例子中的一对多喝多对一的方式在实际上并不经常使用。实践中，我们经常使用`sync.WaitGroup`来实现多对一通知。并且，我们通过关闭通道的方式来实现一对多的通知。请阅读下一小节了解更多信息。

### 通过关闭一个通道实现广播（一对多）通知

上面子章节中所展示的一对多通知的方式在实践中极少被使用，因为有一个更好的方式。通过使用可以从已关闭的通道无限接收值的功能，我们可以通过关闭通道以实现广播通知。

通过上面子章节中的例子，我们可以通过替换三个通道发送操作`ready <-struct{}{}`为一个通道关闭操作`close(ready)`来实现一对多通知。

```go
...

	// ready <- struct{}{}; ready <- struct{}{}; ready <- struct{}{}
	close(ready) // broadcast notifications
...
```

当然，我们也可以通过关闭通道实现一对一的通知。实际上，这是Go中最常用的通知方式。可以从已关闭通道无限接收值的特性将在下面介绍的许多其他用例中使用。

### 使用同一个通道通知多次

和之前一个响应可以返回一连串的结果一样，我们也可以通过使用通道实现多个一对一的通知。逻辑很简单，所以不需要再使用例子暂时如何操作。

### 定时器，调度通知

通过使用通道可以非常容易的实现一个一次性的定时器。

一个自定义的一次性定时器实现：

```go
package main

import (
	"fmt"
	"time"
)

func AfterDuration(d time.Duration) <- chan struct{} {
	c := make(chan struct{}, 1)
	go func() {
		time.Sleep(d)
		c <- struct{}{}
	}()
	return c
}

func main() {
	fmt.Println("Hi!")
	<- AfterDuration(time.Second)
	fmt.Println("Hello!")
	<- AfterDuration(time.Second)
	fmt.Println("Bye!")
}
```

实际上，`time`标准库中的`After`函数通过一个更加高效的实现提供了上述例子中相同的功能。我们应该使用该函数来代替我们自己编写的自定义函数。

请注意，`<-time.After(aDuration)`将会使当前的goroutine进入到阻塞状态，但是`time.Sleep(aDuration)`并不会。

`<-time.After(aDuration)` 通常用于超时机制。这点将会在下面的章节中介绍到。

## 使用通道作为互斥锁

上面某个例子中已经展示了容量为1的缓冲通道可以被用作一次性的[二进制信号量](https://en.wikipedia.org/wiki/Semaphore_(programming))。实际上，这样的通道也可以被用作多次的二进制信号量，又名，互斥锁，尽管这样的互斥锁在效率上不如`sync`标准库提供的互斥锁高效。

使用单一容量缓冲通道作为互斥锁有两种形式。

1. 锁定发送，通过接收解锁。
2. 锁定接收，通过发送解锁。

在这里，我们只展示只锁发送的例子。需要注意的是，用作互斥量的通道的容量必须是1。

```go
package main

import "fmt"

func main() {
	mutex := make(chan struct{}, 1) // the capacity must be one

	counter := 0
	increase := func() {
		mutex <- struct{}{} // lock
		counter++
		<-mutex // unlock
	}

	increase1000 := func(done chan<- struct{}) {
		for i := 0; i < 1000; i++ {
			increase()
		}
		done <- struct{}{}
	}

	done := make(chan struct{})
	go increase1000(done)
	go increase1000(done)
	<-done; <-done
	fmt.Println(counter) // 2000
}
```

## 使用通道作为计数信号量

容量大于1的可缓冲通道可以被用作为[计数信号量](https://en.wikipedia.org/wiki/Semaphore_(programming))。计数信号量可以被视为多所属者锁（multi-owner locks）。如果一个通道的容量是`N`，那么它可以被看作是在任意时间拥有至多`N`个所属者的信号量。二进制信号量（互斥量）是一种特殊的计数信号量，每个二进制信号量在任意时间至多只有一个所属者。计数信号量通常被用于吞吐量限制以及确认资源配额。

和将通道作为互斥量使用一样，获得通道信号量的一个所有权也有两种形式。

1. 通过发送获取所有权，通过接收操作释放。
2. 通过接收获取所有权，通过发送操作释放。

通过从通道接收值来获取所有权的示例。

```go
package main

import (
	"log"
	"time"
	"math/rand"
)

type Seat int
type Bar chan Seat

func (bar Bar) ServeConsumer(customerId int) {
	log.Print("-> consumer#", customerId, " enters the bar")
	seat := <- bar // need a seat to drink
	log.Print("consumer#", customerId, " drinks at seat#", seat)
	time.Sleep(time.Second * time.Duration(2 + rand.Intn(6)))
	log.Print("<- consumer#", customerId, " frees seat#", seat)
	bar <- seat // free the seat and leave the bar
}

func main() {
	rand.Seed(time.Now().UnixNano())

	bar24x7 := make(Bar, 10) // the bar has 10 seats
	// Place seats in an bar.
	for seatId := 0; seatId < cap(bar24x7); seatId++ {
		bar24x7 <- Seat(seatId) // none of the sends will block
	}

	for customerId := 0; ; customerId++ {
		time.Sleep(time.Second)
		go bar24x7.ServeConsumer(customerId)
	}
	for {time.Sleep(time.Second)} // sleeping != blocking
}
```

上面的例子中，只有拿到seat的消费者才能drink。所以在给定的时间内至多只有十个消费者能够drinking。

在`main`函数中最后一个`for`循环用来避免程序退出。但是有一个更好的方式来处理这个工作，这个将会在下节介绍。

尽管在给定的时间内至多只有十个消费者可以drinking，但在同一时间bar可能会服务于多于十个的消费者。有一些消费者在等待空闲的seats。虽然每个消费者goroutine消耗的资源比系统线程少得多，但是大量的goroutines消耗的总资源是不可忽视的。所以最好是在有空闲的seat可用的情况下再创建消费者goroutine。

```go
... // same code as the above example

func (bar Bar) ServeConsumerAtSeat(customerId int, seat Seat) {
	log.Print("consumer#", customerId, " drinks at seat#", seat)
	time.Sleep(time.Second * time.Duration(2 + rand.Intn(6)))
	log.Print("<- consumer#", customerId, " frees seat#", seat)
	bar <- seat // free the seat and leave the bar
}

func main() {
	rand.Seed(time.Now().UnixNano())

	bar24x7 := make(Bar, 10) // the bar has 10 seats
	// Place seats in an bar.
	for seatId := 0; seatId < cap(bar24x7); seatId++ {
		bar24x7 <- Seat(seatId) // none of the sends will block
	}

	for customerId := 0; ; customerId++ {
		time.Sleep(time.Second)
		seat := <- bar24x7 // need a seat to serve next consumer
		go bar24x7.ServeConsumerAtSeat(customerId, seat)
	}
	for {time.Sleep(time.Second)} // sleeping != blocking
}
```

在上面优化的版本中，将有大约十个真正的消费者goroutines共存。

通过发送获取所有权的方式比较简单。并没有放置seats的步骤。

```go
package main

import (
	"log"
	"time"
	"math/rand"
)

type Consumer struct{id int}
type Bar chan Consumer

func (bar Bar) ServeConsumer(c Consumer) {
	log.Print("-> consumer#", c.id, " starts drinking")
	time.Sleep(time.Second * time.Duration(10 + rand.Intn(10)))
	log.Print("<- consumer#", c.id, " leaves the bar")
	<- bar // leaves the bar and save a space
}

func main() {
	rand.Seed(time.Now().UnixNano())

	bar24x7 := make(Bar, 10) // can serve most 10 consumers
	for customerId := 0; ; customerId++ {
		time.Sleep(time.Second)
		consumer := Consumer{customerId}
		bar24x7 <- consumer // try to enter the bar
		go bar24x7.ServeConsumer(consumer)
	}
	for {time.Sleep(time.Second)}
}
```

通道信号量的用例有一个变种。在上面的两个例子中，尽管吞吐量被限制了，但是可能会堆积很多的请求（消费者）。这并不总是一个好主意，有时最好通过使用下面即将介绍的try-receive或try-send机制来建议在队列中的消费者去其他的bars。

## Ping-Pong（对话）

有时候，将在两个goroutines之间来回处理一段数据，或者传递消息。这就像是一个ping-pong游戏或者是两个goroutines正在对话。

一个打印一系列斐波那契数列的例子：

```go
package main

import "fmt"
import "time"
import "os"

type Ball uint64

func Play(playerName string, table chan Ball) {
	var lastValue Ball = 1
	for {
		ball := <- table // get the ball
		fmt.Println(playerName, ball)
		ball += lastValue
		if ball < lastValue { // overflow
			os.Exit(0)
		}
		lastValue = ball
		table <- ball // bat back the ball
		time.Sleep(time.Second)
	}
}

func main() {
	table := make(chan Ball)
	go func() {
		table <- 1 // throw ball on table
	}()
	go Play("A:", table)
	Play("B:", table)
}
```

## 在通道内封装通道

有时候，我们可以使用一个通道类型作为另一个通道的元素类型。在下面的例子中，`<-chan chan<- int`是一个元素类型为仅发送类型通道`chan<- int`的仅接收通道。

```go
package main

import "fmt"

var counter = func (n int) chan<- chan<- int { // chan<- (chan<- int)
	requests := make(chan chan<- int) // chan (chan<- int)
	go func() {
		for request := range requests {
			if request == nil {
				n++ // increase
			} else {
				request <- n // retrieve
			}
		}
	}()
	return requests // implicitly converted to chan<- (chan<- int)
}(0)

func main() {
	increase1000 := func(done chan<- struct{}) {
		for i := 0; i < 1000; i++ {
			counter <- nil
		}
		done <- struct{}{}
	}

	done := make(chan struct{})
	go increase1000(done)
	go increase1000(done)
	<-done; <-done

	request := make(chan int, 1)
	counter <- request
	fmt.Println(<-request) // 2000
}
```

尽管上面例子中的封装实现可能不够高效，但是在有些场景下却是非常有用的。

## 检查通道的容量和长度

我们可以使用内建函数`cap`和`len`来检查一个通道的容量和长度，尽管在实践中我们极少使用它。我们极少使用它的原因是当使用`len`函数来检查一个通道的长度时，该长度可能会在检查之后发生变化。极少使用`cap`函数的原因是对我们来说，一个通道的容量或许并不是那么的重要。

然而，在有些场景下我们也许会需要这两个函数。

例如，有时候，我们想要接收在非关闭的通道`c`中 缓冲的所有的值，此时并没有人会再发送值，那么我们就可以使用下面的代码来接收余下的所有值。

```go
for len(c) > 0 {
	value := <-c
	// use value ...
}
```

我们也可以使用下面即将介绍的try-receive机制来做这样的事情。这两种方式的效率几乎是相同的。

有时候，一个goroutine可能希望将一些值写入到缓冲的通道`c`中，直到该通道已满，而且不会在结束时进入阻塞状态，并且该goroutine是该通道的唯一发送者，那么我们可以使用如下的代码进行这样的工作：

```go
for len(c) < cap(c) {
	c <- aValue
}
```

## 永久阻塞当前的goroutine

`select`机制在Go中是一个独一无二的特性。它为并发编程带来了很多的模式和技巧。关于如何使用`select`机制，请阅读文章[channels in Go](https://go101.org/article/channel.html#select)。

我们可以使用空选择代码块`select {}`来永久阻塞当前的goroutine。这是`select`机制最简单的用例。实际上，上面的例子中的一些代码`for {time.Sleep(time.Second)}`可以使用`select {}`替换。

通常，`select{}`用于阻止主goroutine退出，因为一旦主goroutine退出，整个程序也将退出。

例子：

```go
package main

import "time"

func DoSomething() {
	for {
		// do something ...
		time.Sleep(time.Hour) // sleeping is not blocking
	}
}

func main() {
	go DoSomething()
	select{}
}
```

顺便说下，也有[some other ways](https://go101.org/article/summaries.html#block-forever)来使一个goroutine进入永久阻塞状态，但是`select{}`是最简单的一个。


## 尝试发送和尝试接收

在Go中，有一个`default`分支和只有一个`case`分支的选择块被称作一个尝试发送或尝试接收的通道操作，接收或发送取决于跟在`case`关键字后面的语句是一个接收操作还是一个发送操作。

- 如果跟在`case`关键字后的是一个通道发送操作，那么该选择代码块被称为尝试发送操作。如果该发送操作会被阻塞，那么`default`分支将会执行（发送失败），否则，唯一的`case`分支将会被执行，且发送操作将成功结束。

- 如果跟在`case`关键字后的是一个通道接收操作，那么该选择代码块被称为尝试接收操作。如果该发送操作会被阻塞，那么`default`分支将会执行（接收失败），否则，唯一的`case`分支将会被执行，且接收操作将成功结束。

尝试发送和尝试接收操作永远不会阻塞。

标准的Go编译器对尝试发送和尝试接收操作做了一些优化，它们的执行效率远高于多case选择代码块。

下面是一个展示尝试发送和尝试接收操作是如何工作的。

```go
package main

import "fmt"

func main() {
	type Book struct{id int}
	bookshelf := make(chan Book, 3)

	for i := 0; i < cap(bookshelf) * 2; i++ {
		select {
		case bookshelf <- Book{id: i}:
			fmt.Println("succeed to put book", i)
		default:
			fmt.Println("failed to put book")
		}
	}

	for i := 0; i < cap(bookshelf) * 2; i++ {
		select {
		case book := <-bookshelf:
			fmt.Println("succeed to get book", book.id)
		default:
			fmt.Println("failed to get book")
		}
	}
}

/*
Output:

succeed to put book 0
succeed to put book 1
succeed to put book 2
failed to put book
failed to put book
failed to put book
succeed to get book 0
succeed to get book 1
succeed to get book 2
failed to get book
failed to get book
failed to get book
*/
```

之后的子章节将会展示更多的尝试发送和尝试接收的用例。

### 检查一个无缓冲通道是否关闭而不用阻塞当前的goroutine

假设保证没有值可以被发送到一个无缓冲的通道，我们可以使用下面例子中的代码来检查一个无缓冲的通道是否已经关闭，而不用阻塞当前的goroutine，这里元素类型`T`是相应的通道类型。

```go
func IsClosed(c chan T) bool {
	select {
	case <-c:
		return true
	default:
	}
	return false
}
```

用这种方式来检查一个无缓冲通道是否关闭在Go的并发编程实践中很流行。


### 峰值限制

我们在上节[use channels as counting semaphores](https://go101.org/article/channel-use-cases.html#semaphore)的结尾提到了，如果目前的bar例没有可用的seats，最好是建议新来的消费者去其他的bar。这就被称作峰值限制。

下面的例子是关于[use channels as counting semaphores](https://go101.org/article/channel-use-cases.html#semaphore)的修改版本。

```go
...
	bar24x7 := make(Bar, 10) // can serve most 10 consumers
	for customerId := 0; ; customerId++ {
		time.Sleep(time.Second)
		consumer := Consumer{customerId}
		select {
		case bar24x7 <- consumer: // try to enter the bar
			go bar24x7.ServeConsumer(consumer)
		default:
			log.Print("consumer#", customerId, " goes elsewhere")
		}
	}
...
```
