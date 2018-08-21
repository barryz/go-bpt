> 译自：[Type Embedding](https://go101.org/article/type-embedding.html)。


# 类型嵌套

从文章[structs in Go](https://go101.org/article/struct.html)中，我们知道结构体可以有很多个字段。每个字段由一个字段名称和字段类型组成。实际上，有时候，一个字段可以只由一个字段类型构成。这样的字段被称为**嵌套字段**。有些文章会将嵌套字段称为匿名字段。

本文将解释类型嵌入的目的以及类型嵌套中的各种细节。

## 类型嵌套长啥样？

类型嵌套的一个例子：

```go
package main

func main() {
	type S = []int
	var x struct {
		string // a defined non-pointer type
		error  // a defined interface type
		*int   // an unnamed pointer type
		S      // an alias of an unnamed pointer type

	}
	x.string = "Go"
	x.error = nil
	x.int = new(int)
	x.S = []int{}
}
```

上述例子中，一个结构体中有四个嵌套字段。

实际上，每个嵌套字段都有一个隐式的名称。这四个嵌套字段的非限定类型名称是它们的名称或者是它们的基类型名称。例如，上述列子中的四个嵌套字段的各自的名称分别为`string`，`int`，`error`和`S`。

## 哪些类型可以嵌套

下面的类型无法嵌套进结构体：

- 已定义的指针类型。
- 未命名类型，除了未命名的一级指针类型，其基类型是非接口和非定义类型。

其他的类型都可以被嵌套。具体来说，以下类型可以嵌入到结构类型中：

- 一个非指针类型的[命名类型](https://go101.org/article/type-system-overview.html#named-type)。
- 或一个未命名的指针类型，它的基类型是一个命名的非指针非接口类型。
