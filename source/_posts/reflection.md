title: Java反射的一点总结
date: 2017-04-05 21:05:11
categories:
tags:
---
平时写业务一般是用不到反射的，但是框架内部会有非常多的反射，像Spring这一类。自己也去尝试实现过一个简单的框架，也有非常多的反射代码，反射跟普通的对象创建方法调用不同，在性能上不如前者，今天尝试着了解一下反射如何进行优化。<!--more-->在我有限的认识里，主要两点:
1. Class,Method,Field对象缓存。这一点好理解，这些对象的获取都是比较耗时的，不要每次使用都去获取一遍。
2. 使用动态字节码技术，如cglib，它生成的代理类实际是被代理类的子类，在子类中调用父类的方法。
为了验证cglib能快多少我简单的写了个测试,环境是jdk8：

```
public class SampleBean {
    public String echo(String name) {
        return name;
    }
}

        测试代码：
        FastClass fastClass = FastClass.create(SampleBean.class);
        FastMethod fastMethod = fastClass.getMethod(SampleBean.class.getMethod("echo", String.class));
        SampleBean myBean = new SampleBean();
        long start = System.currentTimeMillis();
        for (int i = 0; i < 1000000; i++) {
            fastMethod.invoke(myBean, new Object[]{"haha"+i});
            //fastMethod.invoke(myBean, new Object[]{"haha"});
        }
        System.out.println("fastmethod:" + (System.currentTimeMillis() - start));
        
        start = System.currentTimeMillis();
        Method m = SampleBean.class.getMethod("echo", String.class);
        for (int i = 0; i < 1000000; i++) {
            m.invoke(myBean, "haha"+i);
            //m.invoke(myBean, "haha");
        }
        System.out.println("reflect:"+(System.currentTimeMillis() - start));
        
        start = System.currentTimeMillis();
        for (int i = 0; i < 1000000; i++) {
            myBean.echo("haha"+i);
            //myBean.echo("haha");
        }
        System.out.println("normal:"+(System.currentTimeMillis() - start));
```
输出很有意思：fastmethod:196 > reflect:139 > normal:72,cglib是最慢的？！！！如果使用注释的代码跑，fastmethod:38 > reflect:33 > normal:25,时间省了很多但三者的大小顺序没变，时间变少我猜测是百万次调用的参数不变被优化了。
于是我怀疑我的测试代码是不是有问题，在网上看了一通发现Java的benchmark是不容易做的，要剔除无关的操作，还要做好JVM的JIT预热，而且有多种gc算法。所以上述所谓的性能测试就是扯淡，不靠谱的...
而对于我颇感意外的cglib不如Java自身反射api快的结论，在cglib的文档上也能看到一点端倪：
> However, the newer versions of the HotSpot JVM (and probably many other modern JVMs) know a concept called inflation where the JVM will translate reflective method calls into native version's of FastClass when a reflective method is executed often enough.  I would therefore recommend to not use FastClass on modern JVMs, it can however fine-tune performance on older Java virtual machines.

最后，关于Java的benchmark在[stackoverflow](http://stackoverflow.com/questions/504103/how-do-i-write-a-correct-micro-benchmark-in-java)上有一个讨论; 各个字节码框架与Java自身的反射的性能[对比](https://github.com/neoremind/dynamic-proxy),可以参考。结论就是在高版本的Java中反射的性能其实已经挺不错了！

