# Goroutine泄漏 - 被遗弃的通道接收者

## 介绍

Goroutine泄漏在Go程序中是一种非常常见的内存泄漏的案例。在我[之前的博客中](https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html)，我已经介绍过Goroutine泄漏，并列举了一个Go开发人员时常会犯错的例子。今天，我们会继续这个话题，本文会介绍另外一种可能会导致Goroutine泄漏的场景。

## 泄漏：被遗弃的通道接收者

___在这个例子中，你将会看到有多个Goroutine阻塞等待从一个永远不会发送值的通道中接收值___。

本篇博客中的程序将会启动多个Goroutines从某个文件中处理一批记录。每个Goroutine将会从`input`通道中接收一个值，之后会向`output`通道发送一个新值。

### 清单1

[https://play.golang.org/p/Jtpla_UvrmN](https://play.golang.org/p/Jtpla_UvrmN)

```go
35 // processRecords is given a slice of values such as lines
36 // from a file. The order of these values is not important
37 // so the function can start multiple workers to perform some
38 // processing on each record then feed the results back.
39 func processRecords(records []string) {
40
41     // Load all of the records into the input channel. It is
42     // buffered with just enough capacity to hold all of the
43     // records so it will not block.
44
45     total := len(records)
46     input := make(chan string, total)
47     for _, record := range records {
48         input <- record
49     }
50     // close(input) // What if we forget to close the channel?
51
52     // Start a pool of workers to process input and send
53     // results to output. Base the size of the worker pool on
54     // the number of logical CPUs available.
55
56     output := make(chan string, total)
57     workers := runtime.NumCPU()
58     for i := 0; i < workers; i++ {
59         go worker(i, input, output)
60     }
61
62     // Receive from output the expected number of times. If 10
63     // records went in then 10 will come out.
64
65     for i := 0; i < total; i++ {
66         result := <-output
67         fmt.Printf("[result  ]: output %s\n", result)
68     }
69  }
70
71 // worker is the work the program wants to do concurrently.
72 // This is a blog post so all the workers do is capitalize a
73 // string but imagine they are doing something important.
74 //
75 // Each goroutine can't know how many records it will get so
76 // it must use the range keyword to receive in a loop.
77 func worker(id int, input <-chan string, output chan<- string) {
78     for v := range input {
79         fmt.Printf("[worker %d]: input %s\n", id, v)
80         output <- strings.ToUpper(v)
81     }
82     fmt.Printf("[worker %d]: shutting down\n", id)
83
}
```

在清单1的第39行，我们定义了一个名为`processRecords`的函数，它接收一个`[]string`类型的参数。在第46行，我们创建了一个名为`input`的缓冲通道。然后在第47、48行运行一个循环从`records`中分别取出每个值并发送至`input`通道。该通道在创建时已经预分配了足够的容量，所以向该通道发送值的操作并不会阻塞。这个通道可以看作是多个Goroutine分发值的管道。

然后在第56-60行，该程序创建了一些Goroutines分别从该管道上接收值以进行各自的工作。在第56行，我们创建了一个名为`output`的缓冲通道；这个通道用于各Goroutine向其发送值。第57-59行则运行了一个循环，创建了`runtime.NumCPU()`个Goroutines。`input`，`output`通道同时和局部变量`i`一同传递给由单独Goroutine运行的`worker`函数。

`worker`函数的定义在第77行。该函数的签名定义了一个类型`<- chan string`的`input`入参，这表示该参数是一个仅接收通道。而类型为`chan <- string`的`output`入参则表明其是一个仅发送通道。

在`worker`函数内部，我们在第78行使用了`range`循环从`input`通道中接收值。一个通道上的`range`循环操作只会在该通道已关闭且没有值时才会结束。对于每一次迭代，从`input`通道接收到的值都会在第79行赋给局部变量`v`，并打印出来。然后在第80行，`worker`函数将`v`传递给`strings.ToUpper()`函数生成一个新的值插入到`output`通道中。

回到`processRecords`函数，函数已经执行到第65行并开始运行另一个循环。该循环迭代`output`通道并接收该通道上的值。在第66行，该函数使用当前函数所在的Goroutine从`output`通道中接收值。而接收到的值将会在第67行打印出来。当所有值接收完毕，并打印之后，该循环就会终止。

表面上看起来，运行此程序来转换数据似乎能正常工作。但实际上，该程序泄漏了一些Goroutines。该程序根本执行不到82行来宣告`worker`函数已经执行完毕。甚至在`processRecords`函数返回后，每个`worker`Goroutine仍然在第78行等待`iuput`通道。在一个通道上`range`迭代接收值，___直到该通道关闭且没有值之后___。问题就在于该程序没有关闭`input`通道。

## 修复：通道信号完成

修复这个问题只需增加一行代码：`close(input)`。关闭一个通道意味着向该通道的接收者广播了一条”不会再有数据发送“的通知。在适合关闭通道的地方就是在该通道最后一个值发送完成之后。如清单2中50行所示：

### 清单2

[https://play.golang.org/p/QNsxbT0eIay](https://play.golang.org/p/QNsxbT0eIay)

```go
45     total := len(records)
46     input := make(chan string, total)
47     for _, record := range records {
48         input <- record
49     }
50     close(input)
```

关闭一个仍然有值的通道是合法的。通道关闭的仅仅是发送操作，而不是接收操作。`worker`Goroutine运行`range input`来接收通道中的值，知道该通道被通知已经关闭。这才可以让workers在终止之前退出循环。

## 结论

与上篇博客提到的一样，虽然Go提供了非常简单的Goroutine使用方式，但作为开发，你仍然需要小心使用它。在本文中，我列举了另一个非常容易造成Goroutine泄漏的案例。事实上，在使用Goroutine进行并发编程时，还有很多的陷阱会造成Goroutine泄漏。之后的博客将会讨论这些陷阱。和往常一样，我仍需要重申下忠告：[永远不要在不知如何停止Goroutine的前提下启动Goroutine](https://dave.cheney.net/2016/12/22/never-start-a-goroutine-without-knowing-how-it-will-stop)。

> Concurrency is a useful tool, but it must be used with caution.

并发是一把利器，但必须谨慎使用。

---

Referenced By: [goroutine-leaks-the-abandoned-receivers](https://www.ardanlabs.com/blog/2018/12/goroutine-leaks-the-abandoned-receivers.html)
