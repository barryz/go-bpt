> 译自[Type-Unsafe Pointers](https://go101.org/article/unsafe.html)。

# 类型不安全的指针

我们已经通过[pointers in Go](https://go101.org/article/pointer.html)学习到了Go中的指针。通过那篇文章，我们知道，和C中的指针相比较，Go中的指针有很多的限制。例如，Go指针不能参与算术运算。对于两个任意的指针类型，它们的值很可能不能相互转换。

在那篇文章中介绍到的指针实际上是类型安全的指针。尽管类型安全的指针的各种限制为我们编写Go代码提供了便利，同时也让我们在某些情况下编写高效的代码带来了阻碍。

实际上，到目前为止（Go1.10），Go也提供了类型不安全的指针，这类指针不受类型安全指针的限制。类型不安全的指针在Go中也被称为不安全的指针。Go中不安全的指针和C语言中的指针很相似，它们很强大，同时又很危险。在一些场景下，我们可以借助不安全指针来编写高效的代码。另一方面，使用不安全指针也很容易编写无效的代码，而且很难检测到。

使用不安全指针的最大风险来自于这样一个事实，即不安全的机制不受[Go1.x兼容性指南](https://golang.org/doc/go1compat)的保护。现在依赖于不安全指针的代码在之后的Go版本中可能无法正常工作。

如果你真的希望因为有些原因需要使用不安全的指针来有效改进代码，你不仅应该知道上述的风险，你也应该按照官方的说明文件，清楚地理解每个安全指针使用所产生的影响，因此，你可以使用不安全指针编写安全的代码，至少在目前标准Go编译器SDK1.10中是可以使用的。

##  关于unsafe标准库

Go为不安全指针提供了一个特殊的类型种类。我们必须导入[unsafe标准库](https://golang.org/pkg/unsafe/)后才能使用不安全指针。`unsafe.Pointer`类型定义如下

```go
type Pointer *ArbitraryType
```

诚然，这不是一个通常意义上的类型定义。这里的`ArbitraryType`仅仅提示了了`unsafe.Pointer`值可以在Go中被转换成任何安全的指针的值，反之亦然。换句话说，`unsafe.Pointer`和C中的`void*`很相似。

Go中不安全的指针意味着底层类型是unsafe.Pointer类型的类型。

不安全指针的零值也是使用预先定义的标识符`nil`表示。

unsafe标准库同样提供了三个函数。

- `func Alignof(variable ArbitraryType) uintptr`，用于获取值的对齐地址。请注意，相同类型的结构体字段值和非字段值的对齐值可能不同，但对于标准Go编译器，它们始终相同。

- `func Offsetof(selector ArbitraryType) uintptr`，用来获取一个结构体字段值的地址偏移量。此处的偏移量是相对于字段值的地址而言的。对于同一程序中相同类型的结构体值的相同的对应字段，返回结果应始终相同。

- `func Sizeof(variable ArbitraryType) uintptr`，用来一个值的大小（也叫做值的类型大小）。该函数对于同一个程序中相同类型所有的值返回的结果总是相同的。

注意上述函数返回结果的类型都是`uintptr`。下面我们将学习到`uintptr`可以转成`unsafe.Pointer`，反之亦然。

尽管这三个函数的返回结果在同一程序中是一致的，但它们在不同架构，不同操作系统，交叉编译器和交叉编译器版本下返回结果可能又是不同的。

这三个函数的调用将始终在编译器求值。他们的值可以分配给常量。

使用上述三个函数的例子：

```go
package main

import "fmt"
import "unsafe"

func main() {
	var x struct {
		a int64
		b bool
		c string
	}
	const M, N = unsafe.Sizeof(x.c), unsafe.Sizeof(x)
	fmt.Println(M, N)   // 16 32

	fmt.Println(unsafe.Alignof(x.a)) // 8
	fmt.Println(unsafe.Alignof(x.b)) // 1
	fmt.Println(unsafe.Alignof(x.c)) // 8

	fmt.Println(unsafe.Offsetof(x.a)) // 0
	fmt.Println(unsafe.Offsetof(x.b)) // 8
	fmt.Println(unsafe.Offsetof(x.c)) // 16
}
```

请注意，上述打印的结果都是基于Liunx amd64平台下Go1.10的标准编译器。

## 不安全指针相关转换规则

Go编译器允许下列显式的转换。

- 一个安全的指针可以显式地转换成一个不安全指针，反之亦然。
- 一个`uintptr`值可以显式地转换成一个不安全指针，反之亦然。

通过这些转换，我们可以在任意两个安全的指针之间、任意两个`uintptr`之间，一个任意类型的安全指针和`uintptr`值之间应用这些转换。不安全指针在这些转换中扮演了桥梁的角色。

尽管，这些转换在编译器都是合法的，但它们在运行时并不都是合法（安全）的。这些转换破坏了整个Go类型系统（不安全部分除外）试图维护的内存的安全性。我们必须按照后面章节部分中列出的说明来编写带有不安全指针的有效Go的代码。


## 在Go中我们需要了解的一些事实

在介绍如何合法（合理）使用不安全指针之前，我们需要知道Go中的一些事实。

### 事实1： 不安全指针是指针， `uintptr`值是整型

任何一个非空的安全或不安全指针都引用了另一个值。然而，`uintptr`值不会引用任何值，它们仅仅是个普通整型，但通常每个值都存储了一个内存地址。

Go是一个支持自动垃圾回收的语言。当一个Go程序运行时，Go运行时将检查哪些值不再使用，并不时收集为这些未使用的值分配的内存，[GC](https://go101.org/article/memory-block.html#when-to-collect)。在这个检查过程中指针扮演了很重要的角色。如果某个值无法从任何仍在使用的值中引用，则Go运行时认为它是一个未使用的值，可以安全地进行垃圾回收。

因为`uintptr`是整型，所以它们能进行算术运算。

下面小节中的例子将会展示指针和`uintptr`值的区别。

### 事实2： 不再使用的值可能会在任何时候被回收

在运行时时，垃圾回收器可能会在任意时间运行，所以每当一个值不可用时，不可用值的分配的[内存块](https://go101.org/article/memory-block.html)可能会被[随时收集](https://go101.org/article/memory-block.html#when-can-collect)。


例子：

```go
package main

import "fmt"
import "unsafe"

var x *int
var y unsafe.Pointer
var z uintptr

func main() {
	var vx, vy, vz int
	x = &vx
	y = unsafe.Pointer(&vy)
	z = uintptr(unsafe.Pointer(&vz))

	// At the time, even if its address of vz is stored in z.
	// vz has already become unused. Gargage collector can
	// collect the memory allocated for it now.
	// On the other hand. vx and vy are still in using,
	// for they are reachable from the x and y pointers.

	// uinptr can be used as operands of arithmetic operators.
	z++
	z &^= 1 // <=> z = z &^ 1

	fmt.Println(x, y, z)
}
```

### 事实3：我们可以使用`runtime.KeepAlive`函数调用来标记一个值当前仍在使用（可达）

为了标记一个值仍在使用，我们应该将另一个引用该值的值作为参数传递给`runtime.KeepAlive`函数调用。一般用值的指针作为该函数的参数。

在下面的代码中，对最后一小节中的示例进行了小的修改。

```go
func main() {
	var vx, vy, vz int
	x = &vx
	y = unsafe.Pointer(&vy)
	z = uintptr(unsafe.Pointer(&vz))

	// do other things ...

	// vz is still reachable at least up to here, so
	// it will not be garbage collected now for sure.
	runtime.KeepAlive(&vz)
}
```

### 事实4：`*unsafe.Pointer`是一个通用的安全指针类型

是的，`*unsafe.Pointer`是一个未命名的安全指针类型。它的基类型是`unsafe.Pointer`。因为它是一个安全指针，通过上述的转换规则，它可以被转换成一个`unsafe.Pointer`类型，反之亦然。

例子：

```go
package main

import "unsafe"

func main() {
	x := 123 // of type int
	p := unsafe.Pointer(&x)
	pp := &p // of type *unsafe.Pointer
	p = unsafe.Pointer(pp)
	pp = (*unsafe.Pointer)(p)
}
```

## 如何正确使用不安全指针

`unsafe`标准库文档中列举了[6个不安全指针的使用模式](https://golang.org/pkg/unsafe/#Pointer)。下面将逐个解释。

### 模式1： 将`*T1`转换成不安全指针，再将不安全指针转换成`*T2`

正如上面提到的，我们使用上述的不安全指针的转换规则，我们可以将类型`*T1`的值转换成`*T2`，这里的`*T1`和`*T2`是任意类型。然而，只有当类型`T1`的尺寸不大于类型`T2`时，我们才能进行这样的转换。并且这样的转换才是有意义的。

因此，我们还可以通过使用此模式实现类型`T1`和`T2`之间的转换。
