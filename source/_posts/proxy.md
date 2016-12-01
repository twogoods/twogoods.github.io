title: 关于动态代理
date: 2016-11-02 15:34:34
categories: java
tags: 动态代理
---
一般的程序不太会用到动态代理，在写库或者框架的时候用的多一些，我呢硬着头皮在试着完成一个类似spring的框架，过程中自然碰到了动态代理的一些小问题。  <!--more-->

```
@Configuration
public class WebAppConfig extends WebMvcConfigurerAdapter {
    @Value("${name}")
    private String name;
    
    @Bean
    AuthorizeInterceptor authorizelInterceptor() {
        return new AuthorizeInterceptor(name);
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authorizelInterceptor())
                .addPathPatterns("/user/**");
        super.addInterceptors(registry);
    }
}
```
我是这么理解这段代码的：@Bean注解的方法里new出来的对象会仍到IOC里，下面调用了上面注解的方法会在IOC里拿出这个Bean。想改变方法的执行，首先想到的是动态代理。  

* JDK的动态代理，文中例子里下面这个方法addInterceptors()调用authorizelInterceptor()方法，在执行authorizelInterceptor()这个方法的时候是原模原样执行的不会再走一次代理。
* Cglib，方法之间调用都会走代理，适合这个例子。

解析`@Bean`最先想到的方法是这样：创建普通Bean一样创建实例，然后调用所有的`@Bean`注解的方法，方法得到的Bean放入IOC，最后创建一个这个类的代理对象放入IOC，也就是定义的这个Bean的实例在IOC里实际是一个代理对象。(这个方法有问题，像上面这个例子，`addInterceptors()`会调用`authorizelInterceptor()`普通对象调用的时候会创建一个`AuthorizeInterceptor`,`authorizelInterceptor()`调用的时候又会创建一个)。为解决这个问题，我们可以一开始就创建代理对象，代理对象调用方法的时候，先查看IOC里有没有这个Bean，没有就创建，有就直接返回IOC里的实例，这样就确保一个Bean只产生了一个实例。  
我们一开始就创建一个这个类的代理对象，又发现这个类注入了一个属性值，方法里也会使用这个属性，而我在查看创建的代理对象的field时发现并没有`name`这个field，所以没办法直接设置这个值，这时只能调用`setter`即`setName()`来赋上值。然而如果我类里没有创建`setter`那就无法赋值，好在Cglib可以动态的为属性加上`setter`方法来解决这个问题。  
最后，以上是我拿到这个问题自己的一些思考，还并没有看过Spring的实现，有时间去了解一下补上spring的实现。

---

[代码参考](https://github.com/twogoods/tiny4j/blob/master/core/src/test/java/com/tg/tiny4j/core/aop/CglibDynamicAopProxyTest.java)
