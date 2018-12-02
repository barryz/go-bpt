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

因为任何被嵌入的类型都必须是结构体类型，文章[structs in Go](https://go101.org/article/struct.html)中已经提到了一个可寻址的结构体值可以通过该结构体值的指针访问。所以下面的代码在Go中也是合法的。

```go
func main() {
	var c C
	pc = &c

	// The following 4 lines are equivalent.
	fmt.Println(pc.B.A.x)
	fmt.Println(pc.B.x)
	fmt.Println(pc.A.x)
	fmt.Println(pc.x)

	// The following 4 lines are equivalent.
	pc.B.A.MethodA()
	pc.B.MethodA()
	pc.A.MethodA()
	pc.MethodA()
}
```

同样地，`gaga.PrintName`可以被看作是`gaga.Person.PrintName`的缩写形式。但是，如果我们认为它不是缩写形式也是可以的。毕竟，类型`Singer`确实拥有一个`PrintName` 方法尽管这个方法是隐式声明的(请阅读下下节内容了解细节)。出于类似的原因，选择器`(&gaga).PrintName`和`(gaga).SetAge`也可以看作或不看作是`(&gaga.Person).PrintName`和`(&gaga.Person).SetAge`的缩写形式。

注意，我们仍然可以使用选择器`gaga.SetAge`，但是仅当`gaga`是类型`Singer`的一个可寻址的值时。它仅仅是`(&gaga).SetAge`的语法糖。可以阅读[methods calls](https://go101.org/article/method.html#call)了解更多细节。

在上面的例子中，`c.B.A.x`被称为选择器`c.x`,`c.B.x`和`c.A.x`的完整形式`MethodA`被称为选择器`c.MethodA` ，`c.B.MethodA`和`c.A.MethodA`的完整形式。

如果选择器的完整形式中的每一个中间名称都对应于一个嵌入字段，则选择器中的中间名称的数量被称为选择器的深度。上面两个示例中使用的选择器的深度是2。

## 选择器的隐藏和冲突
对于一个值`x`，它的很多完整形式的选择器都可能有相同的最后项`y`(y是显式声明的字段或方法)，并且这些选择器的每个中间名称都代表了一个嵌入字段。对于这种案例，

- 只有深度最浅的完整形式的选择器可以缩写为`x.y`。换句话说，`x.y`表示了最浅深度的完整形式的选择器。其他完整形式选择器被最小深度的全形式选择器**遮盖**。

- 如果有一个以上的完整形式的选择器具有最小深度，那么这些完整形式选择器中没有一个可以缩短为`x.y`这种形式。我们说这些最小深度的完整形式选择器正在**碰撞**（冲突）。

如果一个方法选择器被另一个方法选择器遮盖，且这两个方法相应的方法签名是相同的，我们可以说第一个方法被另一个方法覆盖。

例如，假设A，B和C是三个已定义的类型。

```go
type A struct {
	x string
}
func (A) y(int) bool {
	return false
}

type B struct {
	y bool
}
func (B) x(string) {}

type C struct {
	B
}
```

那么下例中的代码将不能编译通过。原因是选择器`v1.A.x`和`v1.B.x`冲突了，所以它们不能同时被缩写成`v1.x`。`v1.A.y`和`v1.B.y`也是相同的情况。

```go
var v1 struct {
	A
	B
}

func f1() {
	_ = v1.x
	_ = v1.y
}
```

下面的代码可以编译通过。选择器`v2.C.B.x`被`v2.A.x`遮盖，所以选择器`v2.x`实际上是`v2.A.x`的缩写形式。因为相同的原因，选择器`v2.y`是`v2.A.y`的缩写形式，而不是`v2.C.B.y`的缩写形式。

```go
var v2 struct {
	A
	C
}

func f2() {
	fmt.Printf("%T \n", v2.x) // string
	fmt.Printf("%T \n", v2.y) // func(int) bool
}
```

## 嵌入类型的隐式方法

如上面提到的，类型`Singer`和*`Singer`都有一个`PrintName`方法。并且`*Singer`方法还有一个`SetAge`方法。然而，我们绝没有为这两个类型显式声明这些方法。那么它们（这些方法）从哪里而来？

实际上，

- 对于被嵌入类型的每个方法，如果该方法的选择器既不被其他选择器遮蔽，也不会和其它选择器冲突，那么编译器将会隐式地为被嵌入的类型声明一个相关的方法。因此，编译器还将隐式地为基类型是嵌入类型结构体的未命名指针类型[声明一个对应的方法](https://go101.org/article/method.html#implicit-pointer-methods)。

- 假设结构体类型`S`嵌入了一个命名类型`T`，对于类型`*T`的每个方法，如果该方法的选择器既不被其它选择器遮蔽，也不会和其它选择器冲突，那么编译器将会为类型`*S`隐式声明一个相应的方法。

简单来说，

- 类型`struct{T}`和类型`*struct{T}`将会获取（继承）类型`T`的所有方法。
- 类型`*struct{T}`也将会获取（继承）类型`*T`的所有方法。
- 类型`struct{*T}`和类型`*struct{*T}`将会获取（继承）类型`*T`的所有方法。因为类型`T`的方法集是类型`*T`的方法集的子集。我们也可以说类型`struct{*T}`和类型`*struct{*T}`将会获取（继承）类型`T`的所有方法。

下面是类型`Singer`和类型`*Singer`隐式声明的一些方法。

```go
func (s Singer) PrintName() {
	s.Person.PrintName()
}

func (s *Singer) PrintName() {
	(*s).Person.PrintName()
}

func (s *Singer) SetAge(age int) {
	(&s.Person).SetAge(age) // <=> (&((*s).Person)).SetAge(age)
}
```

通过文章[methods in Go](https://go101.org/article/method.html)中，我们知道我们不能显式地为基类型是未定义结构体类型的未定义结构体类型和未定义的指针类型声明方法。但是通过类型嵌套，这样的未定义类型也可以拥有方法。

另一个展示隐式声明的方法的例子。

```go
package main

import "fmt"
import "reflect"

type F func(int) bool
func (f F) Validate(n int) bool {
	return f(n)
}
func (f *F) Modify(f2 F) {
	*f = f2
}

type B bool
func (b B) IsTrue() bool {
	return bool(b)
}
func (pb *B) Invert() {
	*pb = !*pb
}

type I interface {
	Load()
	Save()
}

func main() {
	PrintTypeMethods := func(t reflect.Type) {
		fmt.Println(t, "has", t.NumMethod(), "methods:")
		for i := 0; i < t.NumMethod(); i++ {
			fmt.Print("   method#", i, ": ", t.Method(i).Name, "\n")
		}
	}

	var s struct {
		F
		*B
		I
	}

	PrintTypeMethods(reflect.TypeOf(s))
	fmt.Println()
	PrintTypeMethods(reflect.TypeOf(&s))
}

// Output:
/*
struct { main.F; *main.B; main.I } has 5 methods:
   method#0: Invert
   method#1: IsTrue
   method#2: Load
   method#3: Save
   method#4: Validate

*struct { main.F; *main.B; main.I } has 6 methods:
   method#0: Invert
   method#1: IsTrue
   method#2: Load
   method#3: Modify
   method#4: Save
   method#5: Validate
*/
```

如果一个结构体嵌套了一个实现了一个接口类型的类型（被嵌套的类型可能是接口类型本身），那么通常结构体类型也实现了接口类型，例外情况是当由接口类型指定的方法和其他方法或字段有遮盖或冲突时。

请注意，类型只会获取其包含的类型的方法（直接或间接嵌入）。例如，在以下代码中，

- 类型`Age`没有方法，因为它没有嵌入任何类型。
- 类型`X`有两个方法，`IsOdd`和`Double`。`IsOdd`是通过嵌套类型`MyInt`获取到的。
- 类型`Y`没有方法，因为它的嵌套类型`Age`没有方法。
- 类型`Z`只有一个方法，`IsOdd`。因为它包含了类型`MyInt`。它没有从类型`X`获取方法`Double`，因为它不包含类型`X`。

```go
type MyInt int
func (mi MyInt) IsOdd() bool {
	return mi%2 == 1
}

type Age MyInt

type X struct {
	MyInt
}
func (x X) Double() MyInt {
	return x.MyInt + x.MyInt
}

type Y struct {
	Age
}

type Z X
```

## 接口类型嵌套接口类型

不仅结构类型可以嵌入其他类型，接口类型也可以。但是接口类型只能嵌入命名接口类型。有关详细信息，请阅读[interfaces in Go](https://go101.org/article/interface.html#embedding)。

## 一个有趣的类型嵌套例子

最后，让我们看一个有趣的例子，这个例子将会产生死循环和栈溢出。如果你已经理解了上面说的[多态性](https://go101.org/article/interface.html#polymorphism)，那么就会非常容易理解它为什么会产生死循环。

```go
package main

type I interface {
	m()
}

type T struct {
	I
}

func main() {
	var t T
	var i = &t
	t.I = i
	i.m() // will call t.m(), then will call i.m() again, ...
}
```
