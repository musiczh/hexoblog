---
title: 关于Handler同步屏障你可能不知道的问题	#标题
date: 2021/3/17 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - android
 - handler
					
categories:	#分类
 - Android

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

很高兴遇见你 ~

关于handler的内容，基本每个android开发者都掌握了，网络中的优秀博客也非常多，我之前也写过一篇文章，读者感兴趣可以去看看：[传送门](https://blog.csdn.net/weixin_43766753/article/details/108968666)。

这篇文章主要讲Handler中的同步屏障问题，这也是面试的热门问题。很多读者觉得这一块的知识很偏，实战中并没有什么用处，仅仅用来面试，包括笔者。我在[Handler机制](https://blog.csdn.net/weixin_43766753/article/details/108968666)一文中写到：`其实同步屏障对于我们的日常使用的话其实是没有多大用处。因为设置同步屏障和创建异步Handler的方法都是标志为hide，说明谷歌不想要我们去使用他`。


笔者在前段时间面试时被问到这个问题，之后重新思考了这个问题，发现了一些不一样的地方。结合了一些大佬的观点，发现同步屏障这个**机制**，并不如我们所想完全没用，而还是有他的长处。这篇文章则表达一下我对同步屏障机制的思考，希望对你有帮助。

文章主要内容是：介绍同步屏障，分析系统api，如何正确使用他。

那么，我们开始吧。


## 什么是同步屏障机制

**同步屏障机制是一套为了让某些特殊的消息得以更快被执行的机制**。

注意这里我在同步屏障之后加上了**机制**二字，原因是单纯的同步屏障并不起作用，他需要和其他的Handler组件配合才能发挥作用。

这里我们假设一个场景：我们向主线程发送了一个UI绘制操作Message，而此时消息队列中的消息非常多，那么这个Message的处理可能会得到延迟，绘制不及时造成界面卡顿。同步屏障机制的作用，是让这个绘制消息得以越过其他的消息，优先被执行。

MessageQueue中的Message，有一个变量`isAsynchronous`，他标志了这个Message是否是异步消息；标记为true称为异步消息，标记为false称为同步消息。同时还有另一个变量`target`，标志了这个Message最终由哪个Handler处理。

我们知道每一个Message在被插入到MessageQueue中的时候，会强制其`target`属性不能为null，如下代码：
```java
MessageQueue.class

boolean enqueueMessage(Message msg, long when) {
  // Hanlder不允许为空
  if (msg.target == null) {
      throw new IllegalArgumentException("Message must have a target.");
  }
  ...
}
```
而android提供了另外一个方法来插入一个特殊的消息，强行让`target==null`：
```java
private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        // 把当前需要执行的Message全部执行
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        // 插入同步屏障
        if (prev != null) { 
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```
代码有点长，重点在于：**没有给Message赋值target属性，且插入到Message队列头部**。当然源码中还涉及到延迟消息，我们暂时不关心。**这个target==null的特殊Message就是同步屏障**

MessageQueue在获取下一个Message的时候，如果碰到了同步屏障，那么不会取出这个同步屏障，而是会遍历后续的Message，找到第一个异步消息取出并返回。这里跳过了所有的同步消息，直接执行异步消息。为什么叫同步屏障？因为它可以屏蔽掉同步消息，优先执行异步消息。

我们来看看源码是怎么实现的：

```java
Message next() {
    ···
    if (msg != null && msg.target == null) {
        // 同步屏障，找到下一个异步消息
        do {
            prevMsg = msg;
            msg = msg.next;
        } while (msg != null && !msg.isAsynchronous());
    }
    ···
}
```

如果遇到同步屏障，那么会循环遍历整个链表找到标记为异步消息的Message，即isAsynchronous返回true，其他的消息会直接忽视，那么这样异步消息，就会提前被执行了。

**注意，同步屏障不会自动移除，使用完成之后需要手动进行移除，不然会造成同步消息无法被处理**。我们可以看一下源码：
```java
Message next() {
    ...
    // 阻塞时间
    int nextPollTimeoutMillis = 0;
    for (;;) {
        // 阻塞对应时间 
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 同步屏障，找到下一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            // 如果上面有同步屏障，但却没找到异步消息，
            // 那么msg会循环到链表尾，也就是msg==null
            if (msg != null) {
                ···
            } else {
                // 没有消息，进入阻塞状态
                nextPollTimeoutMillis = -1;
            }
            ···
        }
    }
}
```
可以看到如果没有即时移除同步屏障，他会一直存在且不会执行同步消息。因此使用完成之后必须即时移除。但我们无需操心这个，后面就知道了。


## 如何发送异步消息

上面我们了解到了同步屏障的作用，但是会发现`postSyncBarrier`方法被标记为`@hide`，也就是我们无法调用这个方法。那，讲了这么多有什么用？


![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb997212eaac4c548cfd61889350d55b~tplv-k3u1fbpfcp-watermark.image)

咳咳~不要慌，但我们可以发异步消息啊。在系统添加同步屏障的时候，不就可以趁机上车了，是吧。

添加异步消息有两种办法：

- 使用异步类型的Handler发送的全部Message都是异步的
- 给Message标志异步

给Message标记异步是比较简单的，通过`setAsynchronous`方法即可。

Handler有一系列带Boolean类型的参数的构造器，这个参数就是决定是否是异步Handler：

```java
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    // 这里赋值
    mAsynchronous = async;
}
```

在发送消息的时候就会给Message赋值：

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();
	// 赋值
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

但是异步类型的Handler构造器是标记为hide，我们无法使用，但在api28之后添加了两个重要的方法：
```java
public static Handler createAsync(@NonNull Looper looper) {
    if (looper == null) throw new NullPointerException("looper must not be null");
    return new Handler(looper, null, true);
}

    
public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback) {
    if (looper == null) throw new NullPointerException("looper must not be null");
    if (callback == null) throw new NullPointerException("callback must not be null");
    return new Handler(looper, callback, true);
}
```
通过这两个api就可以创建异步Handler了，而异步Handler发出来的消息则全是异步的。

```java
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

## 如何正确使用

上面我们似乎漏了一个问题：**系统什么时候添加同步屏障？**。

异步消息需要同步屏障的辅助，但同步屏障我们无法手动添加，因此了解系统何时添加和删除同步屏障是非常必要的。只有这样，才能更好地运用异步消息这个功能，知道**为什么要用和如何用**。


了解同步屏障需要简单了解一点屏幕刷新机制的内容。放心，只需要了解一丢丢就可以了。

我们的手机屏幕刷新频率有不同的类型，60Hz、120Hz等。60Hz表示屏幕在一秒内刷新60次，也就是每隔16.6ms刷新一次。屏幕会在每次刷新的时候发出一个 `VSYNC` 信号，通知CPU进行绘制计算。具体到我们的代码中，可以认为就是执行`onMesure()`、`onLayout()`、`onDraw()`这些方法。好了，大概了解这么多就可以了。

了解过 view 绘制原理的读者应该知道，view绘制的起点是在 `viewRootImpl.requestLayout()` 方法开始，这个方法会去执行上面的三大绘制任务，就是测量布局绘制。但是，重点来了：
> **调用`requestLayout()`方法之后，并不会马上开始进行绘制任务，而是会给主线程设置一个同步屏障，并设置 ASYNC 信号监听。
> 当 ASYNC 信号的到来，会发送一个异步消息到主线程Handler，执行我们上一步设置的绘制监听任务，并移除同步屏障**


这里我们只需要明确一个情况：**调用requestLayout()方法之后会设置一个同步屏障，知道ASYNC信号到来才会执行绘制任务并移除同步屏障**。（这里涉及到Android屏幕刷新以及绘制原理更多的内容，本文不详细展开，感兴趣的读者可以点击文末的连接阅读。）

那，这样在等待ASYNC信号的时候主线程什么事都没干？是的。这样的好处是：**保证在ASYNC信号到来之时，绘制任务可以被及时执行，不会造成界面卡顿**。但这样也带来了相对应的代价：
- 我们的同步消息最多可能被延迟一帧的时间，也就是16ms，才会被执行
- 主线程Looper造成过大的压力，在VSYNC信号到来之时，才集中处理所有消息

改善这个问题办法就是：**使用异步消息**。当我们发送异步消息到MessageQueue中时，在等待VSYNC期间也可以执行我们的任务，让我们设置的任务可以更快得被执行且减少主线程Looper的压力。

可能有读者会觉得，异步消息机制本身就是为了避免界面卡顿，那我们直接使用异步消息，会不会有隐患？这里我们需要思考一下，什么情况的异步消息会造成界面卡顿：**异步消息任务执行过长、异步消息海量。**

如果异步消息执行时间太长，那即时是同步任务，也会造成界面卡顿，这点应该都很好理解。其次，若异步消息海量到达影响界面绘制，那么即使是同步任务，也是会导致界面卡顿的；原因是MessageQueue是一个链表结构，海量的消息会导致遍历速度下降，也会影响异步消息的执行效率。所以我们应该注意的一点是：
> **不可在主线程执行重量级任务，无论异步还是同步**。

那，我们以后岂不是可以直接使用异步Handler来取代同步Handler了？是，也不是。

同步Handler有一个特点是会遵循与绘制任务的顺序，设置同步屏障之后，会等待绘制任务完成，才会执行同步任务；而异步任务与绘制任务的先后顺序无法保证，在等待VSYNC的期间可能被执行，也有可能在绘制完成之后执行。因此，我的建议是：**如果需要保证与绘制任务的顺序，使用同步Handler；其他，使用异步Handler**。


## 最后

技术深挖，总是能学到一些更加不一样的知识。当知识的广度越来越广，知识之间的联系会迸发出不一样的火花。

第一次学习Handler，仅仅知道可以发送消息并执行；第二次学习Handler，知道了其在Android消息机制重要地位；第三次学习Handler，知道了原来Handler和屏幕刷新机制还有这么一个联系。

温故而知新，古人诚不欺我。

如果文章对你有帮助，还希望可以点赞鼓励一下作者。

## 推荐文献
- [“终于懂了” 系列：Android屏幕刷新机制—VSync、Choreographer 全面理解](https://cloud.tencent.com/developer/article/1685247):胡飞洋作者一篇关于绘制和屏幕刷新很好的文章，阅读后可以对异步消息有更加深层次的理解。
- [RxAndroid 2.1.0 has a new API](https://www.zacsweers.dev/rxandroids-new-async-api/):2018年RxAndroid决定添加异步api的一篇文章，解释了为什么要使用异步消息。



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信告知。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

