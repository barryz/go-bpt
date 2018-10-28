> 译自：[Memory Layouts](https://go101.org/article/memory-layout.html)。

# 内存布局

这篇文章将会介绍Go中对内存对齐的保证。我们有必要知道正确使用`sync/atomic`标准库中关于64位操作的相关函数所做的保证。

Go是一门类C语言，所以你会发现本文我们讨论的很多概念都和C有关。

## Go中的类型对齐保证

类型对齐（或值地址对齐）保证是Go规范对Go编译器的要求。如果一个类型`T`的对齐保证位`N`，那么类型`T`的每个值的地址在运行时必须是`N`的倍数。我们也可以说类型`T`的值的地址被承诺是`N`字节对齐的。

实际上，Go中每个类型都有两层的对齐保证，一个用于该类型作为其他（结构体）类型的一个字段，另一种情况则是针对其他情况（比如当该类型用于变量声明，数组元素类型等等）时。我们把前一种对齐类型称为字段对齐保证，后一种类型对齐称为通用对齐保证。

在编译期间，对一个类型`T`来说，当`t`不是类型`T`的一个字段时，我们可以调用`unsafe.Alignof(t)`来获取这个类型的通用对齐保证，当`x.t`是类型`T`的一个字段值时，我们可以调用`unsafe.Alignof(x.t)`来获取它的字段对齐保证。

在运行时，对一个类型`T`的值`t`来说，我们可以调用`reflect.ValueOf(t).Type().Align()`来获取类型`T`的通用对齐保证，调用`reflect.ValueOf(t).Type().FieldAlgin()`来获取类型`T`的字段对齐保证。

对目前的Go编译器而言，一个类型的通用对齐保证和字段对齐保证是恒等的。但对gccgo编译器来说，这句话是错误的。

本文之后的内容使用的保证说法就是特指对齐保证。

Go规范仅仅提到了[一点关于类型对齐保证](https://golang.org/ref/spec#Size_and_alignment_guarantees):

> 最小化保证以下对齐属性
> 1. 对任意类型的变量`x`来说：`unsafe.Alignof(x)`至少为1。
> 2. 对结构体类型的变量`x`来说：`unsafe.Alignof(x)`是该类型中所有字段中`unsafe.Alignof(x.f)`的最大值，最小值也是1。
> 3. 对数组类型的变量`x`来说： `unsafe.Align(x)`和该数组类型的元素类型的对齐值相同。

该规范里的保证没有指定任何类型的确切的对齐保证。它仅仅指定了一些最小化的保证。对相同的编译器而言，确切类型的对齐保证在不同的架构平台或不同的编译器版本上都有可能是不同的。

对于目前的（1.11）版本的Go标准编译器而言，对齐保证是这样的：

```bash
type                               alignment guarantee
------                             ------
bool, byte, uint8, int8            1
uint16, int16                      2
uint32, int32, float32, complex64  4
arrays and structs                 depend on element types and field types
other types                        size of a native word
```

这里，一个本地字（或机器字）在322位平台上是4个字节大小，在64位平台上8个字节大小。

这意味着，对于当前的Go标准编译器而言，其他类型的对齐保证可以是4或这8，这取决于构建的目标平台。

通常，在Go编程中我们无需关心值的地址对齐，除非我们想编写可移植的程序并使用`sync/atomic`标准库中的64位相关的函数。

## 64位字的原子分配保证

_（在这篇文章中，***64位字***意味着该类型的底层类型为`int64`或`uint64`的值。）_

在[sync/atomic文档](https://golang.org/pkg/sync/atomic/#pkg-note-BUG)的结尾处，它提到：

> - 在 X86-32平台上，64位的函数在Pentium MMX之前指令是不可用的。
> 在非Linux ARM平台，64位函数在ARMv6k core之前指令是不可用的。
>　
