# 对“异步 Python 和数据库”的回应

- Source: [Response to "Asynchronous Python and Databases"][source]
- Author: [A. Jesse Jiryu Davis][author]
- Date: 2015-04-01

- 翻译日期：2018-10-24
- 翻译者：张凯朝

几周之前，SQLAlchemy 的作者 Mike Bayer 发表了一篇优秀文章[“异步 Python 与数据库”]["Asynchronous Python and Databases"]，在这篇文章中，他写道：

> 异步编程只是架构上的一种潜在方法，除了编写 HTTP 或聊天服务器或其他特别需要并发保持大量任意缓慢或空闲的TCP连接（其中“任意”意味着，我们不关心单个连接是慢速，快速还是空闲，无论如何都可以保持吞吐量）的应用程序之外，它绝不是我们应该一直使用的方法，甚至大多数时间都不应该使用。

这里写得很好。如果你正在为非常慢的或不活动的网络连接提供服务的话，并且这些网络连接必须保持打开状态使得能无限期地等待事件，那么异步在这种情况下一般都会比为每一个 socket 开启一个线程的方式更好。相反地，如果你的服务器端应用的典型工作是快速处理请求和响应的话，那么异步*可能不*适合。还有，如果服务器端暴露在公网上的话，无论如何，[慢速的 loris 攻击][slow loris attack]都会使得它忙于处理这种异步擅长的工作。因此，面对这种攻击类型的攻击者，你至少需要在服务器端的前面使用一个非阻塞的前端服务器，比如 Nginx。

异步还不仅用于服务器。对于打开很大数量连接并无限等待事件的客户端来说，使用异步将会有更好的纵向扩展能力。这在客户端通常不太常见，不过对于像网络爬虫这种需要巨量 I/O 操作的程序来说，你可能就会意识到异步的确会有优势。

一般原则是：如果你没有同时控制 socket 的两端，那么其中一端可能会是任意慢的，也许是恶意的慢。你控制的那一端最好能有效地处理缓慢的网络连接。

但是如果这是你的应用连接到你自己的数据库呢？这时候，你同时控制着两端，并且你能够保证所有的数据库请求都能被快速处理，正如 Mike 的测试所展示的，你的应用程序可能不需要花太多的时间在等待数据库响应上，他使用 Postgres 进行了测试，不过配置良好的 MongoDB 实例也具有相似的响应能力，那么当你使用一个低延迟的数据库的时候，你的优先点将是你的应用程序的原始速度，而不是其可扩展性。在这种情况下，异步不是正确的选择，至少在 Python 中不是。因为一个服务于低延迟网络连接的小型线程池通常会比异步框架更快。

基于我自己的测试以及我和 Tornado 的作者 Ben Darnell（译者注：Ben 其实只是当前及长期的维护者）的讨论，我同意 Mike 的文章中的观点。正如我[去年在 PyCon 上说的那样][said at PyCon last year]，异步最小化了每个空闲的网络连接中的资源，此时你正在等待某些事件在不确定的未来发生。它的重大利好并不在于它更快，在很多情况下都不是。

Mike 似乎提倡这样的策略：将数据库驱动程序的异步 API 从异步*实现*中分离开来。例如，在 asyncio 中，重要的是你可以使用以下的代码读取来自数据库的数据：

```python
@asyncio.coroutine
def my_query_method():
    # 在等待数据库响应的时候，使用“yield from”会解锁事件循环（event loop）
    result = yield from my_db.query("query")
```

但是*没有*必要使用非阻塞 sockets 和 asyncio 的事件循环（event loop）来重新实现驱动程序本身。如果 db.query 将你的操作推迟到线程池中，并在准备就绪时将结果注入到主线程的事件循环（event loop）中，那么这样可能会更快，并且可以很好地满足于你需要的小量的数据库连接。

那么，我为 MongoDB 和 Tornado 编写的异步驱动程序 [Motor][] 怎么样呢？经过一些努力，我编写了 Motor 来为 [Tornado][] 应用程序提供支持 MongoDB 的异步 API，并使用了 Tornado 的事件循环（event loop）实现对 MongoDB 的非阻塞的连接。（Motor 在内部使用 [greenlets][] 简化后一个任务，不过 greenlets 和这个讨论无关。）如果 Mike Bayer 的文章的观点是对的，我相信这一点，那么 Motor 是多余的吗？

有了 Motor，我实现了两个目标。其中一个是必要的，不过我正在考虑另外一个之中。必要的那个目标是为使用 MongoDB 的 Tornado 应用程序提供异步 API 支持，Motor 成功地做到了这一点。但是我想知道，如果 Motor 使用线程池和阻塞 sockets 而不是 Tornado 的事件循环（event loop）来和 MongoDB 通信的话，是否会有更好的吞吐量。如果我重新开始的话，尤其现在 [concurrent.futures][] 的线程池更主流，我可能会使用线程替代。在一些基准测试中可能会快 10% 或者 20%，并且也能简化未来的开发。今年的晚些时候，我希望能抽出一些时间来试验这个方法的性能和可维护性，以用于 Motor 的未来版本。

----

分类：Mongo, Motor, Programming, Python

[source]: https://emptysqua.re/blog/response-to-asynchronous-python-and-databases/
[author]: https://twitter.com/jessejiryudavis

["Asynchronous Python and Databases"]: http://techspot.zzzeek.org/2015/02/15/asynchronous-python-and-databases/
[slow loris attack]: http://en.wikipedia.org/wiki/Slowloris_%28software%29
[said at PyCon last year]: https://emptysqua.re/blog/pycon-2014-video-what-is-async/
[Motor]: http://motor.readthedocs.org/
[Tornado]: http://www.tornadoweb.org/
[greenlets]: http://greenlet.readthedocs.org/
[concurrent.futures]: http://pythonhosted.org/futures/
