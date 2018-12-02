# 长轮询- 概念与思考

在本文中，我们将深入了解名为“长轮询”的技术，它是如何实现的，如何在你自己的应用程序中使用它，以及我们如何在*Ably*使用的。

## 简要的历史 - 长轮询如何而来

在web技术发展的早期阶段，[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)被当做是一个简单的请求-应答的协议，因为它被设计为向感兴趣的读者分发结构化文档的一种方式，每个文档通常都会链接到在读者闲暇时间才会浏览的其他文档。请求-响应的模型是完全正确的策略，因为浏览器只会请求用户希望看到的资源内容，并且它的工作完成的很好。当时的技术相较于今天来说非常有限，保持连接开放不仅是web服务器负载管理的一个重要问题，而且无论怎么说都有一定的价值，在早期网站旨在于解决的问题。早期的浏览器如*Mosaic*和*Netscape Navigator*的第一个版本就是基于这个思想实现的，它不支持*JavaScript*或其他可能将web定位为分布式资源共享网络的特性。

在1995年时，[Netscape 社区](https://en.wikipedia.org/wiki/Netscape)雇佣了*Brendan Eich*来为*NetScape Navigator*浏览器实现脚本的兼容性，大概过了10天左右，*JavaScript* 语言诞生了。与现代 *JavaScript* 相比，它作为一门语言最初的作用非常有限，而且它与浏览器文档对象（DOM）的交互更加有限。*JavaScript* 主要的功能用于提供有限的增强功能来丰富浏览器文档的消费能力，例如执行浏览器表单的验证以及为现有的文档插入动态的HTML。

随着[浏览器战争](https://en.wikipedia.org/wiki/Browser_wars)升温，微软的IE浏览器发展到了第4版以后，对最健壮的特性集的争夺最终导致了微软引入了所有的浏览器已经普遍支持了十多年的*XMLHttpRequest*对象。

![](https://assets.ably.io/assets/concepts/browser-wars-5cb095a93cf518de18640b3f1f80ee503fc5256968e675d9fa6eec6a1b33f219.png)

`XMLHttpRequest`可以被看做是web界的黑天鹅事件，因为它为web开发人员提供了开始构建真正动态web应用程序的潜力，这些应用程序可以在后台与服务器通信，而不会中断用户的浏览体验。由于浏览器DOM的功能也在显著地扩展，特别是随着*Internet Explorer*发展到第6版，动态web“应用程序”开始变得普遍，提供了从丰富的基于web的电子邮件界面到浏览器内的网站内容管理系统的所有内容。

就像他们说的那样，“进步导致进步”，自然而然地，开发人员开始探索如何实现应用程序，使其功能具有更多“实时”方面的特性。基于web的聊天室以及简单的游戏是那个时期的一些例子。HTTP协议使得这些用例的实现非常具有挑战性，并且经常看到应用程序反复轮询服务器以检查新数据，因为服务器无法在新数据可用时提前通知用户。考虑到这种方法的极度低效，将HTTP请求-响应模型操作为更实时的媒体的创造性方法开始出现。在这些技术中，最流行的可能就是“长轮询”。

## 长轮询是如何工作的

[Long polling](https://en.wikipedia.org/wiki/Push_technology#Long_polling)本质上来说是原始轮询技术的一种更加高效的实现方式。重复创建请求给服务端会非常浪费资源，因为每个新进的请求都必须要创建连接，需要解析HTTP头部，必须处理新的查询的数据，以及必须返回一个响应给客户端（通常来说没有数据）。建立的连接需要关闭，还需要清理一些资源。而不是必须为每个客户端多次重复此过程，直到给定的客户端的新数据可用，长轮询是一种技术，服务端选择让客户端连接尽可能长时间保持打开状态，只有在数据可用或超时阈值达到时才交付响应。

具体的实现一般来说是服务端的概念。在客户端这一侧，只需要管理到服务端的单一连接。当收到响应时，此客户端可以初始化一个新的请求，并根据需要多次重复此过程。这和基本轮询唯一的不同在于，对客户端而言，执行基本轮询的客户端可能故意在每个请求中间留一个短暂的时间窗口，以减少其在服务端的负载，与不支持长轮询的服务器相比，它可能以不同的假设来响应超时。对于长轮询，可以将客户端配置为在监听响应时允许较长的超时时间（通过`Keep-Alive`头部设置）--这通常来说是避免可见的，因为超时时间通常用于指示与服务端通信的问题。

![](https://assets.ably.io/assets/concepts/long-polling-f6e3a73a589fe25d7c7b622a8487a2e8a27a11f00b22b574abb021fbcd7ac2db.png)

除了这些问题之外，客户端需要做的其他事情与参与基本轮询几乎没有什么不同。相比之下，服务端需要管理多个连接的未解析状态。并且它可能需要实现保存会话状态的策略，当使用多个服务器和负载均衡时（通常我们称之为“会话粘性”）。它还需要优雅地处理连接超时的问题，这比*design-for-purpose*（如Websockets，这是一个经过多年轮询作为伪实时通信的传统技术之后建立起来的一个标准。）更容易出现。

## 使用长轮询时的注意事项

由于长轮询实际上只是一种应用于底层的请求-响应机制，因此它的实现具有一定的复杂性。在考虑系统架构的设计时，你需要考虑各种问题。

### 消息有序以及交付保证

[可靠的消息有序](https://www.ably.io/documentation/core-features/authentication)可能会成为长轮询的一个问题，因为有可能来自同一客户端的多个情况都在处理的情况。例如，如果一个客户端有两个浏览器标签页访问了相同的服务端资源，并且客户端应用程序将数据持久化存储到本地，比如`localStorage`或者`IndexedDb`，那么就不能保证重复的数据会被写入多次。如果客户端实现的是一次使用多个连接，那么也有可能发生这种情况，无论是有意为之，还是代码出现了bug。

另外一个问题是，即使服务端发送一个响应，但是由于网络或者浏览器问题可能会阻止消息被成功接收。除非我们实现了某种消息回执确认的机制，否则对服务器的后续调用可能会导致消息的遗漏。

依赖于不同服务器的实现，一个客户端实例对消息的回执确认可能会导致另一个客户端实例根本就收到预期的消息，因为服务器可能会错误地认为客户端已经收到了它所期望的数据。

在任何实时消息传递的系统中实现健壮的长轮询特性时，我们都需要考虑这些问题。

### 性能和扩展性

不幸的是，这样的一个复杂的系统是很难高效扩展的。为了维护一个给定的客户端的会话状态，这个状态必须要在位于负载均衡器之后的所有服务器上共享 - 这是一个很复杂的架构任务 - 或者，同一个会话中的后续客户端请求必须要路由到其初始请求的服务器上。

这种确定性的“粘性”路由在设计上是有问题的，尤其是在基于IP地址执行路由时，可能导致集群中某个服务器出现不适当的负载（热点节点），而其他的服务器处于空闲状态。这样就造成了负载不够均衡。这也可能导致其成为一个潜在的拒绝服务攻击（denial-of-service attack）的载体 - 这个问题需要更进一步的基础设施层来缓解，否则可能就没有必要了。

### 设备支持和回退

现在（文本撰写于2018年），长轮询对于web应用程序的开发来说可能已经不相关，因为实时通信标准（如Websockets和WebRTC）已经广泛应用。也就是说，在某些情况下，某些网络上的代理和路由器会阻塞Websocket和WebRTC连接，或者网络连接会使得诸如此类的长连接协议变得不那么实用。此外，对于某些客户端群体来说，可能仍有许多设备和客户端在使用旧的标准，缺乏新标准支持。对于这些问题，长轮询可以作为一个很好的备选，以确保支持各种情况。

由于长轮询是在*XMLHttpRequest*后端实现的，而*XMLHttpRequest*在设备支持方面几乎是通用的，这些设备在现代web上仍然具有不可忽视的重要性，因此通常不需要支持更多的回退。若有例外情况必须处理，或者一个服务器可以查询新数据但不支持长轮询(更不用说其他更现代的技术标准)，基本轮询也只能是有限的使用，那么可以使用*XMLHttpRequest*实现,或通过*JSONP*简单的HTML脚本标签。

如果确实需要长轮询，那么可以通过*JSONP*实现长轮询。尽管如此，考虑到在实施这些方法所花费的时间和精力（更不用说资源消耗的低效），在开发新的应用程序和系统体系架构时，我们应该仔细评估方案是否值得增加成本。在大多数情况下，你可能会对更现代的东西（如Websocket）有独有的支持。又或者，你可能希望将这类问题的管理工作移交给专业的云提供商，例如[Ably Realtime](https://www.ably.io/)。

## 开源的长轮询解决方案

大多数的开源库都不是独立于其他传输来实现长轮询的，因为一般来说，长轮询通常伴随着其他传输策略，或者作为备选方案，又或者当长轮询不起作用时，这些传输策略作为备选方案。在2018年之后，独立的长轮询库更加不常见，因为面对一些更加现代的替代方案时，这种技术正在加速退出历史舞台。

不过，下面有几种语言的开源版本可供选择：

### Go: [golongpoll](https://github.com/jcuga/golongpoll)

一个Go语言实现的长轮询功能较为良好的库。

```go
import  "github.com/jcuga/golongpoll"

// This launches a goroutine and creates channels for all the plumbing

manager, err := golongpoll.StartLongpoll(golongpoll.Options{})  // default options

// Pass the manager around or create closures and publish:

manager.Publish("subscription-category", "Some data.  Can be string or any obj convertible to JSON")

manager.Publish("different-category", "More data")

// Expose events to browsers

// See subsection on how to interact with the subscription handler

http.HandleFunc("/events", manager.SubscriptionHandler)

http.ListenAndServe("127.0.0.1:8081", nil)
```

你可以从[GitHub仓库](https://github.com/jcuga/golongpoll)中找到有关此库的更多信息。

### Python: [A simple COMET server](https://github.com/jedisct1/Simple-Comet-Server)

这是一个Python语言的最小化的HTTP长轮询库。它支持多通道和跨域请求。语言版本要求为 Python >= 2.5， 并且使用了[Twisted 框架](https://twistedmatrix.com/trac/)。不同于其他语言的长轮询库，此服务器作为一个代理运行在客户端与服务器之间。当服务器想推送消息给客户端时，它可以向HTTP端点发出请求，该端点很容易隐藏在防火墙后面，以确保服务器端进程只能访问该端点。

该库的*README*提供了一些例子：

- 注册通道
- 发布消息至一个通道
- 注册客户端
- 监控服务器
- 从通道中读取数据

你可以从[GitHub仓库](https://github.com/jedisct1/Simple-Comet-Server)中找到有关此库的更多信息。

## Ably 和长轮询

实际上，我们已经在客户端库中实现了一套健壮的传输协议，可以与服务器进行通信，而*WebSockets*是实时通信的主要协议。但是，当客户端上没有*WebSockets*这样的新标准时，长轮询完全可以作为自动版本回退的传输。可以了解更多关于[Ably Realtime Platform](https://www.ably.io/platform)的信息。

## 引用和延伸阅读

本文译自：[Long Polling Concepts and Considerations](https://www.ably.io/concepts/long-polling)

- [WebSockets – A Conceptual Deep-Dive](https://www.ably.io/documentation/concepts/websockets)

- [IETF: Known Issues and Best Practices for the Use of Long Polling and Streaming in Bidirectional HTTP](https://tools.ietf.org/id/draft-loreto-http-bidirectional-07.html)

- [The Myth of Long Polling](https://blog.baasil.io/why-you-shouldnt-use-long-polling-fallbacks-for-websockets-c1fff32a064a)

- [HTML5 WebSocket: A Quantum Leap in Scalability for the Web](http://websocket.org/quantum.html)

- [Wikipedia: Push Technology](https://en.wikipedia.org/wiki/Push_technology#Long_polling)




