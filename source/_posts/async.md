title: 异步编程杂谈
date: 2017-09-09 23:50:57
categories: java
tags: 异步编程
---
响应式编程、RxJava、Vert.x、异步编程、事件驱动、NodeJs、Callback、高性能，一堆很诱惑人的名词，折腾了一段时间，总结一些内容也提出一些疑问。<!--more-->
## 异步编程
关于异步和同步，阻塞与非阻塞，给一个在IO模型里的定义：
> 阻塞与非阻塞关注的是线程的状态，IO操作会不会把当前线程挂起。
同步和异步关注的是消息通知机制，调用者需不需要关注这个数据返回。

关于IO模型这里有[一篇](http://twogoods.github.io/2017/04/27/IO-Model/)，它在说的操作系统里的IO模型。它似乎跟我们日常开发过程中的一些地方有区别，如Java的这段代码，它非阻塞吧，它是异步吗？应该也是吧，开了异步线程去执行了啊。但是再去套一下上面的异步与非阻塞的定义，似乎说它是同步非阻塞才对啊，在这里我们调用者是**关注**这个结果的嘛。
`executor.submit(()->sql.query("select * from table"));`


说到异步非阻塞，总要提NodeJs，它是怎么实现异步IO的呢？下面摘一段《深入浅出NodeJs里的解释》：
> 通过让部分线程进行阻塞IO或者非阻塞IO加轮询技术(NIO,epoll)来完成数据获取，让一个线程进行计算处理，通过线程之间的通信将IO得到的数据进行传递，这就轻松实现了异步IO(尽管它是模拟的)

这段话似乎和我们上面的代码例子很像哦，它通过用异步线程的方式来执行IO操作，就像这段话最后的括号所说，尽管它是模拟的(这里是跟操作系统提供的AIO做比较)。
这么看来实现异步也不难嘛，造一个线程池，把IO操作放进独立的线程里去执行，这就是一个非常朴素的异步思想啊。

## Vert.x 
Vert.x 是Java语言下的一个异步框架，跟NodeJs非常非常的像，我们简单了解一下它是这么实现的。如下数据库查询操作，和Nodejs的写法几乎一样吧！

```
 client.getConnection(res -> {
       SQLConnection connection = res.result();
       connection.query("SELECT count(1) FROM T_User", res2 -> {
           connection.query("SELECT count(1) FROM T_Book", res3 -> {
               System.out.println(res2.result().getRows() + "--" + res3.result().getRows());
           });
       });
   });
```
我们只看`getConnection`操作，其他都类似。`getConnection`传入一个回调函数

```
 public SQLClient getConnection(Handler<AsyncResult<SQLConnection>> handler) {
        Context ctx = this.vertx.getOrCreateContext();
        // 在线程池中取线程执行真正的拿数据库连接的工作
        this.exec.execute(() -> {
            Future res = Future.future();
            try {
                Connection conn = this.ds.getConnection();
                Object execMetric = null;
                SQLConnection sconn = new JDBCConnectionImpl(ctx, this.helper, conn, metrics, execMetric, this.rowStreamFetchSize);
                res.complete(sconn);
            } catch (SQLException var9) {
                res.fail(var9);
            }
            ctx.runOnContext((v) -> {
                //取到数据库连接执行回调函数，
                //这个回调函数又会在另一个线程里执行，这个线程是eventloop线程
                res.setHandler(handler);           
             });
        });
        return this;
    }
```
看到这里，如果让你封装一个异步操作的API，相信你也能很快做出来。Ok，简单了解了实现，那么一个有意思的问题来了。上面的源代码中，获得数据库连接是在通过`this.exec.execute`在一个线程池中执行的，那回调函数为什么在`ctx.runOnContext`事件循环线程中执行，继续使用`this.exec.execute`执行不行吗？其实在当前的这个场景下看起来是可以的，更大的原因是我们有一系列的回调函数，我们都会让回调函数在EventLoop中执行，这样可以很方便的重新回到事件循环中去。可以[参考stackoverflow](https://stackoverflow.com/questions/46030041/some-questions-about-thread-pool-in-vert-x/46030664#46030664)。

再看一下NodeJs异步的实现方式，Vert.x 和它是不是几乎是一样的。
![Node异步IO实现](http://oepm97cib.bkt.clouddn.com/node.jpeg)
## callback
在说callback之前要先说说另一个话题，为什么利用异步非阻塞IO的NodeJs一出来就标榜高性能，到底高性能在哪了？你会不会觉得我这个问题问的莫名其妙，异步非阻塞的我一个操作调用了就可以立马去做下一个操作了不需要等待，当然性能高啦！前面这句话该怎么理解呢？我用自己的思维方式尝试着理解了一下：

```
//一个耗时15ms,一个10ms
data = sql.query("select * from table");
count= sql.query("select * from table");
//time=15+10

future=executor.submit(()->sql.query("select * from table"));
future2=executor.submit(()->sql.query("select * from table"));
//time=max(15,10)
```
这样抽象过后，从本质上来说这是并行带来的好处。我们把关注点放在`sql.query`这个IO操作上，如果这个IO是阻塞的，那这里我们只享受到了两个query并行带来的好处，并没有享受到非阻塞IO带来的好处。试想一下，我们开了两个线程，可是这两个线程在执行query操作的时候都阻塞了，经历了线程被挂起再被唤醒的过程，如果系统大量的都是这种操作，那么线程切换的开销也是巨大的。所以总结一下，一个是多个操作可以并行所带来的优势，一个还是非阻塞IO带来的硬件资源充分利用的优势。(以上，欢迎指出错误。)

当我知道了这些，似乎获得了魔法，急着跑回到Java世界想用异步改造我的代码，可结果是....大量的业务逻辑都是前后依赖，没什么并行度，强行上异步没得到啥好处的同时引入了复杂度。我们说callback是异步世界写同步代码所呈现的形式，如果你的代码里类似的callback多，那说明这个操作内部并行度本身就不高，并行带来的好处不大。现在Java里大大小小的框架大都没有异步API，没有非阻塞IO，改造往往是从入门到放弃。延伸一下，[为什么JDBC操作不用NIO?](https://www.zhihu.com/question/23084473)

好，现在我们再聊callback，异步编程的风格将函数的业务重点从放回值转移到了回调函数中。依然用Vert.x的那段代码，这里callback套了三层。为什么会嵌套？由于各个操作会依赖于前一个操作，不可避免的就嵌套了，因此在异步为主导的编程框架里，就容易写成callback一层一层嵌套的形式。

```
client.getConnection(res -> {
       SQLConnection connection = res.result();
       connection.query("SELECT count(1) FROM T_User", res2 -> {
           connection.query("SELECT count(1) FROM T_Book", res3 -> {
               System.out.println(res2.result().getRows() + "--" + res3.result().getRows());
           });
       });
   });
```
在一般的Java项目里我们会用同步的方式会这么写：

```
SQLConnection connection=client.getConnection();
data1=connection.query("SELECT count(1) FROM T_User");
data2=connection.query("SELECT count(1) FROM T_Book");
```
有了比较，我们可以更一针见血的说，callback的产生是因为要在异步为主导的世界里写同步代码而不得不采取的做法。当然啦，callback hell 很难受，NodeJs提供了多个API来解决这个问题，我不太熟这个所以不打算深入介绍，但是上面这个Java写的例子我们有必要解决一下。
## 响应式编程和异步编程
响应式编程的两个特点是异步与流，它帮助你用最简便的方式实现异步，并且它将一切看作流，从这个流出发，过滤、组合、变换。即将到来的JDK9也会实现响应式编程的规范带来[Flow API](https://community.oracle.com/docs/DOC-1006738)，随JDK9一起到来的Spring5正式版中也将引入对响应式web编程的[支持](https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/reactive-web.html#webflux)。如果你是一个新手，建议参考这篇关于[响应式编程介绍](http://blog.csdn.net/womendeaiwoming/article/details/46506017)。

我们利用RxJava 尝试着解决一下callabck的问题。还是SQL操作的代码：

```
lient.getConnection(res -> {
       SQLConnection connection = res.result();
       connection.query("SELECT count(1) FROM T_User", res2 -> {
           connection.query("SELECT count(1) FROM T_Book", res3 -> {
               System.out.println(res2.result().getRows() + "--" + res3.result().getRows());
           });
       });
   });
```
RxJava:

```
Observable<SQLConnection> conObservable = Observable.create(emitter -> {
            client.getConnection(res -> {
                emitter.onNext(res.result());
                emitter.onComplete();
            });
        });

Observable<String> data1 = conObservable.concatMap(connection -> {
  return Observable.create(observableEmitter -> {
      connection.query("SELECT count(1) FROM T_User", res2 -> {
          observableEmitter.onNext(res2.result().getRows().toString());
          observableEmitter.onComplete();
      });
  });
});

Observable<String> data2 = conObservable.concatMap(connection -> {
  return Observable.create(observableEmitter -> {
      connection.query("SELECT count(2) FROM T_User", res2 -> {
          observableEmitter.onNext(res2.result().getRows().toString());
          observableEmitter.onComplete();
      });
  });
});

 Observable.zip(data1, data2, (d1, d2) -> d1 + d2).subscribe(System.out::println);
```
Java8 CompletableFuture 也同样可以实现：

```
CompletableFuture<SQLConnection> promise = new CompletableFuture<>();
client.getConnection(res -> promise.complete(res.result()));
promise.thenCompose(conn -> {
  CompletableFuture<String> future = new CompletableFuture<>();
  conn.query("SELECT count(1) FROM T_User", data -> {
      future.complete(data.result().getRows().toString());
  });
  return future;
}).thenCompose(s -> {
          CompletableFuture<String> future = new CompletableFuture<>();
          promise.thenAccept(conn -> {
              conn.query("SELECT count(1) FROM T_User", data -> {
                  future.complete(data.result().getRows().toString() + "  " + s);
              });
          });
          return future;
      }
).thenAccept(System.out::println);
```
虽然解决了层层嵌套的问题，但代码量实在是有点多，没办法，这就是Java，当然如果你有更好的办法请告诉我。

异步编程也会带来一些不好处理的地方，典型的一个就是异常处理，异步执行的代码出异常了如何通知到调用的线程。NodeJs是通过在回调函数中传入error来处理异常，类似`db.query('select * from table , function(err,result){}`。Java编程中虽然RxJava还是封装了不错的异常处理机制，但比起同步模式下的编程，异步代码中的异常处理会是一个痛点。

欢迎指正。

参考：
http://emacoo.cn/backend/reactive-overview/
http://www.streamis.me
https://stackoverflow.com/questions/22412960/vert-x-and-bio-threading?rq=1


