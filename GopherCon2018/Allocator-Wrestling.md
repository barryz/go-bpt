> 译自[GopherCon 2018 - Allocator Wrestling](https://about.sourcegraph.com/go/gophercon-2018-allocator-wrestling)。
By Beyang Liu for the GopherCon Liveblog on August 28, 2018

---

# 总结

一个Go的内存分配器和垃圾回收器的旋风之旅，并提供相关优化工具和技巧。

- 内存分配器和垃圾回收器设计十分巧妙！
- 单个内存分配速度很快但不是免费的。
- 垃圾回收器能够暂停某些特定的goroutines，尽管**STW**暂停非常短暂。
- 结合相关工具对于了解程序正在发生的事情非常重要： 关闭GC的基准测试。
- 使用CPU分析器发现热点分配区域。
- 使用内存分析器了解内存分配次数/字节数。
- 使用执行追踪器了解GC模式。
- 使用逃逸分析器了解为什么会发生分配。

---

Go是一门内存托管的语言，这意味着大多数时间你都不需要担心手动内存管理，因为Go运行时为你做了大量的工作。然而，动态内存分配并不是免费的，并且程序的分配模式能够显著影响其性能。但是，从语言的语法来看，性能影响并不明显。本次演讲介绍了可用于检测和解决分配器瓶颈的技术和工具。

## 一个启发的例子

__Honeycomb__ 的一个存储/查询的架构图：

![](https://user-images.githubusercontent.com/1646931/44677781-0b35f500-a9f4-11e8-80ec-0171103c0690.png)

该架构的设计/性能目标是：

- subsecond / single-digit second query latency
- hundreds of millions / billions of rows per query
- flexible queries, on-the-fly schema changes

使用本次演讲中使用的一些技术，它们能够在一些关键的基准测试中获得高达2.5倍的性能提升。

![](https://user-images.githubusercontent.com/1646931/44685103-df246f00-aa07-11e8-99d1-4fe0a376c69d.png)

## 本次演讲的大纲

1. 分配器内部细节：内存分配器和垃圾回收器是如何工作的？
2. 工具： 理解分配器模式的工具。
3. 我们可以改变什么？提高内存效率的策略。

## 分配器内部细节

### 内存布局

根据它们的生命周期，对象可能会被分配在goroutine *栈上* 或 *堆上*。Go编译器决定对象将分配在什么地方，但一般的规则是，包含函数调用的对象需要在堆上分配。

![](https://user-images.githubusercontent.com/1646931/44678193-0cb3ed00-a9f5-11e8-9dba-4529535a0274.png)

那么堆中到底存储了什么？它是怎么达到那里的？谁在持续地追踪这一切？

想要回答这些问题，我们应该考虑堆内存分配器的作者可能想到的设计目标：

- 目标：高效地满足给定大小的分配操作，但是需要避免内存碎片

  - 解决方案：在块中分配大小相同额对象

- 目标：普通情况下需要避免锁

  - 解决方案：维护每个CPU的缓存

- 目标：高效回收空闲的内存

  - 解决方案：使用位图（bitmaps）作为元数据，同时运行GC。堆被分成两级别的结构： **arenas** 和 **spans** ：

![](https://user-images.githubusercontent.com/1646931/44678582-ee022600-a9f5-11e8-9165-e3ec3b2dd3c4.png)

一个全局的`mheap`结构体来持续追踪它们：

```go
type mheap struct {
  arenas [1 << 22]*heapArena       // Covering map of arena frames
  free []mSpanList                 // Lists of fully free spans
  central []mcentral               // Lists of in-use spans
  // plus much more
}
```

**Arenas** 是可用内存地址的粗略部分（在64位平台是64MB）。 我们以 **arenas** 为单位分配OS的内存。对于每一个 **arena**，`heapArena` 都有一些元数据：

```go
type mheap struct {
  arenas [1 << 22]*heapArena    // Covering map of arena frames
  // ...
}

type heapArena struct {
   // page-granularity map to spans
  spans [pagesPerArena]*mspan
  // pointer/scalar bitmap (2bits/word)
  bitmap [heapArenaBitmapBytes]byte
}
```

大多数的堆对象存在于一个 **span** 中：包含固定大小类别的对象的一系列页（pages）。

![](https://user-images.githubusercontent.com/1646931/44678675-29045980-a9f6-11e8-9168-3180e6d1927a.png)

```go
type span struct {
  startAddr  uintptr
  npages     uintptr
  spanclass  spanClass

  // allocated/free bitmap
  allocBits *gcBits
  // ...
}
```

有大约70种不同的 **span** 大小类别。

每个`P`都有一个`mcache`，其中包含每个大小类别的 **span**。 理想情况下， 分配操作可以直接从`mcahce`中解决（所以它们很快）。在Go中，一个`P`是一个调度上下文。通常每个核心有一个`P`，每次最多只有一个goroutine在`P`中运行：

![](https://user-images.githubusercontent.com/1646931/44678716-4507fb00-a9f6-11e8-8cac-cd36b62017d6.png)

如果想分配，我们需要在已缓存的`mspan`中找到第一个空闲对象，然后返回其地址：

![](https://user-images.githubusercontent.com/1646931/44679087-34a45000-a9f7-11e8-968e-009f8632b2db.png)

假设我们需要96个字节。首先，我们在`mcahce`中查找带有96字节对象的`mspan`。分配之后，结果看起来像如下这样：

![](https://user-images.githubusercontent.com/1646931/44679136-57366900-a9f7-11e8-92b9-be845e8fa153.png)

这中设计意味着“大多数”的内存分配操作是快速且无锁的。这里只有3个快速的步骤：

1. 寻找一个具有正确大小的已缓存的 **span** （`mcache.mspan[sizeClass]`）
2. 在 **span** 中寻找下一个空闲对象
3. 必要时候更新堆的位图（bitmaps）

所以现在我们已经涵盖了分配器的3个设计目标中的2个： (1)高效地满足给定大小的分配操作，但是需要避免内存碎片，(2)普通情况下需要避免锁。但是关于目标(3)高效回收空闲的内存是如何实现的？换言之，什么是垃圾回收器》

## 垃圾回收器

我们必须在对象不再被引用时找到它们并且回收它们。

Go语言使用**三色并发标记清除**垃圾回收器，这听起来很吓人，但事实并非如此。“标记-清除”意味着垃圾收集器被分为了标记阶段，为收集对象去标记，以及清除阶段，用来实际释放内存。

在一个三色垃圾收集器中，对象被标记成白色，灰色，和黑色。刚开始，所有的对象都是白色的。我们首先开始标记所有goroutine栈和全局变量。每当我们达到一个对象时，我们就会将其标记为灰色。

![](https://user-images.githubusercontent.com/1646931/44679351-f8bdba80-a9f7-11e8-8b0e-dbe1f7f17724.png)

当一个对象所有的引用都标记完成时，我们将这个对象标记成黑色。

![](https://user-images.githubusercontent.com/1646931/44679443-3b7f9280-a9f8-11e8-849a-5aec0b828b3d.png)

最终结果就是，所有的对象不是白色就是黑色：

![](https://user-images.githubusercontent.com/1646931/44679461-45a19100-a9f8-11e8-81f1-8952fbfe5190.png)

最后，标记为白色的对象可以被清除和释放。

很简单，是不是？好吧，它留下了一些问题：

- 我们如何知道一个对象的引用有哪些？
- 我们如何真正地去标记一个对象？

Go使用位图（bitmaps）来获取此类元数据。

### 指针识别

假如我们有个如下的结构体：

```go
type Row struct {
  index int
  data []interface{}
}
```

在内存中，这个结构体看起来是像这样的：

![](https://user-images.githubusercontent.com/1646931/44679558-90bba400-a9f8-11e8-89c6-9ad3ec1273e6.png)

那么，垃圾收集器是如何知道它指向的其他对象？即，哪个字段是指针？

需要记住的是堆对象实际上是存在于一个 **arena** 中的。这个 **arena** 的位图告诉我们它的那些字是指针：

![](https://user-images.githubusercontent.com/1646931/44679578-a16c1a00-a9f8-11e8-9a42-31bd426f0bb4.png)

### 标记状态

简单来说，标记状态保存在一个 **span** 的`gcMark`位中：

![](https://user-images.githubusercontent.com/1646931/44679775-1a6b7180-a9f9-11e8-8522-9e2cde8c65ac.png)

一旦我们完成了标记，未标记的对应于空闲槽位，因此我们可以将该标记状态交换为`alloc`位。

![](https://user-images.githubusercontent.com/1646931/44679878-5bfc1c80-a9f9-11e8-8030-89a0fcde8762.png)

这就意味着Go的清除操作通常会非常地快（虽然标记会增加开销，但这是我们必须要关注的）。

垃圾收集器是并发的，但这有点歧义。如果我们由于不小心的实现，程序可能不经意地阻止了垃圾收集器。例如，请使用下面的代码：

```go
type S struct {
  p *int
}

func f(s *S) *int {
  r := s.p
  s.p = nil
  return r
}
```

当该代码中函数的第一行执行之后，我们会得到这样的一个状态：

![](https://user-images.githubusercontent.com/1646931/44680059-eba1cb00-a9f9-11e8-968d-cdcfd2b36d5a.png)

在返回语句中，GC的状态可以看成如下这样：

![](https://user-images.githubusercontent.com/1646931/44680146-1b50d300-a9fa-11e8-80ec-8c526b2a8c31.png)

然后返回值将是一个实际上垃圾收集器可以释放的内存的实时指针！

为了防止这种情况放生，编译器将指针的写入转换为写屏障的潜在调用，非常粗暴：

![](https://user-images.githubusercontent.com/1646931/44680255-6ec32100-a9fa-11e8-9c0e-c915d5e89679.png)

Go编译器中有很多这样巧妙的优化可以让这个过程并发执行，但是我们不会在这里深入讲解。

但是，我们大概可以知道，就处理器的开销而言，标记可能是一个潜在的问题。

当GC在标记时，写屏障是打开的。

在标记期间，约25%的`GOMAXPROCS`被用于**后台标记任务**。但有额外的goroutine可能会被强制进行**辅助标记**：

![](https://user-images.githubusercontent.com/1646931/44680297-931efd80-a9fa-11e8-9521-406c3dc288e7.png)

那么为什么需要**辅助标记**呢？一个快速分配的goroutine可以逃过后台标记。所以在标记期间分配时，每个分配都会收取一个goroutine的费用（费用？？）。如果它有债务，它必须在继续之前做一部分标记工作：

```go
func mallocgc(size uintptr, ...) unsafe.Pointer {
   // ...
   assistG.gcAssistBytes -= int64(size)
	if assistG.gcAssistBytes < 0 {
       // This goroutine is in debt. Assist the GC to
       // this before allocating. This must happen
       // before disabling preemption.
       gcAssistAlloc(assistG)
    }
// ...
```

## 总结

我们已经了解到了很多关于运行时的知识：

- 运行时将数据分配在 **span** 中以避免内存碎片。
- 每CPU缓存加快了分配速度，但分配器仍然需要做一些工作。
- GC是并发的，但是写屏障和辅助标记可能会使程序变慢。
- GC的工作和*可扫描*的堆成比例（分配一个大的标量buffer要分配一个大的指针buffer开销小的多，因为你必须要跟踪指针）

## 工具

看起来动态内存分配有一些开销。那是不是意味着减少分配总是会提高性能？这得看情况。内建的内存分析器可以告诉我们哪里正在进行分配，但这无法回答因果问题， “减少分配是否会产生影响？” 想要回答这些，我们可以使用其他三个工具：

- 原生测试
- 使用pprof进行样本分析
- go工具追踪

### 原生测试

如果你想查看分配器和垃圾收集器增加了多少开销，你可以打开下列运行时的标志位：

- `GOGC=off`：关闭垃圾收集器
- `GODEBUG=sbrk=1`：使用简单的持久化分配器替换整个分配器，该分配器从OS中获取大块（big block)，并在分配时为你提供连续的切片。

![](https://user-images.githubusercontent.com/1646931/44680546-3f60e400-a9fb-11e8-88dd-093c9346f3e0.png)

这看起来可能有点愚蠢，但它是一种可以提供加速潜力的廉价方法。也许30%的加速不值得，或许这也可能是一次令人信服的机会。

这样的尝试会有一些问题：

- 不能再生产环境中使用，需要做基准测试
- 对于有大堆的程序来说不可靠：持久化分配器可能对这些程序执行的不好

然而，执行这样的检查是值得的。

### 分析（profiling）

一个pprof的CPU分析文件通常可以显示`runtime.mallocgc`所花费的时间。

贴士：

- 在pprof的web UI中使用火焰图来查看
- 如果二进制文件中不包含pprof，也可以使用Linux的`perf`工具

![](https://user-images.githubusercontent.com/1646931/44680673-b1392d80-a9fb-11e8-96b3-60079912fdee.png)

![](https://user-images.githubusercontent.com/1646931/44680690-b7c7a500-a9fb-11e8-893f-ec9f03225681.png)

它为我们提供了行级别的属性，即我们正在进行的CPU高开销的分配有哪些，并且可以帮助我们了解GC/分配器产生开销的根本原因。

这种做法也会有一些问题：

- 程序可能不是CPU密集型
- 分配可能不在关键路径上
- 后台标记时间（`gcBgMarkWorker`）会有误导作用（这里花的时间并不一定意味着有减速）

### go工具追踪

*执行追踪器*可能是我们可以理解分配细节及其影响的最佳工具。

执行追踪器在一个很短的时间窗口内捕获非常详细的运行时事件：

```bash
curl localhost:6060/debug/pprof/trace?seconds=5 > trace.out
```

可以在web UI中查看它的可视化输出：

```bash
go tool trace trace.out
```

尽管它看起来有点密集。。。

![](https://user-images.githubusercontent.com/1646931/44680802-fd846d80-a9fb-11e8-95da-16cbe5739faf.png)

当GC运行时，蓝色条框在顶层显示，而下方条框显示内部发生的一些情况。

记住，最上方的GC不意味着程序被阻塞了，但GC内部发生的事情很有趣！

![](https://user-images.githubusercontent.com/1646931/44680937-4fc58e80-a9fc-11e8-8705-a6a840072e56.png)

![](https://user-images.githubusercontent.com/1646931/44680955-5ce27d80-a9fc-11e8-8dd3-165b13ea6871.png)

最小的mutator利用率曲线还可以显示服务是否以不同的四分位数（25%）进行GC绑定。（“Mutator”在这里的意思是“不是GC”。）

总结

关闭分配器，CPU分析器分析，执行追踪器的基准测试给我们的感觉就是：

- 分配/GC是否会影响性能
- 哪些地方花费了大量的时间进行分配
- GC期间程序的吞吐量如何变化

## 我们能改变什么？

如果我们得出结论：分配才是程序效率低下的原因，那我们能做些什么？ 下面是一些高级策略用以提升程序运行效率：

- 限制指针
- 批量分配
- 复用对象（比如：`sync.Pool`）

### 那么只调整`GOGC`会怎么样呢？

- 绝对能够提升程序吞吐量，但是。。。
- 如果我们想优化程序的吞吐量，GOGC并不能表示真正的目标：“使用所有可用的内存，但没有更多内存使用”
- 实时的堆大小通常来说不会很小
- 高GOGC似的避免OOM更加困难

好吧，现在我们可以通过重构代码来改善分配/GC的行为。。。

### 限制指针

可以让Go编译器告诉我们为什么变量会分配到堆上：

```bash
go build -gcflags="-m -m"
```

但是它的输出有点多。

[https://github.com/loov/view-annotated-file](https://github.com/loov/view-annotated-file)有助于我们简化这些输出：

![](https://user-images.githubusercontent.com/1646931/44684310-f8c4b700-aa05-11e8-8963-6dffa42c5257.png)

有时候，很容易就能避免虚假的堆内存分配！

![](https://user-images.githubusercontent.com/1646931/44684321-02e6b580-aa06-11e8-892a-f7214bcd6127.png)

![](https://user-images.githubusercontent.com/1646931/44684378-3590ae00-aa06-11e8-9bda-a3c0dba5cfb4.png)

除了避免虚假的堆内存分配，避免使用内部指针的结构体也有助于垃圾收集器：

![](https://user-images.githubusercontent.com/1646931/44684432-4fca8c00-aa06-11e8-9258-0b1307d6d0e5.png)

为什么会出现这样的差异？因为`time.Time`结构体内部隐藏了一个邪恶的指针：

```go
type Time struct {
  wall uint64
  ext in64
  loc *Location
}
```

### 批量分配

即使分配器中的快速路径已经被优化到了极致，我们仍然需要在每个分配上做一些工作：

- 防止自己被别人抢先一步
- 检查自身是否需要协助GC
- 计算`mcache`中的下一个空闲槽位
- 设置堆内位图的比特位
- 等等。。。

这一系列的工作使得每次分配最终会耗时大约20ns，在这样的情形下，某些案例中我们可以通过减少更大的分配来分摊开销：

```go
// Allocate individual interface{}s out of a big buffer
type SlicePool struct {
	bigBuf []interface{}
}

func (s *SlicePool) GetSlice(size int) []interface{} {
	if size >= len(s.bigBuf) {
		s.bigBuf = make([]interface{}, blockSize)
	}
	res := s.bigBuf[:size]
	s.bigBuf = s.bigBuf[size:]
	return res
}
```

这样做的危险在于任何实时的引用将使整个slab存活：

![](https://user-images.githubusercontent.com/1646931/44684603-b94a9a80-aa06-11e8-9249-e2ad311f5587.png)

而且，这些都不是并发安全的。它们仅仅适合一些重度分配的goroutines。

### 对象回收复用

考虑下面的存储引擎架构：

![](https://user-images.githubusercontent.com/1646931/44684697-f7e05500-aa06-11e8-9650-7364d17b5401.png)

这个架构：

- 很简单，容易理解
- 最大化的并发
- 制造了成吨的垃圾

优化： 显式地回收复用已分配的内存块：

![](https://user-images.githubusercontent.com/1646931/44684763-21997c00-aa07-11e8-9bd2-8ebbc58ef8cb.png)

```go
var buf RowBuffer
select {
  case buf = <-recycled:
  default:
    buf = NewRowBuffer()
}
```

一个更加复杂的版本是使用`sync.Pool`

- 维护回收对象的切片，被CPU共享（需要运行时支持）
- 允许在快速路径中无锁地get/put
- 在每次的GC时都被清除

**警告**： 必须非常小心设零或覆盖已回收的内存。

## 回顾

下面是经过连续几轮优化后的不同基准测试图表：

![](https://user-images.githubusercontent.com/1646931/44685103-df246f00-aa07-11e8-99d1-4fe0a376c69d.png)

总结下：

- 内存分配器和垃圾回收器设计十分巧妙！
- 单个内存分配速度很快但不是免费的。
- 垃圾回收器能够暂停某些特定的goroutines，尽管**STW**暂停非常短暂。
- 结合相关工具对于了解程序正在发生的事情非常重要： 关闭GC的基准测试。
- 使用CPU分析器发现热点分配区域。
- 使用内存分析器了解内存分配次数/字节数。
- 使用执行追踪器了解GC模式。
- 使用逃逸分析器了解为什么会发生分配。

## 延伸阅读

建议延伸阅读：

**Allocation efficiency in high-performance Go services** Achille Roussel / Rick Branson [segment.com/blog/allocation-efficiency-in-high- performance-go-services/](segment.com/blog/allocation-efficiency-in-high- performance-go-services/)

**Go 1.5 concurrent garbage collector** pacing Austin Clements [golang.org/s/go15gcpacing](https://docs.google.com/document/d/1wmjrocXIWTr1JxU-3EQBI6BK6KgtiFArkG47XK73xIQ/edit#)

**So You Wanna Go Fast** Tyler Treat [bravenewgeek.com/so-you-wanna-go-fast/](https://bravenewgeek.com/so-you-wanna-go-fast/)
