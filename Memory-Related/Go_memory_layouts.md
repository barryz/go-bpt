> 译自：[Memory Layouts](https://go101.org/article/memory-layout.html)。

# 内存布局

这篇文章将会介绍Go中对内存对齐的保证。我们有必要知道正确使用`sync/atomic`标准库中关于64位操作的相关函数所做的保证。

Go是一门类C语言，所以你会发现本文我们讨论的很多概念都和C有关。

## Go中的类型对齐保证

类型对齐（或值地址对齐）保证是Go规范对Go编译器的要求。如果一个类型`T`的对齐保证为`N`，那么类型`T`的每个值的地址在运行时必须是`N`的倍数。我们也可以说类型`T`的值的地址被承诺是`N`字节对齐的。

实际上，Go中每个类型都有两类的对齐保证，一个用于该类型作为其他（结构体）类型的一个字段，另一种情况则是针对其他情况（比如当该类型用于变量声明，数组元素类型等等）时。我们把前一种对齐类型称为字段对齐保证，后一种类型对齐称为通用对齐保证。

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

这里，一个本地字（或机器字）在32位平台上是4个字节大小，在64位平台上8个字节大小。

这意味着，对于当前的Go标准编译器而言，其他类型的对齐保证可以是4或这8，这取决于构建的目标平台。

通常，在Go编程中我们无需关心值的地址对齐，除非我们想编写可移植的程序并使用`sync/atomic`标准库中的64位相关的函数。

## 64位字的原子分配保证

_（在这篇文章中，***64位字***意味着该类型的底层类型为`int64`或`uint64`的值。）_

在[sync/atomic文档](https://golang.org/pkg/sync/atomic/#pkg-note-BUG)的结尾处，它提到：

> - 在 X86-32平台上，64位的函数在Pentium MMX之前指令是不可用的。
> 在非Linux ARM平台，64位函数在ARMv6k core之前指令是不可用的。
>　在ARM平台和X86-32平台上，调用者有责任来安排原子访问的64位字的64位对齐。变量的第一个字或者在一个已分配的结构体，数组，或切片的第一个字可以依赖64位对齐。

_（这里，最后一句中的64位是一个追加的上下文。虽然上面引用的文本显示在该文档的错误区域中，但它们描述的内容却不是错误的。）_

所以，在一些非常老的CPU平台上，64位的原子操作是不受支持的。如果在这些平台上使用64位的函数将会导致程序崩溃。如果希望程序能够支持这些平台，我们应该使用`sync.Mutex`来替代使用64位字的同步。`sync.Mutex`中的方法相较于原子函数来说，比较低效。

在64位的CPU平台上，64位字通常是8字节对齐的，所以我们可以安全地调用64位函数。

我们需要注意的是对一些不太旧的32位架构平台进行64位字的分配。64位原子函数在这些平台是受支持的，但是这些平台上64位字的值地址并不能保证是8字节对齐的。为地址不是8字节对齐的64位字的值调用64位原子函数会导致程序在运行时产生恐慌。

但是，在这些不太旧的32位平台上，官方文档保证了 ___变量的第一个字或者在一个已分配的结构体，数组，或切片的第一个字可以依赖64位对齐。___

那么，字 ***被分配*** 是什么意思？你可以认为一个 ***被分配*** 的值是一个 ***被声明*** 的变量或者是通过`make`和`new`返回的一个值。如果一个切片值是从一个已分配的数组派生出来的，并且该切片的第一个元素也是该数组的第一个元素，那么这个切片值也可以被看作是一个已分配的值。

官方文档提供的保证非常保守。实际上，如果一个值位于一个在堆上分配的[内存块](https://go101.org/article/memory-block.html)的起始位置，那么该值的第一个字可以依赖于64位对齐。

并且，***依赖于8字节对齐*** 并不意味着一个全局变量或一个已分配结构体或数组中的第一个64位字的地址在32位体系架构上始终是8字节对齐的。它只是意味着如果以原子的方式访问此类值中的第一个64位字，则编译器必须保证该值的内存对齐在运行时必须是8字节对齐的。如果从不以原子的方式去访问它，则它的地址不保证是8字节对齐的。

还有更多其他的64位字能够被原子访问。

按照常理来说，一个合格的编译器应该保证一个数组/切片的所有元素都是被原子访问的，如果这个数组/切片的元素类型是一个64位字类型，并且该数组/切片中有一个元素是可以被原子访问的话，尽管官方文档并没有做这样的保证。

下面是一个关于64位字能够被安全地或不安全地进行原子访问的例子。

```go
type (
	T1 struct {
		v uint64
	}

	T2 struct {
		_ int16
		x T1
		y *T1
	}

	T3 struct {
		_ int16
		x [6]int64
		y *[6]int64
	}
)

// below, the "safe"s in comments mean "safe to be accessed atomically
// on both 64-bit and 32-bit architectures".

var a int64    // a is safe
var b T1       // b.v is safe
var c [6]int64 // c[0] is safe

var d T2 // d.x.v is unsafe
var e T3 // e.x[0] is unsafe

func f() {
	var f int64           // f is safe
	var g = []int64{5: 0} // g[0] is safe

	var h = e.x[:] // h[0] is unsafe

	// here, d.y.v and e.y[0] are both safe, for they are allocated.
	d.y = new(T1)
	e.y = &[6]int64{}

	_, _, _ = f, g, h
}

// In fact, all elements in c, g and e.y.v are safe to be accessed
// atomically, though the official documentations don't make the
// guarantees.
```

包的维护者应该注意有64位字段的可导出类型可以被原子访问。例如，将[expvar标准库](https://golang.org/pkg/expvar/)中的`Int`和`Float`类型嵌入到可在32位体系结构上运行的用户代码中是不安全的。

如果你不能确定一个64位字是否能被原子访问，你可以在运行时使用`[15]byte`类型的值来确定一个64位字的地址。

```go
package mylib

import (
	"unsafe"
	"sync/atomic"
)

type Counter struct {
	x [15]byte // instead of "x uint64"
}

func (c *Counter) xAddr() *uint64 {
	// the return must be 8-byte aligned.
	return (*uint64)(unsafe.Pointer(
		uintptr(unsafe.Pointer(&c.x)) + 8 -
		uintptr(unsafe.Pointer(&c.x))%8))
}

func (c *Counter) Add(delta uint64) {
	p := c.xAddr()
	atomic.AddUint64(p, delta)
}

func (c *Counter) Value() uint64 {
	return atomic.LoadUint64(c.xAddr())
}
```

通过使用这种解决方案，`Counter`类型可以被安全地嵌入到其他类型中，甚至在32位的平台中。这个方案的缺点是每个`Counter`类型的值都会浪费7个字节的空间。`sync`标准库是使用一个`[3]uint32`[值来代替这种技巧的](https://github.com/golang/go/blob/master/src/sync/waitgroup.go#L23)。这个技巧在标准Go编译器和gccgo编译器上运行良好，但是[可能不能运行在其他第三方的Go编译器上](https://github.com/golang/go/issues/27577)。

## 未来可能的变化

*Russ Cox*，Go团队的核心成员，[已经提议64位字的地址应该始终为8字节对齐](https://github.com/golang/go/issues/599)，无论是在32平台上，还是64位平台上，为了使Go编程更加简单。到目前版本为止（Go 1.11），这个提案还未被通过。

## 类型值的大小和结构体字段填充

Go规范针对[类型尺寸大小](https://golang.org/ref/spec#Size_and_alignment_guarantees)作了以下保证：

```bash
type                               size in bytes
------                             ------
byte, uint8, int8                   1
uint16, int16                       2
uint32, int32, float32              4
uint64, int64, float64, complex64   8
complex128                         16
```

Go规范并未为上述类型之外的类型的值尺寸大小作任何保证。对于目前的标准Go编译器（Go 1.11）来说，`bool`值只有一个字节大小，并且`int/uint/uintptr`指针值是一个本地字大小。[这篇文章](https://go101.org/article/value-copy-cost.html#value-sizes)列出了由标准Go编译器已经确定的不同类型的值大小的完整清单。

标准Go编译器将确保类型值的大小是该类型的对齐保证的倍数。

要满足之前部分提到的类型对齐保证，Go编译器和运行时可能会在结构体的字段值之间填充一些字节。这使得结构体类型的值大小可能不是该结构体类型的所有字段大小之和。

下面是一个展示如何在结构体字段之间填充字节的示例：

```go
// The alignments of type T1 and T2 are the same as
// the largest alignment of their field types (int64),
// 8 on AMD64 OS and 4 on i386 OS.

type T1 struct {
	a int8
	// To make b 8-aligned on AMD64 OS and 4-aligned on i386 OS,
	// 7 bytes padded on AMD64 OS and pad 3 bytes padded on i386 OS here.
	b int64
	c int16
	// To make the size of T1 values is a multiple of the alignment of T1,
	// 6 bytes padded on AMD64 OS and pad 2 bytes padded on i386 OS here.
}
// the sizes of T1 values are 24 on AMD64 OS and 16 on i386 OS.

type T2 struct {
	a int8
	// To make c 2-aligned,
	// 1 byte padded on both AMD64 and i386 OS here.
	c int16
	// To make b 8-aligned on AMD64 OS and 4-aligned on i386 OS,
	// 4 bytes padded on AMD64 OS here. No padding on i386 OS.
	b int64
}
// the sizes of T2 values are 16 on AMD64 OS and 12 on i386 OS.
```

尽管`T1`和`T2`有相同个字段集合，但是它们的大小却是不同的。

对于标准Go编译器而言，一个有趣的事实是，有时候零尺寸大小的字段也会影响到结构体填充。请阅读[the question in the unoffical Go FAQ](https://go101.org/article/unofficial-faq.html#final-zero-size-field)来获取详情。
