> 译自[Code Blocks And Identifier Scopes](https://go101.org/article/blocks-and-scopes.html)。

本文将解释Go中的代码块和声明作用域。

_（请注意，本文中代码块的定义和声明作用域和[Go规范](https://golang.org/ref/spec)中的略微不同。）_

# 代码块

在Go的项目中，有四种代码块种类：

- **全局代码块** 包含了所有项目源代码。
- 每个包有一个**包代码块**，包含了所有源代码，也包括了该包中的import声明。
- 每个文件有一个**文件代码块**，包含了该文件所有的源码，包括该文件中的import声明。
- 通常，每个大括号对`{}`表示了一个**局部代码块**。这种形式的代码块被称为显式的局部代码块。复合字面值中和类型定义中的`{}`不是局部代码块。

有一些关键字可以打开一些隐式的代码块。

- 函数的输入参数和输出结果可以被视为在函数定义时一对大括号`{}`所包围的局部代码块声明，尽管它们在大括号`{}`之外。
- `if`，`switch`和`for`关键字将会打开一个嵌套的局部代码块。其中一个是隐式的，另外的是显式的。显式的局部代码块嵌套在隐式的局部代码块内部。跟随在`if`，`switch`或`for`关键字之后简单语句中以短变量声明形式声明的变量是在隐式代码块中声明的。
- 一个`else`关键字将会打开一个显式的或隐式的代码块，它嵌套在由`if`对应关键字打开的隐式代码块中。如果`else`关键字之后紧跟着另一个`if`关键字，那么由该`else`关键字打开的代码块是隐式的。
- 每个`case`和`default`关键字将打开一个隐式的代码块，它嵌套在由其对应的`switch`或`select`关键字打开的显式代码块内部。

非嵌套在其他局部代码块的局部代码块被称为顶级局部代码块。顶级局部代码块都是函数体。

代码块层级：

- 包代码块嵌套在全局代码块内部。
- 文件代码块同样也是直接嵌套在全局代码块内部，而不是嵌套在包代码块内部。（这个解释和Go规范以及`go/*`标准库中是不同的。）
- 每个顶级局部代码块嵌套在一个包代码块和文件代码块内部。（这个解释与Go规范以及`go/*`标准库中是不同的。）
- 一个非顶级的局部代码块必须嵌套在另一个局部代码块内部。

下图展示了各个代码块之间的层级关系：

![](https://go101.org/article/res/blocks.png)

代码块主要用于解释声明的源代码元素标识符的范围。

## 源代码元素声明的位置

有六种可以被声明的源代码元素：

- 包导入（import）
- 类型
- 常量
- 变量
- 函数
- 标签

标签常用于`break`，`continue`和`goto`语句中。

下表表示了源码元素中哪些代码块可以被直接声明：

||The Universe Block|Package Blocks|File Blocks|Local Blocks|
|-----|-----|-----|-----|-----|
|predeclared(built-in elements)①|Yes||||
|package imports|||Yes||
|types (non built-in)||Yes|Yes|Yes|
|constants (non built-in)||Yes|Yes|Yes|
|variables (non built-in)②||Yes|Yes|Yes|
|functions (non built-in)③||Yes|Yes||
|labels||||Yes|

① 预声明的元素记录在`builtin`包中。

② 包括结构体字段变量。

③ 包括类型方法。

请注意：

- 包导入绝对不能在包代码块和局部代码块中声明。
- 函数不能在局部代码块中声明（匿名函数可以包含在局部代码块内部，但是没有方法可以声明它们）。
- 标签只能在局部代码块中声明。并且包含标签声明和标签引用的最内层函数代码块必须相同。
- 所有隐式的局部代码块中一些情况有特殊要求：

  - 如果跟在`if`，`switch`或`for`关键字后面的是声明语句，那么它必须是一个短变量声明。任何声明都不能直接包含在由这些关键字打开的隐式代码块中。
  - 如果跟在`select`代码块中的`case`关键字后面的是声明语句，那么它必须是一个短变量声明。

- 如果两个标签声明的最里面所包含的功能代码块是相同的，那么这两个标签不能重复。
- 除了标签，如果两个声明的标识符最里面包含的代码块是相同的，那么这两个标识符不能相等。

_（顺便说下，标准库`go/*`认为文件代码块只应该包含包导入声明。）_

在包代码块中声明的源代码元素，但在任意局部代码块之外的元素，被称为包级别的源代码元素。包级别的源代码元素可以是常量，类型，变量或者函数，但不能是标签和包导入。

每个包级别的源代码元素标识符直接包含在包代码块和文件代码块中。

## 已声明的源代码元素的作用域

已声明的标识符的作用域是该标识符的可识别范围。下面是所有的源代码元素标识符的[作用域定义](https://golang.org/ref/spec#Declarations_and_scope)：

- 预定义/内建标识符的作用域是全局代码块。
- 包导入标识符的作用域是包含该标识符的文件代码块。
- 一个表示包级别声明的常量，类型，变量或函数（但不是方法）的标识符的作用域是包代码块。特别地，如果该标识符表示的是一个常量或者变量，那么该标识符声明处不在该标识符的作用域内。
- 一个表示方法接收者，函数参数或返回值变量的标识符的作用域是相应的函数体（局部代码块）。
- 在局部代码块中声明的常量或变量标识符的作用域从其声明语句的末尾开始，并在该代码块最内层的末尾结束。
- 在局部代码块中定义的类型标识符的作用域从其定义处开始，并在该代码块最内层的末尾结束。
- 在局部代码块中声明的类型别名标识符的作用域从其声明处末尾开始，并在该代码块最内层的末尾结束。
- 标签的作用域是包含标签声明的最内层函数的局部代码块的主体，但不包括嵌套在该函数内部的所有匿名函数体。

使用空标识符声明的元素并未真正的声明且没有作用域。

__（预声明的`iota`只能在常量声明中可见。）__

你可能已经注意到局部类型定义和局部变量/常量/别名声明之间声明的标识符的微小差异。这种差异意味着一个已定义的类型在其定义体中可以引用其本身。下面是更加清晰地展示它的例子：

```go
package main

func main() {
	// var v int = v   // error: undefined: v
	// const C int = C // error: undefined: C
	/*
	type T = struct {
		*T // invalid recursive type alias T
	}
	*/

	// Following type definitions are all valid.
	type T struct {
		*T
		x []T
	}
	type A [5]*A
	type S []S
	type M map[int]M
	type F func(F) F
	type Ch chan Ch
	type P *P

	// ...
	var s = make(S, 3)
	s[0] = s
	s = s[0][0][0][0][0][0][0][0]

	var m = M{}
	m[1] = m
	m = m[1][1][1][1][1][1][1][1]

	var p P
	p = &p
	p = ***********************p
}
```

包级别声明和局部声明作用域的不同之处：

```go
package main

// Here the two identifiers at each line are the same one.
// They are all not the predeclcared identifiers.
/*
const iota = iota // error: constant definition loop
var true = true   // error: typechecking loop
*/

var a = b   // can reference variables declared later
var b = 123

func main() {
	// The right identifiers in the next two lines
	// are the predecalred ones.
	const iota = iota // ok
	var true = true   // ok
	_ = true

	// The following lines fail to compile.
	/*
	var c = d // can't reference variables declared later
	var d = 123
	_ = c
	*/
}
```

## 标识符屏蔽（shadowing）

除了标签之外，在外部代码块中声明的标识符可以被嵌套在外部代码块中的代码块中声明的同样的标识符屏蔽。

标签不能被屏蔽。

如果一个标识符被屏蔽了，则它的作用域将会排除掉它所屏蔽的标识符的作用域。

下面是一个有趣的例子。代码中包含了4个已声明的变量`x`。每个在深层次代码块中声明的`x`将会屏蔽在较浅的代码块中声明的`x`。

```go
package main

import "fmt"

var p0, p1, p2, p3, p4, p5 *int
var x = 9999 // x#0

func main() {
	p0 = &x
	var x = 888  // x#1
	p1 = &x
	for x := 70; x < 77; x++ {  // x#2
		p2 = &x
		x := x - 70 //  // x#3
		p3 = &x
		if x := x - 3; x > 0 { // x#4
			p4 = &x
			x := -x // x#5
			p5 = &x
		}
	}

	fmt.Println(*p0, *p1, *p2, *p3, *p4, *p5) // 9999 888 77 6 3 -3
}
```

在某些情况下，当标识符屏蔽遇到短变量声明时，一些新的gophers可能会对短变量声明中的变量是否为新声明的变量感到困惑。以下示例（包含错误的情况）显示了Go中的著名的陷阱。几乎每个gopher都在使用Go的前期陷入过此陷阱。

```go
package main

import "fmt"
import "strconv"

func parseInt(s string) (int, error) {
	n, err := strconv.Atoi(s)
	if err != nil {
		// Some new gophers may think err is an already declared
		// variabled in the following short variable declaration.
		// However, both b and err are new declared here in fact.
		b, err := strconv.ParseBool(s)
		if err != nil {
			return 0, err
		}
		// If exection goes here, a nil error is expected to be
		// returned. But in fact, the outer non-nil error will be
		// returned. The scope of the inner "err" ends at the end
		// of the if clause.
		if b {
			n = 1
		}
	}
	return n, err
}

func main() {
	// Output: 1 strconv.Atoi: parsing "TRUE": invalid syntax
	fmt.Println(parseInt("TRUE"))
}
```

我们可以使用`go vet -shadow`命令列出可能的错误屏蔽的用法。

Go只有[25个关键字](https://go101.org/article/keywords-and-identifiers.html#keyword)。关键字不能用作标识符。Go中许多熟悉的单词并不是关键字，例如`int`，`bool`，`string`，`len`，`cap`，`nil`等。它们仅仅是预定义（内建）的标识符。这些预先声明的标识符在全局代码块中声明，因此自定义标识符可以屏蔽它们。这是一个令人讨厌的例子，其中许多预先声明的标识符都被屏蔽。并且它编译、运行都是正常的。

```go
package main

import (
	"fmt"
)

const len = 3      // shadows the built-in function identifier "len"
var true = 0       // shadows the built-in const identifier "true"
type nil struct {} // shadows the built-in variable identifier "nil"
func int(){}       // shadows the built-in type identifier "int"

func main() {
	fmt.Println("a weird program")
	var print = fmt.Println

	var fmt = [len]nil{{}, {}, {}} // shadows the package import "fmt".
	// var n = len(fmt) // sorry, "len" is a constant.
	var n = cap(fmt)    // use built-in cap function instead, :(

	// the "for" keyword will open one implicit local code block and
	// one explicit local code block.
	for true := 0; true < n; true++ {
		var false = fmt[true] // shadows the built-in const
		                      // the built-in identifier "false".
		var true = true+1 // the new declared "true" variable
		                  // shadows the iteration variable "true".
		// fmt.Println(true, false) // sorry, "fmt" is an array.
		print(true, false)
	}
}

/* output:
a weird program
1 {}
2 {}
3 {}
*/
```

是的，这个例子很极端。它包含许多不良的做法。标识符屏蔽很有用，但请不要滥用它。
