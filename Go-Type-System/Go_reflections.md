> 译自[Reflections in Go](https://go101.org/article/reflection.html)。

# Go中的反射

Go语言是对一门对反射支持良好的静态语言。本文将介绍`reflect`标准库中关于反射相关的功能。很多关于反射的细节都会被提及。

在阅读本文之前，读者可以先去阅读[Go类型系统预览](https://go101.org/article/type-system-overview.html)和[Go中的接口](https://go101.org/article/interface.html)这两篇文章，这对理解本文中的反射会很有帮助。

## Go中的反射预览

通过[之前的文章](https://go101.org/article/generic.html)，我们知道Go语言对于自定义类型和函数的泛型缺乏良好的支持。Go中的反射在Go的编程过程中带来了很多动态的功能，这在一定程度上弥补了缺乏自定义泛型的问题（尽管反射的方式的效率不高）。很多标准库，例如`fmt`和`encoding`包，都大量依赖于反射功能。

我们可以通过`reflect`标准库中的`Type`类型和`Value`类型的值来检查Go中的值。下面两小节的内容将会解释如何使用这两个类型的值。

Go反射的设计目的之一是任何非反射的操作也应该可以通过反射方式实现。出于各种原因，到目前为止这个目标并非100％实现（Go 1.10）。然而，很多的非反射的操作都可以通过反射的方式来实现了。我们可以做一些通过非反射方式无法实现的一些操作。以下各节中提到的操作将不能非反射方式且只能通过反射方式实现。

## `reflect.Type`的类型和值

在Go中，我们可以通过调用`reflect.TypeOf`函数从任意一个非接口的值创建一个`reflect.Type`的值。`reflect.TypeOf`函数调用将会转换它唯一的一个`interface{}`类型的参数，然后检索该`interface{}`值的动态类型的信息，并且将该类型信息存储到一个`reflect.Type`类型的返回值当中。我们可以认为`reflect.Type`的返回值表示了`interface{}`动态值的类型。

无法将`reflect.TypeOf`函数调用返回的结果表示为接口类型的`reflect.Type`值。我们不得不使用间接的方法得到这样一个`reflect.Type`值。这中间接的方法将在第二个例子中展示。

`reflect.Type`类型是一个接口类型。它[指定了一些方法](https://golang.org/pkg/reflect/#Type)。我们可以调用这些方法来检查由`reflect.Type`接收者值表示的类型的信息。其中一些方法适用于[各种类型](https://golang.org/pkg/reflect/#Kind)，其中的一些方法只使用于特定类型。为错误类型的类型调用一个指定类型的方法会引起panic。请阅读`reflect`标准库的文档了解更多的详情信息。

例子：

```go
package main

import "fmt"
import "reflect"

func main() {
	type A = [16]int16
	var c <-chan map[A][]byte
	tc := reflect.TypeOf(c)
	fmt.Println(tc.Kind())    // chan
	fmt.Println(tc.ChanDir()) // <-chan
	tm := tc.Elem()
	ta, tb := tm.Key(), tm.Elem()
	fmt.Println(tm.Kind(), ta.Kind(), tb.Kind()) // map array slice
	tx, ty := ta.Elem(), tb.Elem()

	// byte is an alias of uint8
	fmt.Println(tx.Kind(), ty.Kind()) // int16 uint8
	fmt.Println(tx.Bits(), ty.Bits()) // 16 8
	fmt.Println(tx.ConvertibleTo(ty)) // true
	fmt.Println(tb.ConvertibleTo(ta)) // false

	// Slice and map types are uncomparable.
	fmt.Println(tb.Comparable()) // false
	fmt.Println(tm.Comparable()) // false
	fmt.Println(ta.Comparable()) // true
	fmt.Println(tc.Comparable()) // true
}
```

上面的例子中，我们使用`Elem`方法来获取一些容器类型（通道类型，map类型，切片和数组类型）的元素类型。实际上，我们也可以使用这个方法来获取指针类型的基类型。例如：

```go
package main

import "fmt"
import "reflect"

type T []interface{m()}
func (T) m() {}

func main() {
	tp := reflect.TypeOf(new(interface{}))
	tt := reflect.TypeOf(T{})
	fmt.Println(tp.Kind(), tt.Kind()) // ptr slice

	// Get two interface Types indirectly.
	ti, tim := tp.Elem(), tt.Elem()
	fmt.Println(ti.Kind(), tim.Kind()) // interface interface

	fmt.Println(tt.Implements(tim))  // true
	fmt.Println(tp.Implements(tim))  // false
	fmt.Println(tim.Implements(tim)) // false

	// All types implement any blank interface type.
	fmt.Println(tp.Implements(ti))  // true
	fmt.Println(tt.Implements(ti))  // true
	fmt.Println(tim.Implements(ti)) // true
	fmt.Println(ti.Implements(ti))  // true
}
```

我们可以通过反射来获取一个类型的字段类型和方法信息。我们也可以通过反射获取一个函数类型的参数和返回值信息。

```go
package main

import "fmt"
import "reflect"

type F func(string, int) bool
func (f F) Validate(s string) bool {
	return f(s, 32)
}

func main() {
	var x struct {
		n int
		f F
	}
	tx := reflect.TypeOf(x)
	fmt.Println(tx.Kind())     // struct
	fmt.Println(tx.NumField()) // 2
	tf := tx.Field(1).Type
	fmt.Println(tf.Kind())               // func
	fmt.Println(tf.IsVariadic())         // false
	fmt.Println(tf.NumIn(), tf.NumOut()) // 2 1
	fmt.Println(tf.NumMethod())          // 1
	ts, ti, tb := tf.In(0), tf.In(1), tf.Out(0)
	fmt.Println(ts.Kind(), ti.Kind(), tb.Kind()) // string int bool
}
```

除了`TypeOf`函数，我们还可以使用`reflect`标准库中的些其他函数来创建表示某些复合类型的`reflect.Type`值。

```go
package main

import "fmt"
import "reflect"

func main() {
	ta := reflect.ArrayOf(5, reflect.TypeOf(123))
	fmt.Println(ta) // [5]int
	tc := reflect.ChanOf(reflect.SendDir, ta)
	fmt.Println(tc) // chan<- [5]int
	tp := reflect.PtrTo(ta)
	fmt.Println(tp) // *[5]int
	ts := reflect.SliceOf(tp)
	fmt.Println(ts) // []*[5]int
	tm := reflect.MapOf(ta, tc)
	fmt.Println(tm) // map[[5]int]chan<- [5]int
	tf := reflect.FuncOf(
		[]reflect.Type{ta},
		[]reflect.Type{tp, tc},
		false)
	fmt.Println(tf) // func([5]int) (*[5]int, chan<- [5]int)
	tt := reflect.StructOf([]reflect.StructField{
		{Name: "Age", Type: reflect.TypeOf("abc")},
	})
	fmt.Println(tt)            // struct { Age string }
	fmt.Println(tt.NumField()) // 1
}
```

还有很多`reflect.Type`相关的方法在上面的例子中没有被列举出来，读者可以通过`reflect`标准库文档来了解它们。

注意，到目前为止（Go1.10），还有能通过反射的方式创建接口类型的方法。这是一个Go中一个已知的限制。

另外一个限制是，尽管我们可以通过反射创建一个嵌入了其他类型作为匿名字段的结构体类型，该结构体类型可能会也可能不会获取到嵌入类型的方法，并且创建具有匿名字段的结构体类型甚至可能在运行时出现panic。换言之，使用匿名字段创建结构体类型的行为部分依赖于编译器。

第三个限制就是我们不能通过反射来创建一个新的类型。

## `reflect.Value`的类型和值

同样的，我们也可以通过调用`reflect.ValueOf`函数从一个任意的非接口值中创建一个`reflect.Value`的值。`reflect.ValueOf`函数调用将会转换它唯一的一个`interface{}`类型的参数，然后检索该`interface{}`值的动态值（和动态类型）的信息，并且将该信息存储到一个`reflect.Value`类型的返回值当中。我们可以认为`reflect.Value`的返回值表示了该`interface{}`值的动态值的类型。

不能让`reflect.ValueOf`函数调用返回的`reflect.Value`值来表示接口值。我们必须调用其他`reflect.Value`值的一些方法获得`reflect.Value`值来表示接口值。

`reflect.Value`声明了[很多方法](https://golang.org/pkg/reflect/)。我们可以调用这些由`reflect.Value`作为接收者的方法来检查（并操作）表示的值的信息。 其中一些方法适用于所有类型的值，其中一些是指定类型的。为一个错误类型的Go值调用一种指定类型的方法会引起panic。

每个`reflect.Value`值都有个`CanSet`方法，该方法返回一个是否能够被修改（分配的）的由`reflect.Value`表示的Go的值，我们可以调用`reflect.Value`值的`Set`方法来修改该值。请注意，通过`reflect.ValueOf`函数调用直接返回的`reflect.Value`值是只读的。

例子：

```go
package main

import "fmt"
import "reflect"

func main() {
	n := 123
	p := &n
	vp := reflect.ValueOf(p)
	fmt.Println(vp.CanSet(), vp.CanAddr()) // false false
	vn := vp.Elem() // get the value referenced by vp
	fmt.Println(vn.CanSet(), vn.CanAddr()) // true true
	vn.Set(reflect.ValueOf(789)) // <=> vn.SetInt(789)
	fmt.Println(n) // 789
}
```

结构体中非导出字段不能通过反射来修改。

```go
package main

import "fmt"
import "reflect"

func main() {
	var s struct {
		X interface{} // an exported field
		y interface{} // a non-exported field
	}
	vp := reflect.ValueOf(&s)
	// If vp represents a pointer. the following
	// line is equivalent to "vs := vp.Elem()".
	vs := reflect.Indirect(vp)
	// vx and vy both represent interface values.
	vx, vy := vs.Field(0), vs.Field(1)
	fmt.Println(vx.CanSet(), vx.CanAddr()) // true true
	// vy is addressable but not modifiable.
	fmt.Println(vy.CanSet(), vy.CanAddr()) // false true
	vb := reflect.ValueOf(123)
	vx.Set(vb)     // okay, for vx is modifiable.
	// vy.Set(vb)  // will panic, for vy is not modifiable.
	fmt.Println(s) // {123 <nil>}
	fmt.Println(vx.IsNil(), vy.IsNil()) // false true
}
```

`reflect`标准库中还有一些其他的`reflect.Value`相关的方法。这些函数一一对应于内建函数或非反射函数。以下示例演示如何将自定义泛型函数绑定到不同的函数值中。

```go
package main

import "fmt"
import "reflect"

func InvertSlice(args []reflect.Value) (result []reflect.Value) {
	inSlice, n := args[0], args[0].Len()
	outSlice := reflect.MakeSlice(inSlice.Type(), 0, n)
	for i := n-1; i >= 0; i-- {
		element := inSlice.Index(i)
		outSlice = reflect.Append(outSlice, element)
	}
	return []reflect.Value{outSlice}
}

func Bind(p interface{}, f func ([]reflect.Value) []reflect.Value) {
	// Get dynamic Values of interface Values by using the Elem method.
	invert := reflect.ValueOf(p).Elem()
	invert.Set(reflect.MakeFunc(invert.Type(), f))
}

func main() {
	var invertInts func([]int) []int
	Bind(&invertInts, InvertSlice)
	fmt.Println(invertInts([]int{2, 3, 5})) // [5 3 2]

	var invertStrs func([]string) []string
	Bind(&invertStrs, InvertSlice)
	fmt.Println(invertStrs([]string{"Go", "C"})) // [C Go]
}
```

注意，从以上几个例子中，我们应该知道`reflect.Value`类型的`Elem`方法不仅可以用来获取一个指针值引用的值，还可以用来获取一个接口值的动态值。

如果一个`reflect.Value`值表示的是一个函数（方法）值，那么我们调用`reflect.Value`值的`Call`方法将会调用它对应的函数（方法）。

```go
package main

import "fmt"
import "reflect"

type T struct {
	A, b int
}

func (t T) AddSubThenScale(n int) (int, int) {
	return n * (t.A + t.b), n * (t.A - t.b)
}

func main() {
	t := T{5, 2}
	vt := reflect.ValueOf(t)
	vm := vt.MethodByName("AddSubThenScale")
	results := vm.Call([]reflect.Value{reflect.ValueOf(3)})
	fmt.Println(results[0].Int(), results[1].Int()) // 21 9

	neg := func(x int) int {
		return -x
	}
	vf := reflect.ValueOf(neg)
	fmt.Println(vf.Call(results[:1])[0].Int()) // -21
	fmt.Println(vf.Call([]reflect.Value{
		vt.FieldByName("A"), // panic if the field name is "b"
	})[0].Int()) // -5
}
```

请注意，使用`reflect.Value`表示的非导出的结构体字段不能用作反射函数的参数。如果上述例子中的代码`vt.FieldByName("A")`替换为`vt.Fields("b")`，那么程序将会panic。

一个关于通道值的反射例子：

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan string, 2)
	vc := reflect.ValueOf(c)
	vc.Send(reflect.ValueOf("C"))
	succeeded := vc.TrySend(reflect.ValueOf("Go"))
	fmt.Println(succeeded) // true
	succeeded = vc.TrySend(reflect.ValueOf("C++"))
	fmt.Println(succeeded) // false
	fmt.Println(vc.Len(), vc.Cap()) // 2 2
	vs, succeeded := vc.TryRecv()
	fmt.Println(vs.String(), succeeded) // C true
	vs, sentBeforeClosed := vc.Recv()
	fmt.Println(vs.String(), sentBeforeClosed) // Go false
	vs, succeeded = vc.TryRecv()
	fmt.Println(vs.String(), succeeded) // <invalid Value> false
}
```
`TrySend`和`TryRecv`方法对应了一个`case-one-default`[选择控制流代码块](https://go101.org/article/channel.html#select)。

我们可以使用`reflect.Select`函数在运行时模拟有动态的`case`分支的`select`代码块。

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan int, 1)
	vc := reflect.ValueOf(c)
	succeeded := vc.TrySend(reflect.ValueOf(123))
	fmt.Println(succeeded, vc.Len(), vc.Cap()) // true 1 1

	vSend, vZero := reflect.ValueOf(789), reflect.Value{}
	branches := []reflect.SelectCase{
		{Dir: reflect.SelectDefault, Chan: vZero, Send: vZero},
		{Dir: reflect.SelectRecv, Chan: vc, Send: vZero},
		{Dir: reflect.SelectSend, Chan: vc, Send: vSend},
	}
	selIndex, vRecv, sentBeforeClosed := reflect.Select(branches)
	fmt.Println(selIndex)         // 1
	fmt.Println(sentBeforeClosed) // true
	fmt.Println(vRecv.Int())      // 123
	vc.Close()
	// Remove the send case branch this time, for it may cause panic.
	selIndex, _, sentBeforeClosed = reflect.Select(branches[:2])
	fmt.Println(selIndex, sentBeforeClosed) // 1 false
}
```

还有很多和`reflect.Value`相关的方法未在上面的例子中说明，请阅读`reflect`标准库文档获取详情。

## 更多关于反射的细节

### `reflect.DeepEqual(x, y)`和`x == y`的结果可能是不同的

函数`reflect.DeepEqual(x, y)`当两个参数不同时，返回的结果恒为`false`，而即使两个操作的类型不同，`x == y`也可能返回`true`。

第二个区别是具有相同类型的两个指针参数调用`DeepEqual`时，返回的两个指针引用的两个值是否深度相等。所以该调用在这两个指针不相等的情况下可能会返回`true`。

第三个区别是，如果比较的两个参数在同一个循环引用链路内，则`DeepEqual`调用的结果可能不正确。

第四个区别是，函数调用`reflect.DeepEqual(x, y)`通常不会出现panic，而如果两个操作数都是接口值并且它们的动态类型相同且不可比较，则`x == y`将发生panic。

注意，只有当两个参数都是`nil`并且它们的类型相同时，`DeepEqual`调用才永远返回`true`。

例子：

```go
package main

import "fmt"
import "reflect"

func main() {
	type Book struct {page int}
	x := struct {page int}{123}
	y := Book{123}
	fmt.Println(reflect.DeepEqual(x, y)) // false
	fmt.Println(x == y)                  // true

	z := Book{123}
	fmt.Println(reflect.DeepEqual(&z, &y)) // true
	fmt.Println(&z == &y)                  // false

	type T struct{p *T}
	t := &T{&T{nil}}
	t.p.p = t // form a cyclic reference chain.
	fmt.Println(reflect.DeepEqual(t, t.p)) // true
	fmt.Println(t == t.p)                  // false

	var f1, f2 func() = nil, func(){}
	fmt.Println(reflect.DeepEqual(f1, f1)) // true
	fmt.Println(reflect.DeepEqual(f2, f2)) // false

	var a, b interface{} = []int{1, 2}, []int{1, 2}
	fmt.Println(reflect.DeepEqual(a, b)) // true
	fmt.Println(a == b)                  // panic
}
```

### 值`[]MyByte`可以通过反射转换成`[]byte`值

假设`MyByte`类型的底层类型是一个预定义类型`byte`，我们知道Go类型系统禁止`[]MyByte`和`[]byte`之间的转换。但是看起来，`reflect.Value`类型的`Bytes`方法的实现部分违反了这个限制，是可以允许将`[]MyByte`值转换为`[]byte`的。

例子：

```go
package main

import "bytes"
import "fmt"
import "reflect"

type MyByte byte

func main() {
	var mybs = []MyByte{'a', 'b', 'c'}
	var bs []byte

	// bs = []byte(mybs) // this line fails to compile

	v := reflect.ValueOf(mybs)
	bs = v.Bytes() // okay. Violating Go type system.
	fmt.Println(bytes.HasPrefix(bs, []byte{'a', 'b'})) // true

	bs[1], bs[2] = 'r', 't'
	fmt.Printf("%s \n", mybs) // art
}
```

我觉得这个违反没有害处。相反，它会带来一些好处。例如，通过这种特例，我们可以使用`bytes`标准库中的函数来操作`[]MyByte`值。
