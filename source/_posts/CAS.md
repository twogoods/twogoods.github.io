title: CAS中的遇到的一些问题
date: 2017-05-14 22:48:34
categories: java
tags: 并发
---
看并发看到了无锁的CAS，都说无锁没有锁竞争带来的开销，也没有线程间频繁调度的开销，比基于锁的方式有更好的性能。小白看了很激动，这么好的工具赶紧用起来。于是想试着把以前的一段并发代码改一改。<!--more-->

```
public class BenchController {
    private Random random = new Random();
    private static Object[] lockObj;
    private static AtomicReference<Integer>[] locks;
    static {
        lockObj = new Object[100];
        for (int i = 0; i < lockObj.length; i++) {
            lockObj[i] = new Object();
        }

        locks = new AtomicReference[100];
        for (int i = 0; i < locks.length; i++) {
            locks[i] = new AtomicReference<Integer>(null);
        }
    }

    @RequestMapping("a/{id}")
    @ResponseBody
    public long a(@PathVariable("id") int id) throws Exception {
        long start = System.currentTimeMillis();
        int index = id % 100;
        long inner = 0;
        synchronized (lockObj[index]) {
            inner = test();
        }
        long result = System.currentTimeMillis() - start;
        System.out.println("all: " + result + " inner: " + inner);
        return result;
    }

    @RequestMapping("b/{id}")
    @ResponseBody
    public long b(@PathVariable("id") int id) throws Exception {
        long start = System.currentTimeMillis();
        id = id % 100;
        AtomicReference<Integer> lock = locks[id];
        int b = 0;
        while (!lock.compareAndSet(null, id)) {
            b = 1 + 1;
        }
        long inner = test();
        boolean flag = lock.compareAndSet(id, null);
        long result = System.currentTimeMillis() - start;
        System.out.println("all: " + result + " inner: " + inner + " flag:" + flag);
        System.out.println(b);
        return result;
    }

    public long test() throws Exception {
        long innerstart = System.currentTimeMillis();
        Thread.sleep(200);
        System.out.println(System.currentTimeMillis() - innerstart);
        return System.currentTimeMillis() - innerstart;
    }
}
```
代码的场景很简单，多个用户可能对一个id的订单做操作，而业务上只能有一个用户操作成功，当初实现的时候用的`a()`方法，基于无锁的思路我添加了b。用Jmeter每秒发20个同id的请求，结果很意外，有锁代码明显好于无锁的代码。当然这绝不是一个靠谱的性能测试办法，但这也抛给我一个问题，无锁不是号称性能很好嘛，为什么这里这么差？再往下走我发现了一个怪异的问题，在使用无锁的情况下我特地独立出来的`test()`里打印的睡眠时间和200毫秒相差巨大...这里非常的怪异，sleep有几个毫秒的误差是正常的，然而实际的误差有的实在太大。花了很长时间，问了问别人，我发现了这其中的一些问题：在我的机器上每秒20个线程请求无锁代码，我的cpu被占满了，再仔细想一想就会发现CAS操作虽然免去了线程调度的开销，然而不能忽略它不断尝试直到成功的那段死循环对CPU负载的影响，CAS下的所有线程都是活跃线程（这里可以联系一下JVM锁优化里的自旋锁，线程没拿到锁不会马上挂起，而是自旋等待一会儿，如果这时拿到锁就省去了线程调度的开销）。回到这个例子当中，200毫秒的睡眠不是一个短的时间，每秒20个请求远远超出了处理能力，有大量的线程占用着“无意义”的CPU计算资源。

后来我又看到了这篇讨论AtomicInteger的[文章](http://ifeve.com/enhanced-cas-in-jdk8/),文中显式的调用`compareAndSet()`方法，结果是不如`synchronized`来的好。当然我已经不止一次被告知在测试细微性能差别的时候Main方法往往是不靠谱的,那应该怎么测试呢？jmh。
## jmh
编写测试代码，使用注解，然后jmh完成剩下一切。
Mode:

* Mode.Throughput 指定多少次迭代，计算总时间 op/s
* Mode.AverageTime 方法执行的平均时间 op/s
* Mode.SampleTime 不测量总时间，测量某一部分调用的时间，所以结果中你能看到 p0.5，p0.9
* Mode.SingleShotTime 每次迭代只执行一次

state:
 
* Scope.Thread:线程私有
* Scope.Benchmark：线程共享

Dead-Code Elimination (DCE)

    @Benchmark
    public double measureWrong() {
        Math.log(x1);//注意编译器优化，这一行无意义，会被优化掉，使用后面两种方式
        return Math.log(x2);
    }
    @Benchmark
    public double measureRight_1() {
        return Math.log(x1) + Math.log(x2);
    }
    @Benchmark
    public void measureRight_2(Blackhole bh) {
        bh.consume(Math.log(x1));
        bh.consume(Math.log(x2));
    }
    --------------------------------------------
    private double x = Math.PI;
    private final double wrongX = Math.PI;
    @Benchmark
    public double measureWrong_1() {
        // 值被预测，不执行计算
        return Math.log(Math.PI);
    }
    @Benchmark
    public double measureWrong_2() {
        // final类型依旧被优化
        return Math.log(wrongX);
    }

    @Benchmark
    public double measureRight() {
        return Math.log(x);
    }
## 测试
使用微基准测试框架JMH再测试一遍：

```
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5)
@Measurement(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
@BenchmarkMode(Mode.AverageTime)
@State(Scope.Benchmark)
public class Test {
    private AtomicInteger atomic;
    private Object lock = new Object();
    private int i = 0;
    @Setup
    public void setUp() {
        atomic = new AtomicInteger(0);
    }

    @Fork(1)
    @Threads(10)
    @Benchmark
    public int incrementAtomic() {
        return atomic.incrementAndGet();
    }

    @Fork(1)
    @Threads(10)
    @Benchmark
    public int cas() {
        int i;
        do {
            i = atomic.get();
        } while (!atomic.compareAndSet(i, i + 1));
        return i + 1;
    }

    @Fork(1)
    @Threads(10)
    @Benchmark
    public int incrementSync() {
        synchronized (lock) {
            ++i;
        }
        return i;
    }
}
```

```
显式使用compareAndSet性能并不好，至于为什么，文末的讨论能提供一些帮助
Benchmark             Mode  Cnt     Score    Error  Units
Test.cas              avgt   10  1071.773 ± 21.083  ns/op
Test.incrementAtomic  avgt   10   363.205 ±  1.198  ns/op
Test.incrementSync    avgt   10   748.514 ±  8.889  ns/op
```
再给出最早的那个例子

```
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5)
@Measurement(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
@BenchmarkMode(Mode.AverageTime)
@State(Scope.Benchmark)
public class CASBench {
    private int count=0;
    private Integer id=24;
    private static Object[] lockObj;
    private static AtomicReference<Integer>[] locks;

    @Setup
    public void setUp() {
       lockObj = new Object[100];
        for (int i = 0; i < lockObj.length; i++) {
            lockObj[i] = new Object();
        }

        locks = new AtomicReference[100];
        for (int i = 0; i < locks.length; i++) {
            locks[i] = new AtomicReference<Integer>(null);
        }
    }

    @Fork(1)
    @Threads(10)
    @Benchmark
    public int sync() throws Exception {
        int index = id % 100;
        synchronized (lockObj[index]) {
            count++;
        }
        return count;
    }
    @Fork(1)
    @Threads(10)
    @Benchmark
    public int cas() throws Exception {
        AtomicReference<Integer> lock = locks[id % 100];
        while (!lock.compareAndSet(null, id)) {
        }
        count++;
        lock.compareAndSet(id, null);
        return count;
    }
}
结果：
Benchmark      Mode  Cnt     Score     Error  Units
CASBench.cas   avgt   10  3441.711 ± 220.316  ns/op
CASBench.sync  avgt   10   796.758 ±  26.033  ns/op
```
从结果中可以看到，文章开头的那个改造并不是一个好的选择。还是菜鸟，关于并发，CAS还是应该去多看看juc的代码。

