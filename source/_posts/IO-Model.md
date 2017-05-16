title: 简单聊聊异步同步
date: 2017-04-27 14:37:29
categories: java
tags: IO
---
我们先看一段伪代码：

```
data = sql.query("select * from table");//15毫秒
count= sql.query("select * from table");//10毫秒
```
<!--more-->上面这段代码有两个耗时操作，他们顺序执行，完成需要25毫秒的时间，我们把这样的称为同步阻塞调用。
```
future=executor.submit(()->sql.query("select * from table"));
future2=executor.submit(()->sql.query("select * from table"));
```
我们启用线程池，配合Future，`future.get()`会阻塞到操作完成，最终两个操作可以从25毫秒缩短到15毫秒，在实习当中不少同事就会把这样的代码叫做异步的，或许这样的说法久了大家就默认了，但这本质上不是异步。
### IO模型
讲这个前自然要先说说阻塞与非阻塞，异步与同步。

> 阻塞与非阻塞关注的是线程的状态，IO操作会不会把当前线程挂起。
> 同步和异步关注的是消息通知机制，调用者需不需要关注这个数据返回。

以下是4种IO模型:

![](http://oepm97cib.bkt.clouddn.com/io1.gif)
![](http://oepm97cib.bkt.clouddn.com/io2.gif)
![](http://oepm97cib.bkt.clouddn.com/io3.gif)
![](http://oepm97cib.bkt.clouddn.com/io4.gif)

IO多路复用的一个好处就是以前多个用户线程的阻塞变为了一个线程上的阻塞，单个线程就可以同时处理多个网络连接的IO以支持大量的并发.虽然I/O多路复用的函数也是阻塞的，但I/O多路复用是阻塞在select，epoll这样的系统调用之上，而没有阻塞在真正的I/O系统调用如recvfrom之上。我们的NIO是同步非阻塞的，在IO多路复用那张图上我们看到我们还是主动发起系统调用将数据从内核拷贝到用户空间，这里就说明是同步的，此时的用户线程没有挂起。

推荐阅读:
[nio是同步非阻塞IO](https://www.zhihu.com/question/27991975)
[同步异步阻塞非阻塞](https://www.zhihu.com/question/19732473)
[io模型](http://blog.csdn.net/historyasamirror/article/details/5778378)
