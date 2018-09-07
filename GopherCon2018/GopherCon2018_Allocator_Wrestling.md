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

-  
