---
title: 	Android事件分发机制五：面试官你坐啊		#标题
date: 2021/1/26 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - android
 - 事件分发
					
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

很高兴遇见你~

事件分发系列文章已经到最后一篇了，先来回顾一下前面四篇，也当个目录：

- [Android事件分发机制一：事件是如何到达activity的？](https://juejin.cn/post/6918272111152726024) : 从window机制出发分析了事件分发的整体流程，以及事件分发的真正起点
- [Android事件分发机制二：viewGroup与view对事件的处理](https://juejin.cn/post/6920883974952714247) : 源码分析了viewGroup和view是如何分发事件的
- [Android事件分发机制三：事件分发工作流程](https://juejin.cn/post/6921238915143696392) : 分析了触摸事件在控件树中的分发流程模型
- [Android事件分发机制四：学了事件分发有什么用？](https://juejin.cn/post/6922020192662863886) : 从实战的角度剖析事件分发的运用

本文是最后一篇，主要是模拟面试情况提出一些问题以及解答，也当是整个事件分发知识的回顾。读者也可以尝试一下看看这些问题是否都能解答出来。



## 面试开始

1. 学过事件分发吗，聊聊什么是事件分发

   > 事件分发是将屏幕触控信息分发给控件树的一个套机制。
   > 当我们触摸屏幕时，会产生一些列的MotionEvent事件对象，经过控件树的管理者ViewRootImpl，调用view的dispatchPointerEvnet方法进行分发。

   

2. 那主要的分发流程是什么：

   > 在程序的主界面情况下，布局的顶层view是DecorView，他会先把事件交给Activity，Activity调用PhoneWindow的方法进行分发，PhoneWindow会调用DecorView的父类ViewGroup的dispatchTouchEvent方法进行分发。也就是**Activity->Window->ViewGroup**的流程。ViewGroup则会向下去寻找合适的控件并把事件分发给他。

   

3. 事件一定会经过Activity吗？

   > 不是的。我们的程序界面的顶层viewGroup，也就是decorView中注册了Activity这个callBack，所以当程序的主界面接收到事件之后会先交给Activity。
   > 但是，如果是另外的控件树，如dialog、popupWindow等事件流是不会经过Activity的。只有自己界面的事件才会经Activity。

   

4. Activity的分发方法中调用了onUserInteraction()方法，你能说说这个方法有什么作用吗？

   > 好的。这个方法在Activity接收到down的时候会被调用，本身是个空方法，需要开发者自己去重写。
   > 通过官方的注释可以知道，这个方法会在我们以任意的方式**开始**与Activity进行交互的时候被调用。比较常见的场景就是屏保：当我们一段时间没有操作会显示一张图片，当我们开始与Activity交互的时候可在这个方法中取消屏保；另外还有没有操作自动隐藏工具栏，可以在这个方法中让工具栏重新显示。

   

5. 前面你讲到最后会分发到viewGroup，那么viewGroup是如何分发事件的？

   > viewGroup处理事件信息分为三个步骤：拦截、寻找子控件、派发事件。
   >
   > 事件分发中有一个重要的规则：一个触控点的一个事件序列只能给一个view处理，除非异常情况。所以如果viewGroup消费了down事件，那么子view将无法收到任何事件。
   >
   > viewGroup第一步会判读这个事件是否需要分发给子view，如果是则调用onInterceptTouchEvent方法判断是否要进行拦截。
   > 第二步是如果这个事件是down事件，那么需要为他寻找一个消费此事件的子控件，如果找到则为他创建一个TouchTarget。
   > 第三步是派发事件，如果存在TouchTarget，说明找到了消费事件序列的子view，直接分发给他。如果没有则交给自己处理。

   

6. 你前面讲到“一个触控点的一个事件序列只能给一个view处理，除非异常情况”,这里有什么异常情况呢？如果发生异常情况该如何处理？

   > 这里的异常情况主要有两点：1.被viewGroup拦截，2.出现界面跳转等其他情况。
   >
   > 当事件流中断时，viewGroup会发送一个ACTION_CANCEL事件给到view，此时需要做一些状态的恢复工作，如终止动画，恢复view大小等等。

   

7. 那既然说到ACTION_CANCEL类型，那你可以说说还有什么事件类型吗？

   > 除了ACTION_CANCEL，其他事件类型还有：
   >
   > - ACTION_MOVE：当我们手指在屏幕上滑动时产生此事件
   > - ACTION_UP：当我们手指抬起时产生此事件
   >
   > 此外多指操作也比较常见：
   >
   > - ACTION_POINTER_DOWN: 当已经有一个手指按下的情况下，另一个手指按下会产生该事件
   > - ACTION_POINTER_UP: 多个手指同时按下的情况下，抬起其中一个手指会产生该事件。
   >
   > 一个完整的事件序列是从ACTION_DOWN开始，到ACTION_UP或者ACTION_CANCEL结束。
   > **一个手指**的完整序列是从ACTION_DOWN/ACTION_POINTER_DOWN开始，到ACTION_UP/ACTION_POINTER_UP/ACTION_CANCEL结束。

   

8. 哦？说到多指，那你知道ViewGroup是如何将多个手指产生的事件准确分发给不同的子view吗

   > 这个问题的关键在于MotionEvent以及ViewGroup内部的TouchTarget。
   >
   > 每个MotionEvent中都包含了当前屏幕所有触控点的信息，他的内部用了一个数组来存储不同的触控id所对应的坐标数值。
   >
   > 当一个子view消费了down事件之后，ViewGroup会为该view创建一个TouchTarget，这个TouchTarget就包含了该view的实例与触控id。这里的触控id可以是多个，也就是一个view可接受多个触控点的事件序列。
   >
   > 当一个MotionEvent到来之时，ViewGroup会将其中的触控点信息拆开，再分别发送给感兴趣的子view。从而达到精准发送触控点信息的目的。

   

9. 那view支持处理多指信息吗？

   > View默认是不支持的。他在获取触控点信息的时候并没有传入触控点索引，也就是获取的是MotionEvent内部数组中的第一个触控点的信息。多指需要我们自己去重写方法支持他。

   

10. 嗯嗯...那View是如何处理触摸事件的？

    > 首先，他会判断是否存在onTouchListener，存在则会调用他的onTouch方法来处理事件。如果该方法返回true那么就分发结束直接返回。而如果该监听器为null或者onTouch方法返回了false，则会调用onTouchEvent方法来处理事件。
    >
    > onTouchEvent方法中支持了两种监听器：onClickListener和onLongClickListener。View会根据不同的触摸情况来调用这两个监听器。同时进入到onTouchEvent方法中，无论该view是否是enable，只要是clickable，他的分发方法都是返回true。
    >
    > 总结一下就是：先调用onTouchListener，再调用onClickListener和onLongClickListener。

    

11. 你前面多次讲到分发方法和返回值，那你可以讲讲主要有什么方法以及他们之间的关系吗？

    > 嗯嗯。核心的方法有三个：dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent。
    >
    > 简单来说：dispatchTouchEvent是核心的分发方法，所有分发逻辑都在这个方法中执行；onInterceptTouchEvent在viewGroup负责判断是否拦截；onTouchEvent是消费事件的核心方法。viewGroup中拥有这三个方法，而view没有onInterceptTouchEvent方法。
    >
    > - viewGroup
    >   1. viewGroup的dispatchTouchEvent方法接收到事件消息，首先会去调用onInterceptTouchEvent判断是否拦截事件
    >      - 如果拦截，则调用自身的onTouchEvent方法
    >      - 如果不拦截则调用子view的dispatchTouchEvent方法
    >   2. 子view没有消费事件，那么会调用viewGroup本身的onTouchEvent
    >   3. 上面1、2步的处理结果为viewGroup的dispatchTouchEvent方法的处理结果，没有消费则返回false并返回给上一层的onTouchEvent处理，如果消费则分发结束并返回true。
    > - view
    >   1. view的dispatchTouchEvent默认情况下会调用onTouchEvent来处理事件，返回true表示消费事件，返回false表示没有消费事件
    >   2. 第1步的结果就是dispatchTouchEvent方法的处理结果，成功消费则返回true，没有消费则返回false并交给上一层的onTouchEvent处理
    >
    > 简单来说，在控件树中，每个viewGroup在dispatchTouchEvent方法中不断往下分发寻找消费的view，如果底层的view没有消费事件则会一层层网上调用viewGroup的onTouchEvent方法来处理事件。
    >
    > 同时，由于Activity继承了Window.CallBack接口，所以也有dispatchTouchEvent和onTouchEvent方法：
    >
    > 1. activity接收到触摸事件之后，会直接把触摸事件分发给viewGroup
    > 2. 如果viewGroup的dispatchTouchEvent方法返回false，那么会调用Activity的onTouchEvent来处理事件
    > 3. 第1、2步的处理结果就是activity的dispatchTouchEvent方法的处理结果，并返回给上层

    

12. 看来你对事件分发了解得挺多的，那你在实际中有运用到事件分发吗？

    > 嗯嗯，有的。举两个例子。
    >
    > 第一个需求是要设计一个按钮块，按下的时候会缩小高度变低同时变得半透明，放开的时候又会回弹。这个时候就可以在这个按钮的onTouchEvent方法中判断事件类型：down则开启按下动画，up则开启释放动画。同时注意接收到cancel事件的时候要恢复状态。
    >
    > 第二个是滑动冲突。解决滑动冲突的核心思路就是把滑动事件根据具体的情况分发给viewGroup或者内部view。主要的方法有外部拦截法和内部拦截法。
    > 外部拦截法的思路就是在viewGroup中判断滑动的情况，对符合自身滑动的事件进行拦截，对不符合的事件不拦截，给到内部view。内部拦截法的思路要求viewGroup拦截除了down事件以外的所有事件，然后再内部view中判断滑动的情况，对符合自身滑动情况的时间设置禁止拦截标志，对不符合自身滑动情况的事件则取消标志让viewGroup进行拦截。

    

13. 那外部和内部拦截法该如何选择呢？

    > 在一般的情况下，外部拦截法不需要对子view进行方法重写，比内部拦截法更加简单，推荐使用外部拦截法。
    >
    > 但如果需要在子view判断更多的触摸情况时，则使用内部拦截法可更加方法子view处理情况。

    

14. 前面一直聊到触摸事件，那你知道一个触摸事件是如何从触摸屏幕开始产生的吗？

    > 额....在屏幕接收到触摸信息后，会把这个信息交给InputServiceManager去处理，最后通过WindowManagerService找到符合的window，并把触摸信息发送给viewRootImpl,viewRootImpl经过层层封和处理之后，产生一个MotionEvent事件分发给view。

    

15. 可以具体讲讲前面IMS处理的流程吗？

    > 啊。。这。。。嗯。。。。不会。。。

    

16. 你还有什么想问的吗？

    > 可不可以。。。。给我个小小的点赞再走？

    

17. 下次一定。

    > =_=....



## 最后

关于面试，我一直坚持的一个观点就是：**可以面向面试知识点学习，但不可面向面试题目答案学习** 。把相关热门题目的答案背诵下来可以忽悠到一些面试官，但现在基本上都不是简单的询问什么是事件分发，而会给一个具体的需求让我们思考等等。背诵面试题短期可能会让我们好像学到了很多，但事实上，我们什么都没学到。

事件分发系列文章到此完结。有疑问欢迎评论区交流，希望文章对你有帮助~

# 都看到这了，要不给作者留下个点赞再走？



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

