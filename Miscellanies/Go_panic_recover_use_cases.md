> 译自：[Some Panic/Recover Use Cases](https://go101.org/article/panic-and-recover-use-cases.html)。

# Panic和Recover （恐慌和恢复）

Go语言不支持异常处理，而是在Go编程中使用显式的错误处理。然而，事实上，Go语言支持异常抛出/捕获类似的机制。该机制叫做恐慌/恢复。

我们可以调用内建函数`panic`来创建一个恐慌使得当前的goroutine处于恐慌的状态。该恐慌只在当前的goroutine中存在。同一个goroutine可能会共存多个恐慌。

使用内建函数`recover`可以恢复恐慌。一旦一个goroutine中的所有恐慌都被恢复，该goroutine将会再次进入平静的状态。

恐慌是函数返回的另一种方式。一个恐慌显式地出现在一个函数调用中时，该函数调用将会立即进入退出阶段。该函数调用中被推入的延迟函数将会以它们被推入的顺序反向执行。只有在延迟函数数调用`recover`函数，它才能够恢复恐慌。

如果一个恐慌的goroutine退出时并没有得到恢复，那么它将会是整个程序崩溃。

内建函数`panic`和`recover`定义如下：

```go
func panic(v interface{})
func recover() interface{}
```

接口类型和值已经在之前的文章[interfaces in Go]()有过解释。

`recover`函数调用返回的值是`panic`函数接收的参数值。

下面是一个展示如何使用`panic`和`recover`函数的例子：

```go
package main

import "fmt"

func main() {
	fmt.Println("hi!")
	defer func() {
		v := recover()
		fmt.Println("recovered:", v)
	}()
	panic("bye!")
	fmt.Println("unreachable")
}
/*
Output:
hi!
recovered: bye!
*/
```

下面是一个没有恢复恐慌的goroutine退出的例子。所以该程序将会崩溃。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Println("hi!")

	go func() {
		time.Sleep(time.Second)
		panic(123)
	}()

	for {
		time.Sleep(time.Second)
	}
}
/*
Output:
hi!
panic: 123

goroutine 5 [running]:
...
*/
```

Go运行时在很多情况下会产生恐慌，比如除零操作。例如：

```go
package main

func main() {
	a, b := 1, 0
	_ = a/b
}
/*
Output:
panic: runtime error: integer divide by zero

goroutine 1 [running]:
...
*/
```

# 一些Panic/Recover使用案例

## 案例1：避免panic导致整个程序崩溃

这个也许是恐慌/恢复最受欢迎的用法之一。该用法在并发编程中经常使用，特别是CS（client-server)程序。

例子：

```go
package main

import "errors"
import "log"
import "net"

func main() {
	listener, err := net.Listen("tcp", ":12345")
	if err != nil {
		log.Fatalln(err)
	}
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Println(err)
		}
		// Handle each client connection in a new goroutine.
		go ClientHandler(conn)
	}
}

func ClientHandler(c net.Conn) {
	defer func() {
		if v := recover(); v != nil {
			log.Println("client handler panic:", v)
		}
		c.Close()
	}()
	panic(errors.New("just a demo.")) // a demo-purpose panic
}
```

如果我们不恢复每个客户端的处理器goroutine的潜在的恐慌，那么这些潜在的恐慌将会导致整个程序崩溃。

启动服务器并在另一个终端运行`telnet localhost 12345`，我们可以观察到当客户端出现恐慌时，服务端程序不会退出。

## 案例2：自动重启一个崩溃的goroutine

当在一个goroutine中检测到一个恐慌时，我们可以为它创建一个新的goroutine。例子。

```go
package main

import "log"
import "time"

func shouldNotExit() {
	for {
		time.Sleep(time.Second) // simulate a workload
		// Simultate an unexpected panic.
		if time.Now().UnixNano() & 0x3 == 0 {
			panic("unexpected situation")
		}
	}
}

func NeverExit(name string, f func()) {
	defer func() {
		if v := recover(); v != nil { // a panic is detected.
			log.Println(name, "is crashed. Restart it now.")
			go NeverExit(name, f) // restart
		}
	}()
	f()
}

func main() {
	log.SetFlags(0)
	go NeverExit("job#A", shouldNotExit)
	go NeverExit("job#B", shouldNotExit)
	select{} // blocks here for ever
}
```

## 案例3：使用恐慌/恢复来模拟长跳（long jump)语句

有时候，我们可以使用恐慌/恢复来模拟跨函数的长跳语句以及跨函数的返回，尽管这种方式不推荐使用。这种方式会损害代码可读性以及执行效率。唯一的好处就是可以少写一点代码。

在下面的例子中，一旦一个恐慌在一个内部函数中创建，那么执行步骤将会跳转到延迟的`recover`调用处。

```go
package main

import "fmt"

func main() {
	n := func () (result int)  {
		defer func() {
			if v := recover(); v != nil {
				if n, ok := v.(int); ok {
					result = n
				}
			}
		}()

		func () {
			func () {
				func () {
					// ...
					panic(123) // panic on succeeded
				}()
				// ...
			}()
		}()
		// ...
		return 0
	}()
	fmt.Println(n) // 123
}
```

## 案例4：使用恐慌/恢复来减少错误检查

例子：

```go
package main

import "fmt"

func doTask(n int) {
	if n%2 != 0 {
		// Create a demo-purpose panic.
		panic(fmt.Errorf("bad number: %v", n))
	}
	return
}

func doSomething() (err error) {
	defer func() {
		// The ok return must be present here, otherwise,
		// a panic will be created if no errors occur.
		err, _ = recover().(error)
	}()

	doTask(22)
	doTask(98)
	doTask(100)
	doTask(53)
	return nil
}

func main() {
	fmt.Println(doSomething()) // bad number: 53
}
```

上面的代码比下面例子中的代码更加简洁。

```go
func doTask(n int) error {
	if n%2 != 0 {
		return fmt.Errorf("bad number: %v", n)
	}
	return nil
}

func doSomething() (err error) {
	err = doTask(22)
	if err != nil {
		return
	}
	err = doTask(98)
	if err != nil {
		return
	}
	err = doTask(100)
	if err != nil {
		return
	}
	err = doTask(53)
	if err != nil {
		return
	}
	return
}
```

然而，通常，这种恐慌/恢复的使用案例不推荐使用。因为缺乏可读性已经缺乏效率

通常来说，恐慌是逻辑错误，例如一个需要人去关注的错误。逻辑错误是那些应该在运行时绝不会发生的错误。如果它们发生了，那么一定表明代码中存在bug。另一方面，非逻辑错误是一种在运行时很难绝对避免的错误。换言之，它们是一些真实发生的错误。这些错误应该显式地返回，并且需要正确的处理。
