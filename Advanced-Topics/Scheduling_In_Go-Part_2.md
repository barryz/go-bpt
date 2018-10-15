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
