> 译自：[Atomic Operations Provided In The sync/atomic Standard Package](https://go101.org/article/concurrent-atomic-operation.html) :book:

# 标准库`sync/atomic`提供的原子操作

原子操作相较于其他同步机制更加原始。并且它们是无锁的，通常需要在硬件层面去实现原子操作。实际上，它们经常用来实现其他的同步技术。

请注意，下面将要介绍的例子很多都不是并发程序，它们的目的仅是为了演示，用来展示如何使用`sync/atomic`标准库下提供的一系列的原子函数。

##  Go中的原子操作概览

`sync/atomic`标准库为一个整型类型`T`提供了下列5种原子方法。这里的`T`必须是类型`int32`， `int64`，`uint32`，`uint64`和`uintptr`之一。

```go
func AddT(addr *T, delta T)(new T)
func LoadT(addr *T) (val T)
func StoreT(addr *T, val T)
func SwapT(addr *T, new T) (old T)
func CompareAndSwapT(addr *T, old, new T) (swapped bool)
```

例如，下面有5个例子， 用来提供给类型`int32`的进行原子操作。

```go
func AddInt32(addr *int32, delta int32)(new int32)
func LoadInt32(addr *int32) (val int32)
func StoreInt32(addr *int32, val int32)
func SwapInt32(addr *int32, new int32) (old int32)
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
```

下面的例子中，`sync.Atomic`标准库为不安全的指针类型提供了4个原子函数。因为Go1现在不支持自定义泛型，所以这些函数是通过[不安全的指针类型](https://go101.org/article/unsafe.html)`unsafe.Pointer`(对应C中的`void*`)实现的。

```go
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
func SwapPointer(addr *unsafe.Pointer, new T) (old unsafe.Pointer)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
```

对于指针来说，没有`AddPointer`函数。因为Go的指针不支持算术运算。

标准库`sync.Atomic`同样提供了一个类型： `Value`。它相应的指针类型`*Value`有两个方法，`Load`和`Store`。一个`Value`的值可以用于原子性的存储和加载任意类型的值。

```go
func (v *Value) Load() (x interface{})
func (v *Value) Store(x interface{})
```

本文接下来的内容将会介绍如何使用Go中提供的原子操作。

## 整型的原子操作

下面的例子展示了如何使用`AddInt32`函数在一个`int32`类型的值上进行原子`add`操作。在这个例子中，主Goroutine创建了1000个全新的并发执行的Goroutines。每个新建的Groutine都会在整型`n`上执行加1操作。原子操作则保证了在这些Goroutines之间不会产生数据竞争。最后，`1000`这个结果将会打印出来。

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var n int32
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			atomic.AddInt32(&n, 1)
			wg.Done()
		}()
	}
	wg.Wait()

	fmt.Println(atomic.LoadInt32(&n)) // 1000
}
```

原子函数`StoreT`和`LoadT`通常用于实现一个类型（对应的指针类型）的setter和getter方法，在需要并发访问该值的情况下。例如，

```go
type Page struct {
	views uint32
}

func (page *Page) SetViews(n uint32) {
	atomic.StoreUint32(&page.views, n)
}

func (page *Page) Views() uint32 {
	return atomic.LoadUint32(&page.views)
}
```

对于有符号的类型`int32`和`int64`而言，`AddT`函数的第二个参数可以是负数，用来执行原子减少（decrease）操作。但是该如何对无符号的类型`T`的值进行原子减少操作呢？ 例如`uint32`，`uint64`和`uintptr`？对于第二种参数是无符号情况有两种。

1. 对于一个无符号类型`T`的变量`v`而言，`-v`在Go中是合法的。 所以我们可以在`AddT`函数调用中传递`-v`。

2. 对于一个正数整型常量`c`，将`-v`用作`AddT`函数调用的第二参数是非法的。我们可以使用`^T(c-1)`作为第二个参数。

`^T(c-1)`这个技巧对于无符号变量`v`来说同样适用，但是`^T(v-1)`相对`T(-v)`来说，比较低效。

在`^T(c-1)`这个技巧中，如果`c`是一个类型值并且它的类型能和`AddT`第二参数的类型匹配上，那么可以写成`^(c-1)`这种形式。

例子：

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var (
		n uint64 = 97
		m uint64 = 1
		k int    = 2
	)
	const (
		a        = 3
		b uint64 = 4
		c uint32 = 5
		d int    = 6
	)

	atomic.AddUint64(&n, -m);             fmt.Println(n) // 96 (97 - 1)
	atomic.AddUint64(&n, -uint64(k));     fmt.Println(n) // 94 (95 - 2)
	atomic.AddUint64(&n, ^uint64(a - 1)); fmt.Println(n) // 91 (94 - 3)
	atomic.AddUint64(&n, ^(b - 1));       fmt.Println(n) // 87 (91 - 4)
	atomic.AddUint64(&n, ^uint64(c - 1)); fmt.Println(n) // 82 (87 - 5)
	atomic.AddUint64(&n, ^uint64(d - 1)); fmt.Println(n) // 76 (82 - 6)
	x := b; atomic.AddUint64(&n, -x);     fmt.Println(n) // 72 (76 - 4)
	atomic.AddUint64(&n, ^(m - 1));       fmt.Println(n) // 71 (72 - 1)
	atomic.AddUint64(&n, ^uint64(k - 1)); fmt.Println(n) // 69 (71 - 2)
}
```

`SwapT`函数和`StoreT`函数类似，但是它返回了被交换的值。

`CompareAndSwapT`函数调用仅当当前存储的值与传递的旧值匹配时才会应用存储操作。返回的`bool`值表示了`CompareAndSwapT`函数的存储操作是否成功。

例子：

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var n int64 = 123
	var old = atomic.SwapInt64(&n, 789)
	fmt.Println(n, old) // 789 123
	fmt.Println(atomic.CompareAndSwapInt64(&n, 123, 456)) // false
	fmt.Println(n) // 789
	fmt.Println(atomic.CompareAndSwapInt64(&n, 789, 456)) // true
	fmt.Println(n) // 456
}
```

请注意，到目前为止（Go 1.11），64位的字，即int64和uint64值，它们原子操作要求64位字在内存中是按每8字节对齐的。详情请阅读[内存布局](https://go101.org/article/memory-layout.html)。


## 指针的原子操作
