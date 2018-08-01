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

有些文章会将字段称为成员变量。

相同类型的字段可以放在一起同时声明：

```go
struct {
	title, author string
	pages         int
}
```

结构体的大小是其所有字段类型的大小之和。 空结构体类型的大小恒为0。

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

将字段标签作为注释不是一个好主意。

实践中， 原始的字符串字面量 **\`...\`** 比解释型字符串字面量 `"..."` 更受欢迎。

与C语言不同的是， Go中的结构体不支持联合（union）。

上述的例子中展示的都是未命名的结构体类型，也称为匿名结构体类型。 事实上， 命名结构体类型比匿名结构体更受欢迎。

只有可导出（大写开头）的结构体类型中的可导出的字段才能被其他包导入使用。

字段的标签和字段之间的顺序对于结构体类型来说非常重要。 两个未命名的结构体必须在其具有相同字段声明顺序的前提下才是相等的。 两个字段只有在其具有相同的名称、字段类型、标签的前提下才相等。 需要注意的是： **来自不同包的两个未导出的结构体字段名称始终被视为两个不同的名称**。

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

只有当两个结构体值得类型相同或者两个结构体值得类型具有相同的基类型（需要考虑字段标签）并且这两个类型中的至少一个是[非定义类型](https://go101.org/article/type-system-overview.html#non-defined-type)时， 才能将两个结构体值分配给彼此。


## 结构体字段的可寻址性
