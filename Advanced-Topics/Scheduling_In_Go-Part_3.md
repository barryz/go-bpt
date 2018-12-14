# GO中的调度： 第三部分 - 并发

## 前奏

这篇文章是三部曲系列文章中的第三篇，这个系列的文章将会对Go中调度器背后的机制和语义做深入的了解。本文主要关注并发的部分。

Go调度器系列文章：

- [Go中的调度器：第一部分 - 操作系统调度器](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)

- [Go中的调度器：第二部分 - Go调度器](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)

- [Go中的调度器：第三部分 - 并发](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part3.html)

## 介绍

每当我解决一个问题时，尤其这是一个新问题时，我刚开始并不会考虑并发是否合适。我会先寻找一些解决方案来确保它正常工作。然后是代码的可读性，在技术审阅之后，我才会开始提出并发性是否合理及实用的问题。
对于并发性，有时候它是一个好东西，有时候却未必是。

在本系列文章的[第一部分](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)，我解释了操作系统调度器的机制和语义，如果你有计划编写多线程代码，我认为这方面知识很重要。在本系列文章的[第二部分](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)，我解释了Go中的调度器背后的机制和语义，我相信这对于理解如何在Go中编写并发代码是至关重要的。在本文中，我会将操作系统和Go调度器的机制和语义结合在一起，以便更深入地理解什么是并发，什么不是并发。

本文的目标是：

- 提供语义上的指导，来考虑当前的工作负载是否适合使用并发特性。

- 向你展示如何改变不同工作负载下的语义，来更改你想要做出的工程上的决策。

## 什么是并发

并发意味着“无序”执行。拿到一组本来会有序执行的指令，然后使用无序的执行方法，最终却能得到相同的结果。所以摆在你面前的问题非常明显：无序执行会增加一些”价值“，当我在说”价值“时，我实际的意思是为了复杂度的成本增加足够的性能上的收益。这取决于你具体的问题，有些时候无序执行可能无法实现或者根本没有价值。

再者，理解[并发并不等同于并行](https://blog.golang.org/concurrency-is-not-parallelism)非常的重要。并行意味着同时执行两个或多个指令。并行的概念和并发是完全不同的。只有当你拥有至少两个操作系统（OS)和硬件线程，并且至少需要两个Goroutines时才能实现真正的并行，每个Goroutine在各自的操作系统/硬件线程上独立地执行指令。

___配图1：并发和并行___

![pic1](https://www.ardanlabs.com/images/goinggo/96_figure1.png)

在配图1中，你会看到两个逻辑处理器（P）的关系图，每个逻辑处理器的独立操作系统线程（M)附着到机器上的独立硬件线程（核心）上。同时，你也可以看到有两个Goroutines（G1和G2)在并行地运行，同一时间在它们各自的操作系统/硬件线程上执行它们的指令。在每个逻辑处理器内部，有三个Goroutines轮流共享各自的操作系统线程。所有这些Goroutines都是并行运行的，它们的指令执行时是无序的，且会在操作系统线程上共享时间片。

这就是问题所在，有时候在没有并行的基础上进行并发可能会导致应用程序的吞吐量下降。同样有趣的是，有时候我们在并行的基础上进行使用并发也未必会带来性能上的提升。

## 工作负载

如何才能知道”无序执行“可能会产生收益？了解你所处理的工作负载是一个比较好的起始点。在考虑并发时，有两种重要类型的工作负载需要注意：

- ***CPU密集型***：这种工作负载永远不会导致Goroutines进入等待的状态，这是一种持续不断的计算工作。例如，计算圆周率π的第n位就是一个CPU密集型操作。

- ***IO密集型***：这种工作负载会导致Goroutines自然地进入等待的状态。这类的工作负载一般包括通过网络去访问某些资源，或者向操作系统进行系统调用，或者等待某个事件的发生。如果一个Goroutine需要读取一个文件，那么这就是IO密集型的工作。我个人倾向于将同步事件（互斥锁，原子操作）归为此类，因为这些操作也会导致Goroutines进入等待状态（阻塞）。

如果是CPU密集型的工作，你可能需要并行来提高并发性能。单线程处理多个Goroutines不够高效，因为Goroutines作为工作负载的一部分并不会切换至等待状态。如果Goroutines的数量多于操作系统线程的数量，也可能会减缓代码的执行速度，因为在操作系统线程上切换Goroutines需要考虑延迟成本（切换所花费的时间）。上下文切换会导致一个”STW“事件，因为在切换期间，代码并没有被真正的执行到。

如果是IO密集型的工作，你可能就不需要并行来提高并发性能。单个操作系统线程可以非常高效地处理多个Goroutines，因为Goroutines会因为工作性质而自然地进行等待-执行这种状态的切换。Goroutines的数量多于操作系统线程数时，会增加相关作业的执行速度，因为上下文切换导致的延迟开销并不会产生”STW“事件。你的作业会自然地终止，这允许不同的Goroutine高效地利用同一个操作系统线程，从而不会让操作系统线程处于空闲状态。

那么该如何计算每个操作系统线程上运行多少个Goroutines才能达到最佳吞吐量？如果Goroutines太少，会导致更多的线程空闲时间。Goroutines太多，又会因为上下文切换而产生更高的延迟成本。这个问题超出了本文的范畴，实际上，这个问题应该由你自己思考。

现在来看，最重要的事情是通过检查一些代码来巩固你评估工作负载并适当地使用并发性的能力。

## 数字累加

我们不会使用复杂的用例来解释和理解这些语义。首先，我们看下面代码片段中的`add`函数，它执行的是两个整型数字的求和操作。

___清单1___

[source code](https://play.golang.org/p/r9LdqUsEzEz)

```go
36 func add(numbers []int) int {
37     var v int
38     for _, n := range numbers {
39         v += n
40     }
41     return v
42 }
```

清单1的36行，声明了一个名为`add`函数用来求一个元素为`int`切片的所有元素之和。在37行，它声明了一个名为`v`的变量来表示求和的结果`sum`。然后在38行，该函数开始迭代切片，并依次累加各个元素。最后在41行，该函数返回求和结果。

问题：`add`函数是一个适合无序执行的作业类型吗？我相信答案是肯定的。整型的切片集合可以被划分为多个小集合并且可以被并发地处理。一旦所有的集合都求和完成，就可以将各个集合的结果累加至`sum`，这样和顺序求和的结果是一致的。

但是，这又会引入一个其他的问题。到底需要划分多少个小集合进行并发处理才能达到最佳的系统吞吐量？想要回答这个问题的前提是你必须知道在运行的工作负载`add`具体是什么类型。事实上，`add`函数是一个CPU密集型的作业类型，因为它运行的是纯数学运算，并且不会有其他的问题导致其Goroutine会进入等待状态。这也就意味着为每个操作系统线程分配一个Goroutine即可以达到最佳的吞吐量。

下面清单2是我编写的并发版本的`add`

_注意：你可以使用多种方式来编写并发版本的`add`函数。不用被我写的特定的实现方式所束缚。如果你有更加好的实现方式，并且愿意分享，那真是极好的。_

___清单2___

[source code](https://play.golang.org/p/r9LdqUsEzEz)

```go
func addConcurrent(goroutines int, numbers []int) int {
45     var v int64
46     totalNumbers := len(numbers)
47     lastGoroutine := goroutines - 1
48     stride := totalNumbers / goroutines
49
50     var wg sync.WaitGroup
51     wg.Add(goroutines)
52
53     for g := 0; g < goroutines; g++ {
54         go func(g int) {
55             start := g * stride
56             end := start + stride
57             if g == lastGoroutine {
58                 end = totalNumbers
59             }
60
61             var lv int
62             for _, n := range numbers[start:end] {
63                 lv += n
64             }
65
66             atomic.AddInt64(&v, int64(lv))
67             wg.Done()
68         }(g)
69     }
70
71     wg.Wait()
72
73     return int(v)
74 }
```

在清单2中出现的`addConcurrent`函数以之前`add`函数的并发版本。并发版本使用了26行代码，而之前简单的版本只使用了5行代码。多了很多代码，所以我只关注需要重点理解的代码。

___48行___：每个Goroutine只需要计算一个唯一的，且更加小的列表。小列表的大小是根据总列表元素个数除以Goroutines数量来计算的。

___53行___：创建了一堆Goroutines用来执行累加工作。

___57-59行___：最后一个Goroutine将累加包含了剩余数字的列表，这些数字可能比其它Goroutines中的数字更大。

___66行___：每个小列表的求和运算结果存到最终的sum中。

很明显，并发的版本比顺序处理的版本复杂了很多，但这种复杂性有价值吗？回答此问题最好的方式就是创建一个基准测试。在基准测试里，我使用了1000万个数字集合，并且关闭了垃圾回收器。同时，该基准测试中也包含了一个顺序求值的版本。

___清单3___

```go
func BenchmarkSequential(b *testing.B) {
    for i := 0; i < b.N; i++ {
        add(numbers)
    }
}

func BenchmarkConcurrent(b *testing.B) {
    for i := 0; i < b.N; i++ {
        addConcurrent(runtime.NumCPU(), numbers)
    }
}
```

清单3展示了基准测试的内容。下面是所有Goroutines共用单个操作系统线程的结果。顺序求值版本使用了一个Goroutine，而并发版本使用了和CPU核心数`runtime.NumCPU`一样多的Goroutines（8个）。在这个案例中，并发版本并没有利用到并行性。

___清单4___

```bash
10 Million Numbers using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITHOUT Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 1 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential          1000       5720764 ns/op : ~10% Faster
BenchmarkConcurrent          1000       6387344 ns/op
BenchmarkSequentialAgain     1000       5614666 ns/op : ~13% Faster
BenchmarkConcurrentAgain     1000       6482612 ns/op
```

_注意：在你本地机器上运行基准测试是很复杂的。有很多意外的因素会导致你的测试结果不精确。你需要确保你的机器尽可能地处于空闲状态且多运行几次基准测试。你希望在测试结果中看到一致性，通过测试工具运行两次基准测试，将会使得测试结果达到最一致的状态_。

清单4中的基准测试显示了顺序求值的版本比并发版本在使用单个操作系统线程时每次操作快了约10%-13%。这正是我所期望的，因为并发版本在单个操作系统线程上因为上下文切换和Goroutines管理而造成了一些额外的开销。

下面的清单是当每个Goroutine都有一个独立的操作系统线程为之运行时的结果。顺序求值版本使用了单个Goroutine，并发版本使用了8个（和`runtime.NumCPU`一致）Goroutines。在这种情况下，并发版本开始利用并行特性。

___清单5___

```bash
10 Million Numbers using 8 goroutines with 8 cores
2.9 GHz Intel 4 Core i7
Concurrency WITH Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 8 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential-8                1000      5910799 ns/op
BenchmarkConcurrent-8                2000      3362643 ns/op : ~43% Faster
BenchmarkSequentialAgain-8           1000      5933444 ns/op
BenchmarkConcurrentAgain-8           2000      3477253 ns/op : ~41% Faster
```

清单5中的基准测试显示了并发版本在每个Goroutine使用独立的操作系统线程时，每次操作的耗时比顺序求值版本快了约%41-%43。这也正是我所期望的，因为所有的Goroutines都在并行运行，8个Goroutines同时进行它们各自的工作。

## 排序

理解并非所有的CPU密集型的工作都适合并发是非常重要的。当分离工作或者合并结果的操作都非常昂贵时，这基本上是正确的。冒泡排序是解释这类问题的一个比较好的例子。下面的代码是用Go实现的冒泡排序算法。

___清单6___

[source code](https://play.golang.org/p/S0Us1wYBqG6)

```go
01 package main
02
03 import "fmt"
04
05 func bubbleSort(numbers []int) {
06     n := len(numbers)
07     for i := 0; i < n; i++ {
08         if !sweep(numbers, i) {
09             return
10         }
11     }
12 }
13
14 func sweep(numbers []int, currentPass int) bool {
15     var idx int
16     idxNext := idx + 1
17     n := len(numbers)
18     var swap bool
19
20     for idxNext < (n - currentPass) {
21         a := numbers[idx]
22         b := numbers[idxNext]
23         if a > b {
24             numbers[idx] = b
25             numbers[idxNext] = a
26             swap = true
27         }
28         idx++
29         idxNext = idx + 1
30     }
31     return swap
32 }
33
34 func main() {
35     org := []int{1, 3, 2, 4, 8, 6, 7, 2, 3, 0}
36     fmt.Println(org)
37
38     bubbleSort(org)
39     fmt.Println(org)
40 }
```

清单6是一个用Go编写的冒泡排序算法。这种排序算法会扫描每次迭代时需要交换值的整数集合。依赖于列表的顺序，在对所有的元素进行排序之前，可能会多次遍历集合。

问题：
