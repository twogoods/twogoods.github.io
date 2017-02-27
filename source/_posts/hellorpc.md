title: HelloRPC
date: 2017-02-22 23:01:11
categories: java
tags: RPC
---
在学校里总要拿点什么东西练练手，看了些文章像什么《从零开始写Rpc》，《从零开始写搜索引擎》都是可以一试的。其实前些日子的[tiny4j](https://github.com/twogoods/tiny4j)还有些要改进继续完善的东西，奈何人都是喜新厌旧的，总忍不住再开一坑，所以这次是RPC。<!--more-->  

在开始写之前做了不少准备，了解RPC的一些基本的东西，也找了一些RPC框架在试用，诸如GRPC，thrift，它们都提供跨语言调用的能力，根据写的一些配置便可自动生成一些代码。除了这两个国产框架里也有非常优秀的如阿里开源的dubbo，微博的motan；微博的motan文档很简洁，而且直接支持Springboot的注解形式。马上尝试了一下helloworld，大呼过瘾！不考虑跨语言，对于特定的Java，它的Api设计其简单方便，几个注解搞定所有服务，对于Java，这就是我想要的RPC！代码clone下来一看，传输层用的是netty，为了能够完成这个又跑去搞了本《netty 权威指南》啃了大半本，算是入了个门。  

准备工作差不多了于是开始尝试写，毕竟是第一次写，就不考虑跨语言了，像motan一样配合Spring使用能顺手简单就行。简单梳理的整个RPC调用过程，可以分成以下几层：
![rpc层次](http://i1.piimg.com/4851/71a40fa8d3564fbf.png)
通信包括连接建立、消息传输， 编解码要传输的内容，序列化和反序列化，最后是客户端和服务端的实现，客户调用一个服务的代理实现，发起对服务端的请求。  

基于netty5.0，protostuff、kryo序列化库，commons-pool2实现连接池，服务代理使用jdk自身的Proxy或Cglib，在这样一堆优秀的框架和类库上，整个RPC的构建就简单了不少。
### 使用
贴一段与Springboot整合的代码，完整的在[这里](https://github.com/twogoods/HelloRpc)。
使用`@RpcService`注解服务

```
public interface EchoService {
    String echo(String s);
}

@RpcService
public class EchoServiceImpl implements EchoService {
    @Override
    public String echo(String s) {
        return s;
    }
}
```
`application.yml`配置

```
tgrpc:
    host: 127.0.0.1
    port: 9100
    maxCapacity: 3
    requestTimeoutMillis: 8000
    maxTotal: 3
    maxIdle: 3
    minIdle: 0
    borrowMaxWaitMillis: 5000
```
启动server:

```
@SpringBootApplication
@EnableAutoConfiguration
@EnableRpcServer //启用Server
@ComponentScan()
public class ServerApplication {
    public static void main(String[] args) {
        ApplicationContext applicationContext = SpringApplication.run(ServerApplication.class);
    }
}
```
调用方使用`@RpcReferer`注解.

```
@Component
public class ServiceCall {

    @RpcReferer
    private EchoService echoService;

    public String echo(String s) {
        return echoService.echo(s);
    }
}
```
启动Client发起调用

```
@SpringBootApplication
@EnableAutoConfiguration
@EnableRpcClient //启用Client
@ComponentScan(basePackages = {"com.tg.rpc.springsupport.bean.client"})
public class ClientApplication {
    public static void main(String[] args) {
        ApplicationContext applicationContext = SpringApplication.run(ClientApplication.class);
        ServiceCall serviceCall = (ServiceCall) applicationContext.getBean("serviceCall");
        System.out.println("echo return :" + serviceCall.echo("TgRPC"));
    }
}
```
整个的使用非常简单，当然这中间SpringBoot也占了很大的功劳。
### TODO
当然这还没完，还有很多工作需要做：

- 性能优化
- 服务注册中心

毕竟是一个简单的实现，性能优化方面肯定会去做，但这块又要设计很多的内容：对netty深入的理解，做性能测试找到瓶颈再改进，这些对我来说都是第一次需要慢慢来。
注册中心这个最近已经在做一些工作了，现在考虑会把consul,zookeeper都实现一遍，zookeeper以前就玩过，consul现在已经上手了，基本的服务注册，健康检查的概念及使用都以清晰，接下来会开始实现。

未完，待续，争取全部搞定来个总结。
