> 译自：[How To Gracefully Close Channles](https://go101.org/article/channel-closing.html)

# 如何优雅的关闭通道

之前，我写过一篇文章来解释[Go中的通道的一些规则](https://go101.org/article/channel.html)。该篇文章在[reddit](https://www.reddit.com/r/golang/comments/5k489v/the_full_list_of_channel_rules_in_golang/)和[HN](https://news.ycombinator.com/item?id=13252416)上获得了很多的赞。

我收集了一些关于Go通道的设计和规则的相关批评：

1. 在不修改通道的状态情况下没有简单和通用的方法来检查通道是否关闭。
2. 关闭一个已经关闭的通道会造成运行时恐慌，所以关闭一个不知道是否已经关闭的通道是十分危险的。
3. 向一个已经关闭的通道发送值会造成运行时恐慌，所以在向一个不知道是否已经关闭的通道发送值是十分危险的。

这些批评看起来很合理（实际上不是）。是的，确实没有一个内建的函数来检查一个通道是否已经关闭。

如果你确定没有任何值发送到通道，确实有一种简单的方法可以检查通道是否已经关闭。（这个方法将会在本文的多个示例中反复被使用）

```go
package main

import "fmt"

type T int

func IsClosed(ch <-chan T) bool {
	select {
	case <-ch:
		return true
	default:
	}

	return false
}

func main() {
	c := make(chan T)
	fmt.Println(IsClosed(c)) // false
	close(c)
	fmt.Println(IsClosed(c)) // true
}
```

正如上面提到的，这不是一种通用的方式用来检查通道是否已经关闭。

但是，即使有一个简单的用来关闭通道的函数`closed(chan T) bool`，那它的用处也十分有限，就像内建函数`len`用来检查存储在可缓冲通道的当前值的个数。原因就是当这样的函数返回结果时，被检查的通道的状态可能已经发生改变。如果调用`closed(ch)`返回`true`，则可以停止向该通道`ch`发送值，但如果调用`closed(ch)`返回`false`，则关闭通道或继续向通道发送值是不安全的。

## 通道关闭的原则

使用Go通道时一个通用的原则就是 ___不要在接收端关闭通道，不要在通道拥有多个并发发送者的情况下关闭通道___。换言之，如果发送者是该通道的唯一发送者，那么你应该只关闭该发送者goroutine中的通道。

（下面，我们将上述的原则称之为 ___通道关闭原则___。

当然，这不是关闭通道的通用原则。通用原则是不要发送值到（或关闭）已关闭的通道中。如果任意一个goroutine能保证没有其他goroutines将再发送值到（或关闭）一个非已关闭的非空通道，那么该goroutine就可以安全地关闭该通道。但是，由接收者或通道的众多发送者之一做出这样的保证通常要付出很多努力，并且经常会导致代码更加复杂。相反，保持上述 ___通道关闭原则___ 则很容易理解。

## 粗暴关闭通道的解决办法

如果你确实需要从接收端或从众多发送者之一上关闭一个通道，那么你可以使用`recover`机制来防止可能会产生的恐慌从而导致程序崩溃。下面是一个例子（假设通道的元素类型是T）。

```go
func SafeClose(ch chan T) (justClosed bool) {
	defer func() {
		if recover() != nil {
			// The return result can be altered
			// in a defer function call.
			justClosed = false
		}
	}()

	// assume ch != nil here.
	close(ch)   // panic if ch is closed
	return true // <=> justClosed = true; return

```

这种情况显然打破了 ___通道关闭原则___ 。

同样的办法也适用于向一个潜在已关闭的通道发送值：

```go
func SafeSend(ch chan T, value T) (closed bool) {
	defer func() {
		if recover() != nil {
			closed = true
		}
	}()

	ch <- value  // panic if ch is closed
	return false // <=> closed = false; return
}
```

## 优雅关闭通道的解决方案

上述函数`SafeSend`的一个缺点是它的调用不用作为一个发送操作在`select`块中的`case`后面使用。上述`SendSafe`和`SafeClose`函数的另一个缺点是，很多人，包括我自己在内，会觉得上面的解决方案中使用`panic/recover`和`sync`包不够优雅。下面，我们将介绍一些不使用`panic/recover`或`sync`标准库的一些比较简洁的方案，对于各种各样的情形来说。

_（在下面的例子中，`sync.WaitGroup`用来保证例子程序能够顺利结束。在实践中并不会真正地使用它）。_

### 1. M个接收者，一个发送者，发送者通过关闭数据通道说“不再发送消息”

这是个最简单的情形，仅仅需要发送者在其不需要发送数据时关闭数据通道即可。

```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	// ...
	const MaxRandomNumber = 100000
	const NumReceivers = 100

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int, 100)
解决
	// the sender
	go func() {
		for {
			if value := rand.Intn(MaxRandomNumber); value == 0 {
				// The only sender can close the channel safely.
				close(dataCh)
				return
			} else {
				dataCh <- value
			}
		}
	}()

	// receivers
	for i := 0; i < NumReceivers; i++ {
		go func() {
			defer wgReceivers.Done()

			// Receive values until dataCh is closed and
			// the value buffer queue of dataCh is empty.
			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	wgReceivers.Wait()
}
```

### 2. 一个接收者，N个发送者，接收者通过关闭一个额外的信号通道说”请停止发送更多消息”

这个比上面的例子稍微复杂一点。我们不能让接收者关闭数据通道，因为这样做会打破 ___通道关闭原则___。但是我们可以让接收者关闭一个额外的信号通道来通知发送者停止发送数据。

```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	// ...
	const MaxRandomNumber = 100000
	const NumSenders = 1000

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(1)

	// ...
	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})
		// stopCh is an additional signal channel.
		// Its sender is the receiver of channel dataCh.
		// Its reveivers are the senders of channel dataCh.

	// senders
	for i := 0; i < NumSenders; i++ {
		go func() {
			for {
				// The first select is to try to exit the goroutine
				// as early as possible. In fact, it is not essential
				// for this specified example, so it can be omitted.
				select {
				case <- stopCh:
					return
				default:
				}

				// Even if stopCh is closed, the first branch in the
				// second select may be still not selected for some
				// loops if the send to dataCh is also unblocked.
				// But this is acceptable for this example, so the
				// first select block above can be omitted.
				select {
				case <- stopCh:
					return
				case dataCh <- rand.Intn(MaxRandomNumber):
				}
			}
		}()
	}

	// the receiver
	go func() {
		defer wgReceivers.Done()

		for value := range dataCh {
			if value == MaxRandomNumber-1 {
				// The receiver of the dataCh channel is
				// also the sender of the stopCh channel.
				// It is safe to close the stop channel here.
				close(stopCh)
				return
			}

			log.Println(value)
		}
	}()

	// ...
	wgReceivers.Wait()
}
```

就如注释中提到的，对于这个额外的信号通道来说，它的发送者就是数据通道的接收者。这个额外的信号通道只被其唯一的发送者关闭，所以遵循了 ___通道关闭原则___。

在这个例子里，通道`dataCh`永远不会被关闭。是的，通道不是必须要关闭。一个通道如果没有goroutine引用它，那么它最终会被垃圾回收掉，无论它是否被关闭。因此，关闭通道的优雅并不一定指关一定要关闭通道。

### 3. M个接收者，N个发送者，它们中随机一个通过通知主持人（moderator）关闭一个额外的信号通道说：”让我们结束这个游戏吧”

这是一个非常复杂的情形。我们不能让任何一个接收者或发送者来关闭通道。并且，我们也不能让任何一个接收者通过通知一个额外的信号通道来通知其他所有的发送者、接收者退出这个游戏。因为无论怎么做都会打破 ___通道关闭原则___。然而，我们可以引入一个主持人角色来关闭一个额外的信号通道。此示例中的一个技巧是如何通知主持人角色来关闭其他信号通道：

```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
	"strconv"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	// ...
	const MaxRandomNumber = 100000
	const NumReceivers = 10
	const NumSenders = 1000

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})
		// stopCh is an additional signal channel.
		// Its sender is the moderator goroutine shown below.
		// Its reveivers are all senders and receivers of dataCh.
	toStop := make(chan string, 1)
		// The channel toStop is used to notify the moderator
		// to close the additional signal channel (stopCh).
		// Its senders are any senders and receivers of dataCh.
		// Its reveiver is the moderator goroutine shown below.

	var stoppedBy string

	// moderator
	go func() {
		stoppedBy = <-toStop
		close(stopCh)
	}()

	// senders
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(MaxRandomNumber)
				if value == 0 {
					// Here, a trick is used to notify the moderator
					// to close the additional signal channel.
					select {
					case toStop <- "sender#" + id:
					default:
					}
					return
				}

				// The first select here is to try to exit the goroutine
				// as early as possible. This select blocks with one
				// receive operation case and one default branches will
				// be specially optimized as a try-receive operation by
				// the standard Go compiler.
				select {
				case <- stopCh:
					return
				default:
				}

				// Even if stopCh is closed, the first branch in the
				// second select may be still not selected for some
				// loops (and for ever in theory) if the send to
				// dataCh is also non-blocking.
				// This is why the first select block above is needed.
				select {
				case <- stopCh:
					return
				case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// receivers
	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			defer wgReceivers.Done()

			for {
				// Same as the sender goroutine, the first select here
				// is to try to exit the goroutine as early as possible.
				// This select blocks with one send operation case and
				// one default branches will be specially optimized as
				// a try-send operation by the standard Go compiler.
				select {
				case <- stopCh:
					return
				default:
				}

				// Even if stopCh is closed, the first branch in the
				// second select may be still not selected for some
				// loops (and for ever in theory) if the receive from
				// dataCh is also non-blocking.
				// This is why the first select block is needed.
				select {
				case <- stopCh:
					return
				case value := <-dataCh:
					if value == MaxRandomNumber-1 {
						// The same trick is used to notify
						// the moderator to close the
						// additional signal channel.
						select {
						case toStop <- "receiver#" + id:
						default:
						}
						return
					}

					log.Println(value)
				}
			}
		}(strconv.Itoa(i))
	}

	// ...
	wgReceivers.Wait()
	log.Println("stopped by", stoppedBy)
}
```

这个例子中，我们仍然保持了 ___通道关闭原则___。

需要注意的是通道`toStop`的容量是1。这是为了避免主持人所在的goroutine准备好从`toStop`通道接收信号之前可以发送第一个通知。

我们也可以将`toStop`通道的容量设置成所有发送者和接收者的总和，那么我们就不再需要尝试发送的`select`块来通知这个主持人了。

```go
...
toStop := make(chan string, NumReceivers + NumSenders)
...
			value := rand.Intn(MaxRandomNumber)
			if value == 0 {
				toStop <- "sender#" + id
				return
			}
...
				if value == MaxRandomNumber-1 {
					toStop <- "receiver#" + id
					return
				}
...
```

### 4. 还有更多的情况？

基于上面三种情况可以有很多种情况的变体。例如，基于最复杂的变体的一个变体可能要求接收者从缓冲的数据通道中读取所有剩余的值。这很容易处理，本文不会介绍它。

虽然以上三种情况不能涵盖所有Go通道的使用情况，但它们都是基本情况。实践中的大多数情况可以分为三种情况。

## 结论

没有任何情况会迫使您违反 ___通道关闭原则___。如果你遇到这种情况，请重新考虑您的设计并重构你的代码。

使用Go通道编程有时候就想制作艺术品一样。
