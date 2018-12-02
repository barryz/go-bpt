> 译自[Type-Unsafe Pointers](https://go101.org/article/unsafe.html)。

# 类型不安全的指针

我们已经通过[pointers in Go](https://go101.org/article/pointer.html)学习到了Go中的指针。通过那篇文章，我们知道，和C中的指针相比较，Go中的指针有很多的限制。例如，Go指针不能参与算术运算。对于两个任意的指针类型，它们的值很可能不能相互转换。

在那篇文章中介绍到的指针实际上是类型安全的指针。尽管类型安全的指针的各种限制为我们编写Go代码提供了便利，同时也让我们在某些情况下编写高效的代码带来了阻碍。

实际上，到目前为止（Go1.10），Go也提供了类型不安全的指针，这类指针不受类型安全指针的限制。类型不安全的指针在Go中也被称为不安全的指针。Go中不安全的指针和C语言中的指针很相似，它们很强大，同时又很危险。在一些场景下，我们可以借助不安全指针来编写高效的代码。另一方面，使用不安全指针也很容易编写无效的代码，而且很难检测到。

使用不安全指针的最大风险来自于这样一个事实，即不安全的机制不受[Go1.x兼容性指南](https://golang.org/doc/go1compat)的保护。现在依赖于不安全指针的代码在之后的Go版本中可能无法正常工作。

如果你真的希望因为有些原因需要使用不安全的指针来有效改进代码，你不仅应该知道上述的风险，你也应该按照官方的说明文件，清楚地理解每个安全指针使用所产生的影响，因此，你可以使用不安全指针编写安全的代码，至少在目前标准Go编译器SDK1.10中是可以使用的。

## 关于unsafe标准库

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

`math.Float66bits`函数就是一个例子，此函数可以将一个`float64`的值转换成一个`uint64`的值。这两个值相应的位在内存中的表示是相同的。`math.Float64frombits`函数则可以实现反向转换。

```go
func Float64bits(f float64) uint64 {
	return *(*uint64)(unsafe.Pointer(&f))
}

func Float64frombits(b uint64) float64 {
	return *(*float64)(unsafe.Pointer(&b))
}
```

请注意，函数`Float64bits(aFloat64)`的结果和显式转换`uint64(aFloat64)`的结果是不同的。

在下面的例子中，我们使用模式一将切片类型`[]MyString`转换成类型`[]string`，反之亦然。结果切片和源切片共享底层的元素。这样的转换通过安全的方式是无法实现的。

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	type MyString string
	ms := []MyString{"C", "C++", "Go"}
	fmt.Printf("%s\n", ms)  // [C C++ Go]
	// ss := ([]string)(ms) // compiling error
	ss := *(*[]string)(unsafe.Pointer(&ms))
	ss[1] = "Rust"
	fmt.Printf("%s\n", ms) // [C Rust Go]
	// ms = []MyString(ss) // compiling error
	ms = *(*[]MyString)(unsafe.Pointer(&ss))
}
```

### 模式2： 将不安全指针转换成`uintptr`，然后再使用`uintptr`值

这个模式并不是很有用。通常，我们打印结果`uintptr`值来检查存储在它们中的内存地址。然而，还有其他更加简单的方式实现此功能。所以这个模式不是很重要。

例子：

```go
package main

import "fmt"
import "unsafe"

func main() {
	type T struct{a int}
	var t T
	fmt.Println(&t)                                 // &{0}
	fmt.Printf("%x\n", uintptr(unsafe.Pointer(&t))) // c6233120a8
	println(&t)                                     // 0xc6233120a8
}
```

显然，使用内建函数`println`来打印地址更加简单。

### 模式3： 将不安全指针转换成`uintptr`，使用`uintptr`值进行算术运算， 然后再转换回来。

例子：

```go
package main

import "fmt"
import "unsafe"

type T struct {
	x bool
	y [3]int16
}

const N = unsafe.Offsetof(T{}.y)
const M = unsafe.Sizeof([3]int16{}[0])

// We use a package-level channel variable to make sure the T
// value used in the main function will be allocated on heap.
var c = make(chan *T, 1)

func main() {
	c <- &T{y: [3]int16{123, 456, 789}}
	t := <-c
	p := unsafe.Pointer(t)
	// "uintptr(p) + N + M + M" stores the address of t.y[2].
	ty2 := (*int16)(unsafe.Pointer(uintptr(p) + N + M + M))
	fmt.Println(*ty2) // 789
}
```

对于这样的一个示例，这种转换没什么实际意义。它只是一个教学用途的示例。

请注意，对于这样的示例，上面转换中的第23行不能分离成两行来写，就像下面例子所展示的那样。请阅读代码中的注释寻找原因。

```go
...
	p := unsafe.Pointer(t)
	// Now the t value becomes unused, its memory may
	// be garbage collected at this time. So the following
	// use of the address of t.y[2] may be invalid!
	addr := uintptr(p) + N + M + M
	ty2 := (*int16)(unsafe.Pointer(addr))
	fmt.Println(*ty2)
}
```

这样的bug非常难以检测，这就是为什么使用不安全指针是危险操作。

如果我们确实想把转换操作分成两行来写，我们应该在使用`t.y[2]`之后调用`runtime.KeepAlive`函数并将指针`p`作为参数传递给该函数。像这样：

```go
...
	p := unsafe.Pointer(t)
	addr := uintptr(p) + N + M + M
	ty2 := (*int16)(unsafe.Pointer(addr))
	fmt.Println(*ty2)
	// This following line assures the memory of the value
	// t will not get garbage collected currently for sure.
	runtime.KeepAlive(p)
}
```

另一个需要注意的细节是，不建议将内存块的结束边界存储在指针中（不管是安全的还是不安全的）。这样做可以防止紧跟在前一个内存块之后的另一个内存块被垃圾回收。

### 模式4： 当调用`syscall.Syscall时，`将不安全指针转换成`uintptr`

通过上面的最后一个模式，我们知道下列代码中的行为是非常危险的。

```go
// Assume this function will not inlined.
func DoSomething(addr uintptr) {
	// read or write values at the passed address ...
}
```

上述函数危险的原因是该函数本身不能保证传递的参数地址的值是否会被垃圾回收掉。实际上，可能会有一些新值已经在传递的参数地址中分配。

然而，`syscall`标准库中的函数`Syscall`的原型定义是如下的：

```go
func Syscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err Errno)
```

那么这个函数是如何保证传递的地址`a1`， `a2`和`a3`的值在函数中还未被垃圾回收呢？实际上该函数并不保证这些。只是会由编译器来保证。这是类似于`syscall.Syscall`这样的函数的特权。

我们可以认为编译器会像下面代码中展示的那样为每个`uinptr`参数都追加一个`runtime.KeepAlive`函数调用:

```go
syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
// Compilers will automatically append the following lines for the above line.
runtime.KeepAlive(unsafe.Pointer(uintptr(fd)))
runtime.KeepAlive(unsafe.Pointer(uintptr(unsafe.Pointer(p))))
runtime.KeepAlive(unsafe.Pointer(uintptr(n)))
```

尽管`syscall.Syscall`函数拥有特权，但是调用这个函数也有一个条件。从不安全指针到`uinptr`的转换必须存在于对此函数的调用的参数的字面值表示中。

例如，如果不能保证`p`在调用后仍在使用，则以下调用是无效的。

```go
u := uintptr(unsafe.Pointer(p))
syscall.Syscall(SYS_READ, uintptr(fd), u, uintptr(n))
```

再次重申，在调用其他函数的时候绝不要使用这种模式。在调用其他以`uinptr`为参数的函数时，我们需要手动的加上`runtime.KeepAlive`函数调用。

### 模式5： 将方法`reflect.Value.Pointer`或`reflect.Value.UnsafeAddr`调用的`uinptr`结果转换成不安全指针

标准库`reflect`中的`Value`类型的方法`Pointer`和`UnsafeAddr`都会返回`uinptr`类型的结果，而不是`unsafe.Pointer`类型的结果。这是一个深思熟虑的设计，因为它避免了需要导入`unsafe`标准库才能将这两种函数的调用结果转换为任何安全的指针类型。

这样的设计则要求在（上面两个方法的）调用后必须立即将调用结果转换成不安全指针。 否则，在一个很小的时间窗口内，在调用结果中存储的地址上分配的值可能会丢失其所有的引用并被垃圾回收。

例如，下面的调用是合法的。

```go
p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
```

另一方面，下列的调用是不合法的。

```go
u := reflect.ValueOf(new(int)).Pointer()
p := (*int)(unsafe.Pointer(u))
```

### 模式6：将一个`reflect.SliceHeader.Data`或`reflect.StringHeader.Data`字段转换为不安全指针和反向转换。

因为在上一小节中提及的原因，`reflect`标准库中的`SliceHeader`和`StringHeader`类型的`Data`字段是通过`uintptr`声明的，而不是通过`unsafe.Pointer`声明的。

将一个字符串的指针转换成`StringHeader`指针是合法的，所以我们可以操作字符串的内部结构。同样的，也可以将一个切片的指针转换成`SliceHeader`指针，所以我们也可以操作一个切片的内部结构。

一个使用`reflect.StringHeader`的例子：

```go
package main

import "fmt"
import "unsafe"
import "reflect"

func main() {
	a := [...]byte{'G', 'o', 'l', 'a', 'n', 'g'}
	s := "Java"
	hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
	hdr.Data = uintptr(unsafe.Pointer(&a))
	hdr.Len = len(a)
	fmt.Println(s) // Golang
	// Now s and a share the same byte sequence,
	// which makes the string s become mutable.
	a[2], a[3], a[4], a[5] = 'o', 'g', 'l', 'e'
	fmt.Println(s) // Golang
}
```

一个使用`reflect.SliceHeader`的例子：

```go
package main

import "fmt"
import "unsafe"
import "reflect"
import "runtime"

func main() {
	bs := []byte("Golang")
	var pa *[2]byte // an array pointer
	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	pa = (*[2]byte)(unsafe.Pointer(hdr.Data))
	runtime.KeepAlive(&bs)
	fmt.Printf("%s\n", pa) // &Go
	pa[1] = 'a'
	fmt.Printf("%s\n", bs) // Galang
}
```

如果最后一行`Printf`行不存在，那么`runtime.KeepAlive`调用是不可或缺的。

通常，我们应该只从一个真实（已经存在）的字符串中获取一个`StringHeader`指针，同样的，也应该只从一个真实（已经存在）的切片中获取一个`SliceHeader`指针。我们不应该反着来，例如从`StringHeader`创建一个字符串，或从`SliceHeader`创建一个切片。例如，以下的代码就是无效的。

```go
// Assume p points to a sequence of byte and
// n is the number of bytes in the sequence.
var hdr reflect.StringHeader
hdr.Data = uintptr(unsafe.Pointer(new([5]byte)))
// Now the just allocated byte array has lose all
// references and it can be garbage collected now.
hdr.Len = 5
s := *(*string)(unsafe.Pointer(&hdr))
```

下面的例子展示了如何将一个字节切片转换成字符串（反向转换），通过使用不安全的方法。不同于使用安全的方法在字符串和切片之间转换，在每次转换中不安全的方法不会为结果字符串分配一个新的底层字节序列。

```go
package main

import "fmt"
import "unsafe"
import "reflect"
import "runtime"

func ByteSlice2String(bs []byte) (str string) {
	sliceHdr := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	strHdr := (*reflect.StringHeader)(unsafe.Pointer(&str))
	strHdr.Data = sliceHdr.Data
	strHdr.Len = sliceHdr.Len
	// This KeepAlive line is essential to make the
	// ByteSlice2String function be always valid
	// when it is provided in custom package.
	runtime.KeepAlive(&bs)
	return
}

func main() {
	bs := []byte{'G', 'o', 'l', 'a', 'n', 'd'}
	s := ByteSlice2String(bs)
	fmt.Println(s) // Goland
	bs[5] = 'g'
	fmt.Println(s) // Golang
}
```

`reflect`标准库中的`SliceHeader`和`StringHeader`类型的[文档](https://golang.org/pkg/reflect/#SliceHeader)是类似的。该文档说明了这两个结构体类型在之后的Go版本中可能会被改变。所以对上述例子而言，在不安全规则未改变的情况下， 上述例子也有可能是非法的。幸运的是，目前可用的两个Go编译器（标准Go编译器和`Gccgo`编译器）都能够识别`reflect`标准库中声明的两种类型表示。

同样的，通过不安全的方法也可能将字符串转换成字节切片。然而，我们应该将结果切片视为不可变的值，并永远不要修改它的元素。

同时，对于标准Go编译器而言，目前的Go版本（Go 1.10），对于将字节切片转出字符串有更加高效的实现方式。

```go
func ByteSlice2String(bs []byte) string {
	return *(*string)(unsafe.Pointer(&bs))
}
```

这是自Go1.10以来支持的`strings`标准库下的`Builder`类型中的`String`方法所采用的实现方式。它使用了上面介绍的第一种模式，而且非常的高效。

## 结束语

从上面的内容中，我们可以看到，对于某些场景，不安全的机制可以帮助我们编写更加高效的Go代码。然而，在使用不安全的机制时，很容易引入一些很微妙的错误。一个有这样微妙错误的程序可能会安全运行很长时间，但是会突然表示异常，甚至在运行一段时间后崩溃。这些错误是很难检测和调试的。

我们应该只在必要的时候使用不安全的机制。我们必须非常谨慎地使用它们。特别是，我们应该遵循上述的各种原则。

同样，我们应该意识到上面介绍的不安全的机制可能会在以后的Go版本中发生变化甚至是非法操作，尽管还没有任何证据显示它将会很快发生变化。如果不安全的机制发生变化，上面所介绍的有效的不安全指针的使用模式可能会变得无效。所以，最好能够在根据现有于不安全的机制下的代码，快速地切换到该功能安全方式的实现。
