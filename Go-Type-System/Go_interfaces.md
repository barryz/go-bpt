> 译自[Interfaces in Go](https://go101.org/article/interface.html)。

# Go中的接口

Go中的接口类型是一个特殊的类型。接口类型在Go中扮演了几种重要的角色。首先，接口类型使Go支持值装箱（value boxing）。因此，通过值装箱，反射和多态得到了支持。

本文余下的文章将会介绍接口类型相关的细节。

## 什么是接口类型

一个接口类型指定了一些[方法原型](https://go101.org/article/method.html#method-set)的集合。换句话说，每个接口类型都定义了一个方法集。实际上，我们可以把接口类型看作是一个方法集。对于接口类型中指定的任意一个方法原型，它的名字都不能是空标识符`_`。

接口类型的例子：

```go
// This is an unnamed interface type.
interface {
	About() string
}

// ReadWriter is a defined interface type.
type ReadWriter interface {
	Read(buf []byte) (n int, err error)
	Write(buf []byte) (n int, err error)
}

// Comparable is a named interface alias type.
type Comparable = interface {
	IsEqualTo(another Comparable) bool
}
```

请注意，`ReadWriter`接口类型指定的方法原型中的`error`结果类型是内置接口类型。它的定义如下：

```go
type error interface {
        Error() string
}
```

特别的，一个空方法集的接口类型被称为空接口类型。下面是一个空接口类型的例子：

```go
// An unnamed blank interface type.
interface{}

// Type I is a defined blank interface type.
type I interface{}
```

## 一个类型的方法集

每个类型都有一个关联的[方法集](https://go101.org/article/method.html#method-set%22)。

  - 对于一个接口类型，它的方法集就是它指定的方法原型集合。
  - 对于一个非接口类型，它的方法集就是为其声明的所有方法（显式和隐式声明）的原型集合。

（关于非接口类型的方法，请阅读[methods in Go](https://go101.org/article/method.html)和[type embedding](https://go101.org/article/type-embedding.html)。

为方便起见，类型的方法集通常也称为该类型的任意值的方法集。

两个未命名的接口类型如果他们的方法集相同那么他们本身也是相同的。

## 什么是实现（Implementations）?

对于任意类型`T`的方法集，`T`可能是接口类型，如果是接口类型`I`的方法集的超集，那么我们就认为说类型`T`实现接口`I`。

Go中的实现都是隐式的。不需要在代码中为编译器指定实现需要的关系。Go中没有`Implements`关键字。Go编译器会在需要时自动检查实现的关系。

一个接口类型总是实现了它本身。两个拥有相同方法集的接口类型相互实现了对方。

例如，在下面的例子中，`*Book`结构体指针类型的方法集，整型`*MyInt`和指针类型`*MyInt`都包含了方法原型`About() string`，所以他们全都实现了上面提到的接口类型`interface (About() string)`。

```go
type Book struct {
	name string
	// more other fields ...
}

func (book *Book) About() string {
	return "Book: " + book.name
}

type MyInt int

func (MyInt) About() string {
	return "I'm a custom integer value"
}
```

注意，因为所有的方法集都是空方法集的超集。所以**任何类型都实现了任意的空接口类型**。这是Go中一个重要的事实。

实现的隐式设计使得在其他库的包中定义具体的类型成为可能，例如在标准包中，被动地实现了在用户包中声明的一些接口类型。例如，如果我们定义下面的一个接口类型，那么[标准库中的`database/sql`](https://golang.org/pkg/database/sql/)的类型`DB`和`Tx`将会自动实现这个接口，因为它们都拥有此接口中指定的三个相应的方法。

```go
type DatabaseStorer interface {
	Exec(query string, args ...interface{}) (Result, error)
	Prepare(query string) (*Stmt, error)
	Query(query string, args ...interface{}) (*Rows, error)
}
```

## 值装箱

在Go中，如果类型`T`实现了接口类型`I`，那么类型`T`的任何值都会被隐式地转换成类型`I`。换句话说，类型`T`的任何值都可以[分配](https://go101.org/article/constants-and-variables.html#assignment)给类型`I`的（可寻址的）值。

在这样的转换中（或分配中），

  - 如果类型`T`是非接口类型，则将`T`值的副本装箱（或封装）到结果（或目标）`I`值中。这里拷贝的时间复杂度是`O(n)`，`n`是被拷贝的`T`值的大小。

  - 如果类型`T`也是接口类型，则将`T`值中装箱的值的副本装箱（或封装）到结果（或目标）`I`值中。这里Go编译器会做一个优化，所有时间复杂度是`O(1)`，不是`O(n)`。

装箱值的类型信息也将会存到结果接口值内。

当一个值被装箱进一个接口值时，这个值会被称作该接口值的**动态值**。动态值的类型也被称作是该接口类型的**动态类型**。

一个接口类型的动态类型必须是一个非接口类型。动态值本身是不可变的。我们可以将一个接口值内的动态值替换成另一个动态值，但是没有任何（安全的）方法来修改动态值的任何部分。

有时候动态值和动态类型也被称作是具体值和具体类型。

在Go里，任何接口类型的零值都可以用预先声明的标识符`nil`表示。`nil`接口值没有任何内容。将一个无类型的`nil`分配给一个接口值将会清除该接口值中被装箱的动态值。

（注意，Go中很多非接口类型的零值也使用`nil`表示。非接口`nil`值也可以被装箱进接口值。这种的接口值并不是`nil`接口值。）

因为任何类型都实现了（任何）空接口类型，所以任何非接口的值都可以被装箱进（或分配给）一个空接口值。基于这个原因，空接口类型可以被看作是其他语言中的`any`类型。

当一个无类型的值（除了无类型的`nil`以外）分配给一个空接口值时，这个无类型的值首先会被转换成它的默认值。（或者，我们可以认为无类型值被推导为其默认类型的值。）

让我们看一个示例，这个示例将演示了一些以接口值作为目标值的赋值操作。

```go
package main

import "fmt"

type Aboutable interface {
	About() string
}

// Type *Book implements Aboutable.
type Book struct {
	name string
}
func (book *Book) About() string {
	return "Book: " + book.name
}

func main() {
	// A *Book value is boxed into an Aboutable value.
	var a Aboutable = &Book{"Go 101"}
	fmt.Println(a) // &{Go 101}

	// i is a blank interface value.
	var i interface{} = &Book{"Rust 101"}
	fmt.Println(i) // &{Rust 101}

	// Aboutable implements interface{}.
	i = a
	fmt.Println(i) // &{Go 101}
}
```

请注意，函数`fmt.Println()`的原型是：

```go
func Println(a ...interface{}) (n int, err error)
```

这就是为什么`fmt.Println()`函数可以接收任意类型的参数。

下面的例子将展示如何使用空接口值来装箱任何非接口类型的值。

```go
package main

import "fmt"

func main() {
	var i interface{}
	i = []int{1, 2, 3}
	fmt.Println(i) // [1 2 3]
	i = map[string]int{"Go": 2012}
	fmt.Println(i) // map[Go:2012]
	i = true
	fmt.Println(i) // true
	i = 1
	fmt.Println(i) // 1
	i = "abc"
	fmt.Println(i) // abc

	// Clear the boxed value in interface value i.
	i = nil
	fmt.Println(i) // <nil>
}
```

如上所述，接口值的动态类型的信息也存储在（更准确地来说，是由其引用）接口值中。动态类型信息对于在Go中实现的两个重要功能至关重要。

  - 动态类型信息包括方法表，该方法表存储了由接口类型指定的，并为该接口类型的动态类型声明的所有相应的方法。这个表是Go中实现[多态](https://go101.org/article/interface.html#polymorphism)的关键。

  - 动态类型信息还包括有关动态类型的各种其他信息，例如该动态类型属于什么[kind](https://go101.org/article/type-system-overview.html#type-kinds)，和该动态类型的方法和字段列表，等等。这个信息是Go中实现[反射](https://go101.org/article/interface.html#reflection)的关键。

在运行时，所有类型的信息项目存储在全局区域中。每个接口值只保存对一个类型信息项的引用。每个非接口与接口类型实现的关系对应于一个全局方法表。对标准Go运行时和编译器来说，一个方法表只会在需要的时候才会被创建。

## 多态性

多态性是接口提供的一个关键功能，同时也是Go中的一个重要特性。

当类型`T`的一个非接口值`t`被装箱进类型`I`的接口值`i`时，在接口值`i`上调用由接口类型`I`指定的一个方法实际上将会调用在非接口值`t`上由接口类型`T`声明的对应的方法。换句话说，调用方法`i.m`实际上会调用方法`t.m`。将不同的值装箱进接口值时，接口值的行为会有所不同。这就称为多态性。
