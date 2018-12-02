> 译自[Functions in Go](https://go101.org/article/function.html)

# Go中的函数

之前的文章已经解释过[Functions declarations and calls](https://go101.org/article/function-declarations-and-calls.html)。本文将会解释更多关于Go中函数的概念和细节。

实际上，函数是Go中**一等公民**类型中的一种。换句话说，我们可以把函数（不包括内建函数）作为值来是使用。尽管Go是一门静态语言，Go的函数却是非常灵活的。函数的使用方式和许多动态语言中函数的使用方式很相似。

Go中有一些内建函数。这些函数在`builtin`和`unsafe`标准库中被声明。内建函数和自定义函数有一些不同。其中一项就是内建函数支持通用参数（generic parameters），但是自定义函数不支持。下文将会解释两者之间其他不同点。

## 函数签名和函数类型

在Go里，函数是**一等公民**类型中的一种。函数类型的字面值是由关键字`func`和函数签名字面值组成的。一个函数签名由两个类型列表组成，其中一个是入参类型列表，另一个是返回值类型列表。在函数签名中可以声明参数和返回值类型的名称。但是这些名称是什么并不重要。

我们经常使用函数类型字面值作为一个函数签名的字面值。

函数类型（签名）：

```go
func (a int, b string, c string) (x int, y int, z bool)
```

通过文章[Functions declarations and calls](https://go101.org/article/function-declarations-and-calls.html)，我们已经学习到相同类型的连续变量可以放在一起声明。所以上述例子可以改成：

```go
func (a int, b, c string) (x, y int, z bool)
```

因为参数名字和返回值名字是什么在字面值中不重要，所以上面的例子还可以改成：

```go
func (x int, y, z string) (a, b int, c bool)
```

变量名称可以使用空标识符`_`。所以上面的例子和下面的例子也是等价的：

```go
func (_ int, _, _ string) (_, _ int, _ bool)
```

参数名字要么同时出现要么同时不出现。相同的规则在返回值名称中也同样适用。下面的例子和上面几个例子也是等价的：

 ```go
 func (int, string, string) (int, int, bool) // the standard form
func (a int, b string, c string) (int, int, bool)
func (x int, _ string, z string) (int, int, bool)
func (int, string, string) (x int, y int, z bool)
func (int, string, string) (a int, b int, _ bool)
```

上面所有的例子表示的都是未命名的函数类型。

每个参数列表都必须在字面值`()`括号内部，尽管有时候参数列表是空的。如果返回值列表为空，或者只有一个返回值且没有声明返回值名称，那么返回值列表则不需要字面值`()`符号。

```go
// The following three function types are identical.
func () (x int)
func () (int)
func () int

// The following two function types are identical.
func (a int, b string) ()
func (a int, b string)
```

### 可变参数和可变函数类型

函数中的最后一个参数可以是一个可变长度参数。每个函数可以有一个或多个可变参数。要表示一个参数是可变的，可以使用三点符号`...`作为该参数类型的前缀。例子：

```go
func (values ...int64) (sum int64)
func (seperator string, tokens ...string) string
```

拥有可变参数的函数可以被称为可变函数类型。一个可变函数类型和不可变函数类型不相等。

### 函数类型是不可比较类型

之前已经提到过很多次关于函数类型是不可比较类型的话题。通常，函数值是不可比较的。但是如果函数值为`nil`时又是可以比较的。

因为函数值是不可比较的，所以不能使用它们作为map的key。

## 函数原型

一个函数的原型是由函数的名称和签名组成的。它的字面值是由关键字`func`，函数名称和函数签名组成的。

一个函数原型字面值例子：

```go
func Double(n int) (result int)
```

换句话说，一个函数原型除函数体之外的函数声明。一个函数的声明是由函数原型和函数体组成的。

## 可变函数的声明和调用

通用的函数声明和调用在文章[Functions declarations and calls](https://go101.org/article/function-declarations-and-calls.html)已经解释过。这里我们只关注可变函数的声明和调用。

### 可变函数的声明

可变函数的声明和普通函数声明方式类似。注意，可变函数中的可变参数在函数体中会被看作是一个切片。

```go
// Sum and return the input numbers.
func Sum(values ...int64) (sum int64) {
	// The type of values is []int64.
	sum = 0
	for _, v := range values {
		sum += v
	}
	return
}

// An inefficient string concatenation function.
func Concat(seperator string, tokens ...string) string {
	// The type of tokens is []string.
	r := ""
	for i, t := range tokens {
		if i != 0 {
			r += seperator
		}
		r += t
	}
	return r
}
```

下面，对于使用类型为`...T`声明的可变参数，我们将会认为参数的类型是`[]T`。

实际上，标准库`fmt`中的`Print`，`Printf`和`Println`都是可变函数。

```go
func Print(a ...interface{}) (n int, err error)
func Printf(format string, a ...interface{}) (n int, err error)
func Println(a ...interface{}) (n int, err error)
```

它们可变参数类型是`[]interface{}`。 接口类型和值将在后面文章[interfaces in Go](https://go101.org/article/interface.html)中详细解释。

### 调用可变函数

将参数传递给类型为`[]T`的可变参数有两种方式：

1. 传递一个切片作为唯一参数。这个切片是分配给类型`[]T`的值，而且这个切片必须以三点形式`...`结尾。被传递的切片称作可变参数。

2. 传递零个或多个已分配给类型`[]T`的值的参数。这些参数将被赋值（或转换）成类型为`[]T`的新分配的切片元素。新分配的切片被当作真实的可变参数使用。

注意，这两条规则不能混合使用。

调用可变函数的例子：

```go
package main

import "fmt"

func Sum(values ...int64) (sum int64) {
	sum = 0
	for _, v := range values {
		sum += v
	}
	return
}

func main() {
	a0 := Sum()
	a1 := Sum(2)
	a3 := Sum(2, 3, 5)
	// The above three lines are equivalent to
	// the following three lines.
	b0 := Sum([]int64{}...) // <=> Sum(nil...)
	b1 := Sum([]int64{2}...)
	b3 := Sum([]int64{2, 3, 5}...)
	fmt.Println(a0, a1, a3) // 0 2 10
	fmt.Println(b0, b1, b3) // 0 2 10
}
```

另外一个例子：

```go
package main

import "fmt"

func Concat(seperator string, tokens ...string) (r string) {
	for i, t := range tokens {
		if i != 0 {
			r += seperator
		}
		r += t
	}
	return
}

func main() {
	tokens := []string{"Go", "C", "Rust"}
	langsA := Concat(",", tokens...)        // manner 1
	langsB := Concat(",", "Go", "C","Rust") // manner 2
	fmt.Println(langsA == langsB)           // true
}
```

下面的例子不能编译通过，因为混用了上述两条规则：

```go
package main

func Sum(values ...int64) (sum int64) {
	// ...
	return
}

func Concat(seperator string, tokens ...string) (r string) {
	// ...
	return
}

func main() {
	// The following two lines both fail to compile,
	// for the same error: too many arguments in call.
	_ = Sum(2, []int64{3, 5}...)
	_ = Concat(",", "Go", []string{"C", "Rust"}...)
}
```

## 更多函数声明和调用的内容

### 哪些函数可以重复命名（没有命名冲突）

通常，同一个包内的函数名称不能重复。但是有两个例外。

1. 第一个例外，每个包都可以重复声明多个`init`函数。所有`init`函数的原型都是`func init()`。每个`init`函数都会在运行时包被加载时被调用一次（且只会调用一次）。

2. 另一个例外是使用空白标识符`_`作为名称来声明函数，在这种情况下，被声明的函数永远不会被调用。

### 有些函数调用将会在编译期被求值

大多数的函数会在运行时被求值。但是标准库`unsafe`包中的函数都是在编译期被求值。其他一些内建函数的调用， 比如`len`和`cap`则可能会在编译期被求值，更多细节请参考：[evaluate](https://go101.org/article/summaries.html#compile-time-evaluation)。编译期函数调用被求值生成的结果可以分配给常量。

### 所有的函数参数都是通过拷贝传递

让我们再重申一遍，和Go中所有的值分配一样。Go中所有的函数参数都通过拷贝传递。当一个值被拷贝时， [只有直接部分被拷贝](https://go101.org/article/value-part.html#about-value-copy)。

### 没有函数体的函数声明

我们可以在[Go 汇编](https://golang.org/doc/asm)中实现一个函数。通常，Go汇编源代码存储为以`*.a`结尾的文件。一个使用Go汇编实现的函数仍然需要在`*.go`文件中声明，但只有函数原型需要声明在文件内。必须在`*.go`文件中省略函数声明的主体部分（函数体）。

### 某些带结果的函数不需要返回

如果一个函数有返回结果，那么函数体内的最后一条语句必须是[终止语句](https://golang.org/ref/spec#Terminating_statements)。函数体并不要求一定要包含一个返回语句。例如：

```go
func fa() int {
	a:
	goto a
}

func fb() bool {
	for{}
}
```

### 大多数函数调用的结果可以被丢弃

自定义函数的返回结果可以一起被丢弃掉。内建函数的返回结果，除了`recover`和`copy`以外，不能被丢弃，尽管可以将它们分配给空白标识符`_`时忽略返回结果。无法丢弃结果的函数不能用作`defer`函数调用或`goroutine`调用。

### 使用函数调用作为表达式

对于单个返回结果的函数的调用始终可以用作单个值。例如，它可以作为一个参数嵌入到另一个函数调用中，并且还可以用作单一值出现在任何其他的表达式和语句中。

如果不丢弃多返回结果函数的多个返回值。那么调用只能在两个场景中用作多值表达式。

1. 该调用可以在赋值中用作源值。 但是这个调用不能与赋值中的其他源值混合使用。

2. 该调用可以作为参数嵌套在另一个函数调用中。但是这个调用不能与其他参数混在一起。

```go
package main

func HalfAndNegative(n int) (int, int) {
	return n/2, -n
}

func AddSub(a, b int) (int, int) {
	return a+b, a-b
}

func Dummy(values ...int) {}

func main() {
	// These lines compile okay.
	AddSub(HalfAndNegative(6))
	AddSub(AddSub(AddSub(7, 5)))
	AddSub(AddSub(HalfAndNegative(6)))
	Dummy(HalfAndNegative(6))
	_, _ = AddSub(7, 5)

	// These lines fail to compile.
	_, _, _ = 6, AddSub(7, 5)
	Dummy(AddSub(7, 5), 9)
	Dummy(AddSub(7, 5), HalfAndNegative(6))
}
```

注意，使用多值表达式的规则有[例外](https://go101.org/article/exceptions.html#nest-function-calls)。

## 函数值

正如上面提到的，函数类型也是Go类型系统组成部分之一。函数类型的一个值被称作为函数值。函数类型的零值使用预先声明的`nil`表示。

当我们声明一个自定义函数时，实际上我们同时也声明了一个不可变的函数值。这个函数值使用函数名来标识。函数值的类型可以通过省略函数原型字面量中的函数名来表示。

注意，内建函数不能被当作值。`init`函数同样也不能被当作值。

可以像声明的函数一样调用任何函数值。在开始一个新`goroutine`时调用一个`nil`函数会发生致命错误。这个错误不能被捕捉且会导致整个程序崩溃。对于其他的情况，调用一个`nil`函数将会导致一个可恢复的panic， 包括延迟函数的调用。

从[value parts](https://go101.org/article/value-part.html)的文章中，我们知道非零函数值是多部分值。在一个函数值被分配给另一个值之后，这两个函数将共享同一个底层部分。换句话说，这两个函数代表了相同的内部函数对象。调用这两个函数的效果是一样的。

一个例子：

```go
package main

import "fmt"

func Double(n int) int {
	return n + n
}

func Apply(n int, f func(int) int) int {
	return f(n) // the type of f is "func(int) int"
}

func main() {
	fmt.Printf("%T\n", Double) // func(int) int
	// Double = nil // error: Double is immutable.

	var f func(n int) int // default value is nil.
	f = Double
	g := Apply // let compile deduce the type of g
	fmt.Printf("%T\n", g) // func(int, func(int) int) int

	fmt.Println(f(9))         // 18
	fmt.Println(g(6, Double)) // 12
	fmt.Println(g(6, f))      // 12
}
```

实践中，我们经常分配一个匿名函数给函数变量，于是我们就可以多次调用这个匿名函数。

```go
package main

import "fmt"

func main() {
	// This function returns a function (a closure).
	isMultipleOfX := func (x int) func(int) bool {
		return func(n int) bool {
			return n%x == 0
		}
	}

	var isMultipleOf3 = isMultipleOfX(3)
	var isMultipleOf5 = isMultipleOfX(5)
	fmt.Println(isMultipleOf3(6))  // true
	fmt.Println(isMultipleOf3(8))  // false
	fmt.Println(isMultipleOf5(10)) // true
	fmt.Println(isMultipleOf5(12)) // false

	isMultipleOf15 := func(n int) bool {
		return isMultipleOf3(n) && isMultipleOf5(n)
	}
	fmt.Println(isMultipleOf15(32)) // false
	fmt.Println(isMultipleOf15(60)) // true
}
```

Go中所有的函数都可以视为闭包。这就是为什么各种Go函数的用户体验如此统一，且为什么Go函数与动态语言一样类型的原因。
