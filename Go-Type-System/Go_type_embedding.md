> 译自：[Type Embedding](https://go101.org/article/type-embedding.html)。


# 类型嵌套

从文章[structs in Go](https://go101.org/article/struct.html)中，我们知道结构体可以有很多个字段。每个字段由一个字段名称和字段类型组成。实际上，有时候，一个字段可以只由一个字段类型构成。这样的字段被称为**嵌套字段**。有些文章会将嵌套字段称为匿名字段。

本文将解释类型嵌入的目的以及类型嵌套中的各种细节。

## 类型嵌套长啥样？

类型嵌套的一个例子：

```go
package main

func main() {
	type S = []int
	var x struct {
		string // a defined non-pointer type
		error  // a defined interface type
		*int   // an unnamed pointer type
		S      // an alias of an unnamed pointer type

	}
	x.string = "Go"
	x.error = nil
	x.int = new(int)
	x.S = []int{}
}
```

上述例子中，一个结构体中有四个嵌套字段。

实际上，每个嵌套字段都有一个隐式的名称。这四个嵌套字段的非限定类型名称是它们的名称或者是它们的基类型名称。例如，上述列子中的四个嵌套字段的各自的名称分别为`string`，`int`，`error`和`S`。

## 哪些类型可以嵌套

下面的类型无法嵌套进结构体：

- 已定义的指针类型。
- 未命名类型，除了未命名的一级指针类型，其基类型是非接口和非定义类型。

其他的类型都可以被嵌套。具体来说，以下类型可以嵌入到结构类型中：

- 一个非指针类型的[命名类型](https://go101.org/article/type-system-overview.html#named-type)。
- 或一个未命名的指针类型，它的基类型是一个命名的非指针非接口类型。

下面的例子列举了哪些类型能够嵌入到结构体中：

```go
type Encoder interface {Encode([]byte) []byte}
type Person struct {name string; age int}
type Alias = struct {name string; age int}
type AliasPtr = *struct {name string; age int}
type IntPtr *int

// These types can be embedded in a struct type.
Encoder
Person
*Person
Alias
*Alias
AliasPtr
int
*int

// These types can't be embedded.
*Encoder         // base type is an interface type
*AliasPtr        // base type is a pointer type
IntPtr           // defined pointer type
*IntPtr          // base type is a pointer type
struct {age int} // unnamed type
map[string]int   // unnamed type
[]int64          // unnamed type
func()           // unnamed type
```

一个结构体中不能有两个重名的字段，匿名字段也是如此。通过嵌入字段命名规则，无法将未命名的指针类型与其基类型一起嵌入到相同的结构类型中。例如， `int`和`*int`不能同时嵌入到结构体中。

一个结构体不能嵌入它本身或它的别名类型。接口类型仅仅可以嵌入命名接口类型。

通常，只有嵌入有字段和方法的类型才有意义，尽管也可以嵌入一些没有任何字段和方法的类型。下文部分将解释原因。

## 类型嵌套的意义是什么

类型嵌套的一个主要的目的是将被嵌套类型的功能扩展到嵌套的类型中，所以我们不需要重新实现这部分（被嵌套类型的）功能。

需要其他面向对象的语言使用继承来完成同样的工作。这两种机制都有其[优点和缺点](https://en.wikipedia.org/wiki/Composition_over_inheritance)。这里，本文将不会讨论哪个是更好的。我们应该知道Go选择了类型嵌入机制，两者之间有很大的不同：

- 如果类型`T`继承另一种类型，则类型`T`获得另一种类型的功能。同时，类型`T`的每个值也可以被视为另一种类型的值。

- 如果类型`T`嵌入另一种类型，则其他类型将成为类型`T`的一部分，类型`T`获得另一种类型的功能，但类型T的任何值都不能被视为另一种类型的值。

下面是一个示例，说明嵌入类型如何扩展嵌入类型的功能的。

```go
package main

import "fmt"

type Person struct {
	Name string
	Age  int
}
func (p Person) PrintName() {
	fmt.Println("Name:", p.Name)
}
func (p *Person) SetAge(age int) {
	p.Age = age
}

type Singer struct {
	Person // extends Person by embedding it
	works  []string
}

func main() {
	var gaga = Singer{Person: Person{"Gaga", 30}}
	gaga.PrintName() // Name: Gaga
	gaga.Name = "Lady Gaga"
	(&gaga).SetAge(31)
	(&gaga).PrintName()   // Name: Lady Gaga
	fmt.Println(gaga.Age) // 31
}
```

从上述例子中可以看到，当嵌入了类型`Person`之后，类型`Singer`取得（继承）了类型`Person`的所有方法和字段，并且类型`*Singer`取得（继承）了类型`*Person`的所有方法。这种结论对吗？下面的章节将会回答这个问题。

请注意，一个`Singer`的值不是一个`Person`的值，下面的例子不能编译：

```go
var gaga = Singer{}
var _ Person = gaga
```

## 嵌入类型是否能获得被嵌入类型的字段？

让我们通过使用`reflect`标准包提供的反射功能列出`Singer`类型的所有字段和方法以及上一个示例中使用的`*Singer`类型的方法：

```go
...
import "reflect"
...
func main() {
	t := reflect.TypeOf(Singer{}) // the Singer type
	fmt.Println(t, "has", t.NumField(), "fields:")
	for i := 0; i < t.NumField(); i++ {
		fmt.Print("   field#", i, ": ", t.Field(i).Name, "\n")
	}
	fmt.Println(t, "has", t.NumMethod(), "methods:")
	for i := 0; i < t.NumMethod(); i++ {
		fmt.Print("   method#", i, ": ", t.Method(i).Name, "\n")
	}

	pt := reflect.TypeOf(&Singer{}) // the *Singer type
	fmt.Println(t, "has", pt.NumMethod(), "methods:")
	for i := 0; i < pt.NumMethod(); i++ {
		fmt.Print("   method#", i, ": ", pt.Method(i).Name, "\n")
	}
}

// Output:
/*
main.Singer has 2 fields:
   field#0: Person
   field#1: works
main.Singer has 1 methods:
   method#0: PrintName
main.Singer has 2 methods:
   method#0: PrintName
   method#1: SetAge
*/
```

从结果中我们可以看到，类型`Singer`拥有了`PrintName`方法，类型`*Singer`拥有两个方法，`PrintNamehe`和`SetAge`。但是类型`Singer`并不拥有`Name`字段。那么为什么选择器表达式`gaga.Name`可以合法用于`Singer`值`gaga`？请阅读下一节以了解原因

## 选择器的缩写形式

从文章[structs in Go](https://go101.org/article/struct.html)和[methods in Go](https://go101.org/article/method.html)中，我们已经学习到，对于一个值`x`，`x.y`被称作为一个选择器。这里的`y`是一个字段名和方法名。如果`y`是一个字段名，那么`x`必须是一个结构体值或者是一个指针值。一个选择器是一个表达式，用来表示一个值。如果选择器`x.y`表示的是一个字段，它也可以有自己的字段（如果`x.y`是结构体值）和方法。例如`x.y.z`，`z`在这里可以是字段名或者方法名。

Go中有一个语法糖，*如果选择器的中间名称对应了嵌入字段，则可以从选择器中省略该名称*。这就是为什么嵌入字段又称为匿名字段的原因。

例如：

```go
package main

type A struct {
	x int
}
func (a A) MethodA() {}

type B struct {
	A
}
type C struct {
	B
}

func main() {
	var c C

	// The following 4 lines are equivalent.
	_ = c.B.A.x
	_ = c.B.x
	_ = c.A.x
	_ = c.x

	// The following 4 lines are equivalent.
	c.B.A.MethodA()
	c.B.MethodA()
	c.A.MethodA()
	c.MethodA()
}
```

这也就是为什么上节中表达式`gaga.Name`合法的原因。这个表达式仅仅是`gaga.Person.Name`缩写形式。

 
