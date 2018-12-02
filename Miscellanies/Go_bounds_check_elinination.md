> 译自：[Bounds Check Elimination](https://go101.org/article/bounds-check-elimination.html)。

# Go边界检查

从Go1.7开始，标准Go编译器使用了一个新的编译器后端，它基于静态单一分配形式（SSA)。SSA使得Go编译器通过[边界检查清除BCE](https://en.wikipedia.org/wiki/Bounds-checking_elimination)和[通用子表达式清除CSE](https://en.wikipedia.org/wiki/Common_subexpression_elimination)的优化来生成更加高效的代码。

本文将展示BCE是如何在Go1.7之后的编译器上工作的。

在Go1.7之后，可以通过运行`go build -gcflags="-d=ssa/check_bce/debug=1"`来展示哪些代码行需要检查边界。

## 例子1

```go
// example1.go
package main

func f1(s []int) {
	_ = s[0] // line 5: bounds check
	_ = s[1] // line 6: bounds check
	_ = s[2] // line 7: bounds check
}

func f2(s []int) {
	_ = s[2] // line 11: bounds check
	_ = s[1] // line 12: bounds check eliminatd!
	_ = s[0] // line 13: bounds check eliminatd!
}

func f3(s []int, index int) {
	_ = s[index] // line 17: bounds check
	_ = s[index] // line 18: bounds check eliminatd!
}

func f4(a [5]int) {
	_ = a[4] // line 22: bounds check eliminatd!
}

func main() {}
```

```bash
$ go build -gcflags="-d=ssa/check_bce/debug=1" example1.go
# command-line-arguments
./example1.go:5: Found IsInBounds
./example1.go:6: Found IsInBounds
./example1.go:7: Found IsInBounds
./example1.go:11: Found IsInBounds
./example1.go:17: Found IsInBounds
```

我们可以看到在函数`f2`的12、13行不需要进行边界检查，因为第11行的边界检查确保了第12、13行的索引不会越界。

但是在函数`f1`中，边界检查必须要全部都运行一遍。第5行的边界检查不能确保第6、7行的安全。且第6行的边界检查不能确保第7行的安全。

对于函数`f3`，Go1.7之后的编译器已经知道第二个`s[index]`是绝对安全，在第一个`s[index]`是安全的情况下。

Go1.7之后的编译器同样知道函数`f4`的唯一行也是安全的。

## 例子2

```go
// example2.go
package main

func f5(s []int) {
	for i := range s {
		_ = s[i]
		_ = s[i:len(s)]
		_ = s[:i+1]
	}
}

func f6(s []int) {
	for i := 0; i < len(s); i++ {
		_ = s[i]
		_ = s[i:len(s)]
		_ = s[:i+1]
	}
}

func f7(s []int) {
	for i := len(s) - 1; i >= 0; i-- {
		_ = s[i] // line 22: bounds check
		_ = s[i:len(s)]
	}
}

func f8(s []int, index int) {
	if index >= 0 && index < len(s) {
		_ = s[index]
		_ = s[index:len(s)]
	}
}

func f9(s []int) {
	if len(s) > 2 {
	    _, _, _ = s[0], s[1], s[2]
	}
}

func main() {}
```

```bash
$ go build -gcflags="-d=ssa/check_bce/debug=1" example2.go
# command-line-arguments
./example2.go:22: Found IsInBounds
```

哇哦，有这么多地方需要进行边界检查！

但是等下，为什么编译器认为第10行是安全的，但第15、23行不是安全的？是因为编译器不够智能还是因为有bug？

事实上，编译器在这里的判断是对的！为什么？原因是因为子切片的长度可能大于原始切片的长度。例如：

```go
package main

func main() {
	s0 := make([]int, 5, 10) // len(s0) == 5, cap(s0) == 10

	index := 8

	// In golang, for the subslice syntax s[a:b],
	// the valid rage for a is [0, len(s)],
	// the valid rage for b is [a, cap(s)].

	// So, this line is no problem.
	_ = s0[:index]
	// But, above line is safe can't ensure the following line is also safe.
	// In fact, it will panic.
	_ = s0[index:] // panic: runtime error: slice bounds out of range
}
```

所以语句**if `s[:index]` is safe thne s[index:] is also safe**当`len(s)`等于`cap(s)`时该语句值为`true`。这就是为什么例子3中的函数`fb`和`fc`同样需要边界检查。

Go1.7之后的编译器成功地检测了函数`fa`中`len(s)`是和`cap(s)`相等的。Go团队做了一件多么伟大的事！

## 例子4

```go
// example4.go
package main

import "math/rand"

func fb2(s []int, index int) {
	_ = s[index:] // line 7: bounds check
	_ = s[:index] // line 8: bounds check // not smart enough?
}

func fc2() {
	s := []int{0, 1, 2, 3, 4, 5, 6}
	s = s[:4]
	index := rand.Intn(7)
	_ = s[index:] // line 15 bounds check
	_ = s[:index] // line 16: bounds check eliminatd!
}

func main() {}
```

```bash
$ go build -gcflags="-d=ssa/check_bce/debug=1" example4.go
# command-line-arguments
./example4.go:7:7: Found IsSliceInBounds
./example4.go:15:7: Found IsSliceInBounds
```

在这个例子中，标准Go编译器成功地总结了

- 在函数`fc2`中的第8行代码是安全的。
- 在函数`fc1`中的第15行是安全的情况下，第16行也是安全的。

_（在Go SDK1.9之前，标准Go编译器不能检测第16行是否需要进行边界检查。）_

## 例子5：

尽管目前版本的标准Go编译器仍然不够智能到可以清除一些不必要的边界检查，但我们可以为编译器清除这些不必要的边界检查提供一些提示。

```go
// example5.go
package main

func fd(is []int, bs []byte) {
	if len(is) >= 256 {
		for _, n := range bs {
			_ = is[n] // line 7: bounds check, not smart enough
		}
	}
}

func fd2(is []int, bs []byte) {
	if len(is) >= 256 {
		is = is[:256] // line 14: to avoid bounds check at line 16
		for _, n := range bs {
			_ = is[n] // line 16: bounds check eliminatd!
		}
	}
}

func fe(isa []int, isb []int) {
	if len(isa) > 0xFFF {
		for _, n := range isb {
			_ = isa[n & 0xFFF] // line 24: bounds check, not smart enough
		}
	}
}

func fe2(isa []int, isb []int) {
	if len(isa) > 0xFFF {
		isa = isa[:0xFFF+1] // line 31: to avoid bounds check at line 33
		for _, n := range isb {
			_ = isa[n & 0xFFF] // line 33: bounds check eliminatd!
		}
	}
}

func ff(s []int) []int {
	s2 := make([]int, len(s))
	for i := range s {
		s2[i] = -s[i] // line 41: bounds check, not smart enough
	}
	return s2
}

func ff2(s []int) []int {
	s2 := make([]int, len(s))
	s2 = s2[:len(s)] // line 48: to avoid bounds check at line 50
	for i := range s {
		s2[i] = -s[i] // line 50: bounds check eliminatd!
	}
	return s2
}

func main() {}
```

```bash
$ go build -gcflags="-d=ssa/check_bce/debug=1" example5.go
# command-line-arguments
./example5.go:7: Found IsInBounds
./example5.go:24: Found IsInBounds
./example5.go:41: Found IsInBounds
./example5.go:48: Found IsSliceInBounds
```

## BCE的一个使用案例：高效切片比较

我们都知道在Go中切片类型不支持比较。想要比较两个切片，我们必须自己写比较的代码，或使用`reflect`标准库中的`DeepEqual`函数。但是`reflect.DeepEqual`函数太慢了。

下面是一个自定义的切片比较函数。

```go
func CompareSlices_General(a, b []int) bool {
	if len(a) != len(b) {
		return false
	}

	if (a == nil) != (b == nil) {
		return false
	}

	for i, v := range a {
		if v != b[i] { // here bounds check is needed for b[i]
			return false
		}
	}

	return true
}
```

仍然需要在`for-range`循环体中的`b[i]`上做边界检查以避免索引`i`越界。我们可以略微修改下代码以避免边界检查。

```go
func CompareSlices_BCE(a, b []int) bool {
	if len(a) != len(b) {
		return false
	}

	if (a == nil) != (b == nil) {
		return false
	}

	b = b[:len(a)] // this line is the key.
	for i, v := range a {
		if v != b[i] { // no bounds check for b[i] now!
			return false
		}
	}

	return true
}
```

让我们[基准测试对比这两个函数](https://github.com/go101/go-benchmarks/blob/master/bce/bce_test.go)，结果是：

```bash
Benchmark_SliceComparison/General-4         	 5000000	       287 ns/op
Benchmark_SliceComparison/BCE-4             	 5000000	       251 ns/op
```

BCE的版本比普通的版本快了约12.5%，真的不算太坏。

## 总结

尽管标准Go编译器中的BCE特性不算太完美，但它确实已经能够覆盖很多的场景。毫无疑问Go编译器在之后的版本肯定会做的更好。感谢Go团队增加了如此美妙的特性！

## 引用：

1. [gBounds Checking Elimination](https://docs.google.com/document/d/1vdAEAjYdzjnPA9WDOQ1e4e05cYVMpqSxJYZT33Cqw2g)
2. [Utilizing the Go 1.7 SSA Complier](https://klauspost-talks.appspot.com/2016/go17-compiler.slide)
