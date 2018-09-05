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

### 8.字符串相关转换规则

### 9.`unsafe.Pointer`相关的转换规则

## 值分配规则

## 值比较规则

### 比较的限制

### 两个值是如何被比较的
