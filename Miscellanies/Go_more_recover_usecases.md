> 译自[The right place to call `recover` function](https://go101.org/article/panic-and-recover-more.html)。

# 在正确的地方调用`recover`函数

我们知道`recover`函数只能对其涉及的延迟函数的调用产生影响。然而，在一个延迟函数的调用中，并不是所有的panic都能被`recover`函数捕捉到。下面的内容将会介绍一些无用的`recover`函数调用以及如何在正确的地方使用`recover`函数。

让我们先看一个例子：

```go
package main

import "fmt"

func main() {
	defer func() {
		defer func() {
			fmt.Println("6:", recover())
		}()
	}()
	defer func() {
		func() {
			fmt.Println("5:", recover())
		}()
	}()
	func() {
		defer func() {
			fmt.Println("1:", recover())
		}()
	}()
	func() {
		defer fmt.Println("2:", recover())
	}()
	func() {
		fmt.Println("3:", recover())
	}()
	fmt.Println("4:", recover())
	panic(1)
	defer func() {
		fmt.Println("0:", recover()) // never go here
	}()
}
```

运行它我们会发现，七个运行了`recover`函数的语句什么都没有捕捉到，在程序运行快结束时，该程序会立即崩溃。

```bash
1: <nil>
2: <nil>
3: <nil>
4: <nil>
5: <nil>
6: <nil>
panic: 1

goroutine 1 [running]:
...
```

当然，第0个`recover`函数调用是永远不可达的。对于其他的，我们先来看下[Go 规范](https://golang.org/ref/spec#Handling_panics)中定义的一些检查规则。

当出现下列情况之一时，`recover`函数调用的结果是`nil`。

- `panic`函数的参数是`nil`。
- 当前的goroutine不在恐慌的状态。
- 在延迟函数中`recover`函数没有被直接调用。

我们先忽略第一种情况。第二种情况覆盖了第1，2，3和4种`recover`调用。第三种情况覆盖了第5种`recover`调用。然而，上述中没有任何情况能够覆盖第6种`recover`调用。

我们知道下面的`recover`调用能够捕捉到恐慌。

```go
// example2.go
package main

import (
	"fmt"
)

func main() {
	defer func() {
		fmt.Println( recover() ) // 1
	}()

	panic(1)
}
```

但是，它和之前的例子中的`recover`调用有什么不同？使`recover`调用生效的主要规则到底是什么？

首先，让我们先学习一些概念和事实。

## 概念：函数调用深度，Goroutine执行深度和恐慌（panic）深度

每个函数调用对应了一个调用深度，这与goroutine的入口函数相关。对于一个`main`goroutine，一个函数的调用深度和该程序的入口函数**main**有关。对于其他的goroutine，函数的调用深度和该goroutine中第一个执行的函数调用有关。

例子：

```go
package main

func main() { // depth 0
	go func() { // depth 0
		func() { // depth 1
		}()

		defer func() { // depth 1
			defer func() { // depth 2
			}()
		}()
	}()

	func () { // depth 1
		func() { // depth 2
			go func() { // depth 0
			}()
		}()

		go func() { // depth 0
		}()
	}()
}

```

包含了一个goroutine当前执行点的函数调用深度被称为goroutine的执行深度。

恐慌的深度即是该恐慌传递到函数调用的某个深度处。

## 事实： 恐慌只能传播到具有shadow深度的函数调用中

是的，恐慌只能传播给其调用者。恐慌永远不会向下传播到更深层次的函数调用中。

```go
package main

import "fmt"

func main() { // call depth 0
	defer func() { // call depth 1
		fmt.Println("Now, the panic is still in call depth 0")
		func() { // call depth 2
			fmt.Println("Now, the panic is still in call depth 0")
			func() { // call depth 3
				fmt.Println("Now, the panic is still in call depth 0")
			}()
		}()
	}()

	defer fmt.Println("Now, the panic is in call depth 0")

	func() { // call depth 1
		defer fmt.Println("Now, the panic is in call depth 1")
		func() { // call depth 2
			defer fmt.Println("Now, the panic is in call depth 2")
			func() { // call depth 3
				defer fmt.Println("Now, the panic is in call depth 3")
				panic(1)
			}()
		}()
	}()
}
```

所以，恐慌的深度是单调递减的，该深度永远不会增加。并且，已经存在的恐慌的深度不会大于goroutine的执行深度。

## 事实：一个恐慌会将旧的恐慌抑制在同一深度内

例子：

```go
package main

import "fmt"

func main() {
	defer fmt.Println("program will not crash")

	defer func() {
		fmt.Println( recover() ) // 3
	}()

	defer fmt.Println("now, panic 3 suppresses panic 2")
	defer panic(3)
	defer fmt.Println("now, panic 2 suppresses panic 1")
	defer panic(2)
	panic(1)
}

/*
now, panic 2 suppresses panic 1
now, panic 3 suppresses panic 2
3
program will not crash
*/

```

在这个例子中，恐慌1被恐慌2抑制，恐慌2被恐慌3抑制。所以，在程序最后，只有一个恐慌3存在，所以该程序不会崩溃。

在一个goroutine中，任何时候在同一深度内，至多只有一个活动的恐慌。特别是当执行点在goroutine的调用深度0处运行时，goroutine中最多会有一个活动的恐慌。

## 事实：多个活动的恐慌在同一个goroutine中也有可能共存

例子：

```go
package main

import "fmt"

func main() { // call depth 0
	defer fmt.Println("program will crash, for panic 3 is stll active")

	defer func() { // call depth 1
		defer func() { // call depth 2
			fmt.Println( recover() ) // 6
		}()

		// The depth of panic 3 is 0, and the depth of panic 6 is 1.
		defer fmt.Println("now, there are two active panics: 3 and 6")
		defer panic(6) // will suppress panic 5
		defer panic(5) // will suppress panic 4
		panic(4) // will not suppress panic 3,
		         // for they have differrent depths.
		         // The depth of panic 3 is 0.
		         // The depth of panic 4 is 1.
	}()

	defer fmt.Println("now, only panic 3 is active")
	defer panic(3) // will suppress panic 2
	defer panic(2) // will suppress panic 1
	panic(1)
}

/*
now, only panic 3 is active
now, there are two active panics: 3 and 6
6
program will crash, for panic 3 is stll active
panic: 1
	panic: 2
	panic: 3

goroutine 1 [running]
...
*/

```

在这个例子中，恐慌6，两个活动的恐慌之一，将会被恢复。但是对于另一个活动的恐慌3，将会保留在**main**函数调用的结尾，所以该程序会崩溃。

## 事实：最高深度的恐慌可能会被最先恢复，也可能不会

例子：

```go
package main

import "fmt"

func demo(recoverHighestPanicAtFirst bool) {
	fmt.Println("====================")
	defer func() {
		if !recoverHighestPanicAtFirst{
			// recover panic 1
			defer fmt.Println("panic", recover(), "is recovered")
		}
		defer func() {
			// recover panic 2
			fmt.Println("panic", recover(), "is recovered")
		}()
		if recoverHighestPanicAtFirst {
			// recover panic 1
			defer fmt.Println("panic", recover(), "is recovered")
		}
		defer fmt.Println("now, two active panics coexist")
		panic(2)
	}()
	panic(1)
}

func main() {
	demo(true)
	demo(false)
}

/*
Output:
====================
now, two active panics coexist
panic 1 is recovered
panic 2 is recovered
====================
now, two active panics coexist
panic 2 is recovered
panic 1 is recovered
*/
```

## 那么，让一个`recover`函数调用生效的主要规则是什么？

规则其实十分简单。

__在一个goroutine中，如果`recover`调用的调用者函数是`f`并且`f`调用深度是`d`，那么，为了使`recover`函数调用生效，函数调用`f`必须是一个延迟函数调用且必须有一个活动的恐慌在深度`d-1`内。`recover`调用是否是延迟调用并不重要。__

就是这样，我认为它描述的比Go规范中的更好，现在，你可以向上翻页来检查第一个示例中的第6个`recover`调用未生效的原因。
