> 译自：[Value Conversion, Assignment And Comparison Rules In Go](https://go101.org/article/value-conversions-assignments-and-comparisons.html)。

# Go中的值转换，分配和比较规则

这篇文章将会列举所有关于Go中值的转换，分配和比较的规则。

注意，Go101系列教程中关于转换的定义和Go规范中不完全一致。Go规范中的转换只意为显式转换，而在Go101教程中，转换意味着显式和隐式的转换。

## 值转换规则

在Go中，如果一个值`v`可以显式地转换成类型`T`，那么该转换可能通过表达式`(T)(v)`来完成。有时候，特别当`T`是一个命名类型（一个已定义的类型或类型别名），上面的表达式可以简写为`T(v)`。

我们需要知道的一个事实是，当一个值`x`可以隐式地转换成类型`T`时，那么也意味着`x`可以显式转换成类型`T`。

### 0.明显转换规则

如果两个类型`T1`和`T2`表示的是相同的类型，那么它们的值可以**隐式**地互相转换。

例如：

- 类型`byte`和`uint8`的值可以相互转换。
- 类型`rune`和`int32`的值可以相互转换。
- 类型`[]byte`和`[]uint8`值可以互相转换。

### 1.底层类型相关的转换规则

给定一个非接口值`x`和非接口类型`T`，假设`x`的类型是`Tx`，

- 如果`Tx`和`T`共享相同的底层类型（忽略结构体标签），那么`x`可以显式地转换成`T`，特别地，如果`Tx`或`T`是一个已定义类型且它们的底层类型相同（考虑结构体标签），那么`x`可以隐式地转换成`T`。

- 如果`Tx`和`T`拥有不同的底层类型，但`Tx`和`T`都是[未定义](https://go101.org/article/type-system-overview.html#non-defined-type)的指针类型且它们的基类型共享相同的底层类型（忽略结构体标签），那么`x`可以（且必须）显式地转换成`T`。

_（注意，从Go1.8开始，这两个**忽略的结构体标签**已经生效。）_

普通例子：

```go
package main

func main() {
	// []int, IntSlice and MySlice share
	// the same underlying type: []int
	type IntSlice []int
	type MySlice  []int

	var s  = []int{}
	var is = IntSlice{}
	var ms = MySlice{}
	var x struct{n int `foo`}
	var y struct{n int `bar`}

	// The two lines both fail to compile.
	/*
	is = ms
	ms = is
	*/

	// Must use explicit conversions here.
	is = IntSlice(ms)
	ms = MySlice(is)
	x = struct{n int `foo`}(y)
	y = struct{n int `bar`}(x)

	// Implicit conversions are okay here.
	s = is
	is = s
	s = ms
	ms = s
}
```

指针例子：

```go
package main

func main() {
	type MyInt int
	type IntPtr *int
	type MyIntPtr *MyInt

	var pi = new(int) // type is *int
	var ip IntPtr = pi // ok, same underlying type

	// var _ *MyInt = pi // can't convert implicitly
	var _ = (*MyInt)(pi) // ok, must explicitly

	// var _ MyIntPtr = pi  // can't convert implicitly
	// var _ = MyIntPtr(pi) // can't convert explicitly
	var _ MyIntPtr = (*MyInt)(pi)  // ok
	var _ = MyIntPtr((*MyInt)(pi)) // ok

	// var _ MyIntPtr = ip  // can't convert implicitly
	// var _ = MyIntPtr(ip) // can't convert explicitly
	var _ MyIntPtr = (*MyInt)((*int)(ip))  // ok
	var _ = MyIntPtr((*MyInt)((*int)(ip))) // ok
}
```

### 2.特定通道的转换规则

如果`Tx`是一个双向通道类型，`T`是一个通道类型，`Tx`和`T`拥有相同的元素类型，且`Tx`或`T`不是一个已定义类型，那么`x`可以隐式地转换成`T`。

例子：

```go
package main

func main() {
	type C chan string
	type C1 chan<- string
	type C2 <-chan string

	var ca C
	var cb chan string

	cb = ca // ok, same underlying type
	ca = cb // ok, same underlying type

	var _, _ chan<- string = ca, cb // ok
	var _, _ <-chan string = ca, cb // ok
	var _ C1 = cb // ok
	var _ C2 = cb // ok

	// var _ = C1(ca) // compile error
	// var _ = C2(ca) // compile error
	var _ = C1((chan<- string)(ca)) // ok
	var _ = C2((<-chan string)(ca)) // ok
}
```

### 3.实现相关的转换规则

给定一个值`x`和一个类型`I`，假设`x`的类型（或默认类型）是`Tx`，如果`Tx`实现了`I`，那么`x`可以隐式地转换成类型`I`。转换的结果是一个类型`I`的接口值，它存储了

- 一个`x`的副本，如果`Tx`是一个非接口类型；
- 一个`x`的动态值的副本，如果`Tx`是一个接口类型。

例子：

```go
package main

type Filter interface {
	Filte(n int) bool
}

type MultipleFilter struct {
	Divisor int
}

func (mf MultipleFilter) Filte(n int) bool {
	return n % mf.Divisor == 0
}

func main() {
	var mf = MultipleFilter{11}
	// mf's type MultipleFilter implements Filter,
	// so here implicit conversion is okay.
	var filter = mf

	// all types implement interface{} type, so any
	// value can be converted to interface{} implicitly.

	var n int = 123
	var b bool = true
	var s string = "abc"

	var _ interface{} = n
	var _ interface{} = b
	var _ interface{} = s
	var _ interface{} = mf

	// interface type Filter also implements interface{}
	var _ interface{} = filter
}
```

标准库的很多函数的参数都是`interface{}`。因为所有的类型都实现了`interface{}`，所以任意的值都可以作为`interface{}`类型的参数传递而不需要显式转换。

### 4.类型断言转换规则

（类型切换可以看作是类型断言的一种特殊形式。）

给定一个接口值`x`和一个非接口类型`T`，假设`x`的接口类型是`Ix`，如果`T`实现了`Ix`，那么`x`可以通过类型断言语法转换成`T`：

- 如果`x`的动态类型和`T`是完全相同的类型，那么转换的结果就是`x`的动态值。如果`ok`的结果出现了，那么该`ok`的值是`true`。

- 否则，如果`ok`结果出现了（返回的`ok`的值将是`false`）那么转换的结果将是一个`nil`接口。如果`ok`结果没出现那么转换操作将会panic。

给定一个接口值`x`和一个接口类型`I`，假设`x`的接口类型是`Ix`，那么`x`可以通过类型断言语法转换成`I`，无论`I`是否实现了`Ix`。

- 如果`x`不是一个`nil`接口且`x`的动态类型实现`I`，那么转换的结果（`I`的一个值）存储了`x`的动态值（副本）。如果有`ok`返回结果，它的值将是`true`。

- 否则，如果`ok`结果出现了（返回的`ok`的值将是`false`）那么转换的结果将是一个`nil`接口。如果`ok`结果没出现那么转换操作将会panic。

```go
package main

type I interface {
	f()
}

type T string
func (T) f(){}

func main() {
	var it interface{} = T("abc")
	var is interface{} = "abc"

	// the two are both okay, for T
	// implements both I and interface{}
	var _ T = it.(T)
	var _ I = it.(I)

	// this one is also okay, for
	// string implements interface{}
	var _ string = is.(string)

	// following 3 compile okay but will
	// all panic at run time.
	// _ = is.(I)
	// _ = is.(T)
	// _ = it.(string)
}
```

## 5.无类型值的转换规则

如果无类型值可以表示为类型`T`的值，则可以将无类型值隐式转换为类型`T`。

例子：

```go
package main

func main() {
	var _ []int = nil
	var _ map[string]int = nil
	var _ chan string = nil
	var _ func()() = nil
	var _ *bool = nil
	var _ interface{} = nil

	var _ int = 123.0
	var _ float64 = 123
	var _ int32 = 1.23e2
	var _ int8 = 1 + 0i
}
```

### 6.常量数值的转换规则

_（这条规则和最后一条规则重叠。）_

转换一个常量将产生一个类型常量结果。

给定一个常量`x`和一个类型`T`，如果`x`可以由类型`T`的值表示，那么`x`可以显式地转换成类型`T`，特别地，如果`x`是一个无类型的或它的类型也是`T`，那么`x`可以隐式地转换成类型`T`。

注意，如果`x`是浮点数常量且`T`是一个浮点数类型，那么`T(x)`的转换结果将通过**IEEE 754**相关规则得到一个舍入的`x`的值。

例子：

```go
package main

func main() {
	const I = 123
	const I1, I2 int8 = 0x7F, -0x80
	const I3, I4 int8 = I, 0.0

	const F = 0.123456789
	const F32 float32 = F
	const F32b float32 = I
	const F64 float64 = F
	const F64b = float64(I3)

	const C1, C2 complex64 = F, I
	const I5 = int(C2)
}
```

### 7.非常量数值转换规则

非常量浮点型和整型值可以显式地转换成任意浮点型和整型类型。

非常量复数类型值可以显式地转换成任意复数类型。

注意：

- 复数非常量值不能够转换成浮点类型和整型。
- 浮点类型和整型值不能够转换成复数类型。
- 在非常量值的转换过程中，数据溢出和结果值舍入是被允许的。我们将一个浮点型的值转换成整型的值，将会丢弃该浮点数的小数部分（截断为零）。

```go
package main

import "fmt"

func main() {
var a, b = 1.6, -1.6 // both are float64
fmt.Println(int(a), int(b)) // 1 -1

var i, j int16 = 0x7FFF, -0x8000
fmt.Println(int8(i), uint16(j)) // -1 32768

var c1 complex64 = 1 + 2i
var _ = complex128(c1)
}
```

### 8.字符串相关转换规则

这里所指的字符串类型即为底层类型是内建字符串类型的类型，整型指的是底层类型是整型的类型。

如果一个值的类型（或默认类型）是整型，那么这个值可以显式地转换成字符串类型。

一个字符串值可以显式的转换成底层类型是`[]byte`的切片类型。（即`[]uint8`），反之亦然。

一个字符串值可以显式的转换成底层类型是`[]rune`的切片类型。（即`[]int32`），反之亦然。

请阅读[strings in Go](https://go101.org/article/string.html#conversions)了解更多关于字符串的细节。

### 9.`unsafe.Pointer`相关的转换规则

上面所说的各种转换规则在`unsafe.Pointer`相关的转换规则时，都可以被打破。

一个任意类型的指针值可以转换成一个底层类型是`unsafe.Pointer`的类型。反之亦然。

一个`uintptr`类型值值可以转换成一个底层类型是`unsafe.Pointer`的类型。反之亦然。

请阅读[type-unsafe pointers in Go](https://go101.org/article/unsafe.html)了解更多细节。

## 值分配（赋值）规则

Go规范内定义：

在一个常量定义中，通常也是一个赋值操作，如果目标的常量值没有给定类型，那么该常量可以被视为源值的别名类型，源值也必须是一个常量值。这意味着源值和目标常量值都必须有相同的默认类型。

除了上述的案例，赋值可被视作为一个隐式的转换。

在一次赋值中，如果目标值是一个未指定类型的新声明的变量，那么这个新变量的类型是

- 源值的类型，如果源值是一个已定义类型的值。
- 源值的默认类型，如果源值是一个未定义类型的常量。

隐式转换规则将在最后一节中的转换规则中列举。

注意，参数传递，通道值发送和接收操作都涉及到值分配（赋值）。

对于大多数的值分配，每个源值的[直接部分](https://go101.org/article/value-part.html)被拷贝到目标值的直接部分，除非目标值是一个接口值。

在一次值分配中，如果目标值是接口值但源值不是，那么源值的直接部分将会被拷贝且拷贝的结果将会存储在目标接口值的动态值内。如果源值不是一个指针值，标准编译器将会特殊对待它。如果源值不是一个指针值，源值的直接部分将会被拷贝且该拷贝的一个指针（隐藏且不可变的）将会存储到目标接口值中。

在一次值分配中，如果源值和目标值类型都是接口值，那么源接口值的动态值的直接部分将会被拷贝且该拷贝结果将会以动态值存储到目标接口值中。标准编译器在此处将进行一些优化，以便始终在分配过程中复制指针。

## 值比较规则

Go规范[说明](https://golang.org/ref/spec#Comparison_operators)：

在任意的比较操作中，第一个操作数必须能够赋值给第二个操作数的类型，反之亦然。

所以，比较规则和赋值规则很相似。换言之，两个值可比较的前提就是其中一个值可以隐式地转换成另一个值的类型。

上述的规则并不能覆盖所有的情况，如果比较的两个操作数都是无类型的常量值会出现什么情况？附加的规则其实也很简单：

- 无类型布尔值可以和无类型布尔值进行比较。
- 无类型数值可以和无类型数值进行比较。
- 无类型字符串值可以和无类型字符串值进行比较。

任何比较操作的结果都是无类型的布尔值。

### 比较的限制

到目前为止（Go1.11），上述的比较规则存在一些限制：

- map，切片和函数类型不支持比较。（但这些类型的值可以和`nil`标识符比较）。
- 任何其字段类型是不可比较类型的结构体类型，元素类型是不可比较的任何数组类型都是不支持比较操作的。
- 裸`nil`标识符不能相互比较。

这里是一些不能比较的类型：

```go
[]int
map[string]bool
func() int
struct{x map[int]int} // one field element type is uncomparable
[5][]int              // array element type is uncomparable
```

比较限制的例子：

```go
package main

// The types of the following variables are all uncomparable.
var s []int
var m map[int]int
var f func()()
var t struct {
	s []int
	n int
}
var a [5]map[int]int

func main() {
	// The following lines fail to compile.
	/*
	_ = s == s
	_ = m == m
	_ = f == f
	_ = t == t
	_ = a == a
	_ = nil == nil
	*/

	// The following lines compile okay.
	_ = s == nil
	_ = m == nil
	_ = f == nil
}
```

### 两个值是如何被比较的

我们假设两个值可以比较的，且它们拥有相同的类型T（如果它们有不同的类型，但是其中一个可以隐式地转换成另一个的类型）。

- 如果`T`是一个结构体类型，那么[两个结构体值相应的字段会逐个进行比较](https://go101.org/article/struct.html#comparison)。
- 如果`T`是一个数组类型，那么[两个数组值相应的元素会逐个进行比较](https://go101.org/article/container.html#comparison)。
- 如果`T`是一个接口类型，请阅读[两个接口值是如何比较的](https://go101.org/article/interface.html#comparison)。
- 如果`T`是一个字符串类型，请阅读[两个字符串值是如何比较的](https://go101.org/article/string.html#comparison)。
- 如果`T`是布尔值，数字或指针（安全的或不安全的），那么将逐个比较两个值在内存中占用的相应字节。仅当所有字节比较结果为真时，这两个值才相等。
- 如果`T`是一个通道类型，如果它们都引用相同的底层通道结构，则这两个通道值相等。

请注意，将两个具有相同无法比较的动态类型的接口进行比较会产生panic。上述的前三种情况下的子比较也可以是接口值比较。因此，对前三种情况的比较也可能产生panic。

这是一个比较中会发生一些panic的例子。

```go
package main

func main() {
	type T struct {
		a interface{}
		b int
	}
	var x interface{} = []int{}
	var y = T{a: x}
	var z = [3]T{}

	// Each of the following line can produce a panic.
	_ = x == x
	_ = y == y
	_ = z == z
}
```
请注意，这两个代码行涉及到`z`的编译，可以在任意版本的GoSDK中使用`go build`和`go install`命令进行编译，但无法通过GoSDK1.9和1.10中的`go run`命令进行编译。这是标准Go编译器1.9和1.10的已知错误。
