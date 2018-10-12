> 译自：[Scheduling  In Go](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html) :book:

# Go中的调度 - 第一部分

## 介绍

Go调度器的设计和行为是你的多线程Go程序更加高效。这要归功于Go调度器对操作系统（OS）调度程序的mechanical sympathies。但是，如果你的多线程Go程序的设计和行为上与调度器程序的工作方式上没有mechanical sympathies，那么这一切都不那么重要。了解OS和Go调度器对于如何正确设计一个多线程的软件来说非常重要。

本系列文章将重点介绍调度器在更高级别上的运行机制和语义。我将提供足够多的详细信息，以便你可以看到一些内部的工作原理，方便做出更好的工程上的决策。尽管你需要为多线程程序做出更多的工程决策，机制和语义构成了你所需的基础知识的关键部分。

## 操作系统调度器

操作系统的调度器是软件行业中非常复杂的一环。它们必须考虑到它们所运行的硬件的布局和配置。这包括但不限于存在多个处理器和核心，[CPU缓冲和NUMA](http://frankdenneman.nl/2016/07/06/introduction-2016-numa-deep-dive-series)。没有这些知识，调度器就不可能尽可能高效。最棒的是，你仍然可以开发一个良好的心理模型，来了解操作系统调度器的工作原理，而无需深入研究这些主题。

你的程序只是一系列需要依次执行的机器指令。为了实现这点，操作系统使用了线程的概念。线程的工作是辨别并顺序执行它所分配的指令集。连续执行到该线程没有指令需要执行为止。这就是为什么我将线程称为”一个执行路径”的原因。

每个你运行的程序都会创建一个进程并且每个进程会有一个给定的初始化线程。线程拥有穿件更多线程的能力。这些线程都彼此独立地运行，并且调度的决策实在线程级别进行的。而不是在进程级别。线程可以并发运行（每个线程都面向一个核心），或者并行运行（每个线程同一时间在不同核心上）。线程而且保持了自身的状态，以允许安全，本地和独立执行它们的指令。

如果存在可以执行的线程，那么OS的调度器负责确保核心不在空闲状态。它还必须制造一个假象(illusion)，即所有可以执行的线程是同时执行的。在创建这种假象的过程中，调度器需要运行优先级较高的线程。但是，低优先级的线程并不会被饿死（缺乏执行时间）。调度器还需要通过快速、明智地决策尽可能地最小化调度的延迟。

为了实现这一目标，使用了很多的算法。但幸运的是，该行业有着数十年的工作和经验可以利用。为了更好地理解所有的这些，我们最好描述和定义一些重要的概念。

## 执行指令

[程序计数器(PC)](https://en.wikipedia.org/wiki/Program_counter)，有时候也被称作指令指针(IP)，被用来允许线程可以追踪下一条要执行的指令。在大多数的核心中，PC指向下一条执行而不是当前的指令。

#### 图1

![](https://www.ardanlabs.com/images/goinggo/92_figure1.jpeg)

[https://www.slideshare.net/JohnCutajar/assembly-language-8086-intermediate](https://www.slideshare.net/JohnCutajar/assembly-language-8086-intermediate)

如果你曾经看过Go程序的堆栈追踪，你可能已经注意到每行末尾的这些小十六进制数字。在列表1中查找`+0x39`和`+0x72`。

#### 列表1

```bash
goroutine 1 [running]:
   main.example(0xc000042748, 0x2, 0x4, 0x106abae, 0x5, 0xa)
       stack_trace/example1/example1.go:13 +0x39                 <- LOOK HERE
   main.main()
       stack_trace/example1/example1.go:8 +0x72                  <- LOOK HERE
```

这些数字表示从相应函数的顶部到当前偏移的PC值。`+0x39`PC偏移量表示了在`example`函数内线程将要执行的下一条指令，如果该程序没有发生恐慌的情况下。`+0x72`PC偏移量则表示了在`main`函数内线程将要执行的下一条指令，如果控制权回到`main`函数的情况下。更为重要的，该指针之前的指令会告诉你正在执行的指令。

请观察列表2的程序，该程序导致了列表1中的堆栈追踪。

```bash
https://github.com/ardanlabs/gotraining/blob/master/topics/go/profiling/stack_trace/example1/example1.go

07 func main() {
08     example(make([]string, 2, 4), "hello", 10)
09 }

12 func example(slice []string, str string, i int) {
13    panic("Want stack trace")
14 }
```
十六进制数`+0x39`表示`example`函数内的指令的PC偏移量，该指令比函数的起始指令低了57（基数10）个字节。在下面的列表3中，你可以从`example`函数的二进制中查看`objdump`信息。找到第12条指令，在底部显示的。表明了该指令上方的代码行是对`panic`的调用。

#### 列表3

```bash
$ go tool objdump -S -s "main.example" ./example1
TEXT main.example(SB) stack_trace/example1/example1.go
func example(slice []string, str string, i int) {
  0x104dfa0		65488b0c2530000000	MOVQ GS:0x30, CX
  0x104dfa9		483b6110		CMPQ 0x10(CX), SP
  0x104dfad		762c			JBE 0x104dfdb
  0x104dfaf		4883ec18		SUBQ $0x18, SP
  0x104dfb3		48896c2410		MOVQ BP, 0x10(SP)
  0x104dfb8		488d6c2410		LEAQ 0x10(SP), BP
	panic("Want stack trace")
  0x104dfbd		488d059ca20000	LEAQ runtime.types+41504(SB), AX
  0x104dfc4		48890424		MOVQ AX, 0(SP)
  0x104dfc8		488d05a1870200	LEAQ main.statictmp_0(SB), AX
  0x104dfcf		4889442408		MOVQ AX, 0x8(SP)
  0x104dfd4		e8c735fdff		CALL runtime.gopanic(SB)
  0x104dfd9		0f0b			UD2              <--- LOOK HERE PC(+0x39)
```

记住，PC就是下一条指令，不是当前的指令。列表3是基于amd64的指令的一个非常好的例子，该Go程序的线程负责顺序执行。
