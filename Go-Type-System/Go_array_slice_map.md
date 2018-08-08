> 译自： [Arrays, Slice And Maps in Go](https://go101.org/article/container.html)


# Go中的数组、切片和Map

严格来说， Go中有三种一等公民的容器类型。 它们分别是数组、切片和Map。某些时候， 字符串和channel也被当作容器类型， 但是这篇文章不会涉及这两种类型。 本篇文章提到的容器类型只会是上述列举的三种。

关于这三种容器类型有很多细节， 本文将一一列举。

## 容器类型和值的简单预览

这三种类型的每个值用于存储元素值的集合。 存储在容器值内的所有元素的类型是相同的。 这些相同的类型被称作该容器值的元素类型。

容器中的每个元素都有一个关联的键。每个元素值都可以通过其关联的键来访问。Map类型的key类型必须是[可比较类型](https://go101.org/article/type-system-overview.html#types-not-support-comparison)。 数组和切片的key类型是内建类型`int`。数组和切片元素的key是非负整数， 其标记了这些元素在数组或切片中的位置。这些非负整数的key通常被称为索引。


 每个容器的值都有一个长度（length）用来表示当前的容器存了多少个元素。 数组和切片值的非负整数key的有效范围是从零（包括）到数组或切片的长度值（不包括该值）。 对于Map类型的每一个值，Map值的键值可以是该Map类型的键类型的任意值。

 这三种容器类型之间又有很多的不同点。主要的差异体现在这三种容器类型的内存布局上。从上一篇文章[Value parts](https://go101.org/article/value-part.html)中，我们了解到一个数组值只由一个直接部分组成，但是一个切片或Map值可能会有一个底层部分，这部分是由切片或Map值的直接部分所引用的。

数组和切片的元素都是连续地存储在一个连续内存段中。对于一个数组来说，连续的内存段就是该数组的直接部分。 对于一个切片来说，连续的内存段则存储着该切片的底层部分。 标准Go编译器/运行时实现的Map采用的是HashTable算法。 所以一个Map的所有的元素也是存储在一个底层的连续的内存段中， 但是他们可能不是连续的。这这些连续的内存段中可能会存在很多的间隙（Gap）。

另一种常见的Map实现算法是二叉树算法， 但是下文的描述假设所有的Map底层结构都是HashTable的实现。无论使用何种算法， 与Map元素相关联的Key也同样存储在Map中。


我们可以通过key来访问一个元素。 所有容器类型的元素访问的时间复杂度都是`O(1)`，但是， 通常Map元素访问的速度要比切片和数组元素访问的时间慢上几倍。


需要注意的是，当值被复制时，值的底层部分不会被复制。换句话说， 如果一个值有底层部分， 那么该值的拷贝将会与其共享底层部分。这也是数组和切片/Map之间许多行为差异的根本原因， 下面的文章将会介绍这些差异行为。

## 未命名容器类型的字面量表示

三种未命名容器类型的字面量表示为：

- 数组类型 `[N]T`
- 切片类型 `[]T`
- Map类型 `map[K]T`

这里的

- `T`是任意类型。它定义了容器内元素的类型。只能将指定元素类型的值存储为容器类型元素的值。
- `N`必须是一个非负整数。它定义了数组类型只能存储多少个元素，所以也称之为为数组长度。这说明长度也是数组类型的一部分。比如， `[5]int`和`[8]int`就是两种不同的数组类型。
- `K`是任意可比较的类型。它定义了Map类型的Key的类型。Go中大多数的类型都是可比较的，不可比较的类型可见[列表](https://go101.org/article/type-system-overview.html#types-not-support-comparison)。


例子：

```go
const Size = 32

type Person struct {
	name string
	age  int
}

// Array types.
[5]string
[Size]int
[16][]byte  // element type is a slice type: []byte
[100]Person // element type is a struct type: Person

// Slice types.
[]bool
[]int64
[]map[int]bool // element type is a map type: map[int]bool
[]*int         // element type is a pointer type: *int

// Map types.
map[string]int
map[int]bool
map[int16][6]string     // element type is an array type: [6]string
map[bool][]string       // element type is a slice type: []string
map[struct{x int}]*int8 // element type is a pointer type: *int8,
                        // and key type is a struct type.
```

我们应该避免使用大型结构体作为Map的key。 因为它会大量的时间来进行结构体之间的比较。

所有的切片类型的大小都是一样的。 所有Map类型的大小也是一样的。 数组类型的大小取决于其的长度和元素类型的个数。0长度的数组类型或者0元素的数组类型的大小都是0。


## 容器值的字面量表示

和结构体一样， 容器值也可以通过`T{...}`这样的字面量形式来表示。 这里的`T`表示类型。 下面是一些例子：

```go
// An array value which contains four bool values.
[4]bool{false, true, true, false}

// A slice value which contains three words.
[]string{"break", "continue", "fallthrough"}

// A map value which contains some key-value pairs.
map[string]int{"C": 1972, "Python": 1991, "Java", 1995, "Go": 2009}
```

下面是数组、切片组合字面量的一些变种示例：

```go
// The followings slice composite literals are all equivalent.
[]string{"break", "continue", "fallthrough"}
[]string{0: "break", 1: "continue", 2: "fallthrough"}
[]string{2: "fallthrough", 1: "continue", 0: "break"}
[]string{2: "fallthrough", 0: "break", "continue"}

// The followings array composite literals are all equivalent.
[4]bool{false, true, true, false}
[4]bool{0: false, 1: true, 2: true, 3: false}
[4]bool{1: true, true}
[4]bool{2: true, 1: true}
[...]bool{false, true, true, false}
[...]bool{3: false, 1: true, true}
```

上述例子中最后两个字面量表示中， `...`意为让编译器自身去推断数组的长度。

容器复合字面量的元素值必须可分配给相应的容器元素类型。 Map复合字面量中的key必须可分配给相应的Mapkey类型。

通过上述例子，我们知道元素索引（keys）在数组和切片复合字面量中是可选的。如果指定了索引，则不需要声明该索引的类型为`int`。 但它必须是一个非负整数，可以使用`int`值来表示。如果该索引指定了类型， 则其类型必须是整型。不存在索引的元素可以用前一个元素的索引+1. 如果第一个元素的索引不存在，则其索引为0。


Map类型中的key可以是非常量类型：

```go
var a uint = 1
var _ = map[uint]int {a : 123} // okay
var _ = []int{a: 100}  // error: index must be non-negative integer constant
var _ = [5]int{a: 100} // error: index must be non-negative integer constant
```

一个特定的复合字面量中常量key不能重复。

## 容器类型零值的字面量表示

同结构体一样，数组类型`A`的零值可以通过复合字面量`A{}`表示。例如，类型`[100]int`的零值可以用`[100]int{}`表示。存储在数组零值中所有元素都是数组元素类型的零值。比如说上述例子中，所有元素的零值都是0。

同指针类型一样，切片和Map类型的零值可以用`nil`表示。

顺便说下，也有一部分其他类型的零值可以用`nil`表示， 包括指针、函数、channnel和interface类型。指针类型将在文章[pointers in Go](https://go101.org/article/pointer.html)中解释。其他类型也将在之后的文章中被提及到。

当一个数组变量被声明但却未指定初始值时，那么将会为数组元素的零值分配内存。但是， 对于`nil`的切片和Map来说， 内存却还未分配。

需要注意的是， `[]T{}`表示的是一个空切片类型，基于类型`[]T`，它和`[]T{nil}`是不同的。`map[K]T{}`和`map[K]T{nil}`同理。


## 复合字面量不可寻址但可以拥有地址

复合字面量是不可寻址的值。 我们之前学过[结构体复合字面量可以直接拥有地址](https://go101.org/article/struct.html#take-composite-literal-address)。容器类型同样也适用。

```go
package main

import "fmt"

func main() {
	pm := &map[string]int{"C": 1972, "Go": 2009}
	ps := &[]string{"break", "continue"}
	pa := &[...]bool{false, true, true, false}
	fmt.Printf("%T\n", pm) // *map[string]int
	fmt.Printf("%T\n", ps) // *[]string
	fmt.Printf("%T\n", pa) // *[4]bool
}
```

## 嵌套的复合字面量的简化表达

如果一个复合字面量嵌套了许多其他的复合字面量， 那么这些嵌套的复合字面量可以使用`{...}`这样的形式来简化表示。

例如下面这个切片字面量：

```go
// A slcie value of a type whose element type is *[4]byte.
// The element type is a pointer type whose base type is [4]byte.
// The pointer base type is an array type whose element type is byte.
var heads = []*[4]byte{
	&[4]byte{'P', 'N', 'G', ' '},
	&[4]byte{'G', 'I', 'F', ' '},
	&[4]byte{'J', 'P', 'E', 'G'},
}
```

可以简化表示为：

```go
var heads = []*[4]byte{
	{'P', 'N', 'G', ' '},
	{'G', 'I', 'F', ' '},
	{'J', 'P', 'E', 'G'},
}
```

下面数组字面量：

```go
type language struct {
	name string
	year int
}

var _ = [...]language{
	language{"C", 1972},
	language{"Python", 1991},
	language{"Go", 2009},
}
```

可以简化为：

```go
var _ = [...]language{
	{"C", 1972},
	{"Python", 1991},
	{"Go", 2009},
}
```

还有下例中的Map字面量：

```go
type LangCategory struct {
	dynamic bool
	strong  bool
}

// A value of map type whose key type is a struct type and
// whose element type is another map type "map[string]int".
var _ = map[LangCategory]map[string]int{
	LangCategory{true, true}: map[string]int{
		"Python": 1991,
		"Erlang": 1986,
	},
	LangCategory{true, false}: map[string]int{
		"JavaScript": 1995,
	},
	LangCategory{false, true}: map[string]int{
		"Go":   2009,
		"Rust": 2010,
	},
	LangCategory{false, false}: map[string]int{
		"C": 1972,
	},
}
```

可以简化表示为：

```go
var _ = map[LangCategory]map[string]int{
	{true, true}: {
		"Python": 1991,
		"Erlang": 1986,
	},
	{true, false}: {
		"JavaScript": 1995,
	},
	{false, true}: {
		"Go":   2009,
		"Rust": 2010,
	},
	{false, false}: {
		"C": 1972,
	},
}
```

需要注意的是， 上述的例子中， 每个复合字面量中最后一项的逗号不能省略。 更多的内容可以参考： [the line break rules in Go](https://go101.org/article/line-break-rules.html)。


## 比较容器的值

正如之前文章[overview of Go type system](https://go101.org/article/type-system-overview.html#types-not-support-comparison)中提过的那样，map和切片类型是不可比较的类型。所以map和切片不能当作map的key。

尽管一个切片或map不能和另一个切片或map（或自身）做比较， 但是它们却可以和`nil`值作比较，用来检查一个切片或map是不是该类型的零值。

大多数的数组类型都是可比较的， 除了那些元素类型是不可比较类型的数组。

当比较两个相同数组类型的值时，它们每个元素都会被比较。只有当它们相应的元素全部相等时， 这两个数组类型才是相等的。

例子：

```go
package main

import "fmt"

func main() {
	var a [16]byte
	var s []int
	var m map[string]int

	fmt.Println(a == a)   // true
	fmt.Println(m == nil) // true
	fmt.Println(s == nil) // true
	fmt.Println(nil == map[string]int{}) // false
	fmt.Println(nil == []int{})          // false

	// The following lines fail to compile.
	/*
	_ = m == m
	_ = s == s
	_ = m == map[string]int(nil)
	_ = s == []int(nil)
	var x [16][]int
	_ = x == x
	var y [16]map[int]bool
	_ = y == y
	*/
}
```

## 访问和更改容器元素

语法`v[k]`表示存储在容器`v`中key： `k`相关联的元素。

对于语法`v[k]`来说， 我们假设`v`的类型是切片或者数组，
- 如果`k`是一个常量，那么它必须满足上面说明的复合字面量中的索引要求。特别的， 如果`v`是一个数组，那么`k`的值必须小于数组的长度。

- 如果`k`不是一个常量，那么它必须是一个整型且大于0且小于`len(v)`。否则， Go运行时则会panic。

- 如果`v`是一个nil切片，Go运行时会发生panic。

对于语法`v[k]`来说， 我们假设`v`的类型是map，那么`k`的类型必须是可比较的。
- 如果`k`是一个不可比较的值， 那么则会panic。

- 如果`k`是一个可比较的值， 仅当`v`是`nil` map且`v[k]`被用作赋值操作中目标值是，`v[k]`才会引起panic。

- 如果`v[k]`被用作赋值操作中的源值，不会发生panic，尽管`v`是一个`nil`map。

- 如果`v[k]`被用作赋值操作中的源值且map`v`中没有key这个entry， 那么`v[k]`将会得到map`v`中元素类型的零值。对于大多数的场景来说，`v[k]`被视为单值表达式。 然而， 当`v[k]`被用作赋值操作中的源值时，它则会被视为一个多值表达式，并会产生第二个可选的无类型的bool值， 用来表示map`v`是否含有key`k`相关联的实体。


例子：

```go
package main

import "fmt"

func main() {
	a := [3]int{-1, 0, 1}
	s := []bool{true, false}
	m := map[string]int{"abc": 123, "xyz": 789}
	fmt.Println (a[2], s[1], m["abc"])    // access
	a[2], s[1], m["abc"] = 999, true, 567 // modify
	fmt.Println (a[2], s[1], m["abc"])    // access

	n, present := m["hello"]
	fmt.Println(n, present, m["hello"]) // 0 false 0
	n, present = m["abc"]
	fmt.Println(n, present, m["abc"]) // 567 true 567
	m = nil
	fmt.Println(m["abc"]) // 0

	// The two lines fail to compile.
	/*
	_ = a[3]  // invalid array index 3 (out of bounds)
	_ = s[-1] // invalid slice index -1 (index must be non-negative)
	*/

	// Each of the following lines causes a panic.
	_ = a[n]         // panic: runtime error: index out of range
	_ = s[n]         // panic: runtime error: index out of range
	m["hello"] = 555 // panic: assignment to entry in nil map
}
```
