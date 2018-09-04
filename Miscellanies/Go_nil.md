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

## 两个不同类型的`nil`值可能不可比较

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

## 相同类型的两个`nil`值也可能不可比较

在Go中，map，切片和函数类型不支持比较。所以尝试比较两个不可比较类型的`nil`值是不合法的。

```go
// The following lines fail to compile.
var _ = ([]int)(nil) == ([]int)(nil)
var _ = (map[string]int)(nil) == (map[string]int)(nil)
var _ = (func())(nil) == (func())(nil)
```

但是上述不可互相比较的类型却可以和`nil`作比较。

```go
// The following lines compile okay.
var _ = ([]int)(nil) == nil
var _ = (map[string]int)(nil) == nil
var _ = (func())(nil) == nil
```

## 两个`nil`值可能不相等

如果两个被比较的`nil`值其一是有一个接口类型，而另外一个不是，假设他们是可比较的，那么它们的比较结果总为`false`。原因是非接口值在执行比较操作之前会被转换成接口值的类型。转换后的接口值具有具体的动态类型，但是其他的接口值没有。这就是为什么比较结果总为`false`。

例子：

```go
fmt.Println( (interface{})(nil) == (*int)(nil) ) // false
```

## 从一个`nil`map中检索元素不会panic

从一个`nil`的map中检索元素总会返回一个零元素值。

例如：

```go
fmt.Println( (map[string]int)(nil)["key"] ) // 0
fmt.Println( (map[int]bool)(nil)[123] )     // false
fmt.Println( (map[int]*int64)(nil)[123] )   // <nil>
```

## 迭代`nil`通道、maps，切片和数组指针是合法的

迭代`nil`map和切片的循环次数为零。

迭代一个`nil`的数组指针的循环次数等于该指针相应数组的长度。（然而，如果相应的数组长度不为零，且第二次迭代不能被忽略或省略，那么该迭代将会在运行时panic。）

迭代一个`nil`通道将会永久阻塞。

例如，下面的代码将会打印 `0`，`1`，`2`，`3`和`4`，然后永久阻塞。`Hello`，`world`和`Bye`将永远不会被打印出来。

```go
for range []int(nil) {
	fmt.Println("Hello")
}

for range map[string]string(nil) {
	fmt.Println("world")
}

for i := range (*[5]int)(nil) {
	fmt.Println(i)
}

for range chan bool(nil) { // block here
	fmt.Println("Bye")
}
```

## 通过非接口`nil`参数调用方法不会panic

例子：

```go
package main

type Slice []bool

func (s Slice) Length() int {
	return len(s)
}

func (s Slice) Modify(i int, x bool) {
	s[i] = x // panic if s is nil
}

func (p *Slice) DoNothing() {
}

func (p *Slice) Append(x bool) {
	*p = append(*p, x) // panic if p is nil
}

func main() {
	// The following selectors will not cause panics.
	_ = ((Slice)(nil)).Length
	_ = ((Slice)(nil)).Modify
	_ = ((*Slice)(nil)).DoNothing
	_ = ((*Slice)(nil)).Append

	// The following two lines will also not panic.
	_ = ((Slice)(nil)).Length()
	((*Slice)(nil)).DoNothing()

	// The following two lines will panic. But panics will
	// not be triggered at the time of invoking the methods.
	// It will be triggered in the method bodies.
	/*
	((Slice)(nil)).Modify(0, true)
	((*Slice)(nil)).Append(true)
	*/
}
```

## 如果类型`T`的零值表示为`nil`，则`*new(T)`等于`nil`

例子：

```go
package main

import "fmt"

func main() {
	fmt.Println(*new(*int) == nil)         // true
	fmt.Println(*new([]int) == nil)        // true
	fmt.Println(*new(map[int]bool) == nil) // true
	fmt.Println(*new(chan string) == nil)  // true
	fmt.Println(*new(func()) == nil)       // true
	fmt.Println(*new(interface{}) == nil)  // true
}
```
