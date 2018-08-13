> 译自[Functions in Go](https://go101.org/article/function.html)

# Go中的函数

之前的文章已经解释过[Functions declarations and calls](https://go101.org/article/function-declarations-and-calls.html)。本文将会解释更多关于Go中函数的感念和细节。

实际上，函数是Go中**一等公民**类型中的一种。换句话说，我们可以把函数（不包括内建函数）作为值来是使用。尽管Go是一门静态语言，Go的函数却是非常灵活的。函数的使用方式和许多动态语言中函数的使用方式很相似。

Go中有一些内建函数。这些函数在`builtin`和`unsafe`标准库中被声明。内建函数和自定义函数有一些不同。其中一项就是内建函数支持通用参数（generic parameters），但是自定义函数不支持。下文将会解释两者之间其他不同点。


## 函数签名和函数类型

在Go里，函数是**一等公民**类型中的一种。函数类型的字面值是有关键字`func`和函数签名字面值组成的。一个函数签名是由两个类型列表组成的，其中一个是入参类型列表，另一个是返回值类型列表。在函数签名中可以声明参数和返回值类型的名称。但是这些名称是什么并不重要。

我们经常使用函数类型字面值作为一个函数签名的字面值。

函数类型（签名）：

```go
func (a int, b string, c string) (x int, y int, z bool)
```

通过文章：[Functions declarations and calls](https://go101.org/article/function-declarations-and-calls.html)，我们已经学习到相同类型的连续变量可以放在一起声明。所以上述例子可以改成：

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
