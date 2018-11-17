> 译自： (The Scheduler Saga)[https://about.sourcegraph.com/go/gophercon-2018-the-scheduler-saga]。

# 调度器的冒险故事

Go的调度器是为Go程序提供动力的幕后的魔法机器。它可以高效的运行Goroutines，开可以协调网络IO以及管理内存。这次演说我们将探索Go调度器机制的内部工作。

## 汇总

本次演讲从基本原理上解释了如何为Go创建一个Goroutine的调度程序。它的观点和Go调度器实际上使用的主要思想一致。Go拥有一个Goroutine调度器，因为Goroutine是一种轻量级用户线程，所以Go不能仅仅只依赖于创建很多的内核线程。基于此，Go调度器完成了一下的工作：

- 使用少量的内核线程（创建内核线程开销是非常昂贵的）
- 支持高并发（Go程序应该有创建大量Goroutine的能力）
- 充分利用并行，扩展到N个CPU核心上

如果可以的话，我们非常推荐观看本次演说的视频（视频链接将在文章末尾附加）。视频里有很多插图可以很清晰的解释下面的想法。

```go
func main() {
	for _, u := range images {
		go process(i) // runs goroutines created
	}

	<-ch // pauses and resumes
}

func process(image) {
	go reportMetrics()
	complicatedAlgorithm(image)
	f, err := os.OpenFile() // blocking system calls, network io, runtime tasks garbage collection.
}
```

上面这个注释过的程序解释了Go调度器会出现在什么位置。在创建Goroutines，等待通道，以及处理系统调用阻塞时，Go调度器都会参与工作。

## 动机

所以，为什么Go需要一个调度器？因为Go使用了Goroutines，它是一个用户级的线程，相较于内核线程非常轻量，（创建、销毁）开销非常小。例如，初始Goroutine的栈大小只有2KB。默认内核线程的栈大小有8KB。Goroutines拥有比内核线程更快的创建、销毁、上下文切换。
所以必须存在一个调度器，因为操作系统的调度器只会在CPU上调度内核线程。

### 想法1

首先是一个bad idea：

- 在一个内核线程上复用所有的Goroutines。这不可能有并发和并行。
- 为每个Goroutine创建-销毁一个内核线程。这与Goroutine的理念不符，内核线程太昂贵了。

既然这样，我们应该如何实现这个想法？我们可以使用一个 ***运行队列(runqueue)***。一个运行队列是一个FIFO队列，它包含了需要被调度到内核线程上运行的Goroutines。当我们创建一个Goroutine时，我们将这个Goroutine推入这个运行队列。当Goroutine等待时，它会被放到运行队列上，另一个Goroutine则可以在内核线程上运行（从运行队列里弹出）。

我们不能仅有一个内核线程，否则我们就没办法实现并发和并行，所以我们需要决定在何时创建内核线程。我们可以在当运行队列有Goroutine，且其他所有内核线程都繁忙时创建一个新的内核线程。这样就完成了并发和并行的特性。

所以，到目前为止，我们已经有了一个减少内核线程创建的方案，但问题是：

- 多个内核线程需要访问运行队列，所以运行队列需要锁。
- 无限制的内核线程，假如主Goroutine创建了10000个Goroutines，那么也会创建10000个内核线程。

### 想法2

限制运行Goroutines（GOMAXPROCS）的线程数。我们仍然不会去限制那些被系统调用阻塞的内核线程，我们仍然希望像以前一样限制访问运行队列的内核线程，然后保持内核线程可以重用，并从运行队列中取出可以运行的Goroutine来运行。

那应该怎么限制呢？太高了 = 太多了竞争。 太少了 = 不能充分利用CPU核心。 所以我们应该根据CPU核心数量来作为这个限制数。

这样就解决了无限制的内核线程问题，但是运行队列的争用如何解决？
