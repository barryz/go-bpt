> 译自：[More About Deferred Function Calls](https://go101.org/article/defer-more.html)。

# 延迟函数调用的细节

## 延迟函数调用

延迟函数的调用是一个跟在关键字`defer`之后的函数调用。和Goroutine函数调用一样，该函数调用的所有返回值（如果该函数有返回值）必须在函数调用语句中丢弃。

当一个函数调用被标记为延迟的，它将不会立即执行。它将被推入一个由调用者goroutine维护的延迟调用的栈中。当一个函数`fc`开始返回并进入退出阶段时，将调用所有在函数`fc`中推入的所有延迟函数（此时函数`fc`尚未推出），调用顺序与这些延迟函数推入的顺序相反。一旦所有的延迟函数执行完毕，函数`fc`的调用就会完全退出。

下例将展示延迟函数是如何工作的。在这个例子中，`defer`是一个关键字。

```go
package main

import "fmt"

func main() {
	defer fmt.Println("The third line.")
	defer fmt.Println("The second line.")
	fmt.Println("The first line.")
}

/*
Output:

The first line.
The second line.
The third line.
*/
```

每个goroutine维护了两个调用栈，普通调用栈和延迟调用栈。

- 对于goroutine正常调用栈中的两个相邻的函数调用来说，之后推入的函数将会在之前推入的函数之前调用。普通调用栈中底部的函数将通过`go`关键字调用。

- 对于goroutine延迟调用栈中的函数来说，他们的调用之间并没有任何联系。

下面是一个稍微复杂的例子。这个例子将会每行依次打印`0`到`9`。

```go
package main

import "fmt"

func main() {
	defer fmt.Println("9")
	fmt.Println("0")
	defer fmt.Println("8")
	fmt.Println("1")
	if false {
		defer fmt.Println("not reachable")
	}
	defer func() {
		defer fmt.Println("7")
		fmt.Println("3")
		defer func() {
			fmt.Println("5")
			fmt.Println("6")
		}()
		fmt.Println("4")
	}()
	fmt.Println("2")
	return
	defer fmt.Println("not reachable")
}
```

## 延迟匿名函数可以修改嵌套函数调用的返回结果名称

例子：

```go
package main

import "fmt"

func Triple(n int) (r int) {
	defer func() {
		r += n // modify the return value
	}()

	return n + n // <=> r = n + n; return
}

func main() {
	fmt.Println(Triple(5)) // 15
}
```

## 延迟函数特性的必要性和益处

在上述例子中，延迟函数不是绝对必要的。然而，延迟函数的特性对于panic和recover机制来说是必须的。

延迟函数同样也可以帮助我们写出清晰、健壮的代码。

## 很多带有返回值的内建函数是不能被延迟的

在Go中，自定义函数的返回值可以全部丢弃。然而，对于一些有非空返回值的内建函数来说，他们调用之后的返回值是不能丢弃的（到Go1.11版本为止），除了内建函数`copy`和`recover`。我们之前学习到延迟函数的返回值必须是能丢弃的，所以很多内建函数不能被延迟执行。

幸运的是，需要延迟执行的内建函数非常罕见。据我所知，内建函数`append`有时候需要延迟执行。对于这样的案例，我们可以延迟一个封装了`append`调用的匿名函数。

```go
package main

import "fmt"

func main() {
	s := []string{"a", "b", "c", "d"}
	defer fmt.Println(s) // [a x y d]
	// defer append(s[:1], "x", "y") // error
	defer func() {
		_ = append(s[:1], "x", "y")
	}()
}
```

## 延迟函数的值的求值瞬间

延迟函数中的函数调用可以是一个`nil`函数值。因为这样的原因，在延迟函数推入当前goroutine延迟调用栈之前会发生panic。一个例子：

```go
package main

import "fmt"

func main() {
	defer fmt.Println("reachable")
	var f func() // f is nil by default
	defer f()    // panic here
	// The following lines are dead code.
	fmt.Println("not reachable")
	f = func() {}
```

延迟函数的参数将在该延迟调用被推入到当前goroutine延迟调用栈[之前被求值](https://go101.org/article/defer-more.html#argument-evaluation-moment)。

## 延迟函数可以使代码bug更少，更加简洁

例子：

```go
import "os"

func withoutDefers(filepath string, head, body []byte) error {
	f, err := os.Open(filepath)
	if err != nil {
		return err
	}

	_, err = f.Seek(16, 0)
	if err != nil {
		f.Close()
		return err
	}

	_, err = f.Write(head)
	if err != nil {
		f.Close()
		return err
	}

	_, err = f.Write(body)
	if err != nil {
		f.Close()
		return err
	}

	err = f.Sync()
	f.Close()
	return err
}

func withDefers(filepath string, head, body []byte) error {
	f, err := os.Open(filepath)
	if err != nil {
		return err
	}
	defer f.Close()

	_, err = f.Seek(16, 0)
	if err != nil {
		return err
	}

	_, err = f.Write(head)
	if err != nil {
		return err
	}

	_, err = f.Write(body)
	if err != nil {
		return err
	}

	return f.Sync()
}
```

上面哪个例子看起来更加简洁？明显的，有延迟函数的例子看起来更加简洁明了，而且它看起来更不容易出bug，假如任何`f.Close()`遗忘了怎么办？

下面的例子将展示延迟函数如何使代码减少bug出现的几率。如果下面的`doSomething`函数调用出现了panic，那么函数`f1`退出时会伴随着互斥量的解锁操作。所以函数`f1`会更少地出现bug。

```go
var m sync.Metex

func f1() {
	m.Lock()
	defer m.Unlock()
	doSomething()
}

func f2() {
	m.Lock()
	doSomething()
	m.Unlock()
}
```

## 延迟函数调用导致的性能损耗

使用延迟函数并不总是合适的。到当前为止（Go1.11）为止，对于官方的Go编译器来说，延迟函数在运行时的调用将会导致一些性能损耗，特别是当延迟函数在一个循环内部调用时。

例如，在下面的例子中，方法`CounterB`和`IncreaseB`比方法`CounterA`和`IncreaseA`效率高的多。

```go
import "sync"

type T struct {
	mu sync.Mutex
	n  int64
}

func (t *T) CounterA() int64 {
	t.mu.Lock()
	defer t.mu.Unlock()
	return t.n
}

func (t *T) CounterB() (count int64) {
	t.mu.Lock()
	count = t.n
	t.mu.Unlock()
	return
}

func (t *T) IncreaseA() {
	t.mu.Lock()
	defer t.mu.Unlock()
	t.n++
}

func (t *T) IncreaseB() {
	t.mu.Lock()
	t.n++ // this line will not panic for sure
	t.mu.Unlock()
}
```

但是在B版本的方法中，我们需要保证在`Lock`和`Unlock`之间的代码不会发生panic。

通常来说，实践中，推荐使用A版本的方法。当我们真正地关心程序的性能时，才使用B版本方法。

## 延迟函数调用导致的资源泄露

一个超大的延迟调用栈可能会消耗大量的内存。未执行的延迟调用可能会阻止某些资源的释放。例如，如果在调用以下函数时需要处理大量文件，那么大量的文件处理器（handler）将无法释放。

```go
func writeManyFile(files []File) error {
	for _, file := range files {
		f, err := os.Open(file.path)
		if err != nil {
			return err
		}
		defer f.Close()

		_, err = f.WriteString(file.content)
		if err != nil {
			return err
		}

		err = f.Sync()
		if err != nil {
			return err
		}
	}

	return nil
}
```

对于这样的案例，我们可以使用匿名函数来封装延迟调用，以便延迟函数的调用在更早的时候执行。例子，上面的例子可以重写优化为下面的代码片段

```go
func writeManyFile(files []File) error {
	for _, file := range files {
		if err := func() error {
			f, err := os.Open(file.path)
			if err != nil {
				return err
			}
			defer f.Close()

			_, err = f.WriteString(file.content)
			if err != nil {
				return err
			}

			return f.Sync()
		}(); err != nil {
			return err
		}
	}

	return nil
}
```
