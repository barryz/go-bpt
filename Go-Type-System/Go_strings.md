> 译自： [Strings in Go](https://go101.org/article/string.html)

# Go字符串

和其他编程语言类似， 字符串在Go语言里也是一个非常重要的类型。 本文将会列举关于Go字符串的相关内容。

## 字符串类型的内部结构

对于标准的Go编译器来说， 字符串类型的内部结构声明如下：

```go
type _string struct {
	elements *byte // 底层字节数组
	len      int   // 长度
}
```

通过如上的声明， 我们知道一个字符串实际上是一个字节序列的封装。

需要知道的是， 在Go里， `byte` 是内建类型 `uin8` 的别名。

## 关于字符串一些事实

  - 字符串值可以被用作常量（以及布尔值和所有类型的数字值）。

  - Go支持两种[字符串的字面量声明](https://go101.org/article/basic-types-and-value-literals.html#string-literals)方式， 双引号形式和反引号形式。

  - 字符串的零值是空字符串， 可以用 **""** 和 **\`\`** 表示。

  - 可以通过 `+` 或者 `+=` 来拼接字符串。

  - 字符串类型都是可比较的（使用`==` 或 `!=` 操作符）。 而且和整型和浮点型类型一样， 两个相同类型的字符串可以使用操作符 `>` `<` `>=` `<=` 进行比较。 比较两个字符串时，事实上是比较这两个字符串的底层字节序列， 逐个字节的比较。 如果一个字符串是另一个字符串的前缀且另一个字符串长度较长，则另一个字符串则被视为较大的字符串。

例子：

```go
package main

import "fmt"

func main() {
	const World = "world"
	var hello = "hello"

	// 字符串拼接
	var helloWorld = hello + " " + World
	helloWorld += "!"
	fmt.Println(helloWorld) // hello world!

	// 比较字符串
	fmt.Println(hello == "hello")   // true
	fmt.Println(hello > helloWorld) // false
}
```

更多关于Go中字符串类型和值的细节：

  - 和Java一样， 字符串的内容是不可变的。一旦被创建， 他们的值就不能被修改。 字符串值得长度也是不可变的。 只能通过为其指定另一个字符串的值来覆盖整个字符串的值。

  - 字符串类型没有任何方法（和Go里其他内建的类型一样）， 但是：

    - 标准库 `strings` 包含了很多字符串有关的实用方法。

    - 内建函数 `len` 可以用来获取字符串的长度。

    - 和array、slice、map访问元素方法一样， `aString[i]` 表达式可以用来访问存储在字符串中的某个字符。`aString[i]` 表达式是不可寻址的， 换句话说， `aString[i]` 的值是不可变的。

    - 和array、slice，map的[子切片语法](https://go101.org/article/container.html#subslice)一样，`aString[start:end]` 语法可用来获取字符串的子字符串， 这里的 `start` 和 `end` 是字符串底层字节数的索引。

  - 对于标准的Go编译器来说， 字符串赋值操作中的目标字符串变量和源字符串值在内存中共享同一个底层字节序列。 子字符串表达式 `aString[start:end]` 同样也和原始的 `aString` 共享同一个底层字节序列。

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var helloWorld = "hello world!"

	var hello = helloWorld[:5] // 子字符串
	// 104 is the ascii code (and Unicode) of char 'h'
	fmt.Println(hello[0])         // 104
	fmt.Printf("%T \n", hello[0]) // uint8
	// hello[0] = 'H'             // error: hello[0] 是不可变得
	// fmt.Println(&hello[0])     // error: hello[0] 是不可寻址的

	fmt.Println(len(hello), len(helloWorld))          // 5 12
	fmt.Println(strings.HasPrefix(helloWorld, hello)) // true
```

## 字符串的编码和符文(rune)

在Go里， 所有的字符串都是采用UTF-8编码。 UTF-8编码字符串中的基本单元被称为代码点。大多数的代码点可以被视为我们日常生活中的字符。但是对于某些字符，都是有好几个代码点组成的。

代码点在Go中使用符文（rune）值表示。内建类型 `rune` 是类型 `int32` 的别名。

在编译时，字符串字面量里非法的UFT-8符文值会导致编译错误。 在运行时， Go的运行时不能阻止存储在字符串中的某些字节是UTF-8非法的。换言之， 在运行时， 一个字符串可能是非法的UTF-8编码的。

如上文所述， 每个字符串实际上一个字节序列的封装。 所以字符串中的每个符文（rune）将会存储一个或多个字节（最多四个）。 例如， 每个英文代码点会以一个字节存储在Go的字符串里， 中文代码点存储在Go字符串则需要3个字节。

## 字符串相关的类型转换

在之前的文章： [常量和变量](https://go101.org/article/constants-and-variables.html#explicit-conversion)中， 我们了解到整型类型可以显式地转换成字符串类型，（反过来则无法直接转换）。下面将介绍更多关于字符串转换的相关规则：

1. 一个字符串值可以显式地转换成一个字节切片（byte slice）， 反之亦然。 转换成一个底层类型是`[]byte`的字节切片。

2. 一个字符串值可以显式地转换成一个符文切片（rune slice）， 反之亦然， 转换成一个底层类型是`[]rune`的符文切片。

在符文切片转换到字符串的过程中， 每个切片元素（rune值） 将被转换为其各自的UTF-8编码的字节序列形式。如果切片元素值超出了有效的Unicode代码点的范围， 则这个元素将被视为 `0xFFFD`， Unicode替换字符的代码点。 `0xFFFD` 用UTF-8编码表示为 `\xef\xbf\xdb`，占三个字节。

当一个字符串转换成符文切片时， 存储在字符串中的字节将会被视为由许多Unicode代码点组成的连续UTF-8编码的字节序列。错误的UTF-8编码形式将被转换成符文值 `0xFFFD`。

当一个字符串转换成字节切片时，生成的字节切片（byte slice）只是该字符串底层字节序列的深拷贝版本。 当一个字节切片转换成字符串时， 生成的结果字符串的底层字节序列同样也是该字节切片的深拷贝。 在类似的类型转换中深拷贝行为都需要进行内存的分配操作。 深拷贝必不可少的原因在于切片元素是可变的， 但是存储在字符串中的字节却是不可变的， 因此字节切片和字符串不能共享同一个字节元素。

总之， 字符串和字节切片相互的转换中需要注意的是：

- 允许非法的UTF-8编码字节， 并保持其不变。

- 标准Go编译器对这种转换中的某些特殊情况做了一定的优化， 因此有可能不需要深拷贝。

字节切片和符文切片之间还不能直接进行转换， 我们可以通过以下的方式完成转换：

- 使用字符串作为跳跃点（hop）。 这种方式非常方便但缺乏效率。

- 使用标准库 `unicode/utf8` 中的相关函数。

- 可以使用标准库：[bytes](https://golang.org/pkg/bytes/#Runes)中的 `Runes` 函数来将一个 `[]byte` 转换成 `[]rune`。

```go
package main

import (
	"bytes"
	"unicode/utf8"
)

func Runes2Bytes(rs []rune) []byte {
	n := 0
	for _, r := range rs {
		n += utf8.RuneLen(r)
	}
	n, bs := 0, make([]byte, n)
	for _, r := range rs {
		n += utf8.EncodeRune(bs[n:], r)
	}
	return bs
}

func main() {
	s := "Color Infection is a good game."
	bs := []byte(s) // string -> []byte
	s = string(bs)  // []byte -> string
	rs := []rune(s) // string -> []rune
	s = string(rs)  // []rune -> string
	rs = bytes.Runes(bs) // []byte -> []rune
	bs = Runes2Bytes(rs) // []rune -> []byte
}
```

## 字符串和字节切片相互转换之编译器优化

上文提到过， 字符串和字节切片之间转换底层的字节序列将会被深拷贝。当前版本的标准Go编译器会对这些转换做一些优化措施来避免深拷贝，具体包括：

- 从字符串转换至字节切片， 转换行为会发生在 `for-range` 中的 关键字 `range` 之后的。

- 从字节切片转换至字符串， 使用byte作为map的key的。

- 从字节切片转换至字符串， 用作于比较操作的。

- 从字节切片转换至字符串，用作字符串拼接的， 且至少有一个用于拼接的字符串值为非空的字符串常量。

例子：

```go
package main

import "fmt"

func main() {
	var str = "world"
	// Here, the []byte(str) conversion will not copy
	// the underlying bytes of str.
	for i, b := range []byte(str) {
		fmt.Println(i, ":", b)
	}

	key := []byte{'k', 'e', 'y'}
	m := map[string]string{}
	// Here, the string(key) conversion will not copy the bytes
	// in key. The optimization will be still made even if key
	// is a package-level variable.
	m[string(key)] = "value"
	fmt.Println(m[string(key)]) // value
}
```

另一个例子：

```go
package main

import "fmt"
import "testing"

var s string
var x = []byte{1024: 'x'}
var y = []byte{1024: 'y'}

func fc() {
	// None of the below 4 conversions will
	// copy the underlying bytes of x and y.
	if string(x) != string(y) {
		s = (" " + string(x) + string(y))[1:]
	}
}

func fd() {
	// Only the two conversions in the comparison
	// will not copy the underlying bytes of x and y.
	if string(x) != string(y) {
		s = string(x) + string(y)
	}
}

func main() {
	fmt.Println(testing.AllocsPerRun(1, fc)) // 1
	fmt.Println(testing.AllocsPerRun(1, fd)) // 3
}
```

## 字符串中的 for-range

`for-range` 循环操作也适用于字符串，但需要注意的是，  `for-range` 将在字符串中迭代Unicode代码点（作为rune值）而不是迭代每个字节。 字符串中错误的UTF-8编码表示将会被解释成符文值 `0xFFFD`。

例子：

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ汉字"
	for i, rn := range s {
		fmt.Printf("%v: 0x%x %v \n", i, rn, string(rn))
	}
}
/*
Output:
0: 0x65 e
1: 0x301 ́
3: 0x915 क
6: 0x94d ्
9: 0x937 ष
12: 0x93f ि
15: 0x61 a
16: 0x3c0 π
18: 0x6c49 汉
21: 0x5b57 字
*/
```

需要注意的地方：

1. 迭代时索引值不一定是连续的， 因为一个代码点可能需要多个字节来表示。

2. 第一个字符， `é` 由2个符文值组成（总共3个字节）。

3. 第二个字符，`क्षि` 由4个符文值组成（总共12个字节）。

4. 英文字符，`a` 由1个符文值组成（总共1个字节）。

5. 字符，`π` 由1个符文值组成（总共2个字节）。

6. 每两个中文字符，`你好` 由1个符文值组成（总共3个字节）。

那么该如何在字符串中迭代字节呢？  请看下面的例子：

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ汉字"
	for i := 0; i < len(s); i++ {
		fmt.Printf("The byte at index %v: 0x%x \n", i, s[i])
	}
}
```

如上述例子所看到的， `len(s)` 返回字符串中字节数数量。 时间复杂度为 `O(1)` 。 那么如何获取字符串中的符文数数量呢？ 使用 `for-range` 语句迭代且通过计数方式得到符文数量是一种方式， 使用标准库 `unicode/utf8` 中的 [RuneCountInString](https://golang.org/pkg/unicode/utf8/#RuneCountInString)函数则是另一种方式。两种方式的效率几乎相同。 时间复杂度均为 `O(n)`。

当然， 我们也可以利用上面提到的编译器优化措施来迭代字符串中的字节。 对于标准的Go编译器来说， 这种方式比上面两种方式会更有效。

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ汉字"
	for i, b := range []byte(s) { // here, the underlying bytes are not copied.
		fmt.Printf("The byte at index %v: 0x%x \n", i, b)
	}
}
```

## 字符串拼接方法汇总

除了使用操作符 `+` 来拼接字符串外， 还可以使用如下的函数或者方法拼接字符串：

- 标准库 `fmt` 包中的 `Sprintf` `Sprint` `Sprintln` 可以用来拼接任何类型的值，包括字符串。

- 标准库 `strings` 包中的 `Join` 函数。

- 标准库 `bytes` 包中的 `Buffer` 函数，（或内建函数 `copy`）可以用来构建字节切片，之后可以转换成字符串。

- 从Go1.10开始， 标准库 `strings` 中的 `Builder` 函数可以用来构建字符串。 此方法可以避免生成的结果字符串的底层字节序列的副本。 效率比较高。

标准的Go编译器针对使用操作符 `+` 拼接字符串做了一些优化。 所以通常来说， 在提前获知需拼接字符串长度的前提下，使用 `+` 来拼接字符串是非常方便和高效的。


## 甜点： 使用字符串作为字节切片

从之前关于[array, slices, maps](https://go101.org/article/container.html)相关的文章中， 我们知道， 可以使用内建函数 `copy`  和 `append` 来复制、追加切片元素。 实际上， 作为一个特殊的例子， 如果这两个函数的第一个参数的类型是一个字节切片， 第二个参数则可以是一个字符串，（如果调用 `append()` ， 第二个字符串参数则需要跟三个点 `...` ）。 换言之， 一个字符串可以被当作一个字节切片使用。

例子：

```go
package main

import "fmt"

func main() {
	hello := []byte("Hello ")
	world := "world!"

	// helloWorld := append(hello, []byte(world)...) // the normal way
	helloWorld := append(hello, world...)            // the sugar way
	fmt.Println(string(helloWorld))

	helloWorld2 := make([]byte, len(hello) + len(world))
	copy(helloWorld2, hello)
	// copy(helloWorld2[len(hello):], []byte(world)) // the normal way
	copy(helloWorld2[len(hello):], world)            // the sugar way
	fmt.Println(string(helloWorld2))
}
```

## 关于字符串比较的更多信息

上面已经提及到，两个字符串的比较实际上是这两个字符串底层字节序列的比较。大多数编译器都会对字符串比较进行一些优化.

- 如果比较的两个字符串不相等， 则两个字符串也必须不相等。

- 如果比较的两个字符串的底层字节序列的指针相等， 则比较结果与比较两个字符串的长度的结果相同。

因此， 对于两个相等的字符串， 它们之间比较产生的时间复杂度取决于它们的底层字节序列指针是否相等。如果两个相等的字符串值不共享底层的字节， 则比较这两个字符串的时间复杂度为 `O(n)`， 其中`n`是两个字符串的长度， 否则， 时间复杂度为 `O(1)`。

如上所述， 对于标准Go编译器来说， 在字符串赋值过程中， 目标字符串值和源字符串值将共享内存中的相同的底层字节序列。
因此在分配之后， 比较两个字符串的成本会变得非常小。


例子：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	bs := make([]byte, 1<<26)
	s0 := string(bs)
	s1 := string(bs)
	s2 := s1

	// s0, s1 and s2 are three equal strings.
	// The underlying bytes of s0 is a copy of bs.
	// The underlying bytes of s1 is also a copy of bs.
	// The underlying bytes of s0 and s1 are two different copies of bs.
	// s2 shares the same underlying bytes with s1.

	startTime := time.Now()
	_ = s0 == s1
	duration := time.Now().Sub(startTime)
	fmt.Println("duration for (s0 == s1):", duration)

	startTime = time.Now()
	_ = s1 == s2
	duration = time.Now().Sub(startTime)
	fmt.Println("duration for (s1 == s2):", duration)
}

/*
Output:
duration for (s0 == s1): 10.462075ms
duration for (s1 == s2): 136ns
*/
```

可以看到， 两次比较产生的时间复杂度差距十分明显！ 所以需要避免比较无法共享同一个底层字节序列的长字符串。
