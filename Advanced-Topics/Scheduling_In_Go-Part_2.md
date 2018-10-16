> 译自：[Scheduling In Go - Part II](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html) :book:

# 介绍

在调度系列文章的第一部分中，我解释了操作系统调度器的各个方面，我认为这些方面对于理解Go调度器的语义非常重要。在这篇文章中，我将在语义层面解释Go的调度器的工作原理并关注它的高级行为。Go调度器是一个复杂的系统，一些小的实现细节不重要。主要的是拥有良好的工作和行为方式。这将使你能够做出更好的工程决策。

## 开始一个项目

当你的Go程序开始时，它为主机上标识的每个虚拟核心提供了一个逻辑处理器（P）。如果你的处理器的每个核心有多个硬件线程（[超线程](https://en.wikipedia.org/wiki/Hyper-threading))，每个硬件线程都会被Go程序当作是一个虚拟的核心。为了更好的理解这点，请看下我的MacBook Pro的系统报告。

#### 插图1

![](https://www.ardanlabs.com/images/goinggo/94_figure1.png)

可以看到，我有一个拥有4个物理核心的处理器。这个硬件报告没有公开的是每个物理核心上的硬件线程数。英特尔酷睿i7处理器具有超线程功能，这意味着每个物理内核有2个硬件线程。这将向Go程序报告有8个虚拟核可用于并行执行OS线程。

想要测试它，试试下面的程序：

#### 清单1

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {

    // NumCPU returns the number of logical
    // CPUs usable by the current process.
    fmt.Println(runtime.NumCPU())
}
```

当我在我的机器上运行这个程序时，函数`NumCPU()`返回的结果是8。任何在我机器上运行的Go程序OS都给给其8个P（processor)。

每个P都分配给了一个OS线程("M")。“M”代表了机器。这个线程仍然由操作系统管理，并且操作系统仍需负责将线程放到核心上以便执行，正如[上篇博客](https://github.com/barryz/go101-trans/blob/master/Advanced-Topics/Scheduling_In_Go-Part_1.md)解释的。这意味着如果我在我的机器是运行Go程序，我将拥有8个可用线程来执行我的工作，每个线程都将附加到一个P上。

每个Go程序同事也会给定一个初始的Goroutine（”G“），这是该Go程序的执行路径。一个Goroutine本质上是一个[协程](https://en.wikipedia.org/wiki/Coroutine)，但是在Go里，它就叫Goroutine，所以我们将字母“C”替换成了“G”，然后我们就得到了Goroutine这个单词。你可以认为Goroutine是应用级别的线程并且它们和操作系统的线程在很多方面很类似。正如操作系统需要在核心上进行上下文切换，Goroutines则需要在M上进行上下文切换。

最后一个难题是运行队列。Go调度器中有两个不同的运行队列：全局运行队列（GRQ）和本地运行队列（LRQ）。每个P都有一个LRQ，用于管理在该P的上下文环境中执行的Goroutines。这些Goroutines轮流打开和关闭分配给P的M的上下文。GRQ用于那些还未分配给P的Goroutines。有一个程序会将Goroutines从GRQ移动到LRQ，这点我们将在后面介绍。

插图2集中展示这些组件是如何在一起工作的。

#### 插图2

![](https://www.ardanlabs.com/images/goinggo/94_figure2.png)

## 协作调度器

正如我们在博客的第一部分讨论的，操作系统的调度器是一个抢占式的调度器。从本质上讲，这意味着你无法预测调度器在任何给定的时间将要执行哪个操作。内核在这时候将会作出决策，并且一切都是不确定的。运行在操作系统顶部的应用程序无法控制内核中发生的事情，除非它们利用[原子](https://en.wikipedia.org/wiki/Linearizability)指令和[互斥](https://en.wikipedia.org/wiki/Lock_(computer_science))调用之类的同步原语。

Go调度器是Go运行时的一部分，并且Go运行时已经内置在你的应用程序中。这意味着Go调度器运行在[用户K空间](https://en.wikipedia.org/wiki/User_space)。在内核之上。目前Go的调度器的实现不是抢占式的，实际上是[协作式](https://en.wikipedia.org/wiki/Cooperative_multitasking)的调度器。作为协作调度器意味着**调度器需要明确在代码中的安全点处发生的用户事件以做出相关的调度决策。**

Go协作调度器最大的优点是它看起来，或者说感觉上像是抢占式调度器。你无法预测Go调度器将要执行的操作。这是因为这个协作式的调度程序的决策不是由开发人员控制的，而是在Go运行时由Go自己控制。将Go调度器视为抢占式的非常重要，并且由于调度器是不确定性的，因此这并不是一件容易的事。

## Goroutine状态

和线程一样，Goroutines也有三种高级别的状态。这些状态决定了Go调度器在任何给定的Goroutine中所起的作用。 Goroutine可以处于以下三种状态之一：等待中，可运行和执行中。

__等待中__：这意味着Goroutine已经停止并且在等待某件事情继续下去。（等待的）原因可能是等待操作系统（系统调用）或者同步调用（原子和互斥操作）。这些类型的[延迟](https://en.wikipedia.org/wiki/Latency_(engineering))是造成性能不佳的根本原因。

__可运行__：这意味着Goroutine在M上需要时间，用以执行其指定的指令。如果你有大量的Goroutines需要时间，那么这些Goroutines可能需要等待更长的时间才能获取到时间。此外，随着更多Goroutines争夺时间，任何给定的Goroutine获得的私有时间就缩短了。这种调度器的延迟也是造成性能不佳的原因。

__执行中__：这意味着Goroutine已经在M上，并且已经在执行它的指令。与应用程序相关的工作即将完成。这是每个人都希望看到的。

## 上下文切换

Go调度器需要明确定义的用户空间的事件，这些事件发生在代码中的安全点以进行上下文切换。这些事件和安全点在函数调用中表现出来。函数调用对于Go调度器的状态来说至关重要。今天（Go1.11或更低版本），如果你运行一个没有任何函数调用的紧密循环 __(tight loops: a loop which contains few instructions and iterates many times.)__，你将因为调度器的垃圾回收导致延迟。函数调用在合理的时间范围内发生是至关重要的。

注意：有一个Go1.12的[提议](https://github.com/golang/go/issues/24543)已经被接受: 那就是，在Go调度器中应用非协作抢占技术，以允许抢占紧密循环。

在Go程序中，发生了以下4类事件将会使Go调度器做出调度决策。这并不意味着它总会发生在其中一个事件上。这意味着调度器获得了一个机会。

- 使用关键字`go`
- 垃圾回收
- 系统调用
- 同步和编排


### 使用关键字`go`

关键字`go`用来创建Goroutines。一旦创建了一个新的Goroutine，它将会给调度器一个机会用来做出调度决策。

### 垃圾回收
因为GC使用它们自己的Goroutne运行，这些Goroutines需要在M上获得时间来运行。这就导致了GC创建了大量的调度上的混乱。但是，调度器非常聪明地了解Goroutine正在做什么，调度器因此会做出明智的决策。一个明智的决策是在上下文切换一个Goroutine时，它希望在GC期间接触那些它们不接触堆的堆。当GC运行时，将会做出大量的调度决策。

### 系统调用

如果一个G产生了一个系统调用，那么该G将会阻塞M，有时，调度器能够将G从M上切换出去，并将新的G切换到相同的M. 但是，有时也需要一个新的M来继续执行在P中排队的G(s)。我们将在下节解释这步是如何实现的。

### 同步和编排

如果一个原子，互斥，或者一个通道操作导致了一个G阻塞，那么调度器将会切换一个新的G来运行。一旦这个G能够再次运行，它可以重新排队并最终在M上进行上下文切换。

## 异步系统调用

如果你运行的操作系统有异步处理系统调用的能力，称为[network poller](https://golang.org/src/runtime/netpoll.go)的东西可用于更有效地处理系统调用。这种技术已经通过kqueue(MacOS)， epoll(Linux)或iocp(Windows)实现了。

基于网络的系统调用可以异步处理，在如今我们使用的各种操作系统上。这也是network poller名字的由来，因为它刚开始就是处理网络操作的。通过使用network poller进行网络系统调用，调度器可以防止G在进行系统调用时阻塞M。这有助于保持M的可用性，用以继续执行P的LRQ中的其他G，而不用创建新的M。这有助于操作系统较少调度负载。

查看其工作原理的最佳方法是运行示例，在如今我们使用的各种操作系统上。这也是network poller名字的由来，因为它刚开始就是处理网络操作的。通过使用network poller进行网络系统调用，调度器可以防止G在进行系统调用时阻塞M。这有助于保持M的可用性，用以继续执行P的LRQ中的其他G，而不用创建新的M。这有助于操作系统较少调度负载。

查看其工作原理的最佳方法是运行示例。

#### 插图3

![](https://www.ardanlabs.com/images/goinggo/94_figure3.png)

插图3向我们展示最基本的调度图示。G1正在M上执行，另外LRQ上还有三个G等待在M上获得执行时间。Network poller没有做任何事情，所以是空闲的。

#### 插图4

![](https://www.ardanlabs.com/images/goinggo/94_figure4.png)

在插图4中，G1想要进行一个网络系统调用，所以G1被移动到了network poller处，并且处理该异步网络系统调用。一旦G1移动到了network poller，M就可以执行在LRQ中的一个可用的G。在这个案例中，G2被切换到M上。

#### 插图5

![](https://www.ardanlabs.com/images/goinggo/94_figure5.png)

在插图5中，通过network poller完成了网络异步系统调用之后，G1将会被重新移动到P的LRQ中。一个G1能够被切换回M，该G负责的相关代码就可以被再次执行。这里最大的好处就是，执行一个网络系统调用，不需要额外的M。Network poller有一个操作系统的线程并且它可以高效的处理时间循环。

## 同步系统调用

当一个G想要进行一个无法异步化的系统调用会发生什么？在这个案例中，不能再使用网络轮询器，并且G产生的系统调用将会导致M阻塞。这很不幸，但是没有任何办法去避免这样的事情发生。不能被异步化的系统调用的一个典型的例子就是基于文件的系统调用。如果你使用CGO，可能还有其他情况，调用C函数也可能会阻塞M。

_注意：Windows操作系统确实能够异步化基于文件的系统调用。从技术上讲，在Windows上运行程序时，是可以使用网络轮询器的。_

让我们来看一下同步系统调用（如文件I / O）会导致M阻塞的情况。

#### 插图6：

![](https://www.ardanlabs.com/images/goinggo/94_figure6.png)

图6再次向我们展示了基本的调度图，但是这次G1将进行同步系统调用以阻止M1。

#### 插图7：

![](https://www.ardanlabs.com/images/goinggo/94_figure7.png)

在插图7中，调度器能够识别到G1已经导致了M1的阻塞。此时，调度器将M1与P分离，同时G1仍然附加在M1上。然后调度器将使用一个新的M2来服务P。此时，M2可以切换LRQ中的G2来运行。如果在交换之前已经有存在的M，那么此时交换一个M比创建一个M速度更快。

#### 插图8：

![](https://www.ardanlabs.com/images/goinggo/94_figure8.png)

在插图8中，由G1产生的系统调用已经完成。此时，G1可以重新被移动到LRQ中，并且再次由P服务。M1则放置在旁边备用，以便再次放生这样的（基于文件IO的同步系统调用等等）情况发生。
