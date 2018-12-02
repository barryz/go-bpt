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

### 另一种方式实现First-Response-Wins用例

正如上面提到的，我们可以使用`select`机制（尝试发送）配合一个容量至少为1的缓冲通道来实现first-response-wins用例。例如，

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func source(c chan<- int32) {
	ra, rb := rand.Int31(), rand.Intn(3)+1
	time.Sleep(time.Duration(rb) * time.Second) // sleep 1s, 2s or 3s
	select {
	case c <- ra:
	default:
	}
}

func main() {
	rand.Seed(time.Now().UnixNano())

	c := make(chan int32, 1) // the capacity should be at least 1
	for i := 0; i < 5; i++ {
		go source(c)
	}
	rnd := <-c // only the first response is used
	fmt.Println(rnd)
}
```

请注意，上面例子中的通道的缓冲元素个数至少是一个，这样如果接收/请求方没有及时准备就不会错过第一次发送。

### 第三种实现First-Response-Wins用例的方式

对于一个first-response-wins用例来说，如果源的数量很少，两个或者三个，我们可以使用一个`select`代码块来同时接收源响应。例如，

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func source() <-chan int32 {
	c := make(chan int32, 1) // must be a buffered channel
	go func() {
		ra, rb := rand.Int31(), rand.Intn(3)+1
		time.Sleep(time.Duration(rb) * time.Second)
		c <- ra
	}()
	return c
}

func main() {
	rand.Seed(time.Now().UnixNano())

	var rnd int32
	select{
	case rnd = <-source():
	case rnd = <-source():
	case rnd = <-source():
	}
	fmt.Println(rnd)
}
```

上面介绍的两种方式以及最后子小节的方式同样也可以用作N(1)-to-1通知。

### 超时

在一些请求/响应的场景中，由于各种原因，一个响应可能需要等待很长时间才能返回给请求，有时候甚至不会有响应。对于这样的情况，我们应该使用超时机制返回给客户端一个错误信息。这样的超时机制可以通过`select`机制实现。

下面的代码将展示如何实现一个带有超时机制的请求。

```go
func requestWithTimeout(timeout time.Duration) (int, error) {
	c := make(chan int)
	go doRequest(c) // may need a long time to response

	select {
	case data := <-c:
		return data, nil
	case <-time.After(timeout):
		return 0, errors.New("timeout")
	}
}
```

### 断续器（Ticker）

我们可以使用一个缓冲通道配合一个尝试发送机制来实现一个断续器。。

```go
package main

import "fmt"
import "time"

func Tick(d time.Duration) <-chan struct{} {
	c := make(chan struct{}, 1) // the capacity should be exactly one
	go func() {
		for {
			time.Sleep(d)
			select {
			case c <- struct{}{}:
			default:
			}
		}
	}()
	return c
}

func main() {
	t := time.Now()
	for range Tick(time.Second) {
		fmt.Println(time.Since(t))
	}
}
```

实际上，标准库`time`中有个函数`Tick`提供了相同的功能，但是效率会高的多。我们应该使用此函数以避免使用自定义的断续器。

### 限速（Rate Limiting）

上面中的一个例子已经展示了如何使用尝试发送机制来实现峰值限制。我们同样可以使用尝试发送机制来实现限速。下面是一个从[Go官方wiki](https://github.com/golang/go/wiki/RateLimiting)引用的一个例子。在这个例子中，长时间内每秒处理的平均请求数不会超过10。但是有时候，在某个短周期内，可能会有5个左右的请求被并发处理。

```go
import "time"

type Request interface{}
func handle(Request) {/* do something */}

const RateLimit = 10
const BurstLimit = 5 // 1 means bursts are not supported.

func handleRequests(requests <-chan Request) {
	throttle := make(chan time.Time, BurstLimit)

	go func() {
		tick := time.NewTicker(time.Second / RateLimit)
		defer tick.Stop()
		for t := range tick.C {
			select {
			case throttle <- t:
			default:
			}
		}
	}()

	for reqest := range requests {
		<-throttle
		go handle(reqest)
	}
}
```

### 开关

从[channels in Go](https://go101.org/article/channel.html)文章中，我们已经学习到如果一个goroutine尝试发送（到）或接收（从）一个nil通道时，该goroutine会永久阻塞。通过利用这一事实，我们可以改变`select`代码块中涉及的通道，以影响`select`代码块中的分支选择。

下面是一个使用`select`机制实现的另一个ping-pong例子。在这个例子中，`select`块中涉及的两个通道变量之一是`nil`。对应于`nil`通道的`case`分支将无法确定。我们可以认为这样的`case`分支处于关闭状态。在每个循环步骤的结束时，两个`case`分支的开关状态将会被交换。

```go
package main

import "fmt"
import "time"
import "os"

type Ball uint8
func Play(playerName string, table chan Ball, serve bool) {
	var receive, send chan Ball
	if serve {
		receive, send = nil, table
	} else {
		receive, send = table, nil
	}
	var lastValue Ball = 1
	for {
		select {
		case send <- lastValue:
		case value := <- receive:
			fmt.Println(playerName, value)
			value += lastValue
			if value < lastValue { // overflow
				os.Exit(0)
			}
			lastValue = value
		}
		receive, send = send, receive // switch on/off
		time.Sleep(time.Second)
	}
}

func main() {
	table := make(chan Ball)
	go Play("A:", table, false)
	Play("B:", table, true)
}
```

### 控制代码异常的可能性权重

我们可以在`select`代码块中重复一个`case`分支，以增加相应代码片段的执行可能性的权重。

例子：

```go
package main

import "fmt"

func main() {
	foo, bar := make(chan struct{}), make(chan struct{})
	close(foo); close(bar) // for demo purpose
	x, y := 0.0, 0.0
	f := func(){x++}
	g := func(){y++}
	for i := 0; i < 100000; i++ {
		select {
		case <-foo: f()
		case <-foo: f()
		case <-bar: g()
		}
	}
	fmt.Println(x/y) // about 2
}
```

函数`f`被调用的可能性将会是函数`g`被调用的可能性的两倍。

### 从动态数字中选择

我们可以使用`reflect`标准库中的一些函数在运行时构造一个`select`代码块。这个动态创建的`select`代码块可以拥有任意多个`case`分支。但是请注意，反射的方式相较于常规的方式是十分低效的。

`reflect`标准库同样提供了`TrySend`和`TryRecv`函数来实现one-case-plus-default `select`代码块。

## 数据流操作

本节将介绍一些使用长生命周期的通道进行数据流操作的用例。在实践中有很多与数据流相关的应用场景。例如消息队列（pub/sub），大数据处理（map/reduce），负载均衡以及劳务分工等等。

通常，一个数据流的应用是由很多模块组成的。不同的模块做不同的工作。每个模块拥有自己的一组workers（goroutine），这些workers用来并发地执行由该模块指定的工作。这里是实践中一些模块可能会做的工作：

- 数据生成/收集/加载。
- 数据服务/存储。
- 数据验证/分析。
- 数据聚合/区分。
- 数据组合/分解。
- 数据复制/扩散。

一个模块内的worker可能会从其他的模块接收数据，并且有可能向其他的模块发送数据。换言之，一个模块可能既是生产者也可能是消费者。一个只发送数据至其他模块，但不从其他模块接收数据的，我们称之为仅发送者模块。一个只从其他模块接收数据，但不发送数据至其他模块的，我们称之为仅接收者模块。

众多的模块组成了一个数据流系统。

下面我们将展示一些数据流模块工作者实现。这些实现仅供教学示例，所以它们可能既不高效也不灵活。

### 数据生成/收集/加载

有各类的仅发送者模块。一个仅发送者模块的工作者可以生产数据流

- 通过加载一个文件，从数据库中读取，或者从网络上爬取。
- 通过从各种硬件中收集各种指标。
- 通过生成随机数。
- 等等。

在这里，我们使用一个随机数生成作为示例。生成函数能返回一个结果值但不需要接收任何参数。

```go
import (
	"crypto/rand"
	"encoding/binary"
)

func RandomGenerator() <-chan uint64 {
	c := make(chan uint64)
	go func() {
		rnds := make([]byte, 8)
		for {
			_, err := rand.Read(rnds)
			if err != nil {
				close(c)
			}
			c <- binary.BigEndian.Uint64(rnds)
		}
	}()
	return c
}
```

实际上，这个随机数生成器是一个多返回值的期刊，这个已经在本文之前的内容中介绍过了。

数据生产者可以随时关闭输出流通道以结束数据生成。

### 数据聚合

数据聚合模块工作者将相同数据类型的若干数据流聚合到一个流中。假设数据类型是`int64`，下面的函数将会将任意多的数据流聚合到一个流内。

```go
func Aggregator(inputs ...<-chan uint64) <-chan uint64 {
	output := make(chan uint64)
	for _, in := range inputs {
		in := in // this line is important
		go func() {
			for {
				output <- <-in // <=> output <- (<-in)
			}
		}()
	}
	return output
}
```

更好的实现应该考虑输入流是否已经关闭。（这个也适用于以下其他模块工作者的实现。）

```go
...
		in := in // this line is important
		go func() {
			for {
				x, ok := <-in
				if ok {
					output <- x
				} else {
					close(output)
				}
			}
		}()
...
```

如果需要聚合的数据非常少（两到三个），我们可以使用`select`块来聚合这些数据流。

```go
// Assume the number of input stream is two.
...
	output := make(chan uint64)
	go func() {
		inA, inB := inputs[0], inputs[1]
		for {
			select {
			case v := <- inA: output <- v
			case v := <- inB: output <- v
			}
		}
	}
...
```

### 数据区分

数据区分模块工作者所做的工作与数据聚合模块工作者的工作正好相反。实现一个数据区分工作者很容易，但是在实践中，数据区分工作者并不是很有用，且很少被用到。

```go
func Divisor(input <-chan uint64, outputs ...chan<- uint64) {
	for _, out := range outputs {
		out := out // this line is important
		go func() {
			for {
				out <- <-input // <=> out <- (<-input)
			}
		}()
	}
}
```

### 数据组合

数据组合和数据聚合很相似，但是数据组合工作者会合并不同数据类型的数据流。对于数据聚合来说，两条数据仍然是两条数据。但是对于数据组合来说，几条数据会被组合成一条数据。

下面是一个数据组合工作者的例子，其中来自一个流的两个`uint64`值和来自另一个流的一个`uint64`值组成一个新的`uint64`值。通常来说，在实践中，这些流的通道元素类型都是不同的。这里使用相同的类型只是为了解释时更加方便。

```go
func Composor(inA <-chan uint64, inB <-chan uint64) <-chan uint64 {
	output := make(chan uint64)
	go func() {
		for {
			a1, b, a2 := <-inA, <-inB, <-inA
			output <- a1 ^ b & a2
		}
	}()
	return output
}
```

### 数据分解

数据分解是数据组合的逆过程。分解工作者函数的实现是获取一个输入数据流参数并返回多个数据流结果。

### 数据复制/扩散

数据复制（扩散）可以看作是数据分解的特殊形式。一段数据将被复制，并将每条复制的数据发送到不同的输出数据流中。

例子：

```go
func Duplicator(in <-chan uint64) (<-chan uint64, <-chan uint64) {
	outA, outB := make(chan uint64), make(chan uint64)
	go func() {
		for {
			x := <-in
			outA <- x
			outB <- x
		}
	}()
	return outA, outB
}
```

### 数据计算/分析

数据计算和分析模块的功能变化很大，每个都非常具体。通常来说，这类模块的工作者函数会将输入数据中的每段数据转换成另一段输出数据。

为了简单的演示目的，这里显示了一个工作者示例，它反转每个传输的`uint64`值的每一位。

```go
func Calculator(input <-chan uint64) (<-chan uint64) {
	output := make(chan uint64)
	go func() {
		for {
			x := <-input
			output <- ^x
		}
	}()
	return output
}
```

### 数据验证/过滤

数据验证/过滤模块将会清除一个流中的某些传输的数据。例如，下面的工作者函数将会清除所有非素数。

```go
import "math/big"

func Filter(input <-chan uint64) (<-chan uint64) {
	output := make(chan uint64)
	go func() {
		bigInt := big.NewInt(0)
		for {
			x := <-input
			bigInt.SetUint64(x)
			if bigInt.ProbablyPrime(1) {
				output <- x
			}
		}
	}()
	return output
}
```

### 数据服务/保存（存储）

通常，数据服务或保存模块是一个数据流系统中的最后或最终的输出模块。这里只提供一个简单的工作者示例，它打印从输入流接收的每个数据。

```go
import "fmt"

func Printer(input <-chan uint64) {
	for {
		x, ok := <-input
		if ok {
			fmt.Println(x)
		} else {
			return
		}
	}
}
```

### 数据流系统组装(Assembling)

现在，让我们使用上面的模块工作者函数来组装几个数据流系统。组装数据流系统只是为了创建一些不同模块的工作者，并为每个工作者指定输入流。

数据流系统示例1（线性管道）：

```go
package main

... // the worker functions declared above.

func main() {
	Printer(
		Filter(
			Calculator(
				RandomGenerator(),
			),
		),
	)
}
```

上述数据流系统如下图所示。

![](https://go101.org/article/res/data-flow-linear.png)

数据流系统示例2（有向非循环图管道）：

```go
package main

... // the worker functions declared above.

func main() {
	filterA := Filter(RandomGenerator())
	filterB := Filter(RandomGenerator())
	filterC := Filter(RandomGenerator())
	filter := Aggregator(filterA, filterB, filterC)
	calculatorA := Calculator(filter)
	calculatorB := Calculator(filter)
	calculator := Aggregator(calculatorA, calculatorB)
	Printer(calculator)
}
```
上述数据流系统如下图所示。

![](https://go101.org/article/res/data-flow-dag.png)

更复杂的数据流拓扑可以是任意的图形。例如，数据流系统可具有多个最终输出。但是具有循环图拓扑的数据流系统在现实中很少使用。

从上面的两个例子中，我们可以发现使用通道构建数据流系统非常容易和直观。

从上一个例子中，我们可以发现，在一个数据系统中，如果我们聚合模块中所有工作者的相应的输出流，然后我们可以使用聚合结果流作为下一个模块中所有工作者的相应输入流。通过这种方式，可以轻松地为指定的模块实现扇入和扇出功能。

以上对数据流系统的解释对如何关闭数据流没有太多考虑。请阅读[这篇文章](https://go101.org/article/channel-closing.html)，了解如何优雅地关闭频道。
