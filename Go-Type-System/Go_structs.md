> 译自[Structs In Go](https://go101.org/article/struct.html).

# Go中的结构体

和C语言类似， Go也支持结构体。 本文将介绍Go中结构体相关的概念和知识。

## 结构体类型和结构体类型字面量

每个未命名的结构体类型字面量都以一个`struct`关键字开头， 后面跟着包含一系列字段定义的符号`{}`结尾。每个字段的定义由字段名称和类型组成。 结构体类型的字段可以为空， 我们称之为空结构体。

```go
struct {
	title  string
	author string
	pages  int
}
```

上述例子的结构体有三个字段。`title` 和 `author` 字段的类型是 `string`， `pages` 的类型是 `int`。

在某些文章会将字段称为成员变量。

相同类型的字段可以放在一起同时声明：

```go
struct {
	title, author string
	pages         int
}
```

结构体的大小是其所有字段类型的大小之和。 **空结构体类型的大小恒为0**。

声明结构体时， 每个字段都可以声明一个标签。 字段标签是可选的， 字段标签的默认值是空字符串。

```go
struct {
	Title  string `json:"title"`
	Author string `json:"author,omitempty"`
	Pages  int    `json:"pages,omitempty"`
}
```

每个字段标签的用途取决于应用程序。 上述例子中的字段标签 `json` 可以帮助标准库 `encoding/json` 下的相关函数在将结构体编码为JSON文本或将JSON文本解码为结构体的过程中，确定JSON文本中的字段名称。标准库 `encoding/json` 中的相关函数只会对可导出的（大写开头）的字段进行编解码， 如果一个字段没有指定标签， 则JSON文本中相应字段名称和代码中结构体字段名称相同。

（顺便说下，标签中的 `omitempty` 单词表示如果该字段的值为零值或者是空容器值， 则在JSON文本中不会显示该字段。）

不建议将字段标签作为注释。

实践中， 原始的字符串字面量 **\`...\`** 比解释型字符串字面量 `"..."` 更受欢迎。

与C语言不同的是， Go中的结构体不支持联合（union）。

上述的例子中展示的都是未命名的结构体类型，也称为匿名结构体类型。 事实上， 命名结构体类型比匿名结构体更受欢迎。

只有可导出（大写开头）的结构体类型中的可导出的字段才能被其他包导入使用。

字段的标签和字段之间的顺序对于结构体类型来说非常重要。 两个未命名的结构体必须在其具有相同字段声明顺序的前提下才有可能是相等的。 两个字段只有在其具有相同的名称、字段类型、标签的前提下才相等。 需要注意的是： **来自不同包的两个未导出的结构体字段名称始终被视为两个不同的名称**。

## 结构体值字面量和结构体值操作

在Go中， 诸如 `T{...}` 这样的形式，其中T必须是类型字面量或类型名， 这种的被称为**复合字面量**且可以作为其他类型的字面量值， 包括结构体类型或其他容器类型。

给定的一个结构体类型 `S` 并且按顺序为其声明两个字段： `x int` 和 `y bool`, 那么类型 `S` 的零值可以由如下两种类型的复合结构体字面量表示法：

1. `S{0, false}` 这种形式， 没有字段名称，但是要求值的顺序必须和结构体字段顺序一致。

2. `S{x: 0, y: false}`， `S{y: false, x: 0}`，  `S{x: 0}`，  `S{y: false}` 和 `S{}` 在这样的声明形式中，每个字段都是可选的。没有指定的字段的值将会被设置为该类型的零值。但是一旦指定了某个字段， 那么一定需要以 `FieldName: FieldValue` 这种形式指定。 这时候， 字段的顺序无关紧要。在实践中， 零值形式 `S{}` 是最常用的。

如果 `S` 是需要导出给其他包的结构体类型， 那么推荐使用第二种声明形式，因为后期可能会更改 `S` 的相关字段。 这样就会导致第一种方式的声明失效。

当然， 我们也可以使用结构体复合字面量来表示非零结构体值。

对于类型 `S` 的值 `v`，我们可以使用 `v.x` `v.y` 这种形式来访问值 `v`各个字段的值。

例子：

```go
package main

import (
	"fmt"
)

type Book struct {
	title, author string
	pages         int
}

func main() {
	book := Book{"Go 101", "Tapir", 256}
	fmt.Println(book.pages) // 256

	// Create a book value with another form.
	book = Book{} // title and author are both "", pages is 0
	book = Book{author: "Tapir"} // title is "" and pages is 0
	book = Book{author: "Tapir", pages: 256, title: "Go 101"}

	// Initialize a struct value without
	// using composite literals.
	var book2 Book // <=> book2 := Book{}
	book2.title = "Go 202"
}
```

如果字面量中的最后一项和结束符`}`在同一行， 则复合字面量中的`,`可以省略。 否则， 最后一个`,`是必需的。 （详细的规则请参考： [line break rules](https://go101.org/article/line-break-rules.html))。

```go
var _ = Book {
	author: "Go 101",
	pages: 256,
	title: "Go 101", // here, the "," can not be ommitted.
}

// The last "," in the following line can be ommitted.
var _ = Book{author: "Go 101", pages: 256, title: "Go 101",}
```

## 关于结构体值赋值操作

当将一个结构体值赋值给另一个结构体时， 则与逐个分配字段的效果是相同的。

```go
func f() {
	book1 := Book{pages: 300}
	book2 := Book{"Go 101", "Tapir", 256}

	book2 = book1
	// The above line is equivalent to the following three lines.
	book2.title = book1.title
	book2.author = book1.author
	book2.pages = book1.pages
}
```

只有当两个结构体值得类型相同或者两个结构体值得类型具有相同的基本类型（需要考虑字段标签）并且这两个类型中的至少一个是[非定义类型](https://go101.org/article/type-system-overview.html#non-defined-type)时， 才能将两个结构体值分配给彼此。

## 结构体字段的可寻址性

可寻址的结构体的字段的值也是可以寻址的。 反之， 不可寻址的结构体的字段的值也不可以寻址。不可寻址的结构体的字段无法被修改。 所有的复合字面量，包括结构体复合字面量都是不可寻址的。

例子：

```go
package main

import "fmt"

func main() {
	type Book struct {
		Pages int
	}
	var book = Book{} // book is addressable
	p := &book.Pages  // take the address of the "Pages" field
	*p = 123
	fmt.Println(book) // {123}

	/*
	Book{}.Pages = 123 // This line doesn't compile,
	                   // for "Book{}" is unaddressable.
	*/
}
```

## 复合字面量虽然不能寻址， 但却可以拥有地址

通常来说， 只有可寻址的值才能拥有地址。 但是Go中却有一个语法糖，它允许复合字面量拥有地址。

例子：

```go
package main

func main() {
	type Book struct {
		Pages int
	}
	// Book{100} is unaddressable but can be taken address.
	p := &Book{100} // <=> tmp := Book{100}; p := &tmp
	p.Pages = 200
	// The following line fails to compile.
	// The reason is Book{100} is not addressable,
	// so its field, Pages, is also not addressable,
	/*
	_ = &Book{100}.Pages
	*/
}
```

## 像结构体值一样使用结构体指针

不同于C语言， 在Go语言里， 没有类似 `->` 这样的操作符通过结构体指针来访问结构体字段。 在Go里， 结构体字段的访问操作符仍然是点操作符：`.`。

例子：

```go
package main

func main() {
	type Book struct {
		pages int
	}
	book1 := &Book{100} // book1 is a poiner.
	book2 := new(Book)  // book2 is another pointer.
	book2.pages = book1.pages
}
```

## 关于结构体值的比较操作

大多数的结构体类型都是可比较的， 除了那些有[无法比较的类型](https://go101.org/article/type-system-overview.html#types-not-support-comparison)字段的结构体类型。当比较两个相同类型的结构体类型， 将会对其相应的字段进行比较。只有当所有相关的字段都相等的时候， 结构体值才相等。

只有当两个结构体值能够互相赋值的情况下才能够互相比较。

## 关于结构体值的转换

当两个结构体类型的值 `S1` 和 `S2` 共享相同的基本类型时（这里可以忽略字段标签）那么它们可以互相进行转换。 特别地， 如果 `S1` 或 `S2` 是[未定义类型]()且它们的基本类型是相同的， 那么它们之间的转换是隐式的。

下面的代码片段给定了5个结构体类型 `S0` `S1` `S2` `S3` 和 `S4`，

- 类型 `S0` 的值不能转换成其他4种类型， 反之亦然， 因为它们相对应的字段名称不同。

- `S1` `S2` `S3` 和 `S4` 之间可以两两互相转换。

特别地，

- 类型 `S2` 的值可以隐式的转换成类型 `S3`， 反之亦然。

- 类型 `S2` 的值可以隐式的转换成类型 `S4`， 反之亦然。

但是，

- 类型 `S1` 的值必须显式地转换成类型 `S2`， 反之亦然。

- 类型 `S3` 的值可以显式地转换成类型 `S4`， 反之亦然。

```go
package main

type S0 struct {
	y int "foo"
	x bool
}

type S1 = struct { // S1 is a (non-defined) alias type
	x int "foo"
	y bool
}

type S2 = struct { // S2 is a (non-defined) alias type
	x int "bar"
	y bool
}
type S3 S2 // S3 is a defined type
type S4 S3 // S4 is a defined type

var v0, v1, v2, v3, v4 = S0{}, S1{}, S2{}, S3{}, S4{}
func f() {
	v1 = S1(v2); v2 = S2(v1)
	v1 = S1(v3); v3 = S3(v1)
	v1 = S1(v4); v4 = S4(v1)
	v2 = v3; v3 = v2 // the conversions can be implicit
	v2 = v4; v4 = v2 // the conversions can be implicit
	v3 = S3(v4); v4 = S4(v3)
}
```

实际上， 只有当其中一个结构体值可以隐式地转换成另一个结构体类型时， 它们才能够彼此分配（或比较）。

## 匿名结构体

允许使用匿名结构体类型作为另一个结构体类型的字段类型。 匿名结构体字面量同样可以在复合字面量中使用。

一个例子：

```go
var aBook = struct {
	author struct {
		firstName, lastName string
		gender              bool
	}
	title string
	pages int
}{
	author: struct {
		firstName, lastName string
		gender              bool
	}{
		firstName: "Mark",
		lastName: "Twain",
	},
	title: "The Million Pound Note",
	pages: 96,
}
```

通常来说， 一般不推荐使用在复合字面量中使用匿名结构体。

## 关于结构体类型的更多内容

还有一些关于结构体的高级话题，后面的文章： [type embedding](https://go101.org/article/type-embedding.html) 和 [memory layout](https://go101.org/article/memory-layout.html#size-and-padding) 将会提到这些概念。
