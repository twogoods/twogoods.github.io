title: netty源码简读
date: 2017-05-24 23:06:33
categories: java
tags: netty
---
Java里著名框架的源码往往不容易读，他们在功能上大而全，把多态用到极致，又有很多的性能优化;相应的由于著名所以网上的资源也非常多，慢慢读总还是能体会到整个框架的设计和思路。配合着网上的博客，书籍和源代码，花了三天时间阅读，有了一定的收获，做一下笔记。
看netty之前回顾一下Java NIO，列一下使用Java API开发NIO所需要的几个步骤<!--more-->

```
    selector = Selector.open();//打开一个多路复用器
    servChannel = ServerSocketChannel.open();//打开一个ServerSocketChannel
    servChannel.configureBlocking(false);//设置异步
    servChannel.socket().bind(new InetSocketAddress(port), 1024);//ServerSocketChannel绑定到地址
    //将ServerSocketChannel注册到selector上
    SelectionKey key = servChannel.register(selector, SelectionKey.OP_ACCEPT);
    
    public void run() {
        //事件循环
        while (!stop) {
                selector.select();//在没有channel可用时一直阻塞
                Set<SelectionKey> selectedKeys = selector.selectedKeys();//获取可用的channel
                Iterator<SelectionKey> it = selectedKeys.iterator();
                SelectionKey key = null;
                while (it.hasNext()) {
                    key = it.next();
                    it.remove();
                    handle(key);//每个channel具体的处理逻辑
                }
        }
    }
```
我们可以看到在`run()`方法里开启了我们的Reactor线程，不断的获取注册在selector上的有新事件的channel，这里我们只用一个线程就处理了所有的连接，远比BIO下一个连接一个线程优秀。在循环代码里我们看到channel具体的处理逻辑是一个一个串行执行的，但是channel间没有任何联系，这个过程我们是可以利用线程池让多个channel并行处理来进一步提高性能。讲到线程我们可以联系一下Reactor的线程模型。

Reactor单线程模型:一个Reactor线程处理所有的连接的接入和读写操作，我们上面给出的代码例子就是这个模型。
![Reactor单线程模型](http://oepm97cib.bkt.clouddn.com/reactor1.png)


Reactor多线程模型:由于单线程模型在面对大并发的情况下就显得力不从心了，所以我们在这里引入了线程池，利用多个线程来处理。具体就是我们只用一组线程来处理读写请求，毕竟读写是网络编程里最重要的两个操作，性能好不好，能不能抗住大并发很大的关键在这里；然后我们利用一个线程专门负责处理客户端的连接操作，注意这里只用了一个线程，连接建立起来之后，将channel注册到线程池中的某一个线程的多路复用器上。好，我们一句话总结：前面一个Reacotr线程处理所有的连接建立操作，确认连接后将Channel注册到后面的Reactor线程池的某个线程上，这样大量的读写操作被平均分配到了多个线程上处理，提高了吞吐量。
![Reactor多线程模型](http://oepm97cib.bkt.clouddn.com/reactor2.png)


主从Reactor多线程模型：在多线程模型中，我们只用了一个线程来处理客户端的登陆、握手和安全认证，在一些特殊场景下还是回存在一些性能问题，于是进一步改造把acceptor的线程也改造成线程池，让多个线程来处理客户端接入。
![主从Reactor多线程模型](http://oepm97cib.bkt.clouddn.com/reactor3.png)


我们把Reactor线程模型简单介绍了一下，这是事件循环的模型。我们上面提出的问题是，某个事件循环里`selector.selectedKeys()`这个操作hi返回一系列可用的channel,这些channel的处理可以使用线程池来并发执行吗？这个问题在上面的三个Reactor线程模型中还暂时得不到答案，我们在源码中看一看。
由于netty是对Java NIO的封装所以一上来我的思路是把使用java API写的NIO程序中的每一个步骤看看是在netty的哪个地方实现的，这样一圈下来就能基本搞明白netty的流程。我们重点看Reactor线程模型和事件机制，netty的事件是在一条pipeline中传播的，每个channel都会有一条pipeline，pipeline里第一个是head,最后一个是tail,中间是一系列自定义的处理器，如果一个处理器可以处理，那么事件传递到这里结束，否则继续传递给下一个handle。

```
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     p.addLast(new EchoServerHandler());
                 }
             });
            ChannelFuture f = b.bind(PORT).sync();
            f.channel().closeFuture().sync();
```
以上是netty的最基本的使用方式，首先我们创建了两个`EventLoopGroup`,对应于Reactor线程模型，第一个是Acceptor用于连接的，第二个是负责读写操作的线程池。深入到`EventLoopGroup`的构造器：

```
    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        //只保留了核心的代码
        if (executor == null) {
            //注意newDefaultThreadFactory只是一个一个的创建线程,这个默认的执行器本质上不是线程池，
            //因为事件循环基本上就是死循环，不太会有线程复用的需求
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
        //这里的children就是EventLoop，一个group里有多个EventLoop，我们说它是线程池
        children = new EventExecutor[nThreads];
        for (int i = 0; i < nThreads; i ++) {
            children[i] = newChild(executor, args);
        }
        //EventLoop的选择策略.默认是一个个轮询
        chooser = chooserFactory.newChooser(children);
    }
    
    //看看newChild方法，
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
    }
    
    //NioEventLoop的构造函数
    NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {
        super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
        if (selectorProvider == null) {
            throw new NullPointerException("selectorProvider");
        }
        if (strategy == null) {
            throw new NullPointerException("selectStrategy");
        }
        provider = selectorProvider;//不同操作系统，多路复用器的实现不一样，有不同提供者
        //这里我们看到了第一个步骤，每个Reactor线程都有一个Selector并在这里被打开
        final SelectorTuple selectorTuple = openSelector();
        selector = selectorTuple.selector;
        unwrappedSelector = selectorTuple.unwrappedSelector;
        selectStrategy = strategy;
    }
```
接下来看看，`ServerBootstrap`的构造,利用构建器模式设置它的属性

```
    //group方法设置两个线程池
    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        //第一个acceptor线程组传入父类
        super.group(parentGroup);
        this.childGroup = childGroup;
        return this;
    }
    //设置handle和childhandle,两个的区别后面有介绍
    handler(new LoggingHandler(LogLevel.INFO))
    childHandler(new ChannelInitializer<SocketChannel>()） 
```
现在我们关注`ChannelFuture f = b.bind(PORT).sync()`的`bind`方法

```
 private ChannelFuture doBind(final SocketAddress localAddress) {
        //初始化,注册
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }
        if (regFuture.isDone()) {
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        promise.setFailure(cause);
                    } else {
                        promise.registered();
                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }

    final ChannelFuture initAndRegister() {
        Channel channel = null;
        //通过设置代码里 channel(NioServerSocketChannel.class) 反射通过构造器创建channel
        channel = channelFactory.newChannel();
        init(channel);//初始化
        //取到的这个group是初始化时的第一个group线程组,在其之上注册
        //register是在SingleThreadEventLoop里执行
        ChannelFuture regFuture = config().group().register(channel);
        return regFuture;
    }
    
    //上面反射创建对象，在NioServerSocketChannel的构造器里，我们看到打开一个ServerSocketChannel
    private static ServerSocketChannel newSocket(SelectorProvider provider) {
         return provider.openServerSocketChannel();
    }
    
	//看看init(channel)，这是个多态方法，由于设置的是server端，实现在ServerBootstrap里
	void init(Channel channel) throws Exception {
        //设置ServerBootstrap创建时属性
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            setChannelOptions(channel, options, logger);
        }
        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e : attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }
        //pipeline会在channel创建的时候也默认创建
        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;//是.group()第二个参数设置的workerGroup
        final ChannelHandler currentChildHandler = childHandler;//childHandler
        //往pipeline里添加一系列的handler，这里只是ChannelInitializer，初始化完真正添加到pipeline里
        //ChannelInitializer的initChannel方法在channel被注册的时候调用,这个时候这个handler被加入pipeline
        ChannelHandler test = new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();//外面的handler直接加进去
                if (handler != null) {
                    pipeline.addLast(handler);
                }
                /**
                    这里的eventLoop是channel注册时的那个eventloop(每个eventloop里都有一个selector)
                    这个execute 并没有立马被执行,这个Runnable被扔到了task队列里
                  */
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        /**
                         * ServerBootstrapAcceptor是一个acceptor
                         * 当前的这条pipeline是serversocketchannel的,serversocketchannel主要是accept一个连接用的,
                         * 它不处理childhandler,而且这个serversocketchannel对应的是bosseventloop,这个eventloop只处理新连接的接入
                         *
                         * 在新的连接接入的时候调用这条pipeline(serversocketchannel的pipeline),
                         * 在这条pipeline里我们完成了serversocket.accept(),建立了连接得到的socketchannel,将他注册到了workeventloop上(这个eventloop关注读写操作)
                         * 我们知道每个channel都会有一条pipeline,在ServerBootstrapAcceptor里我们给刚连接的这个socketchannel里的pipeline添加了childhandler来处理用户自定义的读写请求
                         * (连接建立起来的这个channel完成真正的读写,代码中我们也是在childhandler里添加处理读写的handler,)
                         */
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        };
        p.addLast(test);//只是添加了一个ChannelInitializer,具体的handler这个时候还没有添加进去
    }

   //ServerBootstrapAcceptor这个handler会吧建立好链路的channel注册到workeventloop上，具体实现在其channelRead方法里
   public void channelRead(ChannelHandlerContext ctx, Object msg) {
       final Channel child = (Channel) msg;
       //为客户端新连接的这个channel设置pipeline,这里childHandler就是我们会自己定义的那些处理读写的handler
       child.pipeline().addLast(childHandler);
      //给这个channel注册到复用器,注意这里是注册到childGroup,即第二个workeventloop上
      childGroup.register(child).addListener(new ChannelFutureListener() {
          @Override
          public void operationComplete(ChannelFuture future) throws Exception {
              if (!future.isSuccess()) {
                  forceClose(child, future.cause());
              }
          }
      });
   }
pipeline本身是一个双向链表，通过addLast等方法添加handle
最终serversocketchannel的pipelinede 组成是这样的：headhandler--LoggerHandler--ServerBootstrapAcceptor--tailHandler
每一个建立起来的channel中的pipelinede 组成是这样的：headhandler--childrenhandler--tailHandler
```
Channel的init好后看看`register()`

```
 private void register0(ChannelPromise promise) {
      boolean firstRegistration = neverRegistered;
      doRegister();//执行注册
      neverRegistered = false;
      registered = true;
      //这里把ChannelInitializer里的init方法执行,添加handler进去
      pipeline.invokeHandlerAddedIfNeeded();
      safeSetSuccess(promise);
      pipeline.fireChannelRegistered();//注册好，发送事件
      if (isActive()) {
          if (firstRegistration) {
              pipeline.fireChannelActive();
          } else if (config().isAutoRead()) {
              beginRead();
          }
      }
 }
 
protected void doRegister() throws Exception {
   boolean selected = false;
   for (;;) {
       try {
           //真正的执行注册
           //javaChannel() 取出构造器执行时 通过 SelectorProvider的openXXX方法新建的SelectableChannel
           //第三个参数是注册携带的附件,selectionKey中会携带回来
           selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
           return;
       } catch (CancelledKeyException e) {
           if (!selected) {
               eventLoop().selectNow();
               selected = true;
           } else {
               throw e;
           }
       }
   }
}

```
netty在何处开启事件循环呢？事件循环在NioEvevtLoop的`run()`方法里，沿着调用链往上找我们找到了`SingleThreadEventExecutor`的`execute`方法

```
 public void execute(Runnable task) {
        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            //任何一个EventLoop.execute()都会启动这个selector的主循环
            startThread();
            addTask(task);
        }
    }
private void startThread() {
        if (state == ST_NOT_STARTED) {
            //判断是否已经启动过了，事件循环的线程只会启动一次
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                doStartThread();
            }
        }
    }
```
事件循环开始，我们看到`run()`方法里的`processSelectedKeys()`

```
	private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
        if (selectedKeys.isEmpty()) {
            return;
        }
        Iterator<SelectionKey> i = selectedKeys.iterator();
        //这里是for循环，多个SelectionKey可用时，是串行执行
        for (; ; ) {
            final SelectionKey k = i.next();
            final Object a = k.attachment();
            i.remove();
            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }
            if (!i.hasNext()) {
                break;
            }
            if (needsToSelectAgain) {
                selectAgain();
                selectedKeys = selector.selectedKeys();å
                if (selectedKeys.isEmpty()) {
                    break;
                } else {
                    i = selectedKeys.iterator();
                }
            }
        }
    }
    
	//具体的处理逻辑
	private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        try {
            int readyOps = k.readyOps();
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);
                unsafe.finishConnect();
            }
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                ch.unsafe().forceFlush();
            }
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                /**
                 *此处监听读或者连接操作,这里的read()方法是个多态方法,有两个实现,niomessageunsafe 是 nioserversocketchannel 继承链上的,这里的read 变成了accept连接操作
                 * niobyteunsafe 是niosocketchannel 继承链上的,所以这里确实就是读取操作
                */
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
    
    //niomessageunsafe的read()方法里会执行`doReadMessages()`方法，这个方法由NioServerSocketChannel实现
    protected int doReadMessages(List<Object> buf) throws Exception {
        // 接受新的客户端连接
        SocketChannel ch = SocketUtils.accept(javaChannel());
        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);
            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }
        return 0;
    }
```
我们在源码里找到了在Java NIO里要做的每一步操作，在代码中也看到了创建的两个`EventLoopGroup`的作用：一个接受连接，一个处理读写。最上面我提过一个想法，就是一个`selector`得到多个`selectedKeys`时，利用线程来并发处理，在netty源码里我们看到了，这些`selectedKeys`依旧是for循环的方法串行执行的，为什么这样呢？
这个问题解释起来其实很简单，先看一个简单的例子，你只有一个核心的机器，有5个任务要跑，要求最快完成这5个任务，你是在一个线程上顺序跑这5个任务还是5个线程一起跑呢？当然是顺序跑啦，只有一个核，多线程微观上还是串行执行的嘛，而且还要引入线程切换的开销，所以说假设你的任务非常消耗CPU, 那么现在每个CPU都被占满了, 你再增加线程个数, 只能降低系统的效率, 因为线程还需要切换。所以我们在很多线程池的默认设置中看到N,N+1的线程数设置。我们的`NIOEventLoopGroup`默认是线程数是CPU数*2，这些eventloop全部开启，我们的cpu其实已经在这些事件循环上跑满了，再开线程去跑具体的用户业务逻辑并不是好的做法（线程数默认是2N，这么做是因为eventloop是会阻塞的，这个时候cpu空闲了有能力处理其他的线程了，所以这里我们的线程数比核心数多，线程数=CPU核心数/(1-阻塞系数)）。

这里简单的介绍了一下netty的源码，看的时间短设计的内容也有限，对于netty内部还有很多东西值得去琢磨。列一下过程中参考的资料：《netty权威指南》，《Netty5.0架构剖析和源码解读》。
以前主要的工作就是在Spring上堆业务代码，都是在http之上，netty不是因为自己要写一个rpc框架还真的就用不到，也就对网络编程知之甚少。最近看到的一些内容给我开了一扇门，线程，协程，事件驱动，异步这些都是高性能程序常常出现的名词，也会涉及很多操作系统层次的东西，程序员还是应该往下走，在上面堆业务还是进步太慢。


