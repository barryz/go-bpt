# Go 中大堆(Large Heap)的危险

在我使用 Go 工作的这些年里，我偶然发现了一些在Go中如果有大量内存数据会造成头疼的原因。

![](https://cdn-images-1.medium.com/max/1600/1*WzVtMGvjKXZE61P4HYKPOQ.jpeg)

最近的一个问题是我们在*Revelin*中使用的批量特征提取过程中遇到的问题。因我们有大量客户端的原因，
我们发现我们的内存使用率越来越高，所以我们不得不在越发昂贵的服务器f中运行它们。
我起初认为这是内存泄漏或者是其他什么原因，所以我决定深入研究它。

我发现Go的*heap profiler*有个神奇的`-base`选项。它允许你使用当前的*heap profile*与之前的产生的基准线的*heap profile*作比较。
所以我仅仅需要两个*heap profile*，这样就能够确保我找到内存泄漏的根本原因。

好吧，通过*heap profile*，它的确显示出了内存的持续增长，但是却没有告诉我哪一个分配会有内存泄漏的可能。
更令人担忧的是，*heap profile* 显示的分配量远远少于操作系统工具*top*和*ps*显示的分配量。

如果是我的博客的长期读者，会发现我很容易使用一些『unsafe』的Go编程实践，
包括通过使用[mmap系统调用](https://medium.com/p/a7a9430d4d22)直接从操作系统分配内存来[避免堆内存分配](https://medium.com/p/e9bdd9324f0)。
所以这可能k导致了一些『堆外』的内存正在泄漏。

接着就是许多哀号和咬牙切齿的声音，然后是对『不安全』代码的审慎处理。
并且添加一些常规的调用，比如`runtime.ReadMemStats`。`ReadMemStats`会展示当前堆内存占用的大小，
这个大小和从操作系统层面看到的内存占用大小一致，这个大小看起来比*heap profile*显示的要高很多。
奇怪的是，*heap profile* 编号仍然令人非常困惑，但至少`ReadMemStats`调用表明堆外的内存分配并不是泄漏的主要原因。
Go的GC收集器应该知道所有有问题的内存。

接着我开始了新技术的学习（通过采取英勇的措施以及[阅读手册](https://golang.org/pkg/runtime/#ReadMemStats)）。
*heap profiles* 总是在每次GC循环结束时才会采集，而`ReadMemStats`则会在每次调用时就显示当前内存状态。
GC结束时使用的内存更少（因为GC会释放它所能释放的所有内存），所以`ReadMemStats`调用显示的内存总是比*heap profile*显示的内存要多的多。
所以并没有堆外内存泄漏，也没有神秘的Go堆内存计算错误，我只是没有完全理解我使用的工具。

那么到底发生了什么？那么有没有可能是GC延迟了，在你向其分配了大量内存的情况下。
通过阅读，我发现[来自Go团队的这篇博客](https://blog.golang.org/ismmkeynote)似乎暗示了GC不太可能会有延迟。
每当GC落后时，它都会调用一个『GC Paser』，并召集正在进行分配工作的Goroutines来进行GC工作。
它需要做的事情与它正在做的分配量成比例，所以你会认为一切都会平衡。但是在极端情况下并非如此。

这就像往常一样，我的用例又是极端的。

为什么我的用例会比较极端？尽管在过去的几次事件中，我发现了由大堆大小引起的问题，并尝试减少它们，或者将分配从堆中移走，或者使GC对它们不感兴趣。
我仍然有大约50GB的长期的堆上分配，这里面充满了各种指针。
这会导致GC的大量工作。我的程序只是尽可能快地消耗CPU - 它不会等待任何外部的输入。因此，一旦GC落后，那么它就不太可能赶得上GC暂停。

如果你有一个大堆，大量已分配的内存，你需要保持整个生命周期过程中(例如大查找表,或某种内存数据库)需要降低GC工作负载，那么你基本上有两个选择，如下所示。

1. 确保你分配的内存不包含指针。那就意味着没有任何切片，map，字符串，没有`time.Time`，以及没有任何指向其他内存的指针。如果一个分配没有指针，那么它将会
被标记，且GC不会去扫描它。

2. 通过直接调用*mmap*系统调用来将内存分配到堆外。那么GC将不会感知到内存分配的存在。这样做有好有坏。坏处是这个内存实际上不能用于引用正常分配的对象，因为
GC可能会认为这些被引用的对象没有被使用而释放掉它们。

如果你不遵守上述两种做法之一，并且在进程的生命周期内分配了50GB的内存，那么每个GC周期都会扫描这50GB的每一个位。
这就需要一些时间，此外，GC会将其内存使用目标标记为100GB，这将远远超过你实际拥有的内存。

所以，GC抱怨了很长时间。让我们看看在某些的场景下，这有多糟糕。为了给你带来乐趣和启发，我构建了一个可复制的场景，
它可以我的16GB内存2015年的MBP上快速显示问题。

首先它分配了一个拥有15亿个8字节指针的数组。这样就使用了12GB的内存。
然后它使用了所有可用的CPU用来运行一堆workers来循环创建1百万个8字节大小的指针（大约有8MB），并且使用一些CPU资源来进行阶乘计算（仅仅为了好玩）。
理想情况下，使用的内存不会增长太多：循环会自旋地进行几次内存分配，然后GC会被触发并释放它们，然后接着循环，产生更多的分配。等等。
巨大的12GB内存分配只会出现在内存的某一边。

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"sync"
)

func main() {
	// A huge allocation to give the GC work to do
	lotsOf := make([]*int, 15e8)
	fmt.Println("Background GC work generated")
	// Force a GC to set a baseline we can see if we set GODEBUG=gctrace=1
	runtime.GC()

	// Use up all the CPU doing work that causes allocations that could be cleaned up by the GC.
	var wg sync.WaitGroup
	numWorkers := runtime.NumCPU()
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			work()
		}()
	}

	wg.Wait()

	// Make sure that this memory isn't optimised away
	runtime.KeepAlive(lotsOf)
}

func work() {
	for {
		work := make([]*int, 1e6)
		if f := factorial(20); f != 2432902008176640000 {
			fmt.Println(f)
			os.Exit(1)
		}
		runtime.KeepAlive(work)
	}
}

func factorial(n int) int {
	if n == 1 {
		return 1
	}
	return n * factorial(n-1)
}
```

但事实并非如此，内存会持续增长，并且会在不超过一分钟内耗尽内存资源，这时候操作系统会终止进程。
这是我们启用GC跟踪输出时看到的内容。

```bash
GODEBUG=gctrace=1 ./gcbacklog

Background GC work generated

gc 1 @0.804s **21%:** 0.012+4528+0.17 ms clock, 0.099+1.4/9054/27147+1.4 ms cpu, 11444->11444->11444 MB, **11445 MB goal**, 8 P (forced)

gc 2 @5.333s 23%: 0.012+6358+0.086 ms clock, 0.099+0/12716/38112+0.68 ms cpu, 11444->11444->11444 MB, 22888 MB goal, 8 P (forced)

gc 3 @11.764s 31%: 20+53853+1.4 ms clock, 167+37787/107690/0+11 ms cpu, 11505->728829->728783 MB, 22888 MB goal, 8 P

gc 4 @65.676s **40%**: 69+10843+0.036 ms clock, 555+61294/21670/23+0.29 ms cpu, 728844->752155->34785 MB, **1457567 MB goal**, 8 P

Killed: 9
```

[这里](https://golang.org/pkg/runtime/#hdr-Environment_Variables)描述了上述输出的格式。一旦分配初始的数组之后，该进程使用了21%的可用CPU用于GC，而在该进程被杀死之前，这个使用率上升到了40%。
GC内存大小的目标也很快就达到22GB（是我们初始分配的2倍），但随着事情失控的发展，它上升到了1.4TB。

现在，如果我们将初始分配从15亿个8字节指针更改为15亿个8字节整数，那么事情就会完全发生改变。
我们仅仅使用了内存，但并没有包含任何指针。GC的目标达到了22GB，但GC更为频繁地启动并使用更少的CPU，
最重要的是内存目标不会持续增长。

```bash
// Note no *! This now contains no pointers
lotsOf := make([]int, 15e8)
fmt.Println("Background GC work generated")
```

下面是第62轮GC后的数据，在程序启动后95秒采集到的。GC的目标仍然是22GB左右，并且实际上并没有CPU
资源应用于GC。所有事情都是稳定的，它可以永远运行下去。

```bash
gc 61 @93.824s 0%: 4.0+4.5+0.075 ms clock, 32+8.9/8.6/4.0+0.60 ms cpu, 22412->22412->11474 MB, 22980 MB goal, 8 P
gc 62 @95.290s 0%: 14+4.0+0.085 ms clock, 115+4.3/0.39/0+0.68 ms cpu, 22382->22382->11451 MB, 22949 MB goal, 8 P
```

所以，我们在这篇文章中学到了什么？如果你使用GO来进行大数据处理，那么你就不能使用长期的大型堆分配的内存，
或者你必须确保它们不能包含任何指针。这也就意味着没有字符串，没有切片，没有`time.Time`(它包含了一个底层指针)，
没有任何隐藏的指针。我希望跟进一些关于我曾经做过的技巧性地博客文章。

（特别感谢Gophers的#perfomance频道中的一些高人在此过程中给予的帮助。如果你有时间，那么这个频道会有一些很棒的东西在等着你。）

---

原文链接：[Further Dangers of Large Heaps in Go](https://syslog.ravelin.com/further-dangers-of-large-heaps-in-go-7a267b57d487).

---
