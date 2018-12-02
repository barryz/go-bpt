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

## 指针接收者的隐式方法

对于为类型`T`的值接收者声明的每个方法，编译器会将类型为`*T`隐式地声明为具有相同名称的方法。通过上述的例子，`Pages`方法是为类型`Book`声明的，所以编译器将会隐式地为类型`*Book`声明一个同名的Pages方法。这个同名的方法只包含一行代码，那就是对上面介绍的隐式函数`Book.Pages`的调用。

```go
func (b *Book) Pages() int {
	return Book.Pages(*b)
}
```

这就是为什么我不拒绝使用值接收者术语（作为指针接收者的对立面）的原因。毕竟，当我们明确声明一个非指针类型的方法时，实际上声明了两个方法，显式的那个用于非指针类型，另一个隐式的用于相应的指针类型。

正如最后一节中提到的，对于每个声明的方法，编译器都会为其隐式地声明一个函数。因此，对于刚刚提到的隐式声明的方法，编译器声明了以下隐式函数。

```go
func (*Book).Pages(b *Book) int {
	return Book.Pages(*b)
}
```

换句话说，对于每个使用值接收者声明的方法，两个隐式的函数和一个隐式的方法将会在同一时间被声明。

## 方法原型和方法集

一个方法的原型可以被看作是一个除去`func`关键字的[函数原型](https://go101.org/article/function.html#prototype)。我们可以看到每个方法是由关键字`func`，一个接收者参数，一个方法原型和一个方法（函数）体组成。

例如，方法`Pages`和`SetPages`的方法原型表示如下：

```go
Pages() int
SetPages(pages int)
```

每个类型都有一个**方法集**。一个非接口类型的方法集是由已声明的方法的所有方法原型组成。包括显式声明和隐式声明的。对于类型，除了名称为空标识符`_`的那些。

例如，上面例子中Book类型的方法集就是

```go
Pages() int
```

以及类型*Book的方法集就是

```go
Pages() int
SetPages(pages int)
```

对于方法集来说一个方法集的方法原型顺序不重要。

对于一个方法集来说，如果其中的每个方法原型也存在于另一个方法集中，那么我们就说前一个方法集和后一个方法集的子集。如果两个方法集互为对方的子集，那么我们就说这两个方法集是相同的。

给定一个类型`T`，假设它既不是指针类型也不是接口类型，因为上文已经提及的[原因](https://go101.org/article/method.html#implicit-pointer-methods)，类型`T`的方法集总是类型`*T`的方法集的子集。例如，上面提到的`Book`类型的方法集就是`*Book`类型的方法集的子集。

注意，**不同包中的以小写字母开头的非导出方法名将始终被视为两个不同的方法名**。换句话说，一个非导出方法名的方法原型总是不同于其他包中任何方法原型。

方法集在Go的多态性特征中起着重要作用。关于多态，请阅读[下篇文章](https://go101.org/article/interface.html)（interfaces in Go)了解细节。

下列类型的方法集总是为空：

    1. 内建基本类型。

    2. 定义的指针类型。

    3. 基类型是接口或指针类型的未命名指针类型。

    4. 未命名的数组，切片，map，函数和通道类型。

## 方法值和方法调用

实际上方法是特殊的函数。当为一个类型声明一个方法时，该类型的每个值都将拥有一个函数类型的不可变成员。成员名称和方法名相同。函数类型和以方法声明的形式声明的函数相同，但没有接收者部分。成员经常被称作是成员函数。成员函数也可以称为其对应值的方法。

一个方法调用仅仅是这样的一个成员函数的调用。对于一个值`v`来说，它的函数`m`可以通过选择器形式`v.m`来访问。

一个包含方法调用的例子：

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

	fmt.Printf("%T \n", book.Pages)       // func() int
	fmt.Printf("%T \n", (&book).SetPages) // func(int)
	// &book has an implicit method.
	fmt.Printf("%T \n", (&book).Pages)    // func() int

	(&book).SetPages(123)
	fmt.Println(book.Pages()) // 123
	// Call the implicit method of &book.
	fmt.Println((&book).Pages()) // 123
}
```

在上述的例子中，值`book`被称为所有`Pages`和`SetPages`方法调用接收者参数的基础值。

（*不同于C语言，Go中没有`->`操作符用于指针接收者来调用方法*，所以`(&book)->SetPages(123)`在Go中是非法的。）

方法调用使得Go程序清晰易读。实际上，上面的例子可以更简单明了。上述例子中的`(&book).SetPages(123)`可以简化为`book.SetPages(123)`。 但是这怎么可能会发生呢？毕竟，值`book`并没有名为`SetPages`方法调用。啊哈，其实这只是一种语法糖，为了让编程更加便捷。此种语法糖仅适用于`Book`类型是可寻址的。当可寻址的值作为`SetPages`方法调用的接收者参数传递时，编译器将自动获取这些值的地址。

```go
...

// Function results are not addressable in Go.
func MakeBook() Book {
	return Book{}
}

func main() {
	var book Book
	book.SetPages(123) // <=> (&book).SetPages(123)
	fmt.Println(book.Pages()) // 123

	// error: function return results are not addressable.
	MakeBook().SetPages(123)
}
```

正如上面提到的，当为一个类型声明一个方法，该类型的每个值都会拥有一个成员函数。零值也不例外，无论这个零值是否可以用`nil`表示。

例子：

```go
package main

type StringSet map[string]struct{}
func (ss StringSet) Has(key string) bool {
	_, present := ss[key] // Never panic here,
	                      // even if ss is nil.
	return present
}

type Age int
func (age *Age) IsNil() bool {
	return age == nil
}
func (age *Age) Increase() {
	*age++ // If age is a nil pointer, then
	       // dereferencing it will panic.
}

func main() {
	_ = (StringSet(nil)).Has   // will not panic
	_ = ((*Age)(nil)).IsNil    // will not panic
	_ = ((*Age)(nil)).Increase // will not panic

	_ = (StringSet(nil)).Has("key") // will not panic
	_ = ((*Age)(nil)).IsNil()       // will not panic

	// This line will panic. But the panic is not caused
	// by invoking the method. It is caused by the nil
	// pointer dereference within the method body.
	((*Age)(nil)).Increase()
}
```

## 接收者参数通过拷贝传递

和普通的函数参数一样，接收者参数也是通过拷贝传递的。因此，方法调用中对接收者参数的[直接部分](https://go101.org/article/value-part.html)的修改并不会反映到方法的外部。

例子：

```go
package main

import "fmt"

type Book struct {
	pages int
}

func (b Book) SetPages(pages int) {
	b.pages = pages
}

func main() {
	var b Book
	b.SetPages(123)
	fmt.Println(b.pages) // 0
}
```

另一个例子：

```go
package main

import "fmt"

type Book struct {
	pages int
}

type Books []Book

func (books Books) Modify() {
	// Modifications on the underlying part of the receiver
	// will be reflected to outside of the method.
	books[0].pages = 500
	// Modifications on the direct part of the receiver
	// will not be reflected to outside of the method.
	books = append(books, Book{789})
}

func main() {
	var books = Books{{123}, {456}}
	books.Modify()
	fmt.Println(books) // [{500} {456}]
}
```

到这里似乎有些偏离了主题，如果交换上述Modify方法中两行代码的顺序，则两个修改都不会反映到方法体外部。

```go
func (books Books) Modify() {
	books = append(books, Book{789})
	books[0].pages = 500
}

func main() {
	var books = Books{{123}, {456}}
	books.Modify()
	fmt.Println(books) // [{123} {456}]
}
```

这里的原因是`append`调用将分配一个新的内存块用来存储通过接收者参数传递的切片元素的拷贝。分配不会反映到被传递的切片本身。

为了使两个修改都反映到方法体外部，该方法的接收器必须是指针接收器。

```go
func (books *Books) Modify() {
	*books = append(*books, Book{789})
	(*books)[0].pages = 500
}

func main() {
	var books = Books{{123}, {456}}
	books.Modify()
	fmt.Println(books) // [{500} {456} {789}]
}
```

## 方法应该通过指针接收者声明还是值接收者声明？

首先，从上一节开始，我们知道有时我们必须用指针接收者声明方法。

实际上，我们总是可以使用指针接收者声明方法而不会出现任何逻辑问题。这只是程序性能的问题，有时候用值接收器声明方法会更好。

这里有一些因素需要考虑做出使用值接收者还是指针接收者（事实上值接收者和指针接收者都是可接受的）。

    1. 指针拷贝太多可能会导致垃圾回收的工作量增加。

    2. 如果接收者类型的值大小过大，则接收者参数拷贝成本则不可忽略。特别是如果传递的参数是接口值（请参阅[多态性]()以获取更多详细信息），将在参数传递中创建两个拷贝。在这里，我们只需要知道指针值都是[小尺寸值]()。 实际上，对于标准的Go编译器和运行时来说，数组和结构体类型以外的类型都是小尺寸类型。拥有少量字段的结构体也是小尺寸类型。

    3. 如果在多个goroutine中同时调用声明的方法，则混合了值接收者和指针接收者的相同基类型更有可能导致数据竞争。

    4. 标准库`sync`类型的值不应该被复制，因此使用**嵌入了标准库`sync`类型的结构体类型的值接收者方法**是有问题的。

如果很难确定方法是使用指针接收者还是值接收者，那么只需选择指针接收者即可。
