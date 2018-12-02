> 译自：[Concurrency Synchornization_Overview](https://go101.org/article/concurrent-synchronization-overview.html)。

本文将解释什么是同步，并且会列举Go支持的一些同步技术。

# 什么是同步？

通常，在运行时，在并发的程序当中，多个goroutine将会访问同一个值。对于这种情况，我们必须保证在给定的时间某个goroutine能够获取该值的所有权。否则，将有可能发生数据竞争，并且该值的公平性（一致性）也无法得到保证。

我们使用各种数据同步技术来传输或保护goroutine之间的值的所有权，以避免在当前的程序中出现数据竞争。

在这里，更具体的来说，关于值的所有权，

- 一个值的写所有权是独占的。在一个goroutine拥有一个值的写权限的周期内，其他goroutine没有该值的访问权限，无论是读取还是写入。
- 一个值的读所有权是非独占的。在一个goroutine拥有一个值的读权限的周期内，其他goroutine可以安全地读取该值，但是不能写入该值。

_（实际上，更确切地说，为了避免数据竞争，我们所关心的实际上内存段的所有权。在运行时，一个值可能会占据多个内存段。在编写一段并发代码时，我们可能不关心值所占用的所有内存段的所有权。但是，为了方便解释，我们仍将会使用“值的所有权”这样的措辞。）_

# Go支持哪些数据同步技术？

文章[channels in Go](https://go101.org/article/channel.html)已经向我们介绍了如何使用通道来进行同步。除了通道之外，Go还支持另外一些通用的数据同步技术，例如互斥量和原子操作。请阅读下面的文章来获取更多关于Go所支持的同步技术：

- [Channel Use Cases](https://go101.org/article/channel-use-cases.html)
- [How to Graceful Close Channels](https://go101.org/article/channel-closing.html)
- [Concurrency synchronization Techniques Provided In The `sync` Standard Package](https://go101.org/article/concurrent-synchronization-more.html)
- [Atomic Operations Provided In The `sync/atomic` Standard Package](https://go101.org/article/concurrent-atomic-operation.html)

我们还可以通过使用网络和文件IO来进行同步。但是这些技术在单进程程序中是非常低效的。它们常用于进程之间和分布式的同步。Go101教程不会涉及到此类技术。

想要更好地理解这些同步技术，推荐阅读[memory order guarantees in Go](https://go101.org/article/memory-model.html)这篇文章。

Go中的数据同步技术不能防止你编写[improper concurrent code](https://go101.org/article/concurrent-common-mistakes.html)。然而这些技术可能帮助你更加容易地写出正确的并发代码。而且独一无二的通道特性能够使并发变成更加灵活和有趣。
