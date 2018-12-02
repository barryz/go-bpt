> 译自[Memory Leaking Scenarios](https://go101.org/article/memory-leaking.html)。

# Go内存泄露的场景

当使用一门支持垃圾回收的编程语言进行编程时，一般情况下，我们都不需要关心内存泄露的问题，因为语言的运行时将会定期收集无用的内存。然而，我们仍然需要注意一些可能会导致各类内存泄露的特殊场景。本文接下来的部分将会列举到这些场景。

## 由于子字符串引起的内存泄露

Go规范没有指定子字符串表达式中涉及的结果字符串和基本字符串是否应该共享相同的底层[内存块](https://go101.org/article/memory-block.html)来托管这两个字符串的[底层字节序列](https://go101.org/article/string.html)。标准的Go编译器和运行时只是让它们共享了相同的底层内存块。从CPU和内存的开销方面来看，这是一个很好的设计。但是这样在有些情况下可能会导致内存泄露。

例如，当下面例子中的`f`函数调用之后，将会有1M字节左右的内存泄露，直到包级别变量`s0`在其他的地方被修改为止。

```go
var s0 string // a package-level variable

func f(s1 string) {
	// Assume s1 is a string much longer than 50.
	s0 = s1[:50]
	// Now, s0 shares the same underlying memory block of s1.
	// s1 is not alive now, but s0 is still alive.
	// Although there are only 50 bytes used in the block,
	// the fact that s0 is still alive prevent the 1M bytes
	// memory block from being collected.
}
```

为了避免此类的内存泄露，我们应该将子字符串转换成`[]byte`，然后再将`[]byte`转换成`string`。

```go
func f(s1 string) {
	s0 = string([]byte(s1[:50]))
}
```

上面这种避免内存泄露的解决办法的缺点在于，在它转换的过程中，复制了两份50字节大小的数据，并且其中一份数据是不必要的。

我们也可以通过标准Go编译器的[一些优化](https://go101.org/article/string.html#conversion-optimizations)来避免一次没有必要的复制，仅仅只浪费一字节的额外内存。

```go
func f(s1 string) {
	s0 = (" " + s1[:50])[1:]
}
```

上面这种优化方式的缺点就是其所依赖的编译器优化可能会在之后的版本里变得不可用，并且这种优化在其他的编译器里是不合法的。

可以使用第三种方式来避免这类的内存泄露，通过Go1.10版本开始支持的`strings.Builder`。

```go
import "strings"

func f(s1 string) {
	var b strings.Builder
	b.Grow(50)
	b.WriteString(s1[:50])
	s0 = b.String()
	// b.Reset() // if b is used elsewhere,
	             // it must be reset here.
}
```

第三种方式的缺点就是略微有点复杂（和前两种方式比较）。一个好消息是，从Go1.12开始（2019年2月份发布的Go版本），我们可以调用`strings`标准库中的`Repeat`函数来克隆一个字符串。从Go1.12开始，`strings.Repeat`的底层实现将使用`strings.Builder`。

## 子切片导致的内存泄露

和子字符串相似，子切片也可能会导致内存泄露。在下面的代码中，在函数`g`被调用之后，托管`s1`元素的内存块占用的大部分内存将丢失（如果没有更多值引用该内存块时）。

```go
var s0 []int

func g(s1 []int) {
	// Assume the length of s1 is much larger than 30.
	s0 = s1[len(s1)-30:]
}
```

如果我们希望避免这样的内存泄露，我们必须为`s0`重复30个元素，这样`s0`存活就不会阻止托管`s1`元素的内存块被回收掉。

```go
func g(s1 []int) {
	s0 = append([]int(nil), s1[len(s1)-30:]...)
}
```

## 在已死亡的切片元素中不重置指针导致的内存泄露

在下面的代码中，当函数`g`被调用后，为切片`s`第一个元素和最后一个元素分配的内存块将会丢失。

```go
func h() []*int {
	s := []*int{new(int), new(int), new(int), new(int)}
	// do smething ...
	return s[1:3:3]
}
```

只要返回的切片还存活者，它就会阻止托管切片`s`元素的底层内存块被回收掉，因为这样可以防止为`s`的第一个和最后一个元素分配的两个内存块被收集，尽管这两个元素已经死亡了。

如果我们想避免这类的内存泄露，我们必须为已死亡的元素重置指针。

```go
func h() []*int {
	s := []*int{new(int), new(int), new(int), new(int)}
	// do smething ...
	s1 := s[1:3]
	s[0] = nil; s[len(s)-1] = nil
	return s1
}
```

我们经常在[切片元素删除操作](https://go101.org/article/container.html#slice-manipulations)中需要重置一些旧切片元素的指针。

## 由Goroutines挂起导致的真实的内存泄露

有时候，由于一些代码设计上的逻辑错误，一个或多个Goroutines进入永久阻塞状态（挂起）。这类的Goroutines我们称之为挂起的Goroutines。在Go运行时看来，这些Goroutines是仍然存活的。一旦一个Goroutine处于挂起状态，分配给这个Goroutine的资源和被其引用的内存块就永不会被垃圾回收掉。

例如，如果传递一个`nil`通道给下面例子中的函数，这个函数就会变成挂起状态。所以分配给`s`的内存块将永不会被回收。

```go
func k(c <-chan bool) {
	s := make([]int64, 1e6)
	if <-c { // block here for ever if c is nil
		_ = s
		// use s, ...
	}
}
```

如果我们传递给上述例子中的函数的参数是一个非`nil`的通道，并且直到该函数被调用时并没有其它的Goroutine向这个通道发送值，当前的Goroutine也会处于挂起状态。

有时候，我们有意的使得主Goroutine进入挂起状态来避免程序退出。通常来说，其它Goroutines变成挂起状态都是我们不期望出现的。我们应该避免这种情况发生。

## 当不再使用`time.Ticker`时没有停止它而导致的真实的内存泄露

当一个`time.Timer`值不再使用时，它最终将会被回收，但是这对`time.Ticker`值来说，并不成立。在一个`time.Ticker`值不再使用时，我们应该先停止它。

## 由于不当终止导致的真实的内存泄露

为作为循环引用组的成员的值设置终止器可以[防止为循环引用组分配的所有内存块被回收](https://golang.org/pkg/runtime/#SetFinalizer)。 这属于真实的内存泄露。

当下面的函数调用并退出后，分配给`x`和`y`的内存块并不能保证在之后的GC操作中被回收掉。

```go
func memoryLeaking() {
	type T struct {
		v [1<<20]int
		t *T
	}

	var finalizer = func(t *T) {
		 fmt.Println("finalizer called")
	}

	var x, y T

	// SetFinalizer will make x escape to heap.
	runtime.SetFinalizer(&x, finalizer)

	// Following two lines combined will make
	// x and y not collectable.
	x.t, y.t = &y, &x // y also escapes to heap.
}
```

因此，请避免为循环引用组中的值设置终止器。

顺便说一句，我们[不应该使用终止器作为对象析构函数](https://go101.org/article/unofficial-faq.html#finalizers)。

## 由于延迟函数导致的内存泄露

请阅读[这篇文章](https://go101.org/article/defer-more.html#kind-of-resource-leaking)了解详情。
