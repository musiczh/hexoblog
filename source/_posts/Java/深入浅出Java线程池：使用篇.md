---
title: 	深入浅出Java线程池：理论篇 #标题
date: 2021/1/31 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - java
 - 线程池 
 - 多线程
					
categories:	#分类
 - Java

updated: 					#更新日期
author: 一只修仙的猿 #作者
toc: true	#是否显示toc
mathjax:  #数学公式

keywords:				#关键词
description:				#文章描述
top_img:					#文章顶部照片
comments: true				#是否显示评论模块
cover:						#文章缩略图
toc_number: true			#是否显示toc_number
auto_open: true				#是否自动打开toc
copyright: true					#显示文章版权模块
copyright_author: 一只修仙的猿		#文章版权作者
copyright_author_href: 			#文章版权作者链接
copyright_url:						#文章版权文章链接
copyright_info:						#文章版权声明文字

katex:
aplayer:
highlight_shrink: true       #代码框是否打开
---

## 前言

很高兴遇见你~

借助于很多强大的框架，现在我们已经很少直接去管理线程，框架的内部都会为我们自动维护一个线程池。例如我们使用最多的okHttp以及他的封装框架Retrofit，线程封装框架RxJava和kotlin协程等等。为了更好地使用这些框架，则必须了解他的实现原理，而了解他的原理，线程池是永远绕不开的话题。

线程的创建与切换的成本是比较昂贵的。JVM的线程实现使用的是轻量级进程，也就是一个线程对应一个cpu核心。因此在创建与切换线程时，则会涉及到系统调用，是开销比较大的过程。为了解决这个问题，线程池诞生了。

与很多连接池，如sql连接池、http连接池的思想类似，线程池的出现是为了复用线程，减少创建和切换线程所带来的开销，同时可以更方便地管理线程。线程池的内部维护有一定数量的线程，这些线程就像一个个的“工人”，我们只需要向线程池提交任务，那么这些任务就会被自动分配到这些“工人”，也就是线程去执行。

> 线程池的好处：
>
> 1. 减少资源损耗。重用线程、控制线程数量，减少线程创建和切换所带来的开销。
> 2. 提高响应速度。可直接使用线程池中空闲的线程而不必等待线程的创建。
> 3. 方便管理线程。线程池可以对其中的线程进行简单的管理，如实时监控数据调整参数、设置定时任务等。

这个系列文章主要分两篇：理论篇与原理篇。作为理论 篇，顾名思义，主要是了解什么是线程池以及如何正确去使用它。

线程池的主要实现类有两个：ThreadPoolExecutor和ScheduledThreadPoolExecutor，后者继承了前者。线程池的配置参数比较多，系统也预设了一些常用的线程池，放在Executors中，开发者只需要配置简单的参数就可以。线程池的可执行任务有两种对象：Runnable和Callable，这两种对象可以直接被线程池执行，以及他们的异步返回对象Future。

那么以上讲到的，就是本文要讨论的内容。首先，从线程池的可执行任务开始。



## 任务类型

#### Runnable

```java
public interface Runnable {
    public abstract void run();
}
```

Runnable我们是比较熟悉的了。他是一个接口，内部只有一个方法 `run` ，没有参数，没有返回值。当我们提交一个Runnable对象给线程池执行时，他的 `run` 方法会被执行。

#### Callable

```java
public interface Callable<V> {
    V call() throws Exception;
}
```

与Runnable很类似，最大的不同就是他拥有返回值，而且会抛出异常。任务对象适合用于需要等待一个返回值的后台计算任务。

#### Future

```java
public interface Future<V> {
    // 取消任务
    boolean cancel(boolean mayInterruptIfRunning);
    // 任务是否被取消
    boolean isCancelled();
    // 任务是否已经完成
    boolean isDone();
    // 获取返回值
    V get() throws InterruptedException, ExecutionException;
    // 在一定的时间内等待返回值
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

Futrue他并不是一个任务，而是一个计算结果异步返回的管理对象。前面的Callable任务提交给线程池之后，他需要一定的计算时间才能将结果返回，所以线程池会返回一个Future对象。我们可以调用他的 `isDone` 方法来检测是否完成，如果完成可以调用 `get` 方法来获取结果。同时也可以通过 `cancel` 和 `isCancel` 来取消或者判断任务是否被取消。如下图：

![](https://i.loli.net/2021/01/30/DU2alIF7ihmgMZB.png)

1. 当Future未被执行或正在执行中时，get方法会阻塞直到执行完成。如果不希望等待时间过长，可以调用它另外一个带有时间参数的方法，该方法等待指定时间之后，如果任务尚未完成则会抛出TimeoutException异常。
2. 当Future执行完成之后，get方法会返回结果；而此时如果任务是被取消，那么会抛出异常。
3. 当一个任务尚未被执行时，cancel方法会让该任务不会被执行，直接结束。
4. 当任务正在执行，cancel方法的参数如果为false，那么不会中断任务；而如果参数是true则会尝试中断任务。
5. 当任务已完成，取消方法返回false，取消失败。

Futrue接口的具体实现类是FutureTask，该类同时实现了Runnable接口，可以被线程池直接执行。我们可以通过传入一个Callable对象来创建一个FutureTask。如果是Runnable对象，则可以通过 `Executors.callable()` 来构造一个Callable对象，只不过这个Callable对象返回null，所构造出来的Future对象get方法在成功时也会返回null。当FutureTask的 `run` 方法被执行后，其所包含的任务开始执行。

当然，我们也可以单独使用FutureTask，如下：

```java
// 创建一个FutureTask
FutureTask<String> future = new FutureTask<>(() -> {
    Thread.sleep(1000);
    return "一只修仙的猿";
});
// 使用一个后台线程来执行他的run方法
new Thread(future).start();
// 执行结束之后可以通过他的get方法得到返回值
System.out.println(future.get());
```

当然，如果我们要这样使用的话，那为什么不用线程池呢？（手动狗头）

接下来介绍线程池的两个主要类：ThreadPoolExecutor和ScheduledThreadPoolExecutor。



## ThreadPoolExecutor

#### 概述

我们常说的线程池，很大程度上指的就是ThreadPoolExecutor，他也是我们使用最为频繁的线程池。ThreadPoolExecutor的内部核心角色有三个：等待队列、线程和拒绝策略者：

![](https://s3.ax1x.com/2021/01/30/yAALGj.png)

线程池的内部很像一个工厂，等待队列就如同一个流水线，我们的任务就放在里面。 线程分为核心线程和非核心线程，他们就如同一个个的 “工人” ，从等待队列中获取任务进行执行。而如果这个“工厂”已经无法接受更多的任务，那么这个任务就会交给拒绝策略者去处理。

> 这里要特别注意的是，图中我分为两种线程是为了方便理解，而在实际中, **线程本身并没有核心与非核心之分 ，只有线程数大于核心线程数与小于核心线程数之分** 。
>
> 例如核心线程数限制为3，但现在有4个线程正在工作：甲乙丙丁。当甲先执行完成时进入空闲状态时，因为总数超出设定的3，那么甲会被认为非核心线程被关闭。同理，如果丁先执行完成任务，则是丁被认为非核心线程被关闭。

---

ThreadPoolExecutor并不是把每个新任务都直接放到等待队列中，而是有一套既定的规则来执行每个新任务：

![](https://s3.ax1x.com/2021/01/30/yAVggA.png)

- 情况1：在线程数没有达到核心线程数时，每个新任务都会创建一个新的线程来执行任务。
- 情况2：当线程数达到核心线程数时，每个新任务会被放入到等待队列中等待被执行。
- 情况3：当等待队列已经满了之后，如果线程数没有到达总的线程数上限，那么会创建一个非核心线程来执行任务。
- 情况4：当线程数已经到达总的线程数限制时，新的任务会被拒绝策略者处理，线程池无法执行该任务。

---

这里我们可以发现对于线程池中的角色，有各种各样的数量上限，具体的有以下参数：

- 核心线程数corePoolSize：指定核心线程的数量上限；当线程数小于核心线程数，那么线程是不会被销毁的，除非通设置线程池的 `allowCoreThreadTimeOut` 参数为true，那么核心线程在等待 `keepAliveTime` 时间之后，就会被销毁。
- 允许核心线程被回收allowCoreThreadTimeOut ：是否允许核心线程被回收。
- 最大线程数maximumPoolSize：指定线程总数的上限。线程总数=核心线层数+非核心线程数。当等待队列满了之后，新的任务会创建新的非核心线程来执行，直到线程数到达总数的上限。
- 线程闲置时间keepAliveTime：线程空闲等待该时间后会被销毁。非核心线程默认会被销毁，而核心线程则需要开发者自己设定是否允许被销毁。
- 等待队列BlockingQueue：存放任务的队列。我们可以在创建队列的时候设置队列的长度。一般使用的队列有LinkedBlockingQueue、ArrayBlockingQueue、SychronizeQueue、PriorityBlockingQueue等，前两者是普通的阻塞队列，最后一个是优先阻塞队列，剩下的一个是没有容量的队列。
- 拒绝策略者RejectExecutionHandler ：线程池无法处理的任务就会交给他处理。

通过设置线程池的这些参数可以让线程池拥有完全不同的特性，来适应不同的情景。通常我们在ThreadPoolExecutor的构造方法中，会传入这些参数：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    ...
}
```

基本都是我们上面介绍过的参数，有两个参数我们还没见过：

- 时间单位unit：给前面的keepAliveTime指定时间单位。
- 线程工厂threadFactory： 线程池会通过线程工厂来创建新的线程，主要是给线程指定一个有意义的名字。

当然，我们也可以不用全部指定上面的参数，ThreadFactory和RejectedExecutionHandler不指定可以使用默认的参数。



#### 配置ThreadPoolExecutor

经过概述，我们对ThreadPoolExecutor的参数以及内部结构已经非常清楚了，接下来探讨一下如何合理地进行配置。

首先是核心线程数。**核心线程太少，线程之间会互相争夺cpu资源，多个线程之间进行时间片轮换，大量的线程切换工作会增大系统的开销；核心线程太少，会导致一些cpu核心在挂起等待阻塞操作，没法充分利用cpu资源；核心线程数的配置就是要均衡这两个特点。**

关于核心线程数的配置，有一个广为人知的默认规则：

> cpu密集型任务，设置为CPU核心数+1；
> IO密集型任务，设置为CPU核心数*2；

CPU密集型任务指的是需要cpu进行大量计算的任务，这个时候我们需要尽可能地压榨CPU的利用率。此时核心数不宜设置过大，太多的线程会互相抢占cpu资源导致不断切换线程，反而浪费了cpu。最理想的情况是每个CPU都在进行计算，没有浪费。但很有可能其中的一个线程会突然挂起等待IO，此时额外的一个等待线程就可以马上进行工作，而不必等待挂起结束。

IO密集型任务指的是任务需要频繁进行IO操作，这些操作会导致线程长时间处于挂起状态，那么需要更多的线程来进行工作，不会让cpu都处于挂起状态，浪费资源。一般设置为cpu核心数的两倍即可。

当然实际情况还需要根据任务具体特征来配置，例如系统资源有限，有一些线程被挂在后台持续工作，那么这个时候就必须适当减少线程数，从而减少线层切换的次数。

---

第二是阻塞队列的配置。

阻塞队列有4个内置的选择：LinkedBlockingQueue、ArrayBlockingQueue、SychronizeQueue和PriorityBlockingQueue，其次还有比较少用的DelayedWorkQueue。

如果学习过Android的Handler机制的话，会发现他们跟Handler内部的MessageQueue是非常像的。ThreadPoolExecutor中的线程，相当于Handler的Looper；阻塞队列，相当于Handler的MessageQueue。他们都会去队列中获取新的任务，当队列为空时，`queue.poll()` 方法会进行阻塞，直到下一个新的任务来临。

LinkedBlockingQueue、ArrayBlockingQueue都是普通的阻塞队列，尾插头出；区别是后者在创建的时候必须指定长度。
SychronizeQueue是一个没有容量的队列，每一个插入到这个队列的任务必须马上找到可以执行的线程，如果没有则拒绝执行。
PriorityBlockingQueue是具有优先级的阻塞队列，里面的任务会根据设置的优先级进行排序。所以优先级低的任务可能会一直被优先级高的任务顶下去而得不到执行。
DelayedWorkQueue则非常像Handler中的MessageQueue了，可以给任务设置延时。

阻塞队列的选择也是非常重要，一般来说必须要指定阻塞队列的长度。如果使用无限长的队列，那么有可能在大量的任务到来时直接OOM，程序崩溃。

---

第三是线程总数。

在阻塞队列满了之后，如果线程总数还没到达上限，会创建额外的线程来执行。这对应付突然的大量但轻量的任务的时候有奇效。通过创建更多的线程来提高并发效率。

但是同时也要注意，如果线程数量太多，会造成频繁进行线程切换导致高昂的系统开销。

----

第四是线程存活时间。

这里一般指非核心线程的存活时间。这个参数的意义在于，当线程都进入空闲的时候，可以回收部分线程来减少系统资源占用。为了提高线程池的响应速度，一般比较少去回收核心线程。

----

第五是线程工厂。

这个参数比较简单，继承接口然后重写 `newThread` 方法即可。主要的用途在于给创建的线程设置一个有意义的名字。

----

最后一个是拒绝策略者。

ThreadPoolExecutor内置有4种策略：AbortPolicy、CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy。默认是使用AbortPolicy，我们可以在构造方法中传入这些类的实例，在任务被拒绝的时候，会回调他们的 `rejectedExecution` 方法。

- AbortPolicy：丢弃任务并抛出RejectedExecutionException异常。
- DiscardPolicy：也是丢弃任务，但是不抛出异常。
- DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
- CallerRunsPolicy：由调用线程直接执行该任务

当然我们也可以自己实现RejectedExecutionHandler接口来自定义自己的拒绝策略。

#### 线程池的使用

- execute(Runnable command) ：执行一个任务
- Future submit(Runnable/Callable)：提交一个任务，这个任务拥有返回值
- shutDown/shutDownNow：关闭线程池。后者还有去尝试中断正在执行的任务，也就是调用正在执行的线程的interrupt方法
- preStartAllCoreThread：提前启动所有的线程
- isShutDown：线程池是否被关闭
- isTerminad：线程池是否被终止。区别于shutDown，这个状态下的线程池没有正在执行的任务

这些都是比较常用的api，更多的api读者可以去阅读api文档。

#### 监控线程池

ThreadPoolExecutor中有很多参数可以提供我们参考线程池的运行情况：

- largestPoolSize：线程池中曾经到达的最大线程数量
- completedTaskCount：完成的任务个数
- getAliveCount：获取当前正在执行任务的线程数
- getTaskCount：获取当前任务个数
- getPoolSize：获取当前线程数

此外还有很多的get方法来获取相关参数，提供动态的线程池运行情况监控，感兴趣的读者可以去阅读相关的api。

ThreadPoolExecutor中还有提供了一些的回调方法：

- beforeExecute：在任务执行前被调用
- afterExecute：在任务执行之后被调用
- terminated：在线程池终止之后被调用

我们可以通过继承ThreadPoolExecutor来重写这些方法。

## ScheduledThreadPoolExecutor

#### 概述

ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，整体的内部结构和ThreadPoolExecutor是一样的。有一个重点的不同就是阻塞队列使用的是DelayedWorkQueue。他可以根据我们设置的时间延迟来对任务进行排序，让任务按照时间顺序进行执行，和MessageQueue非常像。

ScheduledThreadPoolExecutor实现了ScheduledExecutorService接口，有一系列可以设置延时任务、周期任务的api。

> 定时周期任务我们会想到Timer这个框架。这里简单做个对比：
>
> - Timer是系统时间敏感的，他会根据系统时间来触发任务
> - Timer内部只有一个线程，可能会造成执行任务阻塞无法按时执行；而ScheduledThreadPoolExecutor可以创建多个线程来执行任务
> - Timer遇到异常时会直接抛出，线层终止；而ScheduledThreadPoolExecutor有一个更加完善的异常处理机制。

#### 参数配置

先看到ShcduledhreadPoolExecutor的构造器：

```java
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue(), threadFactory, handler);
}
```

ScheduledThreadPoolExecutor有几个构造器，参数数量最多的是上面这个。父类是ThreadPoolExecutor，所以这里是直接调用到ThreadPoolExecutor的构造器：

- 核心线程数：这个由我们自己传入参数设定
- 线程数上限：这个设置Integer.MAX_VALUE，可以认为是没有上限
- keepAliveTime：10毫秒，基本上只要线程进入空闲状态马上就会被回收
- 阻塞队列：DelayedWorkQueue在上面介绍过了，可以设置延迟时间的阻塞队列
- 线程工厂和拒绝策略由开发者自定义

可以看到ShceduledThreadPoolExecutor的配置还是比较简单的，重点在于他的延时阻塞队列可以设置延时任务；keepAliveTime时间的设置让空闲的线程马上就会被回收。

#### 线程池的使用

ScheduledThreadPoolExecutor有ThreadPoolExecutor的接口，同时还包含了一些定时任务的接口：

- schedule：传入一个任务和延迟的时间，在延迟时间之后开始执行任务
- scheduleAtFixedRate：设置固定时间点指定任务。传入两个时间，第一次执行在延迟初始化的时间后；之后每隔指定时间执行一次。
- scheduledWithFixedDelay：在初始延迟之后，每次执行之后延迟指定时间再次执行。

重点就是上面的三个方法的使用，其他的和ThreadPoolExecutor类似。



## Executors

线程池配置的参数很多，框架内置了一些已经配置好参数的线程池，如果懒得去配置参数，可直接使用内置的线程池，可以使用Executors.newXXX方法来构建。

ThreadPoolExecutor有三个配置好参数的线程池：FixedTthreadPool、CacheThreadPool、SingleThreadExecutor。ScheduledThreadPoolExecutor有两个配置好参数的线程池：ScheduledThreadPoolExecutor和SingleThreadScheduledExecutor。

我们看一下这些线程池：

#### FixedThreadPool

```java
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

这类线程池只有核心线程，数量固定且不会被回收，等待队列的长度没有上限。

这个线程池的特点就是线程数量固定，适用于负载较重的服务器，可以通过这种线程池来限制线程数量；线程不会被回收，也有比较高的响应速度。

但是等待队列没有上限，在任务过多时有可能发生OOM。

#### CacheThreadPool

```java
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```

这类线程池没有核心线程，而且线程数量上限为Integer.MAX_VALUE，可以认为没有上限。等待队列没有长度，每一个任务到来都会分配一个线程来执行。

这类线程池的特点就是每个任务都会被马上执行，在任务数量过大时可能会创建大量的线程导致系统OOM。但是在一些任务数量多但执行时间短的情景下比较适用。

#### SingleThreadExecutor

```java
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

只有一个线程的线程池，等待队列没有上限，每个任务都会按照顺序被执行。适用于对任务执行顺序有严格要求的场景。



#### ScheduledThreadPoolExecutor

```java
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```

和ScheduledThreadPoolExecutor原生默认的构造器差别不大。



#### SingleThreadScheduledPoolExecutor

```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
    return new DelegatedScheduledExecutorService
        (new ScheduledThreadPoolExecutor(1, threadFactory));
}
```

控制线程只有一个。每个任务会按照顺序先后被执行。



#### 小结

可以看到，Executors只是按照一些比较大的情景方向，对线程池的参数进行简单的配置。那可能会问：那我直接使用线程池的构造器自己设置不就完事了？

确实，阿里巴巴开发者手册也是这样建议的。尽量使用原生的构造器来创建线程池对象，这样我们可以根据实际的情况配置出更加规范的线程池。Executors中的线程池在一些极端的情况下都可能会发生OOM，那么我们自己配置线程池时就要尽量避免这个问题。



## 最后

关于线程池的使用总体上就介绍到这里，线程池有非常多的优点，希望下次需要创建线程的时候，不会只记得 `new Thread` 。

下一篇将深入线程池的内部实现原理，如果了解过Android的Handler机制会发现两者的设计几乎一模一样，也是非常有趣的。

希望文章对你有帮助。

## 参考文献

- 《Java并发编程的艺术》：并发编程必读，作者对一些原理讲的很透彻
- 《Java核心技术卷》：这系列的书主要是讲解框架的使用，不会深入原理，适合入门
- [javaGuide](https://snailclimb.gitee.io/javaguide/#/./docs/java/multi-thread/java线程池学习总结?id=_61-%e7%ae%80%e4%bb%8b)：javaGuide，对java知识总结得很不错的一个博客
- [Java并发编程：线程池的使用](https://www.cnblogs.com/dolphin0520/p/3932921.html)：博客园上一位很优秀的博主，文章写得通俗易懂且不失深度

> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

