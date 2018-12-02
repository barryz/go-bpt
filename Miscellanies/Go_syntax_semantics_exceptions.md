> 译自：[Syntax/Sementics Exceptions in Go](https://go101.org/article/exceptions.html)。

# Go的语法/语义异常

本文将介绍Go中的语法/语义糖和异常。在编程中，为了便捷，语法糖是一种特殊的语义/语法异常（例外）。

## 嵌套函数调用

**基本规则**：

如果一个函数调用的返回结果与另一个函数调用的输入参数完全相同，且返回的结果个数不为零，那么第一个函数可以嵌套在第二个函数中。当嵌套的第二个调用时，第一个调用不能与第二个调用的其他参数混淆。

**语法糖**：

如果一个函数函数的滴啊用结果个数只有一个，那么这个函数就可以嵌套进另一个函数，另一个函数仅需至少有一个参数即可，而且这个单返回值的函数调用可以和另一个函数调用的其他参数混在一起。

**例外**：

基本的规则不适用于有多个入参的内建函数，包括`copy`，`delete`，`print`和`println`函数。这些函数的调用都不能将一个多返回值的函数调用作为参数。

*（基本规则可能[应用在](https://github.com/golang/go/issues/15992)`copy`函数，但[绝不能](https://github.com/golang/go/issues/20656)应用在`print`和`println`函数上）*

基本规则作为一个另外（对于标准Go编译器来说）可以应用在内建函数`append`和`complex`上：

对`complex`函数调用不能将一个接口值接收者作为唯一参数使用。

*（这个例外可能是标准Go编译器的一个bug，到目前为止，Go1.11，这个bug仍然存在。）*

例子：

```go
package main

import (
	"fmt"
)

func f0() float64 {return 1}
func f1() (float64, float64) {return 1, 2}
func f2(float64, float64) {}
func f3(float64, float64, float64) {}

type I interface {m()(float64, float64)}
type T struct{}
func (T) m()(float64, float64) {return 1, 2}

func main() {
	// These lines compile okay.
	f2(f0(), 123)
	f2(f1())
	fmt.Println(f1())
	_ = complex(f1())
	_ = complex(T{}.m())
	f2(I(T{}).m())

	// These lines don't compile.
	/*
	f3(123, f1())
	f3(f1(), 123)
	println(f1())
	_ = complex(I(T{}).m())
	*/
}
```

## 未使用的变量

**基本规则**：

局部变量已声明但未使用。

**例外**：

函数参数和返回值变量，也可以看作是局部变量，但是可以允许不被使用。

## 访问结构体字段

**基本规则**：

指针值不能拥有字段。

**语法糖**：

结构体的指针可以访问该结构体的字段。

例子：

```go
package main

type T struct {
	x int
}

func main() {
	var t T
	var p = &t

	p.x *= 2
	// above line is a sugar for
	(*p).x *= 2
}
```

## 方法接收者

**基本规则**：

在类型`*T`上显式定义的方法不是类型`T`的方法。

**语法糖**：

尽管在类型`*T`上显式定义的方法不是类型`T`的方法，但类型`T`可寻址的值可以调用这些方法。

例子：

```go
package main

type T struct {
	x int
}

func (pt *T) Double() {
	pt.x *= 2
}

func main() {
	var t = T{3}

	t.Double() // t.x == 6 now
	// above line is a sugar for
	(&t).Double()
}
```

## 复合字面值取址

**基本规则**：

字面值不可寻址且不能取址。

**语法糖**：

尽管复合字面值不可寻址，但它们可以显式取址。

例子：

```go
package main

type T struct {
	x int
}

func (pt *T) Double() {
	pt.x *= 2
}

func main() {
	// T{3}.Double() // error: T{3} is not addressable
	                 // The last sugar doesn't work here.

	var pt = &T{3} // sugar: explicitly take address of
	               // unaddressable composite literal.
	pt.Double()
}
```

## 已定义的单级指针的选择器

**基本规则**：

通常来说，选择器可以在已定义的指针类型的值上使用。

**语法糖**：

如果`x`是一个已定义的**一级指针**的一个值，且选择器`(*x).f`是一个合法的选择器，那么`x.f`就是`(*x).f`的一个合法的缩写（语法糖）。

选择器绝不能用于多级指针类型的值上，无论该多级指针的类型是否是已定义的。

**语法糖的例外**

只有当`f`表示的是结构体字段时，该语法糖才合法，如果`f`表示的是方法，那么该语法糖则无效。

例子：

```go
package main

type T struct {
	x int
}

func (T) y() {
}

type P *T
type PP **T // a multi-level pointer type

func main() {
	var t T
	var p P = &t
	var pt = &t   // type of pt is *T
	var ppt = &pt // type of ppt is **T
	var pp PP = ppt
	_ = pp

	_ = (*p).x // valid
	_ = p.x    // legal

	_ = (*p).y // valid
	// _ = p.y // illegal

	// Following ones are all illegal.
	/*
	_ = ppt.x
	_ = ppt.y
	_ = pp.x
	_ = pp.y
	*/
}
```

## 一元运算符的优先级

**基本规则**：

对于任何操作数来说，如果之后的操作符不是[二进制操作符](https://golang.org/ref/spec#Operators)，那么其之后的运算符的优先级高于其之前的运算符。

**语法糖**：

假设操作数`ptr`是一个指针值，在表达式`*ptr++`中，操作符`*`的优先级高于操作符`++`，也就是说，`*ptr++`等价于`(*ptr)++`，在Go中是不支持指针的算术运算的。

例子：

```go
package main

import (
	"fmt"
)

type T struct{}

func (t T) M() *int {
	var v = 123
	return &v
}

func main() {
	var t = &T{}
	var a = *t.M()
	var b = *(t.M())
	// The above two lines are equivalent.
	var c = (*t).M()
	fmt.Printf("%T %T %T\n", a, b, c) // int int *int

	var s = [...]int{0, 1, 2}
	var u = &s[0]
	var v = &(s[0])
	// The above two lines are equivalent.
	var w = (&s)[0]
	fmt.Printf("%T %T %T\n", u, v, w) // *int *int int

	// sugar

	var x = new(int)
	*x++ // *x == 1 now
	// The Above line is a sugar for
	(*x)++
}
```

## 容器元素的可寻址性

**基本规则**：

当容器的元素值可以被覆盖时，该容器的元素值是是可寻址的。

**例外**：

map的元素永远不可被寻址，即使它的元素可以被覆盖。

**语法糖**：

切片的元素永远可以被寻址，尽管切片本身有可能不是可寻址的。

例子：

```go
package main

func main() {
	var s = []int{123, 789}
	var p = &s[0] // ok
	_ = p

	var m = map[string]int{"abc": 123}
	_ = m

	// The exception:
	// p = &m["abc"] // error: cannot take addresses of map elements

	// The sugar:
	f := func() []int { // return results are not addressable
		return []int{0, 1, 2}
	}
	_ = &f()[2] // ok
}
```

## 修改不可寻址的值

**基本规则**：

不可寻址的值不能够被修改。换言之，不可寻址的值不应该以目标值出现在赋值操作中。

**例外**：

尽管map的元素值不可寻址，但它们却可以被修改，并且它们可以以目标值出现在赋值操作中。（但是map的元素不能够被部分修改，它们只能够被整个地覆盖或替换。）

例子：

```go
package main

func main() {
	type T struct {
		x int
	}

	var mt = map[string]T{"abc": {123}}
	mt["abc"] = T{x: 789} // replacement is ok.
	// Partial modification is not allowed.
	/*
	mt["abc"].x = 456
	*/
}
```

## 泛型

**基本规则**：

Go不支持泛型。

**例外**：

通过标准库`builtin`和`unsafe`声明的大多数的内建函数是支持泛型的。在Go中使用各种复合类型的类型组合也可以看作是泛型的一种形式。

## 一个包中的函数名

**基本规则**：

一个包中声明的函数名不能重复。

**例外**：

一个包中可以声明多个`init`函数。

## 函数调用

**基本规则**：

可以通过用户代码调用函数。

**例外**：

`init`函数不能通过用户代码调用。

## 函数作为值使用

**基本规则**：

声明的函数可以当做一个函数值使用。

**例外1**：

通过标准库`builtin`和`unsafe`声明的内建函数没有一个是可以当做函数值的，包括非泛型函数`panic`和`recover`。

**例外2**：

`init`函数也不能被当做函数值。

例子：

```go
package main

import "fmt"
import "unsafe"

func main() {
	// These ones are okay.
	var _ = main
	var _ = fmt.Println

	// These ones fail to compile.
	var _ = panic
	var _ = unsafe.Sizeof
}
```

## 函数调用缺少的返回值

**基本规则**：

函数的返回值可以全部缺席，换言之，函数的返回值可以全部丢弃。

**例外**：

通过标准库`builtin`和`unsafe`声明的内建函数如果有返回值的情况下，那么该函数的返回值不能缺席。

**例外的例外**：

内建函数`copy`和`recover`的返回值可以缺席，尽管这两个函数有返回值。

## 声明的变量

**基本规则**：

声明的变量永远都是可寻址的。

**例外**：

[预先声明的](https://golang.org/pkg/builtin/#pkg-variables)`nil`变量不能被寻址。

所以，`nil`是一个不可变变量。

## 内建函数`copy`和`append`的参数一致性

**基本规则**：

内建函数`copy`和`append`调用中的第二个切片参数的类型必须和第一个参数保持一致。（对于`append`调用来说，假设第二个参数是以`aSlice...`传递的。）

**语法糖**：

如果内建函数`copy`和`append`调用中的第一个参数是`[]byte`，那么第二个参数可以是`string`。（对于`append`调用来说，假设第二个参数是以`aSlice...`传递的。）

例子：

```go
package main

func main() {
	var bs = []byte{1, 2, 3}
	var s = "xyz"

	copy(bs, s)
	// above line is a sugar (and an optimization) for
	copy(bs, []byte(s))

	bs = append(bs, s...)
	// above line is a sugar (and an optimization) for
	bs = append(bs, []byte(s)...)
}
```

## 比较操作

**基本规则**：

Go中大多数类型支持比较操作。

**例外**：

Map，切片和函数类型不支持比较。

**例外的例外**：

Map，切片和函数类型的值可以和`nil`标识符比较。

例子：

```go
package main

func main() {
	var s1 = []int{1, 2, 3}
	var s2 = []int{7, 8, 9}
	//_ = s1 == s2 // error: slice values can't be compared
	_ = s1 == nil // ok
	_ = s2 == nil // ok

	var m1 = map[string]int{}
	var m2 = m1
	// _ = m1 == m2 // error: map values can't be compared
	_ = m1 == nil
	_ = m2 == nil

	var f1 = func(){}
	var f2 = f1
	// _ = f1 == f2 // error: function values can't be compared
	_ = f1 == nil
	_ = f2 == nil
}
```

## 比较两个接口值

**基本规则**：

尝试比较两个具有不可比较类型的动态值的接口值将会panic。

**例外**：

如果两个接口值的动态类型不相同，那么比较这两个接口值将不会panic，尽管他们的动态值是不可比较的。

例子：

```go
package main

func main() {
	var s = []int{1, 2, 3}
	var m = map[string]int{}
	var x interface{} = s
	var y interface{} = m
	var z interface{} = nil
	// The dynamic values of x and y are both uncomparable.
	//_ = x == x // will panic
	//_ = y == y // will panic
	_ = x == z // false, will not panic
	_ = x == y   // false, will not panic
}
```

## 空的复合字面值

**基本规则**：

如果类型`T`支持复合字面值，那么`T{}`就是它的零值。

**例外**：

对于一个map或切片类型`T`来说，`T{}`表示的不是它的零值。它的零值使用`nil`表示。

例子：

```go
package main

import "fmt"

func main() {
	// new(T) will return the address of a zero value of type T

	type T0 struct {
		x int
	}
	fmt.Println( T0{} == *new(T0) ) // true
	type T1 [5]int
	fmt.Println( T1{} == *new(T1) ) // true

	type T2 []int
	fmt.Println( T2{} == nil ) // false
	type T3 map[int]int
	fmt.Println( T3{} == nil ) // false
}
```

## 容器元素的迭代

**基本规则**：

只有容器的元素可以被循环，被迭代的值是这些容器的元素。元素的key/index也会随着每个迭代的元素一并返回。

**例外1**：

如果被循环的容器是字符串，那么该字符串的迭代的字节元素将会被替换为符文（rune）。

**例外2**：

当迭代一个通道时，元素的索引（index）并不会随着每个迭代元素一并返回。

**例外3**：

数组指针也可用于迭代数组元素。

顺便提一下，通过数组指针访问数组元素（`pArray[index]`）；从数组指针中派生切片（`pArray[start: end]`）都是合法的。

## 内建类型

**基本规则**：

通常来说，内建类型没有方法。

**例外**：

内建类型`error`有个`Error() string`方法。

## 可导出的标识符

**基本规则**：

不以Unicode大写字母开头的标识符无法导出。

**例外**：

`builtin`包中的以Unicode小写开头的标识符可以导出。

## 值的类型

**基本规则**：

值有一个类型或默认类型。

**例外**：

无类型的`nil`没有类型或默认类型。

由于此例外的原因，无法将无类型的`nil`值分配给未指定类型的新声明的变量。虽然无类型的`nil`可以用作接口值的动态值，但它没有实现任何接口类型。

```go
package main

import "fmt"

func main() {
	const x = 123 // for untyped constant x, its default type is int.

	fmt.Printf("type of %v is %T \n", x, x) // type of 123 is int

	var y = x // compile okay, the type of y is int
	_ = y
	var _ float64 = x    // compile okay
	var _ int32 = x      // compile okay
	var _ complex128 = x // compile okay

	// var _ = nil // compile error: use of untyped nil

	var v interface{}
	fmt.Printf("type of %v is %T \n", v, v) // type of <nil> is <nil>
	var _ interface{} = v.(interface{}) // will panic
}
```

## 常量值

**基本规则**：

常量值不能被改变。常量值不能分配给变量。

**例外**：

`iota`是一个内建的预声明为`0`的常量，但是它的值不是常量。它的值在一个常量声明的组内从`0`开始以一行一行开始递增，递增的过程发生在编译期间。而且`iota`只能在常量声明中使用。它不能用于变量声明和许多其他普通变量可以使用的地方。

## 缺失可选的返回值`ok`可能的行为

**基本规则**：

`ok`返回值存在与否不会影响程序的行为。

**例外**：

如果一个类型断言操作失败，缺少`ok`可选返回值将使当前的goroutine发生panic。

例子：

```go
package main

func main() {
	var ok bool

	var m = map[int]int{}
	_, ok = m[123] // will not panic
	_ = m[123]     // will not panic

	var c = make(chan int, 2)
	c <- 123
	close(c)
	_, ok = <-c // will not panic
	_ = <-c     // will not panic

	var v interface{} = "abc"
	_, ok = v.(int) // will not panic
	_ = v.(int)     // will panic!
}
```
