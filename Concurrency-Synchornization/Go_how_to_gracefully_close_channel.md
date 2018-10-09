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

这种解决方案显然打破了 ___通道关闭原则___ 。

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
