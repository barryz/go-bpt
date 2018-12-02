> 译自： [Arrays, Slice And Maps in Go](https://go101.org/article/container.html)

# Go中的数组、切片和Map

严格来说， Go中有三种"一等公民"的容器类型。 它们分别是数组、切片和map。某些时候，字符串和channel也被当作容器类型，但是这篇文章不会涉及这两种类型。本篇文章提到的容器类型只会是上面列举的三种。

关于这三种容器类型有很多细节，本文将一一展开。

## 容器类型和值的简单预览

这三种类型的用于存储元素值的集合。存储在容器内的所有元素的类型是相同的。这些相同的类型被称作该容器的元素类型。

容器中的每个元素都有一个关联的键（key）。每个元素值都可以通过其关联的键来访问。map类型的key类型必须是[可比较类型](https://go101.org/article/type-system-overview.html#types-not-support-comparison)。数组和切片的key类型是内建类型`int`。数组和切片元素的key是非负整数，其标记了这些元素在数组或切片中的位置。这些非负整数的key通常被称为索引。

 每个容器的值都有一个长度（length）用来表示当前的容器存了多少个元素。数组和切片值的非负整数key的有效范围是从零（包括）到数组或切片的长度值（不包括该值）。对于map类型的每一个值，map值的键值可以是该map类型的键类型的任意值。

 这三种容器类型之间又有很多的不同点。主要的差异体现在这三种容器类型的内存布局上。从上一篇文章[Value parts](https://go101.org/article/value-part.html)中，我们了解到一个数组值只由一个**直接部分**组成，但是一个切片或map值可能会有一个**底层部分**，这部分是由切片或map值的**直接部分**所引用的。

数组和切片的元素都是连续地存储在一个连续内存段中。对于一个数组来说，连续的内存段就是该数组的**直接部分**。对于一个切片来说，连续的内存段则存储着该切片的**底层部分**。标准Go编译器/运行时实现的map采用的是`HashTable`算法。所以一个map的所有的元素也是存储在一个底层的连续的内存段中，但是他们可能不是连续的。这些连续的内存段中可能会存在很多的间隙（Gap）。

另一种常见的map实现算法是二叉树，但是本文的描述的所有的map底层结构都是`HashTable`的实现。无论使用何种算法，与map元素相关联的Key也同样存储在map中。

我们可以通过key来访问一个元素。所有容器类型的元素访问的时间复杂度都是`O(1)`，但是，通常map元素访问的速度要比切片和数组元素访问的时间慢上几倍。

需要注意的是，当值被复制时，值的**底层部分**不会被复制。换句话说，如果一个值有**底层部分**，那么该值的拷贝将会与其共享底层部分。这也是数组和切片/map之间许多行为差异的根本原因，下面的文章将会介绍这些差异行为。

## 未命名容器类型的字面量表示

三种未命名容器类型的字面量表示为：

- 数组类型 `[N]T`
- 切片类型 `[]T`
- Map类型 `map[K]T`

在这里

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

我们应该避免使用大型结构体作为map的key。因为它会大量的时间来进行结构体之间的比较。

所有的切片类型的大小都是一样的。所有map类型的大小也是一样的。数组类型的大小取决于其的长度和元素类型的个数。0长度的数组类型或者0元素的数组类型的大小都是0。

## 容器值的字面量表示

和结构体一样，容器值也可以通过`T{...}`这样的字面量形式来表示。这里的`T`表示类型。下面是一些例子：

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

上述例子中最后两个字面量表示中，`...`意为让编译器自身去推断数组的长度。

容器复合字面量的元素值必须可分配给相应的容器元素类型。map复合字面量中的key必须可分配给相应的map key类型。

通过上述例子，我们知道元素索引（keys）在数组和切片复合字面量中是可选的。如果指定了索引，则不需要声明该索引的类型为`int`。但它必须是一个非负整数，可以使用`int`值来表示。如果该索引指定了类型， 则其类型必须是整型。不存在索引的元素可以用前一个元素的索引+1. 如果第一个元素的索引不存在，则其索引为0。

map类型中的key可以是非常量类型：

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

## 容器类型的赋值分配操作

如果将一个map赋值分配给另一个map， 那么这两个map将共享同一份（底层）元素。

和map的赋值分配一样，如果一个切片分配给另一个切片，它们也将共享同一份（底层）元素。

当一个数组分配给另一个数组时，数组中所有的元素都会从源数组拷贝到目标数组。这两个数组不会共享任何元素。

例子：

```go
package main

import "fmt"

func main() {
	m0 := map[int]int{0:7, 1:8, 2:9}
	m1 := m0
	m1[0] = 2
	fmt.Println(m0, m1) // map[0:2 1:8 2:9] map[0:2 1:8 2:9]

	s0 := []int{7, 8, 9}
	s1 := s0
	s1[0] = 2
	fmt.Println(s0, s1) // [2 8 9] [2 8 9]

	a0 := [...]int{7, 8, 9}
	a1 := a0
	a1[0] = 2
	fmt.Println(a0, a1) // [7 8 9] [2 8 9]
}
```

## 检查容器类型的长度和容量

除了长度这个属性外， 每种容器类型值都有一个容量（capacity）属性。数组的容量恒等于数组的长度。map类型的容量是无限的。于是， 在实践中，只有切片的容量有意义。一个切片的容量永远大于等于其长度。切片容量的含义将在下一章节详细说明。

我们可以使用内建函数`len`来获取容器类型的长度， 使用内建函数`cap`来获取容器类型的容量值。这两个内建函数会返回一个`int`类型的结果。因为map类型的容量是无限的，所以内建函数`cap`并不能用于map类型。

数组类型的容量值和长度值永远不会改变。切片类型的长度值和容量值会在运行时发生改变。所以切片可以看作是动态数组。实践中，切片相较于数组使用时更加灵活且更受欢迎。

例子：

```go
package main

import "fmt"

func main() {
	var a [5]int
	fmt.Println(len(a), cap(a)) // 5 5
	var s []int
	fmt.Println(len(s), cap(s)) // 0 0
	s, s2 := []int{2, 3, 5}, []bool{}
	fmt.Println(len(s), cap(s), len(s2), cap(s2)) // 3 3 0 0
	var m map[int]bool
	fmt.Println(len(m)) // 0
	m, m2 := map[int]bool{1: true, 0: false}, map[int]int{}
	fmt.Println(len(m), len(m2)) // 2 0
}
```

上述例子中，每个切片的长度值和容量值都是相等的。但这并不总是对的。 有些时候切片的长度值和容量值并不相等。这将在下面的章节说详细说明。

## 复习切片类型内部结构定义

为了更好地理解切片类型的值并且更容易解释切片， 我们需要对切片类型的内部结构有深刻的印象。之前的文章[value parts](https://go101.org/article/value-part.html)中，我们已经学习到了切片相关的内部定义：

```go
type _slice struct {
	elements unsafe.Pointer // referencing underlying elements
	len      int            // length
	cap      int            // capacity
}
```

其他编译器/运行时实现的切片的内部定义并一定和上述例子中一样。下面章节引用的切片内部实现基于官方的切片实现。

上述的例子展示了切片类型内部结构并解释了其内存布局已经它的直接部分。`len`字段的直接部分代表了这个切片的长度值， `cap`字段表示了这个切片的容量值。下面的图片描述了一个切片值的内存布局。

![](https://go101.org/article/res/slice-internal.png)

尽管承载（hosting）切片元素的底层内存段可能非常大，但切片可能只关注内存段的子段（sub-segment）。例如，在上面的图片中，切片只关心整个内存段中的灰色部分。

接下来的章节将会解释如何使用内建函数`append`将元素追加到基础切片并派生出一个新的切片。 函数`append`调用产生的新的slice结果有可能与基本切片（base slice）共享起始元素， 共享与否取决于基本切片的容量值（长度值）和追加了多少元素。

像之前图片描述的slice一样。从`len`索引开始到`cap`索引之间的元素是不属于该切片的。它们仅仅是一些冗余的元素槽（redundant element slots)。对于上述图片上的切片来说，它们可能属于其他切片或者另一个数组。

`append`函数调用时：

- 如果追加的元素数大于基础切片的冗余元素槽的数量，一个全新的底层内存段将会被分配给结果切片。因此结果切片和基础切片不会共享任何元素。

- 反之，如果追加的元素数小于基础切片的冗余元素槽的数量，就不会有新的内存段分配给结果切片，并且基础切片的元素同样属于结果切片。换句话说，这两个切片共享一些元素，而且他们所有的元素都托管（host）在同一个底层内存段上。

之后的章节将会展示一张描述上述两种情况的图片。

有很多的方式会导致两个切片的元素会被同一个底层内存段托管。例如切片赋值分配以及下面将要介绍的子切片操作。

注意，通常来说，我们无法单独地去修改切片中这三个字段，除非通过[relection](https://go101.org/article/container.html#modify-slice-length-and-capacity)和[unsafe](https://go101.org/article/unsafe.html)这两种方式。换句话说，通常，想要修改一个切片值， 它的三个字段会被一起修改。通常，这是通过将另一个切片值（相同切片类型）复制/分配给需要修改的切片来实现的。

## 追加和删除容器元素

将键值对（也称为条目entry）追加到map的语法与修改map元素的语法相同。例如，对于`non-nil`map值`m`， 下面的代码：

```go
m[k] = e
```

将会将键值对`(k, e)`放到map`m`中， 如果`m`中不含有跟key`k`相关的条目。 如果`m`中含有跟key`k`相关的条目，那么该操作只是修改key`k`关联的值。

map的底层连续的内存段可能需要重新分配来容纳更多的元素。

内建函数`delete`用来从一个map中删除一个条目。例如，下面的代码片段将会从map`m`中删除key`k`相对应的条目。 如果map`m`中不包含`k`的条目，那么它不会有任何操作。没有panic， 尽管`m`有可能是一个nil map。

```go
delete(m, k)
```

存储在map中的条目是无序的。我们可以认为这种顺序是随机的。

如何在map中追加（put）元素或删除元素：

```go
package main

import "fmt"

func main() {
	m := map[string]int{"Go": 2007}
	m["C"] = 1972     // append
	m["Java"] = 1995  // append
	fmt.Println(m)    // map[C:1972 Python:1991 Go:2007]
	m["Go"] = 2009    // modify
	delete(m, "Java") // delete
	fmt.Println(m)    // map[Go:2009 C:1972]
}
```

数组元素既不能被追加也不能被删除， 尽管可寻址的数组元素可以被修改。

我们可以使用内建函数`append`来向基础切片追加多个元素。`append`函数调用的结果是产生一个的包含基础切片和追加的元素的新切片。请注意，`append`函数调用时，基础切片并不会被修改。当然， 如果我们想的话，我们可以将结果切片分配给基础切片来修改基础切片。

并没有内建方法来删除一个切片的元素。我们必须结合使用`append`函数和子切片特性来达到这一目的。切片元素的删除和插入将会在下面的内容中进行演示。 这里将只会演示如何追加切片元素。

例子：

```go
package main

import "fmt"

func main() {
	s0 := []int{2, 3, 5}
	s1 := append(s0, 7)      // append one element
	s2 := append(s1, 11, 13) // append two elements
	fmt.Println(s0, cap(s0)) // [2 3 5] 3
	fmt.Println(s1, cap(s1)) // [2 3 5 7] 6
	fmt.Println(s2, cap(s2)) // [2 3 5 7 11 13] 6
	s3 := append(s0)         // <=> s3 := s0
	s4 := append(s0, s0...)  // append all elements of s0
	fmt.Println(s3, cap(s3)) // [2 3 5] 3
	fmt.Println(s4, cap(s4)) // [2 3 5 2 3 5] 6
	s0[0], s1[0] = 99, 789
	fmt.Println(s2[0], s3[0], s4[0]) // 789 99 2
}
```

注意， 内建函数`append`是一个[可变参数的函数](https://go101.org/article/function.html#variadic-function)。它有两个参数，第二个参数是一个[可变参数](https://go101.org/article/function.html#variadic-parameter)。在[调用可变参数函数](https://go101.org/article/function.html#variadic-call)时有两种方式。在上面的例子中，第7行、第8行和第12行使用了第一种方式，第13行使用另一种方式。对于前一种方式，我们可以传递零个或者多个元素值作为可变参数。对于后一种方式，我们必须传递一个切片作为唯一的可变参数，并且必须跟随三个点号`...`。我们之后可以学习到如何如何调用可变参数函数。

在这里， 第13行等价于：

```go
s4 := append(s0, s0[0], s0[1], s0[2])
```

对于三个点`...`这种方式，`append`函数并不要求可变参数是一个和第一个参数同类型的切片， 但是他们的元素必须是相同类型的。换句话说，这两个参数对应的切片必须共享同一个底层类型。

上面的例子中：

- 第7行的`append`调用将会分配一块全新的底层内存段给切片`s1`。对于切片`s0`来说，其没有足够的冗余元素槽来存储新追加的元素。这和第13行中的情形是一样的。

- 第8行的`append`调用不会分配新的底层内存段给切片`s2`，因为对于切片`s1`来说，它有足够的冗余元素槽位来存储新追加的元素。

于是， `s1`和`s2`共享某些元素， `s0`和`s3`共享所有的元素， `s4`不和其他切片共享元素。下图描绘了上述代码片段最终执行结果时的内存布局情况：

![](https://go101.org/article/res/slice-append.png)

到Go目前的版本（1.10）为止， `append`函数的第一个参数不能为`nil`。

## 使用内建函数`make`来创建切片和map

除了使用复合字面量来创建map和切片以外，我们也可以使用内建函数`make`来创建它们。内建函数`make`不能用于创建数组类型。

**（内建函数`make`同样可以用来创建channels， 这将会在之后的文章[channels in Go](https://go101.org/article/channel.html)中详细阐述。）**

假设`M`是一个map类型， `n`是一个非负整数，我们可以使用以下两种方式来创建一个新的map类型`M`。 第一种形式创建一个新的空map，分配了足够的空间来容纳N个元素。第二种形式只接受一个参数，在这种情况下， 分配一个具有足够空间来容纳少量元素的新空map。这里的少量依赖于具体的编译器实现。

```go
make(M, n)
make(M)
```

假设`S`是一个切片类型， `length`和`capacity`是两个非负整数，`length`值不大于`capacity`值，我们可以使用以下两种方式来创建一个新的切片类型`S`。

```go
make(S, length, capacity)
make(S, length)
```

第一种形式创建一个新的空切片，并且指定了长度和容量。第二种形式只接受两个参数，在这种情况下， 新创建的切片容量与其长度相同。

`make`函数调用产生的结果切片中的所有元素都被初始化为其零值（切片元素类型的零值）。

一个展示如何使用`make`函数创建切片和map的示例：

```go
package main

import "fmt"

func main() {
	// Make new maps.
	fmt.Println(make(map[string]int)) // map[]
	m := make(map[string]int, 3)
	fmt.Println(m, len(m)) // map[] 0
	m["C"] = 1972
	m["Go"] = 2009
	fmt.Println(m, len(m)) // map[C:1972 Go:2009] 2

	// Make new slices.
	s := make([]int, 3, 5)
	fmt.Println(s, len(s), cap(s)) // [0 0 0] 3 5
	s = append(s, 7, 11)
	fmt.Println(s, len(s), cap(s)) // [0 0 0 7 11] 5 5
	s = make([]int, 2)
	fmt.Println(s, len(s), cap(s)) // [0 0] 2
}
```

## 使用内建函数`new`来分配容器

通过之前的文章[pointers in Go](https://go101.org/article/pointer.html)， 我们学习到也可以通过调用内建函数`new`来分配任意类型的值，并返回一个引用该值的指针。该分配的值是它所属类型的零值。基于这个原因， 使用`new`函数创建map和切片是没有意义的（因为其返回了map或者切片的零值nil）。

使用内建函数`new`来分配一个数组类型的零值并非完全没有意义。但是在实践中却很少有人这么做。

例子：

```go
package main

import "fmt"

func main() {
	m := *new(map[string]int)   // <=> var m map[string]int
	fmt.Println(m == nil)       // true
	s := *new([]int)            // <=> var s []int
	fmt.Println(s == nil)       // true
	a := *new([5]bool)          // <=> var a [5]bool
	fmt.Println(a == [5]bool{}) // true
}
```

在创建容器类型时，复合字面量以及内建函数`make`比内建函数`new`更加流行。

## 容器元素的可寻址性

以下是关于容器元素的可寻址性的一些事实。

- 可寻址的数组的元素也是可寻址的。不可寻址的数组的元素也是不可寻址的。 原因是每个数组的值都只是由一个直接部分构成的。

- 任何切片值的元素总是可寻址的， 无论该切片是否可以寻址。这是因为切片中的元素总是存储在底层值部分中，而且这个底层值部分总是托管在已分配的内存段上。

- 任何map值的元素总是不可寻址的，原因是当map元素的数量增加时，map的底层连续内存段（underlying continuous memory segment）可能需要重新分配。 在重新分配结束后，所有元素都将被移动到其他的位置。

例子：

```go
package main

import "fmt"

func main() {
	a := [5]int{2, 3, 5, 7}
	s := make([]bool, 2)
	pa2, ps1 := &a[2], &s[1]
	fmt.Println(*pa2, *ps1) // 5 false
	a[2], s[1] = 99, true
	fmt.Println(*pa2, *ps1) // 99 true
	ps0 := &[]string{"Go", "C"}[0]
	fmt.Println(*ps0) // Go

	m := map[int]bool{1: true}
	_ = m
	// The following lines fail to compile.
	/*
	_ = &[3]int{2, 3, 5}[0]
	_ = &map[int]bool{1: true}[1]
	_ = &m[1]
	*/
}
```

与大多数其他无法寻址的值（不能修改直接部分）不同，map元素值的直接部分是可以被修改的，但也只能作为一个整体进行修改（覆盖）。 对于大多数的元素类型来说，这不是个大问题。但是， 如果map类型的元素类型是数组类型或结构体类型，事情可能会变的有点违反直觉。

通过上篇文章， [value parts]()， 我们学习到每个结构体和数组类型都是又一个直接部分构成的。于是：

- 如果一个map的元素类型是结构体类型，我们不能单独地修改map元素中的某个字段。

- 如果一个map的元素类型是数组类型，我们不能单独地修改map中数组类型元素中的元素。

例子：

```go
package main

import "fmt"

func main() {
	type T struct{age int}
	mt := map[string]T{}
	mt["John"] = T{age: 29}
	ma := map[int][5]int{}
	ma[1] = [5]int{1: 789}

	// Accessments are ok
	fmt.Println(ma[1][1]) // 789
	fmt.Println(mt["John"].age) // 29

	// The following two modifications fail to compile.
	/*
	ma[1][1] = 123      // error: cannot assign to a[1][1]
	mt["John"].age = 30 // cannot assign to struct field
	                    // mt["John"].age in map.
	*/
}
```

如果相对上述例子中进行预期的修改操作，上述要修改的关联map元素首先需要存储到一个临时的变量当中， 然后按照需求修改这个临时变量， 然后用这个临时变量覆盖（重写）这个map的元素。 例如：

```go
package main

import "fmt"

func main() {
	type T struct{age int}
	mt := map[string]T{}
	mt["John"] = T{age: 29}
	ma := map[int][5]int{}
	ma[1] = [5]int{1: 789}

	t := mt["John"] // a temporary copy
	t.age = 30
	mt["John"] = t // overwrite it back

	a := ma[1] // a temporary copy
	a[1] = 123
	ma[1] = a // overwrite it back

	fmt.Println(ma[1][1], mt["John"].age) // 123 30
```

## 从数组和切片中派生切片

我们可以使用子切片语法从一个数组或切片中派生出一个新的切片。派生出的新的切片元素和它的基础数组（切片）由同一个底层连续的内存段托管。换句话说，派生出的新切片和基础切片（数组）共享一些连续的元素。

有两种子切片语法：（`baseContainer是一个数组或者切片`）

```go
baseContainer[low : high]       // two-index form
baseContainer[low : high : max] // three-index form
```

其中三索引形式等价于：

```go
baseContainer[low : high : cap(baseContainer)]
```

所以，两索引形式是三索引形式的一个特殊表现形式。实践中，两索引形式比三索引形式更加受欢迎。

注意，三索引形式在Go1.2版本之后才被支持的。

在一个子切片的表达式中，`low`， `high` 和 `max` 索引必须满足以下条件：

- `0 <= low <= high <= max <= cap(baseContainer)`

- `low <= len(baseContainer)`

索引如果不满足上述条件可能导致程序在编译器或运行时发生panic或错误， 这里也取决于基础容器的类型以及索引是否是常量。

注意：

- `high`索引可以大于`len(baseContainer)`的值， 但是不能大于`cap(baseContainer)`的值，如果`baseContainer`是一个切片，那么`len(baseContainer)`将小于`cap(baseContainer)`。

- 一个`nil`切片的子切片表达式（表达式中的索引都为0）不会导致程序panic。从`nil`切片中派生出的切片仍然是`nil`切片。

派生出的子切片的长度等于`high - low`，派生出的子切片的容量等于`max - low`。 派生出的子切片可能会比其基础切片要大，但是它的容量绝不会大于其基础切片。

实践中，为了简便，我们通常会省略子切片语法形式中的一些索引。省略的规则是：

- 如果`low`索引为0，那么可以被省略，无论是在两索引还是三索引形式中。

- 如果`high`索引等于`len(baseContainer)`，那么它也可以被省略， 但只能在两索引形式中。

- 在三索引形式中`max`不能被省略。

例如，下面的表达式都是相等的：

```go
baseContainer[0 : len(baseContainer)]
baseContainer[: len(baseContainer)]
baseContainer[0 :]
baseContainer[:]
baseContainer[0 : len(baseContainer) : cap(baseContainer)]
baseContainer[: len(baseContainer) : cap(baseContainer)]
```

一个使用子切片语法的完整例子：

```go
package main

import "fmt"

func main() {
	a := [...]int{0, 1, 2, 3, 4, 5, 6}
	s0 := a[:]
	s1 := s0[:]   // <=> s1 := s0[0:7] <=> s1 := s0
	s2 := s1[1:3] // <=> s2 := s1[1:3] <=> s2 := a[1:3]
	s3 := s2[2:6] // <=> s3 := s2[2:6] <=> s3 := s1[3:7]
	s4 := s0[3:5] // <=> s4 := s0[3:5:7]
	s5 := s0[3:5:5]
	s6 := append(s4, 77)
	s7 := append(s5, 88)
	s8 := append(s7, 66)
	s3[1] = 99
	fmt.Println(len(s2), cap(s2), s2) // 2 6 [1 2]
	fmt.Println(len(s3), cap(s3), s3) // 4 4 [3 99 77 6]
	fmt.Println(len(s4), cap(s4), s4) // 2 4 [3 99]
	fmt.Println(len(s5), cap(s5), s5) // 2 2 [3 99]
	fmt.Println(len(s6), cap(s6), s6) // 3 4 [3 99 77]
	fmt.Println(len(s7), cap(s7), s7) // 3 4 [3 4 88]
	fmt.Println(len(s8), cap(s8), s8) // 4 4 [3 4 88 66]
}
```

下面的图片描绘了上述例子中的切片（数组）的最终内存布局：

![](https://go101.org/article/res/slice-subslice.png)

从上图中我们可以看到`s7`和`s8`被一个不同于其他的底层内存段托管。其他切片的元素被同一个底层内存段`a`托管。

## 使用内建函数`copy`拷贝切片的元素

我们可以使用内建函数`copy`将一个切片的元素拷贝到另一个切片，这两个切片的类型不要求相同， 但是他们的元素类型必须是相同的。换句话说，`copy`函数的两个切片参数必须共享同一个底层类型。`copy`函数的第一个参数是目的切片， 第二个参数是源切片。这两个切片参数可以有元素重叠。`copy`函数返回被拷贝的元素个数， 这个数将是两个切片参数中长度较小的一个。

借助于子切片语法， 我们可使用`copy`函数在两个数组或切片中复制元素。

例子：

```go
package main

import "fmt"

func main() {
 type Ta []int
 type Tb []int
 dest := Ta{1, 2, 3}
 src := Tb{5, 6, 7, 8, 9}
 n := copy(dest, src)
 fmt.Println(n, dest) // 3 [5 6 7]
 n = copy(dest[1:], dest)
 fmt.Println(n, dest) // 2 [5 5 6]

 a := [4]int{} // an array
 n = copy(a[:], src)
 fmt.Println(n, a) // 4 [5 6 7 8]
 n = copy(a[:], a[2:])
 fmt.Println(n, a) // 2 [7 8 7 8]
}
```

实际上，`copy`函数并不是很重要。 我们可以使用内建函数`append`函数来实现它，尽管我们的实现不支持泛型。

```go
// Assume element type is T.
func Copy(dest, src []T) int {
	if len(dest) < len(src) {
		_ = append(dest[:0], src[:len(dest)]...)
		return len(dest)
	} else {
		_ = append(dest[:0], src...)
		return len(src)
	}
}
```

尽管`copy`函数在Go不太重要，但是在某些情况下，它却是非常方便的。

从另一个角度来说， `append`函数似乎也不那么重要，我们也可以使用`make`和`copy`去实现它。

注意，作为一个特例，`copy`函数能够用在[copy bytes from a string to a byte slice](https://go101.org/article/string.html#use-string-as-byte-slice)。

到Go1.10为止，`copy`函数的两个参数都不能为`nil`。

## 容器类型的元素迭代

在Go中， 可以使用`range`语法来迭代容器类型的元素：

```go
for key, element = range aContainer {
	// use key and element ...
}
```

这里的`key`和`element`被称作迭代变量。

赋值符号`=`可以是一个短变量声明符号`:=`，在这种情况下，两个迭代变量都是两个新声明的变量，这些变量只能在`for-range`代码块中是可见的。

和传统的`for`循环体一样，每个`for-range`循环体将会创建两个代码块，一个隐式的代码块和一个使用`{}`形式的显式代码块。显式的代码块嵌套在隐式的代码块内。

和`for`循环一样，`break`和`continue`语句也可以在`for-range`代码块中使用， 例子：

```go

package main

import "fmt"

func main() {
	m := map[string]int{"C": 1972, "C++": 1983, "Go": 2009}
	for lang, year := range m {
		fmt.Printf("%v: %v \n", lang, year)
	}

	a := [...]int{2, 3, 5, 7, 11}
	for i, prime := range a {
		fmt.Printf("%v: %v \n", i, prime)
	}

	s := []string{"go", "defer", "goto", "var"}
	for i, keyword := range s {
		fmt.Printf("%v: %v \n", i, keyword)
	}
}
```

如果上述例子中的迭代变量`i`是在`for-range`代码块外面声明的， 那么它的类型必须是`int`。

`for-range`代码块语法有几个变种：

```go
// Ignore the key iteration variable.
for _, element = range aContainer {
	// ...
}

// Ignore the element iteration variable.
for key, _ = range aContainer {
	element = aContainer[key]
	// ...
}

// The element iteration variable is omitted.
// This form is equivalent to the last one.
for key = range aContainer {
	element = aContainer[key]
	// ...
}

// Ignore both the key and element iteration variables.
for _, _ = range aContainer {
	// This variant is not much useful.
}

// Both the key and element iteration variables are omitted.
// This form is equivalent to the last one.
for range aContainer {
	// This variant is not much useful.
}

```

如果迭代一个`nil`的切片或数组，那么迭代出的键值对的个数为0。

迭代一个map有几个细节需要注意：

- 对于map来说，不保证本此迭代后条目的顺序于下次迭代时相同，尽管这两次迭代时，map未发生修改。Go规范中， map的条目顺序是随机的。

- 如果在迭代期间移除了尚未迭代到的键值对，那么该条目将不会在该迭代中出现。

- 如果在迭代期间创建了一个新的条目，那么该条目将会在本次迭代中出现。

如果保证没有在其他的goroutine中操作map`m`, 那么下面的代码将保证会删除map`m`中所有键值对：

```go
for key := range m {
	delete(m, key)
}
```

另外，数组和切片元素可以通过传统的`for`循环进行迭代：

```go
for i := 0; i < len(anArrayOrSlice); i++ {
	element := anArrayOrSlice[i]
	// ...
}
```

对于一个`for-range`的代码块：

```go
for key, element = range aContainer {...}
```

有三个重要的事实：

1. 被迭代的容器是`aContainer`的拷贝。然而，需要注意的是，只有`aContainer`的[直接部分](https://go101.org/article/value-part.html#about-value-copy)被[拷贝](https://go101.org/article/value-part.html#about-value-copy)。这个被拷贝的容器时匿名的，所以无法修改它。

    1. 如果`aContainer`是一个数组，那么在迭代期间对于数组元素的修改不会反映到迭代变量中来。原因是数组的拷贝不与源数组共享元素。

    2. 如果`aContainer`是一个切片或者map，那么迭代期间对元素的修改会反映到迭代变量中来。 原因是切片（或map）的拷贝会和源切片（或map）共享所有的元素（条目）。

2. `aContainer`的拷贝的键值对将会在每一次迭代时被分配（拷贝）到迭代变量中，所以修改迭代变量的**直接部分**不会反映到存储在`aContainer`中的元素中。

3. 所有的键值对会被分配（拷贝）到**相同**的迭代变量中。

一个解释第1点和第2点的例子：

```go
package main

import "fmt"

func main() {
	type Person struct {
		name string
		age  int
	}
	persons := [2]Person {{"Alice", 28}, {"Bob", 25}}
	for i, p := range persons {
		fmt.Println(i, p)
		// This modification has no effects on the iteration,
		// for the ranged array is a copy of the persons array.
		persons[1].name = "Jack"
		// This modification has not effects on the persons array,
		// for p is just a copy of a copy of one persons element.
		p.age = 31
	}
	fmt.Println("persons:", &persons)
}

// output:
/*
0 {Alice 28}
1 {Bob 25}
persons: &[{Alice 28} {Jack 25}]
*/
```

如果我们将上述例子中的数组替换成切片，然后我们在迭代期间修改切片将会影响到迭代。但是修改迭代变量仍然对被迭代的切片没有任何影响。

```go
package main

import "fmt"

func main() {
	type Person struct {
		name string
		age  int
	}
	persons := []Person {{"Alice", 28}, {"Bob", 25}} // a slice
	for i, p := range persons {
		fmt.Println(i, p)
		// Now this modification has effects on the iteration.
		persons[1].name = "Jack"
		// This modification still has no any real effects.
		p.age = 31
	}
	fmt.Println("persons:", &persons)
}

// output
/*
0 {Alice 28}
1 {Jack 25}
persons: &[{Alice 28} {Jack 25}]
*/
```

下面是一个解释第2点和第3点的例子：

```go
package main

import "fmt"

func main() {
	langs := map[struct{ dynamic, strong bool }]map[string]int{
		{true, false}:  {"JavaScript": 1995},
		{false, true}:  {"Go": 2009},
		{false, false}: {"C": 1972},
	}
	// The key type and element type of this map are both
	// pointer types. Some weird, just for education purpose.
	m0 := map[*struct{ dynamic, strong bool }]*map[string]int{}
	for category, langInfo := range langs {
		m0[&category] = &langInfo
		// This following line has no real effects on langs.
		category.dynamic, category.strong = true, true
	}
	m1 := map[struct{ dynamic, strong bool }]map[string]int{}
	for category, langInfo := range m0 {
		m1[*category] = *langInfo
	}
	// m0 and m1 both contain only one entry.
	fmt.Println(len(m0), len(m1), m1)
	for category, langInfo := range langs {
		fmt.Println(category, langInfo)
}

// output:
/*
1 1 map[{true true}:map[C:1972]]
{true false} map[JavaScript:1995]
{false true} map[Go:2009]
{false false} map[C:1972]
*/
```

拷贝切片和map的代价很小， 但是拷贝一个超大的数组的代价也会非常的大。 所以，通常来说，迭代一个大型的数组并不是一个好主意。我们可以迭代一个从数组派生出的切片，或者迭代一个大型数组的指针。

对于一个数组或切片来说，如果它的元素类型的大小很大， 那么，通常来说，那么在每次的迭代中使用第二个迭代变量来存储迭代的值可能不是一个好主意。对于这样的数组和切片， 我们应该使用**单迭代变量**`for-range`迭代形式的变种或者使用传统的`for`循环来迭代它们的元素。在下面的例子中，函数`fa`中的循环效率比函数`fb`循环效率低得多。

```go
type Buffer struct {
	start, end int
	data       [1024]byte
}

func fa(buffers []Buffer) int {
	numUnreads := 0
	for _, buf := range buffers {
		numUnreads += buf.end - buf.start
	}
	return numUnreads
}

func fb(buffers []Buffer) int {
	numUnreads := 0
	for i := range buffers {
		numUnreads += buffers[i].end - buffers[i].start
	}
	return numUnreads
}
```

## 使用数组指针作为数组

很多时候，我们可以使用数组指针来代替数组。

我们可以遍历指向数组的指针来迭代数组的元素。对于大型数组来说，这种方式很高效，拷贝一个指针比拷贝一个大型数组高效的多。下面的例子中， 两个循环代码块的效率是相同的。

```go
package main

import "fmt"

func main() {
	var a [100]int

	for i, n := range &a { // copying a pointer is cheap
		fmt.Println(i, n)
	}

	for i, n := range a[:] { // copying a slice is cheap
		fmt.Println(i, n)
	}
}
```

当迭代一个`nil`的数组指针会panic。下面的例子中， 最后一个迭代的例子将会产生panic。

```go
package main

import "fmt"

func main() {
	var p *[5]int // nil

	for i, _ := range p { // okay
		fmt.Println(i)
	}

	for i := range p { // okay
		fmt.Println(i)
	}

	for i, n := range p { // panic  because p is a nil pointer for array
		fmt.Println(i, n)
	}
}
```

数组指针也可以用作索引数组元素，通过`nil`数组指针来索引数组元素会产生panic。

```go
package main

import "fmt"

func main() {
	a := [5]int{2, 3, 5, 7, 11}
	p := &a
	p[0], p[1] = 17, 19
	fmt.Println(a) // [17 19 5 7 11]
	p = nil
	p[0] = 31 // panic
}
```

我们也可以从一个数组指针派生出一个切片。从`nil`数组指针派生切片将会产生panic。

```go
package main

import "fmt"

func main() {
	pa := &[5]int{2, 3, 5, 7, 11}
	s := pa[1:3]
	fmt.Println(s) // [3 5]
	pa = nil
	s = pa[1:3] // panic
}
```

我们也可以将数组指针作为参数传递给内建函数`len`和`cap`。 传递`nil`数组指给这两个函数不会产生panic。

## `memclr`优化

假设`t0`是类型`T`零值的字面量表示，并且`a`是一个元素类型为`T`的数组，那么Go编译器会将下面的单一迭代变量`for-range`循环代码块：

```go
for i := range a {
  a[i] = t0
}
```

转换成一个[internal `memclr` call](https://github.com/golang/go/issues/5373)，通常比逐个重置每个元素更快。

当被遍历的容器是切片时，上述的优化同样可以工作。但是， 当被遍历的容器类型是数组指针时，这种优化却不能工作。
所以，如果你想重置一个数组，就不能迭代它的指针。实际上，推荐的方式是迭代从这个数组派生出的切片。像下面这样：

```go
s := a[:]
for i := range s {
	s[i] = t0
}
```

这么建议的原因是因为可能其他的Go编译器可能不进行上述优化，并且正如上说提到的，迭代一个数组将会产生一个数组的拷贝。

## 对内建函数`len`和`cap`的调用可能会在编译期间被求值

如果传递给内建函数`len`或`cap`的参数是一个数组或数组指针，那么这个调用将会在编译期被求值，并且求值的结果是一个类型化的常量（typed constant），默认类型为内置类型`int`。这个结果会被绑定一个命名常量上。例子：

```go
package main

import "fmt"

var a [5]int
var p *[7]string

// N and M are both typed constants.
const N = len(a)
const M = cap(p)

func main() {
	fmt.Println(N) // 5
	fmt.Println(M) // 7
}
```

## 单独修改切片的长度和容量属性

正如上面提到的，通常，一个切片的长度和容量值不能被单独地修改。一个切片值只能通过为其分配另一个切片值来进行整体地覆盖。但是，我们可以通过反射机制来单独修改切片的长度和容量值。

例子：

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	s := make([]int, 2, 6)
	fmt.Println(len(s), cap(s)) // 2 6

	reflect.ValueOf(&s).Elem().SetLen(3)
	fmt.Println(len(s), cap(s)) // 3 6

	reflect.ValueOf(&s).Elem().SetCap(5)
	fmt.Println(len(s), cap(s)) // 3 5
}
```

传递给函数`reflect.SetLen`的参数必须要小于等于参数切片`s`当前的容量值。传递给函数`reflect.SetCap`的参数必须要不小于参数切片`s`当前的长度值且要大于参数切片`s`当前的容量值。

反射操作是非常低效的， 甚至比切片分配赋值操作还慢。

## 切片的其他操作

Go内建方法中不支持切片其他的操作，比如切片克隆， 元素删除和插入。 但我们可以组合各种内建方法来完成这些操作。

本节中，下个例子里，我们假设`s`是需要操作的切片，`T`是该切片元素的类型， `t0`类型`T`字面量表示的零值。

### 克隆切片

到目前的Go版本为止（Go1.10）， 克隆切片最好的方式如下例所示：

```go
sClone := append(s[:0:0], s...)
```

如果切片足够大，那么上述的方式则比下述的方式高效的多：

```go
sClone := make([]T, len(s))
copy(sClone, s)
```

第二种方式不如第一种方式完美，如果`s`是一个`nil`切片，那第二种会导致一个非零切片克隆。

### 删除切片元素某一段

上面提到了切片的元素连续地存储在内存中，并且切片的任何两个相邻的元素之间没有间隙。所以当一个元素删除时：

- 如果必须保留元素顺序，则将被删除元素之后的所有元素向前移动。

- 如果没有不要保留元素顺序，那我们可以将切片最后一个元素移动到被删除的索引处。

下面的例子中，我们假设`from`和`to`是两个合法的索引，`from`不大于`to`，且索引`to`是独占的。

```go
// way 1 (preserve element orders):
s = append(s[:from], s[to:]...)

// way 2 (preserve element orders):
s = s[:from + copy(s[from:], s[to:])]

// Don't preserve element orders:
copy(s[from:to], s[len(s)+from-to:])
s = s[:len(s)+from-to]
```

如果切片元素中包含指针，我们应该重置老切片中尾部元素来避免内存泄漏。

```go
// "len(s)+to-from" is the old slice length.
temp := s[len(s):len(s)+to-from]
for i := range temp {
	temp[i] = t0
}
```

如上面章节提到的，`for-range`循环代码块会被Go编译器当作一个`memclr`调用优化。

### 删除一个切片元素

删除一个切片元素和删除切片元素段方式一样，而且更加简单。

下面的例子中，假设`i`是要删除的元素的索引，且`i`是一个合法的索引。

```go
// Way 1 (preserve element orders):
s = append(s[:i], s[i+1:]...)

// Way 2 (preserve element orders):
s = s[:i + copy(s[i:], s[i+1:])]

// There will be len(s)-i-1 elements being copied in
// in either of the above two ways.

// Don't preserve element orders:
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```

如果切片元素包含了指针，那么在删除操作之后，我们应该重置老切片中最后一个元素以避免内存泄漏。

```go
s[len(s):][0] = t0
```

### 按条件删除切片元素

有时候我们可能会有按照某个条件删除一些切片元素的需求。

```go
// Assume T is a small-size type.
func DeleteElements(s []T, shouldKeep func(T) bool, clear bool) []T {
	result := make([]T, 0, len(s))
	for _, v := range s {
		if shouldKeep(v) {
			result = append(result, v)
		}
	}
	if clear { // avoid memory leaking
		temp := s[len(result):]
		for i := range temp {
			temp[i] = t0 // t0 is a zero value literal of T.
		}
	}
	return result
}
```

### 将一个切片插入到另一个切片中

假设插入的位置是一个合法索引`i`，并且要插入的切片称为`elements`。

```go
// One-line implementation:
s = append(s[:i], append(elements, s[i:]...)...)

// A more efficient but more verbose way:
if cap(s)-len(s) >= len(elements) {
	s = s[:len(s)+len(elements)]
	copy(s[i+len(elements):], s[i:])
	copy(s[i:], elements)
} else {
	x := make([]T, 0, len(elements)+len(s))
	x = append(x, s[:i]...)
	x = append(x, elements...)
	x = append(x, s[i:]...)
	s = x
}

// Push:
s = append(s, elements...)

// Unshift:
s = append(elements, s...)
```

### 插入一定的切片元素

插入一定量的切片元素和插入一整个切片操作形式一样。我们可以使用切片复合字面量构造一个切片，其中包含要插入的元素，然后使用上述方法插入新切片。

### 特殊的删除和插入操作： Push Front/Back, Pop Front/Back

假设推入和弹出的元素是`e`，切片`s`至少有一个元素。

```go
// Pop front (shift):
s, e = s[1:], s[0]
// Pop back:
s, e = s[:len(s)-1], s[len(s)-1]
// Push front:
s = append([]T{e}, s...)
// Push back
s = append(s, e)
```

### 关于上述例子中各种切片骚操作

实际上，需求是多种多样的。对于某些情况，上述示例中方式都不是最有效的方式。有时，上述方法可能无法满足中的某些特定的要求。所以，请学以致用（灵活应用）。这可能是Go不支持将上面各种骚操作以内建方式引入的原因。

## 使用map模拟set（集合）

Go不支持内建类型`set`。然而，用map来模拟一个set是十分方便的。实践中，我们经常使用map类型`map[K]struct{}`来模拟一个set类型。该map的元素`struct{}`类型的大小为0, 这种map类型的值元素不占用任何内存空间。

## 容器相关的操作在内部是不同步的

需要注意的是，所有容器的操作在内部来看都不是同步的。如果不使用任何数据同步技术，多个goroutine可以同时读取容器，但是多个goroutine同时对容器进行操作并且至少有一个goroutine修改容器（写操作）是不可行的。后一种情况会导致数据竞争，甚至会导致线程（goroutine） panic。我们必须在操作容器时手动同步。更多数据同步的内容请参考：[data synchronizations](https://go101.org/article/concurrent-synchronization-overview.html)。
