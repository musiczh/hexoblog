---
title: 深入浅出Java线程池：源码篇	#标题
date: 2021/2/5 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - java
 - 线程池
 - 并发
					
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

在上一篇文章[深入浅出Java线程池：理论篇](https://juejin.cn/post/6923856791117758472)中，已经介绍了什么是线程池以及基本的使用。（本来写作的思路是使用篇，但经网友建议后，感觉改为理论篇会更加合适）。本文则深入线程池的源码，主要是介绍ThreadPoolExecutor内部的源码是如何实现的，对ThreadPoolExecutor有一个更加清晰的认识。

ThreadPoolExecutor的源码相对而言比较好理解，没有特别难以读懂的地方。相信没有阅读源码习惯的读者，跟着本文，也可以很轻松地读懂ThreadPoolExecutor的核心源码逻辑。

本文源码jdk版本为8，该类版本为jdk1.5，也就是在1.5之后，ThreadPoolExecutor的源码没有做修改。

## 线程池家族

Java中的线程池继承结构如下图：(类图中只写了部分方法且省略参数)

![](https://s3.ax1x.com/2021/02/05/yGLPMQ.png)

- 顶层接口Executor表示一个执行器，他只有一个接口：`execute()` ，表示可以执行任务
- ExecutorService在Executor的基础上拓展了更多的执行方法，如`submit()` `shutdown()` 等等,表示一个任务执行服务。
- AbstarctExecutorService是一个抽象类，他实现了ExecutorService的部分核心方法，如submit等
- ThreadPoolExecutor是最核心的类，也就是线程池，他继承了抽象类AbstarctExecutorService
- 此外还有ScheduledExecutorService接口，他表示一个可以按照指定时间或周期执行的执行器服务，内部定义了如`schedule()` 等方法来执行任务
- ScheduledThreadPoolExecutor实现了ScheduledExecutorService接口，同时继承于ThreadPoolExecutor，内部的线程池相关逻辑使用自ThreadPoolExecutor，在此基础上拓展了延迟、周期执行等功能特性

ScheduledThreadPoolExecutor相对来说用的是比较少。延时任务在我们Android中有更加熟悉的方案：Handler；而周期任务则用的非常少。现在android的后台限制非常严格，基本上一退出应用，应用进程很容易被系统干掉。当然ScheduledThreadPoolExecutor也不是完全没有用处，例如桌面小部件需要设置定时刷新，那么他就可以派上用场了。

因此，我们本文的源码，主要针对ThreadPoolExecutor。在阅读源码之前，我们先来看一下ThreadPoolExecutor内部的结构以及关键角色。

## 内部结构

阅读源码前，我们先把ThreadPoolExecutor整个源码结构讲解一下，形成一个整体概念，再阅读源码就不会迷失在源码中了。先来看一下ThreadPoolExecutor的内部结构：

![yGOUkq.md.png](https://s3.ax1x.com/2021/02/05/yGOUkq.md.png)

- ThreadPoolExecutor内部有三个关键的角色：阻塞队列、线程、以及RejectExecutionHandler（这里写个中文名纯粹因为不知道怎么翻译这个名字），他们的作用在理论篇有详细介绍，这里不再赘述。
- 在ThreadPoolExecutor中，一个线程对应一个worker对象，工人，非常形象。每个worker内部有一个独立的线程，他会不断去阻塞队列获取任务来执行，也就是调用阻塞队列的 `poll` 或者 `take` 方法，他们区别后面会讲。如果队列没有任务了，那么就会阻塞在这里。
- workQueue，就是阻塞队列，当核心线程已满之后，任务就会被放置在这里等待被工人worker领取执行
- RejectExecutionHandler本身是一个接口，ThreadPoolExecutor内部有这样的一个接口对象，当任务无法被执行会调用这个对象的方法。ThreadPoolExecutor提供了该接口的4种实现方案，我们可以直接拿来用，或者自己继承接口，实现自定义逻辑。在构造线程池的时候可以传入RejectExecutionHandler对象。
- 整个ThreadPoolExecutor中最核心的方法就是execute，他会根据具体的情况来选择不同的执行方案或者拒绝执行。

这样，我们就清楚ThreadPoolExecutor的内部结构了，然后，我们开始 Read the fucking code 吧。

## 源码分析

#### 内部关键属性

ThreadPoolExecutor内部有很多的变量，他们包含的信息非常重要，先来了解一下。

ThreadPoolExecutor的状态和线程数整合在同一个int变量中，类似于view测量中MeasureSpec。他的高三位表示线程池的状态，低29位表示线程池中线程的数量,如下：

```java
// AtomicInteger对象可以利用CAS实现线程安全的修改，其中包含了线程池状态和线程数量信息
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// COUNT_BITS=29,（对于int长度为32来说）表示线程数量的字节位数
private static final int COUNT_BITS = Integer.SIZE - 3;
// 状态掩码，高三位是1，低29位全是0，可以通过 ctl&COUNT_MASK 运算来获取线程池状态
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;
```

线程池的状态一共有5个：

- 运行running：线程池创建之后即是运行状态
- 关闭shutdown：调用shutdown方法之后线程池处于shutdown状态，该状态会停止接收任何任务，阻塞队列中的任务执行完成之后会自动终止线程池
- 停止stop：调用shutdownNow方法之后线程池处于stop状态。和shutdown的区别是这个状态下的线程池不会去执行队列中剩下的任务
- 整理tidying：在线程池stop之后，进入tidying状态，然后执行 `terminated()` 方法，再进入terminated状态
- 终止terminated：线程池中没有任何线程在执行任务，线程池完全终止。

在源码中这几个状态分别对应：

```java
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

上面的位操作不够直观，转化后如下：

```java
private static final int RUNNING    = 111 00000 00000000 00000000 00000000;
private static final int SHUTDOWN   = 000 00000 00000000 00000000 00000000; 
private static final int STOP       = 001 00000 00000000 00000000 00000000;
private static final int TIDYING    = 010 00000 00000000 00000000 00000000;
private static final int TERMINATED = 011 00000 00000000 00000000 00000000;
```

可以看到除了running是负数，其他的状态都是正数，且状态越靠后，数值越大。因此我们可以通过判断 `ctl&COUNT_MASK > SHUTDOWN` 来判断状态是否处于 stop、tidying、terminated之一。后续源码中会有很多的这样的判断，举其中的一个方法：

```java
// 这里来判断线程池的状态
if(runStateAtLeast(ctl,SHUTDOWN)) {
    ...
}
// 这里执行逻辑，直接判断两个数的大小
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

ps：这里怎么没有使用掩码COUNT_MASK ？因为状态是处于高位，低位的数值不影响高位的大小判断。当然如果要判断相等，就还是需要使用掩码COUNT_MASK的。
```

接下来是ThreadPoolExecutor内部的三个关键角色对象：

```java
// 阻塞队列
private final BlockingQueue<Runnable> workQueue;
// 存储worker的hashSet,worker被创建之后会被存储到这里
private final HashSet<Worker> workers = new HashSet<>();
// RejectedExecutionHandler默认的实现是AbortPolicy
private volatile RejectedExecutionHandler handler;
private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();

```

内部使用的锁对象：

```java
// 这里是两个锁。ThreadPoolExecutor内部并没有使用Synchronize关键字来保持同步
// 而是使用Lock；和Synchronize的区别就是他是应用层的锁，而synchronize是jvm层的锁
private final ReentrantLock mainLock = new ReentrantLock();
private final Condition termination = mainLock.newCondition();
```

最后是内部一些参数的配置，前面都介绍过，把源码贴出来再回顾一下：

```java
// 线程池历史达到的最大线程数
private int largestPoolSize;
// 线程池完成的任务数。
// 该数并不是实时更新的，在获取线程池完成的任务数时，需要去统计每个worker完成的任务并累加起来
// 当一个worker被销毁之后，他的任务数就会被累加到这个数据中
private long completedTaskCount;
// 线程工厂，用于创建线程
private volatile ThreadFactory threadFactory;
// 空闲线程存储的时间
private volatile long keepAliveTime;
// 是否允许核心线程被回收
private volatile boolean allowCoreThreadTimeOut;
// 核心线程数限额
private volatile int corePoolSize;
// 线程总数限额
private volatile int maximumPoolSize;
```

不是吧sir？源码还没看到魂呢，整出来这么无聊的变量？
咳咳，别急嘛，源码解析马上来。这些变量会贯穿整个源码过程始终，先对他们有个印象，后续阅读源码就会轻松畅通很多。

#### 关键方法：execute()

这个方法的主要任务就是根据线程池的当前状态，选择任务的执行策略。该方法的核心逻辑思路是：

1. 在线程数没有达到核心线程数时，会创建一个核心线程来执行任务

   ```java
   public void execute(Runnable command) {
       // 不能传入空任务
       if (command == null)
           throw new NullPointerException();
   
       // 获取ctl变量，就是上面我们讲的将状态和线程数合在一起的一个变量
       int c = ctl.get();
       // 判断核心线程数是否超过限额，否则创建一个核心线程来执行任务
       if (workerCountOf(c) < corePoolSize) {
           // addWorker方法是创建一个worker，也就是创建一个线程，参数true表示这是一个核心线程
           // 如果添加成功则直接返回
           // 否则意味着中间有其他的worker被添加了，导致超出核心线程数;或者线程池被关闭了等其他情况
           // 需要进入下一步继续判断
           if (addWorker(command, true))
               return;
           c = ctl.get();
       }
       ...
   }
   ```

2. 当线程数达到核心线程数时，新任务会被放入到等待队列中等待被执行

3. 当等待队列已经满了之后，如果线程数没有到达总的线程数上限，那么会创建一个非核心线程来执行任务

4. 当线程数已经到达总的线程数限制时，新的任务会被拒绝策略者处理，线程池无法执行该任务。

   ```java
   public void execute(Runnable command) {
       ...
       // 如果线程池还在运行,则尝试添加任务到队列中
       if (isRunning(c) && workQueue.offer(command)) {
           int recheck = ctl.get();
           // 再次检查如果线程池被关闭了，那么把任务移出队列
           // 如果移除成功则拒绝本次任务
           // 这里主要是判断在插入队列的过程中，线程池有没有被关闭了
           if (! isRunning(recheck) && remove(command))
               reject(command);
           // 否则再次检查线程数是否为0，如果是，则创建一个没有任务的非主线程worker
           // 这里对应核心线程为0的情况，指定任务为null，worker会去队列拿任务来执行
           // 这里表示线程池至少有一个线程来执行队列中的任务
           else if (workerCountOf(recheck) == 0)
               addWorker(null, false);
       }
       // 如果上面添加到队列中失败，则尝试创建一个非核心线程来执行任务
       // 如果创建失败，则拒绝任务
       else if (!addWorker(command, false))
           reject(command);
   }
   ```

源码中还设计到两个关键方法：addWorker创建一个新的worker，也就是创建一个线程；reject拒绝一个任务。后者比较简单我们先看一下。

#### 拒绝任务：reject()

```java
// 拒绝任务，调用rejectedExecutionHandler来处理
final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```

默认的实现类有4个，我们依次来看一下：

-  AbortPolicy是默认实现，会抛出一个RejectedExecutionException异常：

  ```java
  public static class AbortPolicy implements RejectedExecutionHandler {
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          throw new RejectedExecutionException("Task " + r.toString() +
                                               " rejected from " +
                                               e.toString());
      }
  }
  ```

- DiscardPolicy最简单，就是：什么都不做，直接抛弃任务。（这是非常渣男不负责任的行为，咱们不能学他，所以也不要用它 [此处狗头] ）

  ```java
  public static class DiscardPolicy implements RejectedExecutionHandler {
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      }
  }
  ```

-  DiscardOldestPolicy会删除队列头的一个任务，然后再次执行自己（挤掉原位，自己上位，绿茶行为？）

  ```java
  public static class DiscardOldestPolicy implements RejectedExecutionHandler {
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          if (!e.isShutdown()) {
              e.getQueue().poll();
              e.execute(r);
          }
      }
  }
  ```

-  CallerRunsPolicy最猛，他干脆在自己的线程执行run方法，不依靠线程池了，自己动手丰衣足食。

	 ```java
  public static class CallerRunsPolicy implements RejectedExecutionHandler {
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          if (!e.isShutdown()) {
              r.run();
          }
      }
  }
  ```

上面4个ThreadPoolExecutor已经帮我们实现了，他的静态内部类，在创建ThreadPoolExecutor的时候我们可以直接拿来用。也可以自己继承接口实现自己的逻辑。具体选择哪个需要根据实际的业务需求来决定。

那么接下来看创建worker的方法。

#### 创建worker：addWorker()

方法的目的很简单：创建一个worker。前面我们讲到，worker内部创建了一个线程，每一个worker则代表了一个线程，非常类似android中的looper。looper的loop()方法会不断地去MessageQueue获取message，而Worker的run()方法会不断地去阻塞队列获取任务，这个我们后面讲。

addWorker() 方法的逻辑整体上分为两个部分：

1. 检查**线程状态**和**线程数**是否满足条件：

   ```java
   // 第一个参数是创建的线程首次要执行的任务，可以是null，则表示初始化一个线程
   // 第二参数表示是否是一个核心线程
   private boolean addWorker(Runnable firstTask, boolean core) {
       retry:
       for (int c = ctl.get();;) {
           // 还记不记得我们前面讲到线程池的状态控制？
           // runStateAtLeast(c, SHUTDOWN)表示状态至少为shutdown，后面类同
           // 如果线程池处于stop及以上，不会再创建worker
           // 如果线程池状态在shutdown时，如果队列不为空或者任务!=null，则还会创建worker
           if (runStateAtLeast(c, SHUTDOWN)
               && (runStateAtLeast(c, STOP)
                   || firstTask != null
                   || workQueue.isEmpty()))
               // 其他情况返回false，表示拒绝创建worker
               return false;
           
   		// 这里采用CAS轮询，也就是循环锁的策略来让线程总数+1
           for (;;) {
               // 检查是否超出线程数限制
               // 这里根据core参数判断是核心线程还是非核心线程
               if (workerCountOf(c)
                   >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                   return false;
               // 利用CAS让ctl变量自增，表示worker+1
               // 如果CAS失败，则表示发生了竞争，则再来一次
               if (compareAndIncrementWorkerCount(c))
                   // 成功则跳出最外层循环
                   break retry;
               // 如果这个期间ctl被改变了，则获取ctl，再尝试一次
               c = ctl.get();  
               // 如果线程池被shutdown了，那么重复最外层的循环，重新判断状态是否可以创建worker
               if (runStateAtLeast(c, SHUTDOWN))
                   // 继续最外层循环
                   continue retry;
           }
       }
       
       // 创建worker逻辑
       ...
   }
   ```

   不知道读者对于源码中的`retry: ` 有没有疑惑，毕竟平时很少用到。他的作用是标记一个循环，这样我们在内层的循环就可以跳转到任意一个外层的循环。这里的retry只是一个名字，改成 `repeat:` 甚至 `a:` 都是可以的。他的本质就是：**一个循环的标记** 。

   

2. 创建worker对象，并调用其内部线程的start()方法来启动线程：

   ```java
   private boolean addWorker(Runnable firstTask, boolean core) {
   	...
       boolean workerStarted = false;
       boolean workerAdded = false;
       Worker w = null;
       try {
           // 创建一个新的worker
           // 创建的过程中内部会创建一个线程
           w = new Worker(firstTask);
           final Thread t = w.thread;
           if (t != null) {
               // 获得全局锁并加锁
               final ReentrantLock mainLock = this.mainLock;
               mainLock.lock();
               try {
                   // 获取锁之后，需要再次检查状态
                   int c = ctl.get();
   				// 只有运行状态或者shutDown&&task==null才会被执行
                   if (isRunning(c) ||
                       (runStateLessThan(c, STOP) && firstTask == null)) {
                       // 如果这个线程不是刚创建的，则抛出异常
                       if (t.getState() != Thread.State.NEW)
                           throw new IllegalThreadStateException(); 
                       // 添加到workerSet中
                       workers.add(w);
                       workerAdded = true;
                       int s = workers.size();
                       // 跟踪线程池到达的最多线程数量
                       if (s > largestPoolSize)
                           largestPoolSize = s;
                   }
               } finally {
                   // 释放锁
                   mainLock.unlock();
               }
               // 如果添加成功，启动线程
               if (workerAdded) {
                   t.start();
                   workerStarted = true;
               }
           }
       } finally {
           // 如果线程没有启动，表示添加worker失败，可能在添加的过程中线程池被关闭了
           if (! workerStarted)
               // 把worker从workerSet中移除
               addWorkerFailed(w);
       }
       return workerStarted;
   }
   ```

经过前面两步，如果没有出现异常，则创建worker成功。最后还涉及到一个方法： `addWorkerFailed(w)` ，他的内容比较简答，顺便提一下吧：

```java
// 添加worker失败
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    // 加锁
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        // 这里会让线程总数-1
        decrementWorkerCount();
        // 尝试设置线程池的状态为terminad
        // 因为添加失败有可能是线程池在添加worker的过程中被shutdown
        // 那么这个时候如果没有任务正在执行就需要设置状态为terminad
        // 这个方法后面会详细讲
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

那么到这里，execute()方法中的一些调用方法就分析完了。阻塞队列相关的方法不属于本文的范畴，就不展开了。那么还有一个问题：worker是如何工作的呢？worker内部有一个线程，当线程启动时，初始化线程的runnable对象的run方法会被调用，那么这个runnable对象是什么？我直接来看worker。

#### 打工人：Worker

首先我们看到他的构造方法：

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

源码很简单，把传进来的任务设置给内部变量firstTask，然后把自己传给线程工厂去创建一个线程。所以线程启动时，Worker本身的run方法会被调用，那么我们看到Worker的 `run()`方法。

```java
public void run() {
    runWorker(this);
}
```

Worker是ThreadPoolExecutor的内部类，这里直接调用到了ThreadPoolExecutor的方法： `runWorker()`来开始执行。那么接下来，我们就看到这个方法。

#### 启动worker：runWorker()

这个方法是worker执行的方法，在线程被销毁前他会一直执行，类似于Handler的looper，不断去队列获取消息来执行：

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 获取worker初始化时设置的任务，可以为null。如果为null则表示仅仅创建线程
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); 
    // 这个参数的作用后面解释，需要结合其他的源码
    boolean completedAbruptly = true;
    try {
        // 如果自身的task不为null，那么会执行自身的task
        // 否则调用getTask去队列获取一个task来执行
        // 这个getTask最终会去调用队列的方法来获取任务
        // 而队列如果为空他的获取方法会进行阻塞，这里也就阻塞了,后面深入讲
        while (task != null || (task = getTask()) != null) {
            try{
            // 执行任务
            ...
            } finally {
                // 任务执行完成，把task设置为null
                task = null;
                // 任务总数+1
                w.completedTasks++;
                // 释放锁
                w.unlock();
            }
        }
    	// 这里设置为false，先记住他
        completedAbruptly = false;
    } finally {
    	// 如果worker退出，那么需要执行后续的善后工作
        processWorkerExit(w, completedAbruptly);
    }
}
```

可以看到这个方法的整体框架还是比较简单的，核心就在于 `while (task != null || (task = getTask()) != null)` 这个循环中，如果 `getTask()` 返回null，则表示线程该结束了，这和Handler机制也是一样的。

上面的源码省略了具体执行任务的逻辑，他的逻辑也是很简单：判断状态+运行任务。我们来看一下：

```java
final void runWorker(Worker w) {
    ...;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // 如果线程池已经设置为stop状态，那么保证线程是interrupted标志
            // 如果线程池没有在stop状态，那么保证线程不是interrupted标志
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 回调方法，这个方法是一个空实现
                beforeExecute(wt, task);
                try {
                    // 运行任务
                    task.run();
                    // 回调方法，也是一个空实现
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            }
            ...
        }
    	completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

在获取到一个任务后，就会去执行该任务的run方法，然后再回去继续获取新的任务。

我们会发现其中有很多的空实现方法，他是给子类去实现的，有点类似于Activity的生命周期，子类需要重写这些方法，在具体的情况做一些工作。当然，一般的使用是不需要去重写这些方法。接下来需要来看看 `getTask()` 是如何获取任务的。

#### 获取任务：getTask()

这个方法的内容可以分为两个部分：判断当前线程池的状态+阻塞地从队列中获取一个任务。

第一部分是判断当前线程池的状况，如果处于关闭状态那么直接返回null来让worker结束，否则需要判断当前线程是否超时或者超出最大限制的线程数：

```java
private Runnable getTask() {
    boolean timedOut = false; 
    // 内部使用了CAS，这里需要有一个循环来不断尝试
    for (;;) {
        int c = ctl.get();
        // 如果处于shutdown状态而且队列为空，或者处于stop状态，返回null
        // 这和前面我们讨论到不同的线程池的状态的不同行为一致
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            // 这里表示让线程总数-1，记住他，后面会继续聊到
            decrementWorkerCount();
            return null;
        }
        
        // 获取目前的线程总数
        int wc = workerCountOf(c);
        // 判断该线程在空闲情况是否可以被销毁：允许核心线程为null或者当前线程超出核心线程数
        // 可以看到这里并没有去区分具体的线程是核心还是非核心，只有线程数量处于核心范围还是非核心范围
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 超出最大线程数或者已经超时；
        // 这里可能是用户通过 setMaximumPoolSize 改动了数据才会导致这里超出最大线程数
        // 同时还必须保证当前线程数量大于1或者队列已经没有任务了
        // 这样就确保了当有任务存在时，一定至少有一个线程在执行任务
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 使用CAS尝试让当前线程总数-1，失败则从来一次上面的逻辑
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        
        // 获取任务逻辑
        ...
    }
}
```

第二部分是获取一个任务并执行。获取任务使用的是阻塞队列的方法，如果队列中没有任务，则会被阻塞：

```java
private Runnable getTask() {
    boolean timedOut = false; 
    // 内部使用了CAS，这里需要有一个循环来不断尝试
    for (;;) {
        // 判断线程池状态逻辑
        ...
        try {
            // 获取一个任务
            // poll方法等待具体时间之后如果没有获取到对象，会返回null
            // take方法会一直等到获取新对象，除非被interrupt
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            // r==null,说明超时了，重新循环
            timedOut = true;
        } catch (InterruptedException retry) {
            // 被interrupt，说明可能线程池被关闭了，重新判断情况
            timedOut = false;
        }
    }
}
```

这里需要重点关注的是阻塞队列的 `poll()` 和 `take()` 方法，他们都会去队列中获取一个任务；但是，`poll()` 方法会阻塞指定时间后返回，而 `take()` 则是无限期阻塞。这里对应的就是有存活时间的线程和不会被销毁的核心线程。

同时注意 `timedOut = true` 是在这一部分被赋值的，当赋值为true之后需要再执行一次循环，在上面的判断中就会被拦截下来并返回false，这在第一部分逻辑介绍了。而如果线程在等待的时候被 `interrupt` 了，说明线程池被关闭了，此时也会重走一次上面判断状态的逻辑。

到这里关于执行的逻辑就讲得差不多了，下面聊一聊线程池关闭以及worker结束的相关逻辑。

#### worker退出工作：processWorkerExit

前面已经介绍 `runWorker()` 了方法，这个方法的主要任务就是让worker动起来，不断去队列获取任务。而当获取任务的时候返回了null，则表示该worker可以结束了，最后会调用 `processWorkerExit()` 方法，如下：

```java
final void runWorker(Worker w) {
    ...
    try {
       ...
    } finally {
    	// 如果worker退出，那么需要执行后续的善后工作
        processWorkerExit(w, completedAbruptly);
    }
}
```

`processWorkerExit()` 会完成worker退出的善后工作。具体的内容是：

1. 把完成的任务数合并到总的任务数,移除worker，尝试设置线程池的状态为terminated：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果不是经过getTask方法返回null正常退出的，那么需要让线程总数-1
    // 这个参数前面一直让你们注意一下不知道你们还记不记得
    // 如果是在正常情况下退出，那么在getTask() 方法中就会执行decrementWorkerCount()了
    // 而如果出现一些特殊的情况突然结束了，并不是通过在getTask返回null结束
    // Abruptly就是突然的意思，那么completedAbruptly就为true,正常情况下在runWorker方法中会被设置为false
    // 那什么叫突然结束？用户的任务抛出了异常，这个时候线程就突然结束了，没有经过getTask方法
    // 这里就需要让线程总数-1
    if (completedAbruptly) 
        decrementWorkerCount();

    // 获取锁，并累加完成的任务总数，从set中移除worker
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    // 尝试设置线程池的状态为terminated
    // 这个方法前面我们addWorker失败的时候提到过，后面再展开
    tryTerminate();
	...
}
```

2. 移除worker之后，如果线程池还没有被stop，那么最后必须保证队列任务至少有一个线程在执行队列中的任务：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    ...
	int c = ctl.get();
    // stop及以上的状态不需要执行剩下的任务
    if (runStateLessThan(c, STOP)) {
        // 如果线程是突然终止的，那肯定需要重新创建一个
        // 否则进行判断是否要保留一个线程
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; 
        }
        // 如果此时线程数<=核心线程数，或者当核心线程可被销毁时，线程数==0且队列不为空
        // 那么需要创建一个线程来执行任务
        addWorker(null, false);
    }
}
```

代码虽然看起来很多，但是具体的逻辑内容还是比较简单的。前面一直提到一个方法 `tryTerminate()` 但一直没有展开解释，下面来介绍一下。

#### 尝试终止线程池：tryTerminate()

这个方法出现在任何可能让线程池进入终止状态的地方。如添加worker失败时，那么这个时候可能线程池已经处于stop状态，且已经没有任何正在执行的worker了，那么此时可以进入terminated状态；再如worker被销毁的时候，可能这是最后一个被销毁的worker，那么此时线程池需要进入terminated状态。

根据这个方法的使用情况其实就已经差不多可以推断出这个方法的内容：**判断当前线程池的状态，如果符合条件则设置线程池的状态为terminated** 。如果此时不能转换为terminated状态，则什么也不做，直接返回。

1. 首先判断当前线程池状态是否符合转化为terminated。如果处于运行状态或者tidying以上状态，则肯定不需要进行状态转换。因为running需要先进入stop状态，而tidying其实已经是准备进入terminated状态了。如果处于shutdown状态且队列不为空，那么需要执行完队列中的任务，所以也不适合状态转换：

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 如果处于运行状态或者tidying以上状态时，直接返回，不需要修改状态
        // 如果处于stop以下状态且队列不为空，那么需要等队列中的任务执行完成，直接返回
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateLessThan(c, STOP) && ! workQueue.isEmpty()))
            return;
        // 到这里说明线程池肯定处于stop状态
        // 线程的数量不等于0，尝试中断一个空闲的worker线程
        // 这里他只中断workerSet中的其中一个，当其中的一个线程停止时，会再次调用tryTerminate
        // 然后又会再去中断workerSet中的一个worker，不断循环下去直到剩下最后一个，workercount==0
        // 这就是 链式反应 。
        if (workerCountOf(c) != 0) { 
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
		
        // 设置状态为terminated逻辑
        ...
    }
}
```

2. 经过上面的判断，能到第二部分逻辑，线程池肯定是具备进入terminated状态的条件了。剩下的代码就是把线程池的状态设置为terminated：

```java
final void tryTerminate() {
    for (;;) {
        // 上一部分逻辑
        ...
        // 首先获取全局锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 尝试把线程池的状态从stop修改为tidying
            // 如果修改失败，说明状态已经被修改了，那么外层循环再跑一个
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 这个方法是一个空实现，需要子类继承重写
                    terminated();
                } finally {
                    // 最后再设置状态为terminated
                    ctl.set(ctlOf(TERMINATED, 0));
                    // 唤醒所有等待终止锁的线程
                    termination.signalAll();
                }
                return;
            }
        } finally {
            // 释放锁
            mainLock.unlock();
        }
        // CAS修改线程池的状态失败，重新进行判断
    }
}
```

当线程池被标记为terminated状态时，那么这个线程池就彻底地终止了。

好了到这里，恭喜你，关于ThreadPoolExecutor的源码解析理解得差不多了。接下来剩下几个常用的api方法：`submit()` 、 `shutdown()/shutdownNow()` 顺便看一下吧，他们的逻辑也是都非常简单。



#### 关闭线程池：shutdown/shutdownNow

关闭线程池有两个方法：

- shutdown：设置线程池的状态为shutdown，同时尝试中断所有空闲线程，但是会等待队列中的任务执行结束再终止线程池。
- shutdownNow：设置线程池的状态为stop，同时尝试中断所有空闲线程，不会等待队列中的任务完成，正在执行中的线程执行结束，线程池马上进入terminated状态。

我们各自来看一下：

```java
// 关闭后队列中的任务依旧会被执行，但是不会再添加新的任务
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 设置状态为shutdown
        advanceRunState(SHUTDOWN);
        // 尝试中断所有空闲的worker
        interruptIdleWorkers();
        // 回调方法，这个方法是个空方法，ScheduledThreadPoolExecutor中重写了该方法
        onShutdown(); 
    } finally {
        mainLock.unlock();
    }
    // 尝试设置线程池状态为terminated
    tryTerminate();
}
```

再看一下另一个方法shutdownNow：

```java
// 关闭后队列中剩余的任务不会被执行
// 会把剩下的任务返回交给开发者去处理
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 检查是否可以关闭线程
        checkShutdownAccess();
        // 设置状态为stop
        advanceRunState(STOP);
        // 尝试中断所有线程
        interruptWorkers();
        // 返回队列中剩下的任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

最后再来看一下和 `execute()`不同的提交任务方法：submit。

#### 提交任务：submit()

submit方法并不是ThreadPoolExecutor实现的，而是AbstractExecutorService，如下：

```java
// runnable没有返回值，创建FutureTask的返回参数传入null
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
// 有参数返回值的runnable
// 最终也是构造一个callable来执行，把返回值设置为result
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
// callable本身就拥有返回值
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

他们的逻辑都几乎一样：调用newTaskFor方法来构造一个Future对象并返回。我们看到newTaskFor方法：

```java
// 创建一个FutureTask来返回
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

可以看到这个方法很简单：构造一个FutureTask并返回，FutureTask也是Future接口目前唯一的实现类。

更加具体关于Future的内容就不展开了，有兴趣的读者可以去了解一下。

## 最后

好了到这里，关于ThreadPoolExecutor的源码分析内容就讲完了。最后让我们再回顾一下吧：

![yGOUkq.md.png](https://s3.ax1x.com/2021/02/05/yGOUkq.md.png)

- ThreadPoolExecutor的整个执行流程从execute方法开始，他会根据具体的情况，采用合适的执行方案
- 线程被封装在worker对象中，worker对象通过runWorker方法，会一直不断地调用getTask方法来调用队列的poll或take方法获取任务
- 当需要退出一个worker时，只要getTask方法返回null即可退出
- 当线程池关闭时，会根据不同的关闭方法，等待所有的线程执行完成，然后关闭线程池。

线程池整体的模型和handler是十分类似的：一个生产者-消费者模型。但和Handler不同的是，ThreadPoolExecutor不支持延时任务，这点在ScheduledThreadPoolExecutor得到了实现；Handler的线程安全采用synchronize关键字，而ThreadPoolExecutor采用的是Lock和一些利用CAS实现线程安全的整型变量；Handler无法拒绝任务，线程池可以；Handler抛出异常会直接程序崩溃，而线程池不会等等。

了解了线程池的内部源码，对于他更加了解后，那么可以根据具体的问题，做出更加合适的解决方案。ThreadPoolExecutor还有一些源码没有讲到，以及ScheduledThreadPoolExecutor、阻塞队列的源码，有兴趣读者可以自行去深入了解一下，拓展关于线程池的一切。

全文到此，假期肝文不容易啊，如果文章对你有帮助，求一个大拇指![yJsExU.png](https://s3.ax1x.com/2021/02/06/yJsExU.png)，赞一下再走呗。

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

