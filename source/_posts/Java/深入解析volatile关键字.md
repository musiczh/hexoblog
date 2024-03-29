---
title: 深入解析volatile关键字	#标题
date: 2020/11/3 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - java
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

很高兴遇见你~ 欢迎阅读我的文章。

volatile关键字在Java多线程编程编程中起的作用是很大的，合理使用可以减少很多的线程安全问题。但其实可以发现使用这个关键字的开发者其实很少，包括我自己。遇到同步问题，首先想到的一定是加锁，也就是synchronize关键字，暴力锁解决一切多线程疑难杂症。但，锁的代价是很高的。**线程阻塞、系统线程调度**这些问题，都会造成很严重的性能影响。如果在一些**合适的场景，使用volatile，既保证了线程安全，又极大地提高了性能**。

那为啥放着好用的volatile不用，偏偏要加锁呢？一个很重要的原因是：不了解什么是volatile。加锁简单粗暴，几乎每个开发者都会用（但不一定可以正确地用），而volatile，可能压根就不知道有这个东西（包括之前的笔者=_=）。那volatile是什么东西？他有什么作用？在什么场景下适用？他的底层原理是什么？他真的可以保证线程安全吗？这一系列问题，是面试常见相关题目，也正是这篇文章要啊解决的问题。

那么，我们开始吧。

## 认识volatile

volatile关键字的作用有两个：**变量修改对其他线程立即可见、禁止指令重排。**

第二个作用我们后面再讲，先主要讲一下第一个作用。通俗点来说，就是我在一个线程对一个变量进行了修改，那么其他线程马上就可以知道我修改了他。嗯？难道我修改了数值其他线程不知道？我们先从实例代码中来感受volatile关键字的第一个作用。

```java
private  boolean stopSignal = false;

public void fun(){
    // 创建10个线程
    for (int i=0;i<=10;i--){
        new Thread(() -> {
            while(!stopSignal){
                // 循环等待
            }
            System.out.println(Thread.currentThread().toString()+"我停下来了");
        }).start();
    }
    new Thread(() -> {
        stopSignal = true;
        System.out.println("给我停下来");
    }).start();
}
```

这个代码很简单，创建10个线程循环等待`stopSignal`，当`stopSignal`变为true之后，则跳出循环，打印日志。然后我们再开另外一个线程，把`stopSignal`改成true。如果按照正常的情况下，应该是先打印“给我停下来”，然后再打印10个“我停下来了”，最后结束进程。我们看看具体情况如何。来，运行：

<img src="https://s1.ax1x.com/2020/11/03/Bs0v90.png" width=300 border="0" />

嗯嗯？为什么只打印两个我停下来了？而且看左边的停止符号，表示这个进程还没结束。也就是说在剩下的线程中，他们拿到的`stopSignal`数据依旧是`false`，而不是最新的`true`。所以问题就是：**线程中变量的修改，对于其他线程并不是立即可见**。导致这个问题的原因我们后面讲，现在是怎么解决这个问题。加锁是个好办法，只要我们在循环判断与修改数值的时候加个锁，就可以拿到最新的数据了。但是前面讲到，锁是个重量级操作，然后再加上循环，这性能估计直接掉下水道里了。最好的解决方法就是：给变量加上volatile关键字，如下：

```java
private volatile  boolean stopSignal = false;
```

我们再运行一下：

<img src="https://s1.ax1x.com/2020/11/03/BsBCB4.png" width=300 border="0" />

诶，可以了，全部停下来了。使用了volatile关键修饰的变量，只要被修改，那么其他的线程均会立即可见。这就是volatile关键字的第一个重要作用：**变量修改对其他线程立即可见**。关于指令重排我们后面再讲。

那么为什么变量的修改是对其他线程不是立即可见呢？volatile为何能实现这个效果？那这样我们可不可以每个变量都给他加上volatile关键字修饰？要解决这些问题，我们得先从Java内存模型说起。系好安全带，我们的车准备开进底层原理了。



## Java内存模型

Java内存模型不是堆区、栈区、方法区那些，而是线程之间如何共享数据的模型，也可以说是线程对共享数据读写的规范。内存模型是理解Java并发问题的基础。限于篇幅这里不深入讲述，只简单介绍一下。不然又是万字长文了。先看个图：

<img src="https://s1.ax1x.com/2020/11/03/ByUsAI.png" width=500 border="0" />

JVM把内存总体分为两个部分：**线程私有以及线程共享，线程共享区域也称为主内存**。线程私有部分不会发生并发问题，所以主要是关注线程共享区域。这个图大致上可以这么理解：

1. 所有共享变量存储在主内存
2. 每条线程拥有自己的工作内存
3. 工作内存保留了被该线程使用的变量的主内存副本
4. 变量操作必须在工作内存进行
5. 不同线程之间无法访问对方的工作内存

简单总结一下，所有数据都要放在主内存中，线程要操作这些数据必须要先拷贝到自己的工作内存，然后只能对自己工作内存中的数据进行修改；修改完成之后，再写回主内存。线程无法访问另一个线程的数据，这也就是为什么线程私有的数据不存在并发问题。

那为什么不直接从主内存修改数据，而要先在工作内存修改后再写回主内存呢？这就涉及到了高速缓冲区的设计。简单来说，处理器的速度非常快，但是执行的过程中需要频繁在内存中读写数据，而内存访问的速度远远跟不上cpu的速度，导致降低了cpu的效率。因而设计出高速缓冲区，处理器可以直接操作高速缓冲区的数据，等到空闲时间，再把数据写回主内存，提高了性能。而JVM为了屏蔽不同的平台对于高速缓冲区的设计，就设计出了Java内存模型，来让开发者可以面向统一的内存模型进行编程。可以看到，这里的工作内存就对应硬件层面的高速缓冲区。



## 指令重排

前面我们一直没讲指令重排，是因为这是属于JVM在后端编译阶段进行的优化，而在代码中隐藏的问题很难去复现，也很难通过代码执行来看出差别。指令重排是即时编译器对于字节码编译过程中的一种优化，受到运行时环境的影响。限于能力，笔者只能通过理论分析来讲这个问题。

我们知道JVM执行字节码一般有两种方式：解释器执行和即时编译器执行。解释器这个比较容易理解，就是一行行代码解释执行，所以也不存在指令重排的问题。但是解释执行存在很大的问题：**解释代码需要耗费一定的处理时间、无法对编译结果进行优化**，所以解释执行一般在应用刚启动时或者即时编译遇到异常才使用解释执行。而**即时编译则是在运行过程中，把热点代码先编译成机器码，等到运行到该代码的时候就可以直接执行机器码，不需要进行解释，提高了性能**。而在编译过程中，我们可以对编译后的机器码进行优化，只要保证运行结果一致，我们可以按照计算机世界的特性对机器码进行优化，如方法内联、公共子表达式消除等等，指令重排就是其中一种。

计算机的思维跟我们人的思维是不一样的，我们按照面向对象的思维来编程，但计算机却必须按照“面向过程”的思维来执行。所以，**在不影响执行结果的情况下，JVM会更改代码的执行顺序**。注意，这里的不影响执行结果，是指在当前线程下。如我们可能会在一个线程中初始化一个组件，等到初始化完成则设置标志位为true，然后其他线程只需要监听该标志位即可监听初始化是否完成，如下：

```java
// 初始化操作
isFinish = true;
```

在当前线程看起来，**注意，是看起来**，会先执行初始化操作，再执行赋值操作，因为**结果是符合预期的**。但是！！！在其他线程看来，这整个执行顺序都是乱的。JVM可能先执行`isFinish`赋值操作，再执行初始化操作；而如果你在别的线程监听`isFinish`变化，就可能出现还未初始化完成`isFinish`却是true的问题。而**volatile可以禁止指令重排，保证在`isFinish`被赋值之前，所有的初始化动作都已经完成**。



## volatile实现原理

为什么volatile可以实现这么神奇的功能？

在写入volatile变量和写入普通变量，两者的不同是**前者在写入之后会增加一条指令，把缓存写入到主内存中** 。这个操作会导致其他线程的备份失效，因此其他线程在读取该变量时，需要到主内存去重新获取新的值。

指令重排，并不是乱排，他需要保证在单线程情况下执行结果是不会改变的。volatile关键字增加的写入指令，让其他的操作必须在此之前完成，才能保证缓存是代码执行后的结果；因此也就可以保证，在赋值之前的代码操作，全都完成。



## volatile的具体定义

上面讲了两个看起来跟我们的主角volatile关系不大的知识点，但其实是非常重要的知识点。

首先，通过Java内存模型的理解，现在知道为什么会出现线程对变量的修改其他线程未立即可知的原因了吧？**线程修改变量之后，可能并不会立即写回主内存，而其他线程，在主内存数据更新后，也并不会立即去主内存获取最新的数据。**这也是问题所在。

被volatile关键字修饰的变量规定：**每次使用数据都必须去主内存中获取；每次修改完数据都必须马上同步到主内存**。这样就实现了每个线程都可以立即收到该变量的修改信息。不会出现读取脏数据旧数据的情况。

第二个作用是禁止指令重排。这里是使用到了JVM的一个规定：**同步内存操作前的所有操作必须已经完成**。而我们知道每次给volatile赋值的时候，他会同步到主内存中。所以，在同步之前，保证所有操作都必须完成了。所以当其他线程监测到变量的变化时，赋值前的操作就肯定都已经完成了。

既然volatile关键字这么好，那可不可以每个地方都使用好了？当然不行！在前面讲Java内存模型的时候有提到，为了提高cpu效率，才分出了高速缓冲区。如果一个变量并不需要保证线程安全，那么频繁地写入和读取内存，是很大的性能消耗。因而，只有必须使用volatile的地方，才使用他。



## volatile修饰的变量一定是线程安全吗

首先明确一下，怎么样才算是线程安全？

1. 一个操作要符合原子性，要么不执行，要么一次性执行完成，在执行过程中不会受其他操作的影响。
2. 对于变量的修改相对于其他线程必须立即可见。
3. 代码的执行在其他线程看来要满足有序性。

从上面分析我们讲了：**代码的有序性、修改的可见性**，但是，缺少了原子性。在JVM中，在主内存进行读写都是满足原子性的，这是JVM保证的原子性，那么volatile岂不是线程安全的？并不是。

JVM的计算过程并不是原子性的。我们举个例子，看一下代码：

```java
private volatile int num = 0;

public void fun(){
    for (int k=0;k<=10;k++){
        new Thread(() -> {
            int a = 10000;
            while (a > 0) {
                a--;
                num++;
            }
            System.out.println(num);
        }).start();
    }
}
```

按照正常的情况，最后的输出应该是100000才对，我们看看运行结果：

<img src="https://s1.ax1x.com/2020/11/03/ByRkFO.png" width=400 border="0" />

怎么才五万多，不应该是10万吗？这是因为，volatile仅仅只是保证了修改后的数据对其他线程立即可见，但是并不保证运算的过程的原子性。

这里的`num++`在编译之后是分为三步：1.在工作区中取出变量数据到处理器 2.对处理器中的数据进行加一操作 3.把数据写回工作内存。如果，在变量数据取到处理器运算的过程中，变量已经被修改了，所以这时候进行自增操作得到的结果就是错误的。举个例子：

变量`a=5`,当前线程取5到处理器，这时`a`被其他线程改成了8，但是处理器继续工作，把5自增得到6，然后把6写回主内存，覆盖数据8，从而导致了错误。因此，**由于Java运算的非原子性，volatile并不是绝对线程安全**。

那什么时候他是安全的：

1. 计算的结果不依赖原来的状态。
2. 不需要与其他的状态变量共同参与不变约束。

通俗点来讲，就是运算不需要依赖于任何状态的运算。因为依赖的状态，可能在运算的过程中就已经发生了变化了，而处理器并不知道。如果涉及到需要依赖状态的运算，则必须使用其他的线程安全方案，如加锁，来保证操作的原子性。



## 适用场景

#### 状态标志/多线程通知

状态标志是很适合使用volatile关键字，正如我们在第一部分举的例子，通过设置标志来通知其他所有的线程执行逻辑。或者是如上一部分的例子，当初始化完成之后设置一个标志通知其他线程。

而更加常用的一个类似场景是线程通知。我们可以设置一个变量，然后多个线程观察他。这样只需要在一个线程更改这个数值，那么其他的观察这个变量的线程均可以收到通知。非常轻量级、逻辑也更加简单。

#### 保证初始化完整：双重锁检查问题

单例模式是我们经常使用的，其中一种比较常见的写法就是双重锁检查，如下：

```java
public class JavaClass {
	// 静态内部变量
   private static JavaClass javaClass;
    // 构造器私有
   private JavaClass(){}
    
   public static JavaClass getInstance(){
       // 第一重判空
       if (javaClass==null){
           // 加锁，第二重判空
           synchronized(JavaClass.class){
               if (javaClass==null){
                   javaClass = new JavaClass();
               }
           }
       }
       return javaClass;
   }
}
```

这种代码很熟悉，对吧，具体的设计逻辑我就不展开了。那么这样的代码是否绝对是线程安全的呢？并不是的，在某些极端情况下，仍然会出现问题。而问题就出在`javaClass = new JavaClass();`这句代码上。

新建对象并不是一个原子操作，他主要有三个子操作：

1. 分配内存空间
2. 初始化Singleton实例
3. 赋值 instance 实例引用

正常情况下，也是按照这个顺序执行。但是JVM是会进行指令重排优化的。就可能变成：

1. 分配内存空间
2. 赋值 instance 实例引用
3. 初始化Singleton实例

赋值引用在初始化之前，那么外部拿到的引用，可能就是一个未完全初始化的对象，这样就造成问题了。所以这里可以给单例对象进行volatile修饰，限制指令重排，就不会出现这种情况了。当然，关于单例模式，更加建议的写法还是利用类加载来保证全局单例以及线程安全，当然，前提是你要保证只有一个类加载器。限于篇幅这里就不展开了。

其他类型的初始化标志，也是可以利用volatile关键字来达到限制指令重排的作用。



## 总结

关于Java并发编程的知识很多，而volatile仅仅只是冰山一角。并发编程的难点在于，他的bug隐藏很深，可能经过几轮测试都不能找到问题，但是一上线就崩溃了，且极难复现和查找原因。因而学习并发原理与并发编程思想非常重要。同时，更要注重原理。掌握了原理和本质，那么其他的相关知识也是手到擒来。

希望文章对你有帮助。

> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

