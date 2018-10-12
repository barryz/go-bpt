# Go 值拷贝内存消耗

Go的程序里会频繁地发生值拷贝的现象. 其中值赋值, 参数传递, channel的发送和接受值都会涉及到值赋值.


## 值大小(size)

值的大小意味着该值直接占用内存的字节数, 值的间接底层的部分不会影响值的大小.

在Go里面, 如果两个值的类型(type)属于同一个类型([kind](https://go101.org/article/type-system-overview.html#type-kinds)), 并且如果这个类型(type)的种类(kind)不是string kind, interface kind,  array kind 和 struct kind, 那么这两个值的大小永远相等.


实际上, 对于标准的Go编译器来说, 两个string的值的大小也是恒等的. 两个interface的值也是恒等的.

对于标准的Go编译器来说, 相同类型(type)的值始终具有相同的值大小. 因此, 通常来说, 我们将值的大小成为值的类型(type)的大小.

数组(array)类型的大小取决于数组元素类型的大小和数组类型的长度. 数组类型的大小是数组元素类型的大小和数组长度的乘积.

结构体类型的大小取决于它的所有字段. 因为在两个相邻的字段之间可能会存在一些填充([padding bytes](https://go101.org/article/memory-layout.html#size-and-padding))字段, 结构体类型的大小通常不会小于所有结构体字段相应类型大小之和.

下表列出了所有种类(kind)的类型大小值. 在此表中, 一个字(word)表示一个本地字, 在32位的OS中为4个字节,在64位的OS中为8个字节.

| Type | Value Size For The Standard Go Compiler(v1.10) | Requirement By Go Specification |
|-----|-----|------|
|bool|1 byte |not specified|
|int8, uint8(byte)|1 byte|1 byte|
|int16, uint16|2 bytes|2 bytes|
|int32(rune), uint32, float32|4 bytes|4 bytes|
|int64, uint64, float64, complex64|8 bytes|8 bytes|
|complex128|16 bytes|16 bytes|
|int, uint|1 word|architecture dependent, either 4 or 8 bytes|
|uintptr|1 word|large enough to store the uninterpreted bits of a pointer value|
|string|2 words|not specified|
|pointer|1 word|not specified|
|slice|3 words|not specified|
|map|1 word|not specified|
|channle|1 word|not specified|
|function|1 word|not specified|
|interface|2 words|not specified|
|struct|the sum of sizes of all fields + number of padding bytes|a struct type has size zero if it contains no fields that have a size greater than zero|
|array|(element value size) * (array length)|an array type has size zero if its element type has zero size|

## 值拷贝内存消耗

一般来说, 值复制的成本开销和值的大小成正比关系. 但是, 值大小并不是计算值复制的唯一因素. 不同的CPU架构可以专门针对具有特定大小的值进行值复制的优化. 实践中， 我们可以将size小于四个words的值视为小尺寸的值. 复制小尺寸的值的成本会很小.

对于标准的Go编译器来说, 除了大型的结构体和数组类型, 大部分Go里面的值都是小尺寸值.

为了避免在参数传递和channel发送/接受操作中产生的大值的复制成本, 我们应该避免使用过大的结构体或数组类型作为函数或者方法的参数类型(也包括方法的接收者类型), 并且也应当避免使用过大的结构体或数组类型作为channel的元素类型. 我们可以使用这些大尺寸类型的指针类型来传参或者channel发送和接收.

从另一方面来说, 我们也应当要考虑在运行时使用过多的指针类型造成的GC压力. 因此, 是否应该使用大尺寸结构体和数组类型本身还是其指针类型依赖于具体的场景和方案.

通常来说, 我们不应该使用其基本类型为slice, map, channel, function, string 和 interface类型的指针类型. Go runtime复制这些内建类型的值的成本很小.

如果一个切片类型的元素类型是大尺寸的, 我们还应该尽量避免使用两次迭代变量的形式来迭代数组和切片元素, 因为每个元素的值都将被复制到第二次迭代中的变量里.

下面是一个对slice元素迭代的不同方式进行的benchmark.

```go
package main_test

import (
        "testing"
)

type Test struct {
        a int64
        b int
        c int64
        d uint32
        e bool
        f struct {
                x int64
        }
}

var SliceX, SliceY, SliceZ = make([]Test, 1000), make([]Test, 1000), make([]Test, 1000)
var sumX, sumY, sumZ int64

func BenchmarkGeneralLoop(b *testing.B) {
        b.ResetTimer()
        for i := 1; i < b.N; i++ {
                sumX = 0
                for j := 0; j < len(SliceX); j++ {
                        sumX += SliceX[j].a
                }
        }
        b.StopTimer()
}

func BenchmarkRangeOneIter(b *testing.B) {
        b.ResetTimer()
        for i := 1; i < b.N; i++ {
                sumY = 0
                for idx := range SliceY {
                        sumY += SliceY[idx].a
                }
        }
        b.StopTimer()
}

func BenchmarkRangeTwoIter(b *testing.B) {
        b.ResetTimer()
        for i := 1; i < b.N; i++ {
                sumZ = 0
                for _, v := range SliceZ {
                        sumZ += v.a
                }
        }
        b.StopTimer()
}
```

运行这个测试之后，我们得到了一下测试数据:

```bash
goos: linux
goarch: amd64
pkg: test
BenchmarkGeneralLoop-4           1000000              1357 ns/op               0 B/op          0 allocs/op
BenchmarkRangeOneIter-4          1000000              1327 ns/op               0 B/op          0 allocs/op
BenchmarkRangeTwoIter-4           500000              3416 ns/op               0 B/op          0 allocs/op
PASS
ok      test    4.470s
```

很明显, 两次迭代中进行元素赋值比其他两个慢了非常多.



----

引用: [Go Value Copy Costs](https://go101.org/article/value-copy-cost.html)
