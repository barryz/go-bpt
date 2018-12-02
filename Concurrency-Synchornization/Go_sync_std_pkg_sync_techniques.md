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

## 类型`sync.Mutex`和`sync.RWMutex`

类型`sync.Mutex`和`sync.RWMutex`都实现了[`sync.Locker`接口](https://golang.org/pkg/sync/#Locker)。

- 所以它们都有`Lock()`和`Unlock()`这两个方法，这两个方法为数据的访问进行锁定和释放。
- 除了`Lock()`和`Unlock()`这两个方法用来为数据访问进行锁定和解锁之外，`*sync.RWMutex`类型还有另外两个方法，`RLock()`和`RUnlock()`，用来进行数据读操作①的锁定和解锁。

①这里所说的 ___数据读操作___ 不能以字面的意思来理解。有些数据的读操作可能会修改一些数据，但是数据读操作所做的修改不会相互干扰。

一个`sync.Mutex`值是一个通常意义上的互斥锁。该值的零值是一个未锁定的互斥量。
一个`sync.Mutex`值只能在其未锁定的状态下被锁定。换言之，一个数据访问者会使用`sync.Mutex`值来排除掉其他数据访问者使用`sync.Mutex`的值。

一个`sync.RWMutex`值是一个读写互斥锁。该值的零值是一个未锁定的互斥量。同时，一个`sync.RWMutex`可以被任意多个数据读取者锁定，但是最多只能被一个通用的数据访问者锁定。换言之，

- 一个数据读取者使用`sync.RWMutex`值不能排除掉其他的数据读取者使用`sync.RWMutex`的值。
- 但是一个数据读取者使用一个`sync.RWMutex`值来排除其他的数据访问者使用`sync.RWMutex`值。
- 一个通用的数据访问者使用一个`sync.RWMutex`值来排除掉任何其他的通用的数据访问者，包括数据读取者，来尝试使用`sync.RWMutex`的值。

通常，

- 一个`sync.Mutex`值用来阻止某些数据被并发地访问。
- 一个`sync.RWMutex`值用来阻止某些数据并发的访问，当该数据被一个数据写入者修改时。

使用`sync.Mutex`的例子：

```go
package main

import "fmt"
import "sync"

type Counter struct {
	m sync.RWMutex
	n uint64
}

func (c *Counter) Increase(delta uint64) {
	c.m.Lock()
	c.n += delta
	c.m.Unlock()
}

// The first two lines in this method can be replaced with:
//    c.m.RLock()
//    defer c.m.RUnlock()
func (c *Counter) Value() uint64 {
	c.m.Lock()
	defer c.m.Unlock()
	return c.n
}

func main() {
	var c Counter
	for i := 0; i < 100; i++ {
		go func() {
			for k := 0; k < 100; k++ {
				c.Increase(1)
			}
		}()
	}

	for c.Value() < 10000 {}
	fmt.Println(c.Value()) // 10000
}
```

上面的例子中，任何`Counter`值的字段`Mutex`将会保证`Counter`值的字段`n`不会被多个goroutines并发的访问。

使用`sync.RWMutex`的例子：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	const N = 10
	var values [N]string

	var m sync.RWMutex
	for i := 0; i < N; i++ {
		i := i
		go func() {
			m.RLock()
			values[i] = string('a' + i)
			m.RUnlock()
		}()
	}

	done := func() bool {
		m.Lock()
		defer m.Unlock()
		for i := 0; i < N; i++ {
			if values[i] == "" {
				return false
			}
		}
		fmt.Println(values) // [a b c d e f g h i j]
		return true
	}
	for !done() {}
}
```

上面的例子中创建了10个“读取者”的goroutines，它们每个都将修改`values`中的一个元素。这10个“读取者”goroutines不会被其他goroutines所干扰，`done`goroutine可以被看作是一个通用访问者。

请注意，上面两个例子中的`m.Lock()`，`m.Unlock()`和`m.RUnlock()`是`(&m).Lock()`，`(&m).Ulock()`和`(&c).RUnlock()`对应的缩写形式。

`sync.Mutex`值也可用于发出通知，但还有许多其他更好的方法可以进行相同的工作。下面是使用`sync.Mutex`值进行通知的示例。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var m sync.Mutex
	m.Lock()
	go func() {
		time.Sleep(time.Second)
		fmt.Println("Hi.")
		m.Unlock() // make a notification
	}()
	m.Lock() // wait to be notified
	fmt.Println("Bye.")
}
```

上面的例子中，文本`Hi`将被保证在文本`Bye.`打印之前被打印。关于`sync.Mutex`和`sync.RWMutex`值所做的内存顺序保证，请阅读[memory order guarantees in Go](https://go101.org/article/memory-model.html#mutex)。

## 类型`sync.Cond`

类型`sync.Cond`为多个goroutines之间的通知提供了一种高效的方式。

每个`sync.Cond`值拥有一个名为`L`的`sync.Locker`的字段。这个字段的值一般是一个`*sync.Mutex`或`*sync.RWMutex`的值。每个`sync.Cond`值同样维护了一个等待状态的goroutines队列。

`*sync.Cond`类型有[三个方法](https://golang.org/pkg/sync/#Cond)， `Wait`，`Signal`和`Broadcast`。

对于一个可寻址的`sync.Cond`的值`c`来说，

- `c.Wait()`只能当`c.L`被锁定之后才能被调用，否则，调用`c.Wait()`将会发生恐慌。一个`c.Wait()`调用将会

  1. 首先将当前调用者goroutine推入到由`c`维护的等待状态队列中，
  2. 然后调用`c.L.Unlock()`来释放锁`c.L`。
  3. 然后使当前调用者goroutine进入阻塞状态①。调用者goroutine将被另一个goroutine通过调用`c.Signal()`或者`c.Broadcast()`来解锁（重新进入运行状态），将调用`c.L.Lock()`（在恢复`c.Wait()`的调用中）以再次尝试获取锁`c.L`，`c.Wait()`调用将在`c.L.Lock()`返回之后退出。

- 一个`c.Signal()`调用将会尝试解锁并删除由`c`维护的等待队列中的第一个goroutine，如果该队列不为空的情况下。

- 一个`c.Broadcast()`调用将会尝试解锁并删除由`c`维护的等待队列中的第一个goroutine，如果该队列不为空的情况下。

①这是对于大多数通用情况下而言的，假设在步骤2和步骤3之间的短时间内没有调用`c.Signal()`和`c.Broadcast()`。

- 如果在步骤2和步骤3之间的短时间内调用了`c.Signal()`，那么`c.Wait()`的调用者goroutine可能不会进入阻塞状态。

- 如果在步骤2和步骤3之间的短时间内调用了`c.Broadcast()`，那么`c.Wait()`的调用者goroutine将肯定不会进入阻塞状态。

请注意`c.Wait()`，`c.Signal()`和`c.Broadcast()`是`(&c).Wait()`，`(&c).Signal()`和`(&c).Broadcast()`对应的缩写形式。

在一个典型的`sync.Cond`用例中，通常，一个goroutine等待某个条件的更改，而其他一些goroutine会进行更改通知。这是一个例子：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var (
	fooIsDone = false // Here, the user defined condition
	barIsDone = false // is composed of two parts.
	cond      = sync.NewCond(&sync.Mutex{})
)

func doFoo() {
	time.Sleep(time.Second) // simulate a workload
	cond.L.Lock()
	fooIsDone = true
	cond.Signal() // called when cond.L locker is acquired
	cond.L.Unlock()
}

func doBar() {
	time.Sleep(time.Second * 2) // simulate a workload
	cond.L.Lock()
	barIsDone = true
	cond.L.Unlock()
	cond.Signal() // called when cond.L locker is released
}

func main() {
	cond.L.Lock()

	go doFoo()
	go doBar()

	checkConditon := func() bool {
		fmt.Println(fooIsDone, barIsDone)
		return fooIsDone && barIsDone
	}
	for !checkConditon() {
		cond.Wait()
	}

	cond.L.Unlock()
}
```

输出为：

```bash
false false
true false
true true
```

因为只有一个goroutine（主goroutine）等待被解除阻塞，两个`cond.Signal()`调用可以用`cond.Broadcast()`替换掉。

上面例子中的一些细节。

- 为避免数据竞争，用户定义的条件的每个部分只应在goroutine获取锁时由goroutine修改。
- 为了保证在主goroutine中`cond.Wait()`之前调用`cond.Signal()`（或`cond.Broadcast()`），

  1. 在主goroutine获取锁的时候goroutine`doFoo`和`doBar`需要被启动。
  2. 当goroutine获得锁时，应该由goroutine调用`cond.Signal()`（或`cond.Broadcast()`。可以在goroutine释放锁之前或之后调用这两种方法。

由`sync.Cond`值监视的用户定义的条件可以为空。对于这种情况，`sync.Cond`值纯粹用于通知。例如，以下程序将打印`abc`或`bac`。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	wg := sync.WaitGroup{}
	wg.Add(1)
	cond := sync.NewCond(&sync.Mutex{})
	cond.L.Lock()
	go func() {
		cond.L.Lock()
		go func() {
			cond.L.Lock()
			cond.Broadcast()
			cond.L.Unlock()
		}()
		cond.Wait()
		fmt.Print("a")
		cond.L.Unlock()
		wg.Done()
	}()
	cond.Wait()
	fmt.Print("b")
	cond.L.Unlock()
	wg.Wait()
	fmt.Println("c")
}
```

如果有需要，多个`sync.Cond`值可以共享同一个`sync.Locker`。但是，这种情况在实践中很少见。
