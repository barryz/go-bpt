> 译自：[Concurrency Synchronization Techniques Provided In The `sync` Standard Package](https://go101.org/article/concurrent-synchronization-more.html)

# 标准库`sync`提供的一些并发同步技术

文章[通道的用例]()介绍了很多关于多个goroutine之间使用通道进行数据同步的用例。实际上，通道并不是Go所能提供的唯一的同步技术。Go中还提供了其他一些数据同步技术。对于某些特定的场景，使用这些同步技术比使用通道更加高效，有时候代码也更加易读。下面我们将介绍标准库`sync`中提供的一些数据同步技术。

`sync`标准库提供了多种类型，可以用于某些特定情况下进行数据同步。并且保证了一些指定的内存顺序。对于这些特殊的情况，这些技术比使用通道的方式更加高效，并且看起来更加简洁。

___（请注意，为了避免一些异常的行为，最好不要拷贝`sync`标准库中的类型的值。）___


## 类型`sync.WaitGroup`

每个`sync.WaitGroup`值内部都维护了一个计数器。该计数器的默认值是零。

类型`*sync.WaitGroup`有[三个方法](https://golang.org/pkg/sync/#WaitGroup)：`Add(delta int)，Done()` 和 `Wait()`。

对于可寻址的`sync.WaitGroup`的值`wg`来说，

- 我们可以调用`wg.Add(delta)`方法来改变`wg`的相应计数器的值。
- 方法`wg.Done()`的调用完全等价于方法调用`wg.Add(-1)`。
- 如果调用`wg.Add(delta)`（或`wg.Done()`）将由`wg`维护的计数器修改为负数，则会发生恐慌。
- 当一个goroutine调用`wg.Wait()`时，

  - 如果由`wg`维护的计数器已经为零，那么调用`wg.Wait()`可以被视为无任何操作。
  - 否则（计数器为正数时），该goroutine将会进入阻塞状态。当另一个goroutine将计数器修改为零时（通常是调用`wg.Done()）`，这个goroutine会再次进入运行状态。

请注意`wg.Add(delta)`，`wg.Done()`和`wg.Wait`是`(&wg).Add(delta)`，`(&wg).Done()`和`(&wg).Wait()`相应的缩写形式。

通常来说，`sync.WaitGroup`值用于一个goroutine等待其他所有goroutine完成各自作业的场景。例子：

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	const N = 5
	var values [N]int32

	var wg sync.WaitGroup
	wg.Add(N)
	for i := 0; i < N; i++ {
		i := i
		go func() {
			values[i] = 50 + rand.Int31n(50)
			fmt.Println("Done:", i)
			wg.Done() // <=> wg.Add(-1)
		}()
	}

	wg.Wait()
	// All elements are guaranteed to be initialized now.
	fmt.Println("values:", values)
}
```

在上面的例子中，主goroutine等待所有其他N个goroutine在`values`数组中填充它们各自的元素值。下面是一个可能的输出结果：

```bash
Done: 4
Done: 1
Done: 3
Done: 0
Done: 2
values: [71 89 50 62 60]
```

我们可以将上述例子中的唯一的`Add`方法分解成多个。

```go
...
	var wg sync.WaitGroup
	for i := 0; i < N; i++ {
		wg.Add(1) // will be invoked N times
		i := i
		go func() {
			values[i] = 50 + rand.Int31n(50)
			wg.Done()
		}()
	}
...
```

`Wait`方法可以被多个goroutines调用。当计数器值变为零时，这些goroutines都将被通知到，通过一个广播的方式。

```go
func() {
	var wg sync.WaitGroup
	wg.Add(1)

	for i := 0; i < N; i++ {
		i := i
		go func() {
			wg.Wait()
			fmt.Printf("values[%v]=%v \n", i, values[i])
		}()
	}

	// The loop is guaranteed to finish before
	// any above wg.Wait calls returns.
	for i := 0; i < N; i++ {
		values[i] = 50 + rand.Int31n(50)
	}
	wg.Done() // will make a broadcast
}
```

## 类型`sync.Once`

一个`*sync.Once`值有一个`Do(f func())`方法，该方法的参数的类型为`func()`。

对于一个可寻址的`sync.Once`的值`o`来说，方法`o.Do`，是方法`(&o).Do`的缩写形式，该方法可以被多个goroutines调用多次。这些`o.Do`调用的参数应该（但不强制要求）是相同的函数值。

在这些`o.Do`调用之间，只有一个参数函数被调用。该被调用的函数被保证在任何`o.Do`方法调用返回之前被返回。换言之，被调用参数函数中的代码将会被保证在任意`o.Do`方法调用返回之前被执行到。

通常来说，一个`sync.Once`值被用来确认在并发变成中某件事是否已经完成且只完成了一次。

例子：

```go
package main

import (
	"log"
	"sync"
)

func main() {
	log.SetFlags(0)

	x := 0
	doSomething := func() {
		x++
		log.Println("Hello")
	}

	var wg sync.WaitGroup
	var once sync.Once
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			once.Do(doSomething)
			log.Println("world!")
		}()
	}


	log.Println("x =", x) // x = 1
}
```

在上面的例子中，`Hello`只会被打印一次，但是`world!`将会被打印5次。并且`Hello`将被保证在这5个`world!`打印之前被打印。
