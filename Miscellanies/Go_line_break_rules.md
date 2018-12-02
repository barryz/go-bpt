> 译自：[Line Break Rules in Go](https://go101.org/article/line-break-rules.html)。

# Go中断行规则

如果你写了足够多的代码，你应该知道在Go编程中我们不能使用随意的代码风格。具体来说，我们不能在任意的字符位置去断行。本文接下来的内容将会列举Go中的断行规则。

## 分号插入规则

实践中我们经常遵循的一个规则就是，我们不应该将[控制流代码块](https://go101.org/article/control-flows.html)之后的花括号`{`放到新的一行内。

例如，下面的`for`循环的代码将会编译失败。

```go
for i := 5; i > 0; i--
	{ // unexpected newline, expecting { after for clause
}
```

如果想编译通过，起始的花括号`{`必须不能放在新行内，想下面这样。

```go
for i := 5; i > 0; i-- {
}
```

然而，上述的规则也会有例外，例如，下面裸的`for`循环代码块也能编译通过。

```go
for
{
  // do something ...
}
```

那么，什么才是Go编程中的基本断行规则呢？在回答这个问题之前，我们应该知道正式的Go语法使用分号`;`作为语句的结束符。然而，我们极少在Go代码中使用分号。原因就是Go代码中大多数的分号是可选的且可以省略。Go编译器在编译时将会为我们自动插入省略的分号。

例如，下面例子中的十个分号都是可选的。

```go
package main;

import "fmt";

func main() {
	var (
		i   int;
		sum int;
	);
	for i < 6 {
		sum += i;
		i++;
	};
	fmt.Println(sum);
};
```

假设上述的程序存储为名为`semicolons.go`的文件，我们可以运行`go fmt semicolons.go`命令来删除该文件中所有不必要的分号。编译器在编译源代码时会自动将删除的分号插入（进内存中）。

那么Go中分号插入的规则是什么？那我们阅读下Go规范中列举的分号规则。

在许多的产品的正式语法中使用分号“;”作为结束符。Go程序会根据下面两条规则可能地省略大多数的分号：

- 当输入被解析成token时，如果该token是下列形式时，在行的最终标记之后，分号会立即自动插入到标记（token）的流中。

    1. 一个标识符

    2. 一个整型，浮点型，虚数，符文（rune）或者字符串字面值

    3. `break`，`continue`，`fallthrough`或`return`关键字之一

    4. `++`，`--`，`)`，`]`或`}`操作符或标点符号之一

- 为了允许复杂语句可以占用一行，在符号`}`或`)`之前可以省略分号。

对于第一条规则列出的场景，当然，我们也可以手动插入分号，就像上面最后一个例子中的分号一样。换言之，这些分号都是可选的。

第二条规则的意思是在一个多项声明语句中在符号`)`之前的分号以及在一个代码块内或类型声明结束符号`}`之前的分号都是可选的。如果没有最后一个分号，那么编译器将会自动插入它们。

第二条规则可以允许我们写出下列合法的代码。

```go
import (_ "math"; "fmt")
var (a int; b string)
const (M = iota; N)
type (MyInt int; T struct{x bool; y int32})
type I interface{m1(int) int; m2() string}
func f() {print("a"); panic(nil)}
```

编译器会为我们自动插入省略的分号，就像下列代码所显示的这样。

```go
var (a int; b string;);
const (M = iota; N;);
type (MyInt int; T struct{x bool; y int32;};);
type I interface{m1(int) int; m2() string;};
func f() {print("a"); panic(nil);};
```

在其他场景中编译器不会插入分号。我们必须在必要的时候手动插入分号。例如，上面示例中每行代码中的第一个分号都是必需的。下面例子中的分号也是必须的。

```go
var a = 1; var b = true
a++; b = !b
print(a); print(b)
```

通过这两条规则，我们知道在`for`关键字之后永远不会自动插入分号。这就是为什么上面例子中的裸`for`循环合法的原因。

分号插入规则的一个后果就是自增或者自减操作必须作为一个语句出现。他们不能作为一个表达式。例如，下面的代码就是不合法的。

```go
func f() {
	a : = 0
	println(a++)
	println(a--)
}
```

上面的代码不合法的原因是因为编译器将代码视为：

```go
func f() {
	a : = 0
	println(a++;)
	println(a--;)
}
```

分号插入规则的另一个后果是：如果我们将方法调用链分解成多个行，则必须要在每个代码行行尾出现符号，所以我们必须要将点号`.`放到每行的行尾。

```go
anObject.
		MethodA().
		MethodB().
		MethodC()
```

我们不能像这样调用方法链

```go
anObject
		.MethodA()
		.MethodB()
		.MethodC()
```

在下面修改的例子中，编译器会在每行的末尾插入一个分号，所以上面的例子和下面的例子是等价的，都是明显不合法的。

```go
anObject;
	.MethodA();
	.MethodB();
	.MethodC();
```

也有很多不恰当的断行例子。如果你认为上述例子中的断行规则难以理解，下面是一个更加简单的规则。这个规则是完整规则的子集。

通常来说，我们应该只在有二元操作，赋值符号，单个点`.`，逗号，分号或任意左括号(`{`，`[`，`{`)之后断行。

分号插入规则能让我们写出更加简洁的代码。同时，他们还可以编写一些合法但是有点怪异的代码，例如，

```go
package main

import "fmt"

func alwaysFalse() bool {return false}

func main() {
	for
	i := 0
	i < 6
	i++ {
		// use i ...
	}

	if x := alwaysFalse()
	!x {
		// do something ...
	}

	switch alwaysFalse()
	{
	case true: fmt.Println("true")
	case false: fmt.Println("false")
	}
}
```

上面三个控制流代码块都是合法的。编译器将会在第9,10,15和20行的行尾插入一个分号。

请注意，上述例子中的`switch-case`代码块将会打印一个`true`来代替`false`。这和下面的例子不一样：

```go
switch alwaysFalse() {
	case true: fmt.Println("true")
	case false: fmt.Println("false")
}
```

如果你使用命令`go fmt`来格式化之前的例子，该命令将会在`alwaysFalse()`调用之后自动追加一个分号，代码会变成如下这样：

```go
switch alwaysFalse();
	{
	case true: fmt.Println("true")
	case false: fmt.Println("false")
}
```

该修改的版本和下面的例子是等价的。

```go
switch alwaysFalse(); true {
case true: fmt.Println("true")
case false: fmt.Println("false")
}
```

这就是为什么它会打印一个`true`。

为你编写的代码经常运行`go fmt`和`go vet`是一个良好的编程习惯。

对于一个十分罕见的情况，分号插入规则也会使一些代码看起来是合法但实则非法的情况。例如，下面两个代码片段都将编译失败。

```go
func fa() {
	var c chan bool
	select {
	case <-c:
	{
		goto A
		A: // compiles okay
	}
	case <-c:
		goto B
		B: // syntax error: missing statement after label
	case <-c:
		goto C
		C: // compiles okay
	}
}
```

```go
func fb() {
	switch 3 {
	case 2:
		goto B
		B: // syntax error: missing statement after label
	case 1:
	{
		goto A
		A: // compiles okay
	}
	case 0:
		goto C
		C: // compiles okay
	}
}
```

上面两个编译报错表明标签定义之后必须要跟随一个语句。但是看上去上述两个例子中的每个标签都没有跟随一个语句。那么为什么只有两个`B:`标签声明处才是非法的？产生这种情况的原因就是，通过上面提到的分号插入规则，编译器会在每个合法的标签声明的结尾处插入一个分号，但不会在两个`B:`标签声明之后插入分号。每个插入的`;`可以被视作一个空语句，这就是为什么`A:`和`C:`标签声明都是合法的。

我们可以为每个`B:`标签声明处手动插入一个分号（一个空语句）来保证编译成功。

因为标签定义也是语句，所以下面的代码也是合法的，尽管标签定义的形式在实践中完全没有意义。

```go
func f() {
	A:B: // has not any usefulness
	for {
		break B
		goto A
	}
}
```

## 逗号（`,`）不会自动插入

在一些包含了多个类似项目的语法形式中，逗号被当作分隔符使用，例如复合字面值，函数参数列表和函数原型中的入参列表、返回值列表等。在这样的语法形式中，如果逗号不是其对应代码行中的最后一个有效字符，那么最后一项之后的最后一个逗号是可选的，否则，逗号就必须存在。

编译器不会在任何情况下自动插入逗号。

例如：下面的代码是合法的。

```go
func f1(a int, b string,) (x bool, y int,) {
	return true, 789
}
var f2 func (a int, b string) (x bool, y int)
var f3 func (a int, b string, // the last comma is required
) (x bool, y int,             // the last comma is required
)
var _ = []int{2, 3, 5, 7, 9,} // the last comma is optional
var _ = []int{2, 3, 5, 7, 9,  // the last comma is required
}
var _ = []int{2, 3, 5, 7, 9}
var _, _ = f1(123, "Go",) // the last comma is optional
var _, _ = f1(123, "Go",  // the last comma is required
)
var _, _ = f1(123, "Go")
```

然而，下面的代码是不合法的，对于编译来说，它将会为代码中的每行插入一个分号，第二行除外。

```go
func f1(a int, b string,) (x bool, y int // error: unexpected newline
) {
	return true, 789
}
var _ = []int{2, 3, 5, 7, 9 // error: unexpected newline
}
var _, _ = f1(123, "Go" // error: unexpected newline
)
```

## 结束语

就像Go中其他特性一样，分号插入规则毁誉参半。有些程序员不喜欢这个规则，因为他们认为这限制了代码风格的自由性。赞扬者则认为这个规则能够使编译速度更快，并且使不同程序员编写的代码保持同一风格，所以很容易理解其他人写的代码。
