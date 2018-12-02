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

存储在接口值的方法表是一个切片。表中的方法查找只是一个切片的索引寻值操作。

（注意，在运行时在`nil`接口上调用方法会导致panic，因为没有已经声明的方法可供调用。）

例子：

```go
package main

import "fmt"

type Filter interface {
	About() string
	Process([]int) []int
}

// A UniqueFilter will remove duplicated numbers.
type UniqueFilter struct{}
func (UniqueFilter) About() string {
	return "remove duplicated numbers"
}
func (UniqueFilter) Process(inputs []int) []int {
	outs := make([]int, 0, len(inputs))
	pusheds := make(map[int]bool)
	for _, n := range inputs {
		if !pusheds[n] {
			pusheds[n] = true
			outs = append(outs, n)
		}
	}
	return outs
}

// A MultipleFilter will only keep numbers which are
// multiples of the MultipleFilter as an int value.
type MultipleFilter int
func (mf MultipleFilter) About() string {
	return fmt.Sprintf("keep multiples of %v", mf)
}
func (mf MultipleFilter) Process(inputs []int) []int {
	var outs = make([]int, 0, len(inputs))
	for _, n := range inputs {
		if n % int(mf) == 0 {
			outs = append(outs, n)
		}
	}
	return outs
}

// With the help of polymorphism, only one "filteAndPrint"
// function is needed.
func filteAndPrint(fltr Filter, unfiltered []int) []int {
	// Call the methods of "fltr" will call the methods
	// of the value boxed in "fltr" actually.
	filtered := fltr.Process(unfiltered)
	fmt.Println(fltr.About() + ":\n\t", filtered)
	return filtered
}

func main() {
	numbers := []int{12, 7, 21, 12, 12, 26, 25, 21, 30}
	fmt.Println("before filtering:\n\t", numbers)

	// Three non-interface values are boxed into three Filter
	// interface slice element values.
	filters := []Filter{
		UniqueFilter{},
		MultipleFilter(2),
		MultipleFilter(3),
	}

	// Each slice element will be assigned to the local variable
	// "fltr" (of interface type Filter) one by one. The value
	// boxed in each element will also be copied into "fltr".
	for _, fltr := range filters {
		numbers = filteAndPrint(fltr, numbers)
	}
}

// Output:
/*
before filtering:
	 [12 7 21 12 12 26 25 21 30]
remove diplicated numbers:
	 [12 7 21 26 25 30]
keep multiples of 2:
	 [12 26 30]
keep multiples of 3:
	 [12 30]
*/
```

在上面的示例中，多态性使得不必为每种过滤器类型编写一个`filteAndPrint`函数。

除了上述优点之外，多态还使包的开发人员可以编写接受任何用户定义类型的可导出函数和方法，只要用户类型实现了导出的函数和方法的接口类型。

实际上，多态性不是语言的基本特征。有其他方法可以实现相同的工作，例如回调函数。但是多态的方式更清晰，更优雅。

## 反射

存储在接口值中的动态类型信息可用于检查和操作由被装箱进接口值中的动态值引用的值。这在编程中被称为反射。

到目前位置（Go1.10），Go还不支持自定义函数和类型的泛型。反射部分地弥补了由于缺乏泛型而造成的不便。

本文将不会介绍[reflect标准库](https://golang.org/pkg/reflect/)中的相关功能。请阅读[reflections in Go](https://go101.org/article/reflection.html)来学习如何使用该包。下面将只介绍Go中内建的反射方法。在Go中，内建的反射操作有类型断言和类型切换来完成。

### 类型断言

Go中涉及到接口值的转换案例有四种：

1. 将一个非接口值转换成接口值，当非接口值必须实现了接口值的类型。

2. 将一个接口值转换成接口值，当源接口值必须实现了目标接口的类型。

3. 将一个接口值转换成非接口值，无论非接口值是否实现了接口值的类型。

4. 将一个接口值转换成接口值，无论源接口值是否实现了目标接口值的类型。

上文已经介绍了头两种转换案例。这两种案例都要求源值类型必须实现了目标接口类型。这类的转换在编译期间就能确定。

这里我们将解释剩下的两种案例。后面这两种案例将会在运行时执行转换，通过使用语法**类型断言**。事实上，这种语法也支持第二种类型的转换案例。

类型断言的表达式形式是`i.(T)`，`i`是一个接口值，而`T`是一个：

- 必须实现了`i`的类型的费接口值，
- 或是任意的接口类型。

在类型断言`i.(T)`中，`i`被称为被断言值，`T`被称作被断言类型。一个类型断言可以成功或失败。

- 如果`T`是一个非接口类型，那么类型断言的目的是获取一个（副本/拷贝）被装箱进接口值的动态值。`T`必须是`i`的动态类型才能确保断言成功。每个这样的转换操作都可以被看作是一个值开箱尝试。

- 如果`T`是一个接口类型，那么类型断言的目的就是使得被装箱进接口值的动态值也被装箱进另一个类型`T`的接口值中。`i`的动态类型必须能转换成接口类型`T`以确保断言成功。换句话说，`i`的动态类型必须实现了接口类型`T`以确保断言成功。

对于大多数的场景，类型断言操作将会产生一个被断言类型的值，类型断言被看作是一个单值表达式。然而，当一个类型断言在值分配中被当做源值表达式使用时，该类型断言会被看作是一个多值表达式，第二个结果值是无类型的布尔值，用来表明类型断言是否成功。

如果一个类型断言失败，那返回结果中的第一项必须是被断言类型的零值，第二项结果（假设它存在）是一个无类型的布尔值`false`。如果类型断言成功，那么第一个结果是被断言接口值的动态值的副本，第二个结果（假设它存在）是一个无类型的布尔值`true`。

如果一个用在单值表达式的类型断言失败，那么将会导致panic。

一个例子（`T`是一个非接口类型）：

```go
package main

import "fmt"

func main() {
	// Compiler will deduce the type of 123 as int.
	var x interface{} = 123

	// Case 1:
	n, ok := x.(int)
	fmt.Println(n, ok) // 123 true
	n = x.(int)
	fmt.Println(n) // 123

	// Case 2:
	a, ok := x.(float64)
	fmt.Println(a, ok) // 0 false

	// Case 3:
	a = x.(float64) // will panic
}
```

另一个例子（`T`是一个接口类型）：

```go
package main

import "fmt"

type Writer interface {
	Write(buf []byte) (int, error)
}

type DummyWriter struct{}
func (DummyWriter) Write(buf []byte) (int, error) {
	return len(buf), nil
}

func main() {
	var x interface{} = DummyWriter{}
	var y interface{} = "abc" // dynamic type is "string"
	var w Writer
	var ok bool

	// DummyWriter implements both Writer and interface{}.
	w, ok = x.(Writer)
	fmt.Println(w, ok) // {} true
	x, ok = w.(interface{})
	fmt.Println(x, ok) // {} true

	// The dynamic type of y is "string", which doesn't
	// implement Writer.
	w, ok = y.(Writer)
	fmt.Println(w, ok) // <nil> false
	w = y.(Writer)     // will panic
}
```

实际上，`i.m(...)`对动态类型为`T`的接口值`i`的方法调用等同于`i.(T).m(...)`的方法调用。

### 类型选择控制流（type switch）

类型语法在Go中可以算是很怪异的。它可以被看作是类型断言的增强版。一个类型选择块的示例：

```go
switch aSimpleStatement; v := x.(type) {
case TypeA:
	...
case TypeB, TypeC:
	...
case nil:
	...
default:
	...
}
```

`aSimpleStatement`，类型选择代码块中可选的一部分。

`case`关键字后面可以跟`nil`标识符，一个或多个类型名称和类型字面值。这些项目中， 同样的项目在类型选择代码块中只能在`case`关键字后面出现一次。

如果`case`关键字后面的类型名或类型字面值不是接口类型，则它必须实现了接口值的接口类型，后面跟`.(type)`。

这里是一个类型切换的例子：

```go
package main

import "fmt"

func main() {
	values := []interface{}{
		456, "abc", true, 0.33, int32(789),
		[]int{1, 2, 3}, map[int]bool{}, nil,
	}
	for _, x := range values {
		// Here, v is declared once, but it denotes
		// different varialbes in different branches.
		switch v := x.(type) {
		case []int: // a type literal
			// The type of v is []int.
			fmt.Println("int slice:", v)
		case string: // one type name
			// The type of v is string.
			fmt.Println("string:", v)
		case int, float64, int32: // multiple type names
			// The type of v is always same as x.
			// In this example, it is interface{}.
			fmt.Println("number:", v)
		case nil:
			// The type of v is always same as x.
			// In this example, it is interface{}.
			fmt.Println(v)
		default:
			// The type of v is always same as x.
			// In this example, it is interface{}.
			fmt.Println("others:", v)
		}
		// Note, each variable denoted by v in the
		// last three branches is a copy of x.
	}
}

// Output:
/*
number: 456
string: abc
others: true
number: 0.33
number: 789
int slice: [1 2 3]
others: map[]
<nil>
*/
```

上面的例子等同于下面的例子：

```go
package main

import "fmt"

func main() {
	values := []interface{}{
		456, "abc", true, 0.33, int32(789),
		[]int{1, 2, 3}, map[int]bool{}, nil,
	}
	for _, x := range values {
		if v, ok := x.([]int); ok {
			fmt.Println("int slice:", v)
		} else if v, ok := x.(string); ok {
			fmt.Println("string:", v)
		} else if x == nil {
			v := x
			fmt.Println(v)
		} else {
			_, isInt := x.(int)
			_, isFloat64 := x.(float64)
			_, isInt32 := x.(int32)
			if isInt || isFloat64 || isInt32 {
				v := x
				fmt.Println("number:", v)
			} else {
				v := x
				fmt.Println("others:", v)
			}
		}
	}
}
```

关于类型切换的一些细节：

- 不同于普通的`switch-case`代码块，`fallthrough`语句不能在类型切换代码块中的分支使用。
- 和普通的`switch-case`代码块一样，如果指定了`default`分支，那这个默认分支可以出现在类型切换代码块中的任何位置。
- 如果我们不关心在类型切换中被断言的值，我们可以使用短形式`switch x.(type) {...}`代替。
- 和普通的`switch-case`代码块一样，零个或多个分支可以出现在类型切换代码块中。一个没有分支的类型切换代码块，`switch a.(type) {}`在运行时是没有任何操作（影响）的。

## 更多关于Go中的接口

### 接口类型嵌入

在Go中，一个接口类型可以嵌入到另一个命名的接口类型中。最后的效果与将嵌入的接口类型指定的方法原型（方法集）在嵌入接口类型的定义体中展开一样。例如，给出以下两个命名的接口类型：

```go
type Ia interface {
	fa()
}

type Ib interface {
	fb()
}
```

然后下面的接口类型：

```go
interface {
	Ia
	Ib
}
```

等同于`Ic`的方法集：

```go
type Ic interface {
	fa()
	fb()
}
```

当目前的Go版本为止（Go 1.10），如果两个接口类型都指定了相同名称的方法原型，则他们不能同时在相同的第三个接口类型中嵌入，尽管这两个方法原型是相同的。例如，下面例子中两个接口类型都是非法的：

```go
interface {
	Ia
	Ic
}

interface {
	Ib
	Ic
}
```

### 涉及的接口值的比较

涉及比较的接口值有两种情况:

1. 非接口值和接口值之间的比较。
2. 两个接口值之间的比较。

对于第一种情况，非接口值的类型必须实现了接口值的类型（假设它是`I`），所以这个非接口值可以转换成（装箱进）一个`I`的接口值。这就意味着一个非接口类型值和接口类型值之间的比较可以转换成两个接口值之间的比较。所以下面仅会解释两个接口值之间的比较。

比较两个接口值实际上是比较它们各自的动态类型和动态值。

比较两个接口值的步骤（使用`==`操作符）：

1. 如果两个接口值其一是一个`nil`接口值，则比较结果取决于另一个接口值是否也是`nil`接口值。
2. 如果两个接口值的动态类型是不同的类型，那么它们比较的接口恒为`false`。
3. 对于两个接口值的动态类型是相同的类型，

	1. 如果相同的类型是一个[不可比较类型](https://go101.org/article/value-conversions-assignments-and-comparisons.html#comparison-rules)，那么将会发生panic。
	2. 否则，比较结果就是这两个接口值动态值的比较结果。

简单来说，两个接口类型只有当它们各自的动态类型和动态值都相等时才相等，且它们的动态值必须是可比较的。

根据上述的规则来看，两个动态值同为`nil`的接口值不一定是相等的。 例子：

```go
package main

import "fmt"

func main() {
	var a, b, c interface{} = "abc", 123, "a"+"b"+"c"
	fmt.Println(a == b) // a case of step 3. Print "false".
	fmt.Println(a == c) // a case of step 3. Print "true".

	var x *int = nil
	var y *bool = nil
	var ix, iy interface{} = x, y
	var i interface{} = nil
	fmt.Println(ix == iy) // false
	fmt.Println(ix == i)  // false
	fmt.Println(ix == i)  // false

	var s []int = nil // []int is an uncomparable type
	i = s
	fmt.Println(i == nil) // a case of step 1. Print "false".
	fmt.Println(i == i)   // a case of step 2. Will panic.
}
```

### 接口值的内部结构

对于官方的Go编译器/运行时来说，空接口和非空接口值是使用两种不同的内部结构来表示的。请阅读[value parts](https://go101.org/article/value-part.html#interface-structure)以获取更多内容。

### 指针动态值和非指针动态值

官方的Go编译器/运行时做了一个优化，使得将指针值装箱进接口值比非指针值装箱进接口值更加高效。对于[小尺寸值](https://go101.org/article/value-copy-cost.html)，这种效率上的差异很小，但是对于大尺寸值，效率差异可能会很大。对于相同的优化，如果基类型是大尺寸类型，则使用指针类型的类型断言也比具有指针类型的基类型的类型断言更有效。

**所以应该避免将大尺寸值装箱进接口值，应该使用指针值代替**。

### `[]T`的值不能直接转换为`[]I`，即使类型`T`实现了接口类型`I`.

例如，有时，我们可能需要将[]string值转换成[]interface类型。不同于其他的编程语言，Go中并没有直接的方式进行这样的转换。我们必须在一个循环中手动地进行转换：

```go
package main

import "fmt"

func main() {
	words := []string{
		"Go", "is", "a", "high",
		"efficient", "language.",
	}

	// The prototype of fmt.Println function is
	// func Println(a ...interface{}) (n int, err error).
	// So words... can't be passed to it as the argument.

	// fmt.Println(words...) // not compile

	// Convert the []string value to []interface{}.
	iw := make([]interface{}, 0, len(words))
	for _, w := range words {
		iw = append(iw, w)
	}
	fmt.Println(iw...) // compiles okay
}
```

## 每个在接口类型中指定的方法都对应一个隐式的函数

对于由接口类型`I`定义的方法集中的每个名为`m`的方法，编译器将隐式地声明一个名为`I.m`的函数，它具有多个参数，第一个参数是类型I的值本身，类似于其他语言中的`this`。额外的参数就是`I.m`中的输入参数。函数调用`I.m(i, ...)`等价于方法调用`i.m(...)`。

例子：

```go
package main

import "fmt"

type I interface {
	m(int)bool
}

type T string
func (t T) m(n int) bool {
	return len(t) > n
}

func main() {
	var i I = T("gopher")
	fmt.Println (i.m(5))                         // true
	fmt.Println (I.m(i, 5))                      // true
	fmt.Println (interface {m(int) bool}.m(i, 5)) // true

	// The following lines compile okay, but will panic at run time.
	I(nil).m(5)
	I.m(nil, 5)
	interface {m(int) bool}.m(nil, 5)
}
```
