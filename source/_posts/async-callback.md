title: 再看callback hell
date: 2018-01-29 13:47:06
categories: java
tags: 异步编程
---
以一个在实际中是非常常见为例，分页查询的rest接口，正常的同步逻辑是这样的：<!--more-->

```
//示例1
SQLConnection connection=client.getConnection();
list = connection.query("select * from user lmit 0,20");
count= connection.query("select count(1) from user");
//组装json
```
查总数和查列表数据没有依赖关系，改造一下

```
//示例2
Executor executor = Executors.newFixedThreadPool(10);
SQLConnection connection=client.getConnection();
listFuture=executor.submit(()->connection.query("select * from user lmit 0,20"));
countFuture=executor.submit(()->connection.query("select count(1) from user"));
//通过 listFuture.get() countFuture.get()  组装json
```
这里executor.submit会立即返回，但你不知道这个任务到底要执行多久，而future.get()依然有阻塞当前执行线程的可能，并不是一种特别好的方式。再看看异步代码会怎么写，以下代码都以Vertx为例

```
//示例3
lient.getConnection(res -> {
       SQLConnection connection = res.result();
       connection.query("select * from user lmit 0,20", res2 -> {
           connection.query("select count(1) from user", res3 -> {
               //组装json
           });
       });
   });
```
可以看到示例3是示例1的异步化写法，导致了callback hell,而且这里又不同于示例2。ok,这里怎么改造一下能解决callback和并行化的问题？主要有三种方法，future、coroutine、rx

```
Future<SQLConnection> connectionFuture = Future.future(future->client.getConnection(future));

Future<ResultSet> countFuture = Future.future();
Future<ResultSet> rowFuture = Future.future();

Future<ResultSet> tmp = Future.future();
connectionFuture.setHandler(res -> {
  SQLConnection connection = res.result();
  connection.query("SELECT count(1) FROM T_User", countFuture.completer());
  connection.query("SELECT * FROM T_User", rowFuture.completer());
});

countFuture.setHandler(res -> {
  tmp.complete(res.result());
});

rowFuture.setHandler(res -> {
  tmp.setHandler(countSet -> {
      ResultSet count = countSet.result();
      ResultSet raw = res.result();
      //组装json
  });
});
```
上面这种呢造了一个中间future,很难受

```
Future<SQLConnection> connectionFuture = Future.future(future->client.getConnection(future));

Future<ResultSet> countFuture = Future.future();
Future<ResultSet> rowFuture = Future.future();
connectionFuture.setHandler(res -> {
  SQLConnection connection = res.result();
  connection.query("SELECT count(1) FROM T_User", countFuture.completer());
  connection.query("SELECT * FROM T_User", rowFuture.completer());
});

CompositeFuture.all(countFuture, rowFuture).setHandler(ar -> {
//只有当all里的future都完成了才会调用handle
  if (ar.succeeded()) {
      ResultSet count = countSet.result();
      ResultSet raw = res.result();
      //组装json
  }
});
```
另外再看看rx系列怎么解决这个问题

```
Observable<String> observable = Observable.create(observableEmitter -> {
  //改造代码成rx风格，使用create，在这个地方写原来的代码
  client.getConnection(res -> {
      SQLConnection connection = res.result();
      connection.query("SELECT count(1) FROM T_User", res2 -> {
          connection.query("SSELECT * FROM T_User", res3 -> {
              observableEmitter.onNext(res2.result().getRows().toString());
              observableEmitter.onComplete();
          });
      });
  });
});

observable.subscribe(s -> System.out::println);
```
当然上面只是硬搬过来而已利用了RX的api，感觉并没有解决callback的问题

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
      connection.query("SELECT * FROM T_User", res2 -> {
          observableEmitter.onNext(res2.result().getRows().toString());
          observableEmitter.onComplete();
      });
  });
});

 Observable.zip(data1, data2, (d1, d2) -> d1 + d2).subscribe(System.out::println);
```
上面这种操作数据库的方式已经很底层了，RX包装后两个查询操作并不会共用一个connection，如何做呢？RX系列感觉很困难，我还没找到好的方法，但是在我们的项目代码里一条SQL往往要封装成一个函数，那么自然每条SQL会用不同的connection，其实这个点不需要纠结，而且当初纠结了好一会儿......

回调在不阻塞任何事情的情况下，解决了Future.get()过早阻塞的问题。RX系列利用观察者模式实现，观察者模式订阅，被观察者如何通知观察者呢？调用观察者注册的handle，某种程度来说这也是回调。

以上是rxjava的代码，spring5webflux里使用了另一个[RX库](http://projectreactor.io/docs/core/release/reference)，使用都差不多。

关于RX系列只能先写到这里，感觉还没有完全get到精髓。据说在很fp的语言上写这个会非常方便，大神给出的从oop到fp之路:java -> kotlin -> scala -> clojure -> haskell

## 协程
看到这里你也发现了，原来三四行的代码，一改异步，再解决callback，居然多了这么多行，此刻内心我是拒绝的.....
那么更好的方案就是：coroutine，协程在异步IO编程模式里能大大简化异步回调的实现与逻辑处理。

**协程**是用户态的线程，非常轻量级，占用内存甚至只需几百byte，系统可以创建上百万的协程，并且在任务切换时对CPU负担非常小；线程的另一个特点是可以在任一地方挂起，自己会保存栈信息，下次执行时恢复，协程上有个经常出现的一个词yield，说的就是挂起这点。

在这里你可能也和我一样有一个疑问，那为什么现在的语言不以协程为执行单位(这么说好像违反了我们对操作系统进程线程的理解)，由于它是用户态的线程，所以遇到系统调用我们在想办法搞别的，这样的话你背后完全可以搞成异步模式而对于程序员来说几乎无感知。谁知道可以告诉我。


>协程的本质上其实还是和上面的方法一样，只不过他的核心点在于调度那块由他来负责解决，遇到阻塞操作，立刻yield掉，并且记录当前栈上的数据，阻塞完后立刻再找一个线程恢复栈并把阻塞的结果放到这个线程上去跑，这样看上去好像跟写同步代码没有任何差别，这整个流程可以称为coroutine，而跑在由coroutine负责调度的线程称为Fiber。比如Golang里的 go关键字其实就是负责开启一个Fiber，让func逻辑跑在上面。而这一切都是发生的用户态上，没有发生在内核态上，也就是说没有ContextSwitch上的开销。

我的个人理解：协程那样搞了之后io操作block的是协程，协程本身yield和恢复的开销非常小，远小于线程上下文切换的开销，协程可以很廉价的被创造出来，但是他本质上还是跑在一个线程上面，一个协程被block线程会转而执行(调度)其他协程。

ok，接下来看一下代码，首先是Java语言，JVM本身没有提供协程的支持，好在有第三方工具的支持[quasar](https://github.com/puniverse/quasar)，至于quasar怎么使用大家看[文档](http://docs.paralleluniverse.co/quasar/)或者刘大神的[博客](http://www.streamis.me/2016/05/04/java-next-generation-1/)，vertx自己也封装了一下quasar，引入Vertx-sync即可。

```
SQLConnection connection = awaitResult(h -> client.getConnection(h));
ResultSet countRes = awaitResult(h -> connection.query("SELECT count(1) FROM T_User", h));
ResultSet listRes = awaitResult(h -> connection.query("SELECT * FROM T_User", h));
```
可以看到它跟示例1一样只有三行代码，就是同步模式下的编程习惯。JVM上有协程的一门语言是[Kotlin](https://www.kotlincn.net/docs/reference/)，经过vertx封装后代码几乎跟上面的一样，但是我第一次看Kotlin语法的时候还是各种不适应，带有fp味道的就是有点抽象......

https://www.npmjs.com/package/async
https://zhuanlan.zhihu.com/p/28046403
https://juejin.im/post/5a3b44bbf265da43062aee37#heading-3
http://www.streamis.me/2016/05/04/java-next-generation-1/



