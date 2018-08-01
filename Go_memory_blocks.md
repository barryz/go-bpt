
> 译自： [Memory Blocks](https://go101.org/article/memory-block.html)

# Go 内存块

Go语言内置了自动内存管理， 比如内存分配和垃圾回收。 所以Go开发者不需要关心底层的内存管理问题。 降低了开发者的心智负担。

对于Go开发者来说， 尽管了解底层的内存管理实现细节不是必须的，但是了解一些概念并通过便准的Go编译器和运行时了解内存管理实现细节中的一些内容对于Go开发者编写高质量的Go代码还是非常有帮助的。

本文将尝试解释Go语言中内存块(memory block) 的在Go标准编译器和运行时的分配，垃圾回收等一些概念和细节。


## Go 内存块(memory block)

内存块是用于在运行时管理值的一个连续的内存段(segment)。 不同的内存块可以大小不同， 用来管理不同的值。 一个内存块可以同时管理多个值的部分， 但每个值只能被一个内存块管理，无论该值的大小有多大。换言之， 对于任意的值部分， 都不可能跨越内存块存储。

有几个原因会导致内存块管理多个值部分：

  - 结构体(struct)值通常会有若干个字段， 因此当为结构体值分配内存块时， 内存块也将管理这些字段值。

  - 数组(array)值通常会有若干个元素， 因此当为数组值分配内存块时， 内存块也将管理这些元素值。

  - 两个切片(slice)共享一个底层数组(array)， 那么他们将被同一个内存块管理， 这两个切片的元素序列甚至可以彼此重叠。


## 内存块管理的值部分的引用

如果一个值部分`v`由另一个值部分引用， 那么另一个值也将间接引用管理`v`的内存块。


## 什么时候会分配内存块？

在Go里， 以下情况会导致内存块的分配：

  - 显式调用内建函数： `new` 和 `make`。 `make` 函数调用将会分配多个内存块用来管理创建的 map，slice， channel值的直接或者间接的部分。

  - 使用相应的字面量创建map， slice， 或者匿名函数。 每个操作上会分配多个内存块。

  - 变量声明。 取决于变量的值类型， 可以在每次声明中分配单个或者多个内存块。

  - 为接口值分配非接口值时（当非接口值不是指针值时）。

  - 字符串拼接。

  - 字符串转换成byte或rune切片时，反之亦然(vice versa)。

  - 整型转换成`string`

  - `append`内建函数调用（当声明时的slice容量不足时）。

  - `map`中添加新的key-value时（因为需要调整基础hash table大小）。


## 内存块具体分配到什么地方？

对于标准Go编译器和运行时来说，每个Go程序， 每个goroutine在运行时都会维护一个内存区域， 我们称之为栈(stack)。 每个goroutine的初始栈都很小。 栈的大小会根据实际情况扩缩容。

(***需要注意一点， 对于标准的Go编译器来说， 每个goroutine的栈大小都有限制。 在Go1.10版本之后， 64位OS上默认的最大栈大小为1GB，32位OS上为256GB。 可以通过 `runtime/debug` 包下的 `SetMaxStack` 函数改变这个大小。***)

在goroutine栈上分配的内存块只能在goroutine内部使用（引用）。这些内存块属于goroutine内部资源。 跨goroutine引用这些内存块是不安全的。 一个goroutine可以访问分配在其栈上的内存块， 而无需使用任何数据同步的技术。

堆(Heap)是每个程序的单例。 分配在堆上的内存块可以被多个goroutine访问。 换言之， 这些内存块可以被并发地访问。它们在并发访问时可能需要同步。

堆是注册内存块的相对保守的地方。 如果编译器检测到内存块被多个goroutine引用或者不能确认内存块是否可以安全地放在栈上， 那么内存块会在运行时被分配到堆上。 这就意味着可以在栈上安全地分配一些值， 也可以在堆上分配一些值。


实际上， 栈对于Go程序来说，并不是必不可少地。 Go的编译器和运行时可以将所有的内存块都分配到堆上。 栈的支持仅仅是为了提高Go程序的运行效率：

- 在栈上分配内存块会比在堆上分配内存块快的多。

- 分配在栈上的内存不需要垃圾回收。

- 栈上的内存块相较于堆上的内存块，对CPU的缓存策略更加友好。

如果在某个地方（堆或者栈）上分配了内存块， 我们也可以说内存块上被管理的值部分分配在同一个区域。

如果函数中的一个值分配到了堆上，我们就可以说这个值逃逸到了堆上。 使用标准Go SDK的情况下， 我们可以使用 `go build -ldflags -m` 命令来检测是否有本地的值在运行时逃逸到了堆上。 正如上面所提到的，目前Go标准编译器里的逃逸分析工具还不是很完美， 很多可以安全地分配在栈上的值还是会被分配到堆上。

当类型为`T`的局部变量逃逸到堆上时， 这也意味着Go运行时还会在当前的goroutine的栈上创建类型为`*T`的隐式指针。 指针的值存储了分配到堆上的内存块的地址（也就是 类型为`T`的局部变量的地址）。 Go编译器已经在编译时将对变量的所有引用替换成为该指针值的反引用。

包级别的全局变量永远不会在栈上分配内存块。 它们是否在堆上分配内存块依赖于编译器的具体实现。 `A value part referenced by the direct part of any global variable and any value part allocated on heap will be allocated on heap if the value part is not the direct part of a global variable. Direct parts of global variables and value parts allocated on heap will never reference value parts allocated on stacks. Value parts allocated on a stack can only be referenced by value parts allocated on the same stack. `

一些例子：

 - 如果结构体的字段值逃逸到了堆上，那么整个结构体的值也会逃逸到堆上。

 - 如果数组的元素的值逃逸到了堆上， 那么整个数组的值也会逃逸到堆上。

 - 如果切片的元素的值逃逸到了堆上， 那么整个切片的所有元素也会逃逸到堆上。

 - 如果一个非`nil`的切片逃逸到堆上， 则其元素也将逃逸到堆上。


另外通过`new`函数既可以在堆上也可以在栈上分配内存块， 这与C++里面的new不同。


## 什么时候内存块会被回收？

包级别全局变量的内存块永远不会被垃圾回收。

当goroutine退出它时，它所拥有的栈将会被整个的回收掉。 所以不需要逐个地收集栈上分配地内存块， 垃圾回收器不会回收栈。

对于分配在堆上的内存块， 只有当全局变量值的直接部分和goroutine栈上分配的任何值不再（直接或者间接）引用它时， 才能被垃圾回收器安全地回收掉。我们将这些将要被回收的内存块称之为未使用的内存块。 这里的全局值不仅包括显式的包级变量，还包括一些运行时的全局值。垃圾回收器将收集堆上未使用的内存块。

下面的例子可以展示内存块被回收的情况：

```go
package main

var p *int

func main() {
	done := make(chan bool)
  // done 将会在main中和后面的goroutine中使用到， 所以
  // 将会被分配到堆上

	go func() {
		x, y, z := 123, 456, 789
		_ = z  // z 会被安全地分配到栈上
		p = &x // x&y 被全局的p引用， 所以它们将被分配到堆上
		p = &y
		p = nil
		// 现在x&y都没有被引用，所以它的内存块将会被收集并回收掉

		done <- true
	}()

	<- done
  // 上面的goroutine已经退出了， done channel 不再被使用，
  // 所以也会被回收掉。

	// ...
}
```

有时智能编译器（例如标准Go编译器） 可能会进行一些优化， 以便比我们预期地更早删除某些引用， 下面是一个例子：

```go
package main

import "fmt"

func main() {
	bs := make([]int, 1000000)

  // 智能编译器能够检测到bs这个切片的底层数组之后将不会再被使用，
  // 于是就将其回收掉。

	fmt.Println(len(bs))
}
```

可以通过阅读[value parts](https://go101.org/article/value-part.html)深入了解slice值的内部结构。

有时候，我们可能希望在某些调用（如上面的`fmt.Println()`）之后保证切片`bs`不被垃圾回收， 我们可以使用`runtime.KeepAlive`函数告诉垃圾回收器我们需要继续使用切片`bs`。

例子：

```go
package main

import "fmt"
import "runtime"

func main() {
	bs := make([]int, 1000000)

	fmt.Println(len(bs))
	runtime.KeepAlive(&bs) // runtime.KeepAlive(bs) 告诉垃圾回收器不需要回收bs
}
```

如果涉及到[unsafe pointer](https://go101.org/article/unsafe.html)， 则经常需要`runtime.KeepAlive`函数调用。

## 什么时候收集内存块？

Go运行时将收集未使用的堆内存（垃圾）以便重用或者释放内存。目前的标准Go编译器使用并发的三色标记-清除 垃圾回收器。

收集器并不是总在运行。 它会在满足了某个阈值的情况下才会被触发。 因此， 未使用的内存块在内使用之后可能不会立即被清除掉。 目前这个回收阈值是由环境变量：`GOGC` 控制的：

>  The GOGC variable sets the initial garbage collection target percentage. A collection is triggered when the ratio of freshly allocated data to live data remaining after the previous collection reaches this percentage. The default is GOGC=100. Setting GOGC=off disables the garbage collector entirely.

此环境变量的值确定了垃圾回收的频率， 并且可以在运行时进行[修改](https://golang.org/pkg/runtime/debug/#SetGCPercent)。小值可能会导致频繁的垃圾回收。 Go程序可以通过调用`runtime.GC`函数显式地运行垃圾回收器。

在标记阶段， 收集器使用上面提到的三色标记算法来分析未使用的内存块：

>  At the start of a GC cycle all objects are white. The GC visits all roots, which are objects directly accessible by the application such as globals and things on the stack, and colors these grey. The GC then chooses a grey object, blackens it, and then scans it for pointers to other objects. When this scan finds a pointer to a white object, it turns that object grey. This process repeats until there are no more grey objects. At this point, white objects are known to be unreachable and can be reused.

这里， 对象指的是值或者内存块。

未使用的内存块将在扫面阶段被扫描， 收集器是非压缩的， 因此它不会移动内存块来重新排列它们。
