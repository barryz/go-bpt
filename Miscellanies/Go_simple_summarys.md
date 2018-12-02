> 译自：[Some Simple Summaries](https://go101.org/article/summaries.html)。

# 简单总结

## 哪些类型的值可能有间接底层类型？

可能拥有间接底层部分的类型有：

- 字符串类型
- 函数类型
- 切片类型
- map类型
- 通道类型
- 接口类型

[这篇文章](https://go101.org/article/value-part.html)列举了上述各种类型的在标准Go编译器和运行时中的内部定义细节。

## 哪些类型可以使用内建函数`len`（包括函数`cap`，`close`，`delete`，`make`）？

||len|cap|close|delete|make|
|-----|-----|-----|-----|-----|
|string|Yes| | | | |
|array (and array pointer)|Yes| Yes| | | |
|slice|Yes| Yes| | | Yes|
|map|Yes| Yes| | Yes| Yes|
|channel|Yes| Yes|Yes | | Yes|

上述中的各个类型都可以使用`for-range`循环结构。

值能够使用内建函数`len`的类型被称为容器类型。

## 内建容器类型的比较

|Type|Can New Elements Be Added into Values?|Are Elements of Values Replcacable?|Are Elements of Values Addressable?|Will Element Accessments Modify Value Lengths?|May Values Have Underlying Parts|
|-----|-----|-----|-----|-----|-----|
|string|No|No|No|No|Yes(1)|
|array|No|Yes(2)|Yes(2)|No|No|
|slice|No(3)|Yes|Yes|No|Yes|
|map|Yes|Yes|No|No|Yes|
|channel|Yes(4)|No|No|Yes|Yes|

(1)仅适用于标准Go编译器/运行时。
(2)仅适用于可寻址的数组值。
(3)切片的长度可以通过函数`reflect.SetLen`来修改。通过这种方式增加切片的长度其实就是在切片中添加新元素。也可以通过分配另一个切片值来改变源切片的长度。
(4)仅适用于还未满的缓冲通道。

## 可以使用复合字面值（`T{...}`）表示值的类型有哪些？

下列类型可以使用复合字面值表示：

|Type(`T`)|Is `T{}` A Zero Value Of `T`?|
|-----|-----|
|struct| Yes|
|array| Yes|
|slice| No (zero value is nil)|
|slice| No (zero value is nil)|

## 可以使用`nil`表示其零值的类型有哪些？

下列类型的零值可以用`nil`来表示：

|Type(`T`)| Size Of `T(nil)`?|
|-----|-----|
|pointer|1 word|
|slice|3 words|
|map|1 word|
|channel|1 word|
|function|1 word|
|interface|2 words|

*（一个字长在32位平台上表示为4个字节，在64位平台上表示为8个字节， 且[间接底层部分](https://go101.org/article/value-part.html)不参与值的大小计算。）*

类型零值的大小和该类型其他值的大小相同。

## 可以为其声明方法的类型有哪些？

请阅读[本文](https://go101.org/article/unofficial-faq.html#types-can-have-methods)获取详情。

## 可以匿名嵌入其他类型的类型有哪些？

请阅读[这篇文章](https://go101.org/article/type-embedding.html#embeddable-types)获取详情。

## 哪些函数的调用可以在编译期被求值？

如果一个函数调用可以在编译期间被求值，那么它的结果可以被用作一个常量。

|Function|Return Type|Are Calls Always Evaluated At Compile Time?|
|-----|-----|-----|
|unsafe.SizeOf|`uintptr`|Yes, always|
|unsafe.AlignOf|`uintptr`|Yes, always|
|unsafe.OffsetOf|`uintptr`|Yes, always|
|len|`int`|Not always (1)|
|cap|`int`|Not always (1)|
|real|`float64`(default type)|Not always (2)|
|imag|`float64`(default type)|Not always (2)|
|complex|`complex128`(default type)|Not always (3)|

(1)通过[Go规范](https://golang.org/ref/spec#Length_and_capacity)：

- 如果`s`是一个字符串，那么表达式`len(s)`是一个常量。
- 如果`s`的类型是数组或指向数组的指针且表达式`s`不包含通道接收操作或（非常量）的函数调用（换言之，`s`可以是数组变量或数组指针变量），则表达式`len(s)`和`cap(s)`是常量。

(2)通过[Go规范](https://golang.org/ref/spec#Constants)：
如果`s`是一个复数常量，那么表达式`real(s)`和`imag(s)`也是常量。

(3)通过[Go规范](https://golang.org/ref/spec#Length_and_capacity)：
如果`sr`和`si`都是一个数值常量，那么表达式`complex(sr, si)`也是常量。

## 无法获取地址的值有哪些？

下面列举的值不能够获取地址：

- 字符串中的字节
- map元素
- 接口值（通过类型断言暴露）的动态值
- 常量值
- 字面量值
- 包级别的函数
- 方法（作为函数值）
- 中间值
  - 函数调用
  - 显式值转换
  - 各种操作，除了指针解引用操作，但是包括：
    - 通道接收操作
    - 子字符串操作
    - 子切片操作
    - 加减乘除等算术运算操作

_请注意，在Go中有个语法糖，`&T{}`。实际上它是`tmp := T{}; (&tmp)`的缩写形式，所以`&T{}`合法并不意味着复合字面值`T{}`是可寻址的。_

顺便说下，下列值可以获取地址（除了刚刚提到的复合字面值）：

- 变量
- 可寻址的结构体的字段
- 可寻址数组的元素
- 任何切片（无论切片是否可以寻址）的元素
- 指针解引用操作
