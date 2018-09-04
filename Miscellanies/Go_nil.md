> 译自：[nils in Go](https://go101.org/article/nil.html)。

# Go中的`nil`

`nil`在Go中是熟悉且重要的预先声明的标识符。它是很多类型的零值的字面值表示。许多新的Go开发者会将`nil`当作其他语言中的`null`（或`NULL`）。这是部分正确的，但`nil`与其他语言中的`null`（`NULL`）又有很多不同的地方。

## `nil`在Go中是一个预先声明的标识符

你可以直接使用`nil`，而不用先声明它们。

## `nil`可以用来表示很多类型的零值

在Go中，`nil`可以用来表示很多类型的零值：

- 指针类型（包括不安全的指针类型）。
- map类型。
- 切片类型。
- 函数类型。
- 通道类型。
- 接口类型。

换言之，在Go里，`nil`可能会是多个值，或多个类型。

## `nil`没有默认类型

Go中其他预先声明的其他标识符都有个一个默认类型。例如，

- `true`和`false`都是`bool`类型。
- `iota`的默认类型是`int`。

但是`nil`没有默认类型，尽管它有很多的类型。编译器必须有足够多的信息才能从上下文中推导出`nil`类型。 例如：

```go
package main

func main() {
	// This following line doesn't compile.
	/*
	v := nil
	*/

	// There must be sufficient information for compiler
	// to deduce the type of a nil value.
	_ = (*struct{})(nil)
	_ = []int(nil)
	_ = map[int]bool(nil)
	_ = chan string(nil)
	_ = (func())(nil)
	_ = interface{}(nil)

	// This lines are equivalent to the above lines.
	var _ *struct{} = nil
	var _ []int = nil
	var _ map[int]bool = nil
	var _ chan string = nil
	var _ func() = nil
	var _ interface{} = nil
}
```

## `nil`在Go中不是一个关键字

预先声明的`nil`不能被遮蔽（shadowed）。

例如：

```go
package main

import "fmt"

func main() {
	nil := 123
	fmt.Println(nil) // 123

	// The following line will compile error, for
	// nil represents an int value now in this scope.
	/*
	var _ map[string]int = nil
	*/
```

_（顺便说下，其它语言中的`nill`和`NULL`也不是关键字）。_

## 不同类型的`nil`值大小也有可能不同

同一个类型的任何值的内存布局总是相同的。同一个类型的`nil`值也不例外。同一个类型的`nil`值的大小总是和该类型的非零值的大小相等。所以不同类型的零值`nil`所表示的值的大小有可能是不同的。

例如：

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var p *struct{} = nil
	fmt.Println( unsafe.Sizeof( p ) ) // 8

	var s []int = nil
	fmt.Println( unsafe.Sizeof( s ) ) // 24

	var m map[int]bool = nil
	fmt.Println( unsafe.Sizeof( m ) ) // 8

	var c chan string = nil
	fmt.Println( unsafe.Sizeof( c ) ) // 8

	var f func() = nil
	fmt.Println( unsafe.Sizeof( f ) ) // 8

	var i interface{} = nil
	fmt.Println( unsafe.Sizeof( i ) ) // 16
}
```

值的大小取决于编译器和架构，上述例子中打印出的值的大小是在64平台上的。而如果在32位平台上，这些大小将会减半。

对于标准Go编译器来说，同一个种类不同类型的`nil`值总是形同的。例如， `[]int`和`[]string`这两个切片类型的`nil`值总是相同的。

## 两个不同类型的`nil`值可能不能被比较

例如：

```go
// Following lines don't compile.
var _ = (*int)(nil) == (*bool)(nil)         // error: mismatched types.
var _ = (chan int)(nil) == (chan bool)(nil) // error: mismatched types.
```

在Go中，两个不同的可比较类型的值只有在其中一个值可以隐式地转换为另一个的类型的值时才能进行比较。 具体来说，有三种情况可以比较两个不同的可比较类型的值：

1. 两个值之一的类型是另一个值的基类型。

2. 两个值之一的类型实现另一个值的类型（必须是接口类型）。

3. 两个值之一的类型是定向通道类型，另一个是双向通道类型，这两种类型具有相同的元素类型，并且这两种类型其一不是已定义的类型。

`nil`值在上述规则中也不例外。

下例中的代码将会编译通过：

```go
type IntPtr *int
// The underlying of type IntPtr is *int.
var _ = IntPtr(nil) == (*int)(nil)

// Every type in Go implements interface{} type.
var _ = (interface{})(nil) == (*int)(nil)

// Values of a directional channel type can be converted to the
// bidirectional channel type which has the same element type.
var _ = (chan int)(nil) == (chan<- int)(nil)
var _ = (chan int)(nil) == (<-chan int)(nil)
```
