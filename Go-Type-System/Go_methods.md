> 译自[Methods in Go](https://go101.org/article/method.html)

# Go中的方法

Go支持一些面向对象的特性。方法是这些特性之一。方法是一个拥有接收者参数的特殊函数。本文将解释方法相关的的概念。

## 方法声明

在Go里，我们可以显式地为类型`T`和`*T`声明方法，对于类型`T`必须满足以下4个条件：

  1. `T`必须是一个已经定义的类型；
  2. `T`必须在于方法声明的同一包内定义；
  3. `T`必须不能为一个指针类型。
  4. `T`必须不能为一个接口类型。（接口类型将在[下篇文章](https://go101.org/article/interface.html)中解释。）

类型`T`和`*T`所有方法中的`T`被称之为接收者基础类型。（下节将解释什么是接收者。）

我们还可以为类型`T`和`*T`的别名类型声明方法。别名类型声明的方法和它们本来的类型声明的方法效果是一样的。

如果一个类型声明了方法，那我们可以说这个类型拥有了自己的类型。

从上面列举的条件中，我们可以得出结论，我们应该明确不会显式地为：

  - 内建基本类型，如`int`和`string`，因为我们不能在内建标准库包内声明方法。
  - 接口类型。但是接口类型可以拥有自己的方法，详情请阅读[下篇文章](https://go101.org/article/type-embedding.html)。
  - 除了上述的指针类型`*T`之外的未命名类型，包括未命名的数组，切片，map，函数，通道和结构体类型。然而，如果一个未命名的结构体类型嵌入了其他具有方法的类型。那么编译器会将隐式地声明一些未命名结构体类型的方法和一个未命名的指针类型。它的基础类型是未命名的结构体类型。详情请阅读[type embedding](https://go101.org/article/type-embedding.html)。

方法的声明和函数的声明相似，但是它有一个额外的参数声明部分。额外的参数声明部分被称作为接收者参数。每个方法的声明必须有且仅有一个接收者参数。接收者参数必须在括号`()`内部且必须位于关键字`func`和方法名之间。接收者参数的类型被称为该方法的接收者类型。

这里是一些方法声明的例子：

```go
// Age and int are two distinct types.
// We can't declare methods for int, but can for Age.
type Age int
func (age Age) LargerThan(a Age) bool {
	return age > a
}
func (age *Age) Increase() {
	*age++
}

// Receiver of custom defined function type.
type FilterFunc func(in int) bool
func (ff FilterFunc) Filte(in int) bool {
	return ff(in)
}

// Receiver of custom defined map type.
type StringSet map[string]struct{}
func (ss StringSet) Has(key string) bool {
	_, present := ss[key]
	return present
}
func (ss StringSet) Add(key string) {
	ss[key] = struct{}{}
}
func (ss StringSet) Remove(key string) {
	delete(ss, key)
}

// Receiver of custom defined struct type.
type Book struct {
	pages int
}
func (b Book) Pages() int {
	return b.pages
}
func (b *Book) SetPages(pages int) {
	b.pages = pages
```

从上面的例子中，我们知道接收者基础类型不仅可以是结构体类型，而且可以是其他类型的种类（kind），例如基本类型和容器类型，只要接收者类型满足上面列出的4个条件即可。

在某些编程语言中，接收者参数名称始终隐式地用`this`表示，但这不是Go中接收者的推荐标识符。

类型`*T`的接收者被称为**指针接收者**，非指针接收者被称为**值接收者**。就个人而言，我不建议将术语指针和术语值的对立起来，因为指针值只是一个特殊值。但是在这里，我也不反对使用指针接收者和值接收者这样的术语。原因将在下面的章节中解释。

方法名可以是空标识符`_`。一个类型可以拥有多个以空标识符命名的方法。但是这样的方法将永远不会被调用。

只有可导出的方法才能从其他的包中被调用。

## 每个方法都对应一个隐藏的函数

对于每个方法的声明，编译器都会为之声明一个相关的隐式函数。对于在上节的最后一个示例中为类型`Book`和类型`*Book`声明的最后两个方法，编译器隐式声明了以下两个函数：


```go
func Book.Pages(b Book) int {
	return b.pages // the body is the same as the Pages method
}

func (*Book).SetPages(b *Book, pages int) {
	b.pages = pages // the body is the same as the SetPages method
}
```

在每个隐式的函数声明中，接收者参数从其相关的方法声明中被移除并且作为第一个参数插入到了一个正常的函数中。两个隐式声明的函数的函数体与它们对应的方法体相同。

这两个隐式的函数名，`Book.Pages`和`(*Book).SetPages`，都是`TypeDenotation.MethodName`形式。由于Go中的标识符不能包含句点特殊字符，因此两个隐式函数名称不是合法标识符，因此这两个函数不能显式地声明。它们只能通过编译器隐式地声明，但是它们可以被用户代码调用：

```go
package main

import "fmt"

type Book struct {
	pages int
}
func (b Book) Pages() int {
	return b.pages
}
func (b *Book) SetPages(pages int) {
	b.pages = pages
}

func main() {
	var book Book
	// Call the two implicit decalred functions.
	(*Book).SetPages(&book, 123)
	fmt.Println(Book.Pages(book)) // 123
}
```
