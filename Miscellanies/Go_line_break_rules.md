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

  - 一个标识符
  - 一个整型，浮点型，虚数，符文（rune）或者字符串字面值
  - `break`，`continue`，`fallthrough`或`return`关键字之一
  -  `++`，`--`，`)`，`]`或`}`操作符或标点符号之一

-  为了允许复杂语句可以占用一行，在符号`}`或`)`之前可以省略分号。

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
