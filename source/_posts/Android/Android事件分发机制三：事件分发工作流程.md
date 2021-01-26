---
title: 	Android事件分发机制三：事件分发工作流程 #标题
date: 2021/1/23 00:00:00 						#建立日期
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

本文是事件分发系列的第三篇。

在前两篇文章中，[Android事件分发机制一：事件是如何到达activity的？](https://juejin.cn/post/6918272111152726024) 分析了事件分发的真正起点：viewRootImpl，Activity只是其中的一个环节；[Android事件分发机制二：viewGroup与view对事件的处理](https://juejin.cn/post/6920883974952714247) 源码解析了viewGroup和view是如何分发事件的。

事件分发的核心内容，则为viewGroup和view对事件的分发，也就是第二篇文章。第二篇文章对源码的分析较为深入，缺乏一个更高的角度来审视事件分发流程。本文在前面的分析基础上，对整个事件分发的工作流程进行一个总结，更好地把握事件是如何在不同的对象和方法之间进行传递。



## 回顾

先来回顾一下整体的流程，以便更好地定位我们的知识。

![](https://i.loli.net/2021/01/23/HbxLquIz75YiQGM.png)

1. 触摸信息从手机触摸屏幕时产生，通过IMS和WMS发送到viewRootImpl
2. viewRootImpl通过调用view的dispatchPointerEvent方法把触摸信息传递给view
3. view通过调用自身的dispatchTouchEvent方法开始了事件分发

图中的view指的是一个控件树，他可以是一个viewGroup也可以是一个简单的view。因为viewGroup是继承自view，所以一个控件树，也可以看做是一个view。

我们今天探讨的工作流程，就是从图中的view调用自身的dispatchTouchEvent开始。



## 主要对象与方法

### 事件分发的对象

这一部分内容在第二篇有详细解析，这里做个简单的回顾。

当我们手机触碰屏幕时会产生一系列的MotionEvent对象，根据触摸的情况不同，这些对象的类型也会不同。具体如下：

- ACTION_DOWN: 表示手指按下屏幕
- ACTION_MOVE: 手指在屏幕上滑动时，会产生一系列的MOVE事件
- ACTION_UP: 手指抬起，离开屏幕、
- ACTION_CANCEL：当出现异常情况事件序列被中断，会产生该类型事件
- ACTION_POINTER_DOWN: 当已经有一个手指按下的情况下，另一个手指按下会产生该事件
- ACTION_POINTER_UP: 多个手指同时按下的情况下，抬起其中一个手指会产生该事件



### 事件分发的方法

事件分发属于控件系统的一部分，主要的分发对象是viewGroup与view。而其中核心的方法有三个： `dispatchTouchEvent` 、`onInterceptTouchEvent` 、 `onTouchEvent` 。那么在讲分发流程之前，先来介绍一下这三个方法。这三个方法属于view体系的类，其中Window.CallBack接口中包含了 `dispatchTouchEvent` 和 `onTouchEvent` 方法，Activity和Dialog都实现了Window.CallBack接口，因此都实现了该方法。因这三个方法经常在自定义view中被重写，以下的分析，如果没有特殊说明都是在默认方法实现的情况下。

#### dispatchTouchEvent

该方法是事件分发的核心方法，事件分发的逻辑都是在这个方法中实现。该方法存在于类View中，子类ViewGroup、以及其他的实现类如DecorView都重写了该方法。

无论是在viewGroup还是view，该方法的主要作用都是处理事件。如果成功处理则返回true，处理失败则返回false，表示事件没有被处理。具体到类，在viewGroup相关类中，该方法的主要作用是把事件分发到该viewGroup所拥有的子view，如果子view没有处理则自己处理；在view的相关类中，该方法的主要作用是消费触摸事件。

#### onInterceptTouchEvent

该方法只存在于viewGroup中，当一个事件需要被分发到子view时，viewGroup会调用此方法检查是否要进行拦截。如果拦截则自己处理，而如果不拦截才会调用子view的 `dispatchTouchEvent` 方法分发事件。

方法返回true表示拦截事件，返回false表示不拦截。

这个方法默认只对鼠标的相关操作的一种特殊情况进行了拦截，其他的情况需要具体的实现类去重写拦截。

#### onTouchEvent

该方法是消费事件的主要方法，存在于view中，viewGroup默认并没有重写该方法。方法返回true表示消费事件，返回false表示不消费事件。

viewGroup分发事件时，如果没有一个子view消费事件，那么会调用自身的onTouchEvent方法来处理事件。View的dispatchTouchEvent方法中，并不是直接调用onTouchEvent方法来消费事件，而是先调用onTouchListener判断是否消费；如果onTouchListener没有消费事件，才会调用onTouchEvent来处理事件。

我们为view设置的onClickListener与onLongClickListener都是在View的dispatchTouchEvent方法中，根据具体的触摸情况被调用。



## 重要规则

事件分发有一个很重要的原则：**一个触控点的事件序列只能给一个view消费，除非发生异常情况如被viewGroup拦截** 。具体到代码实现就是：**消费了一个触控点事件序列的down事件的view，将持续消费该触控点事件序列接下来的所有的事件** 。举个栗子：

当我手指按下屏幕时产生了一个down事件，只有一个view消费了这个down事件，那么接下来我的手指滑动屏幕产生的move事件会且仅会给这个view消费。而当我手机抬起，再按下时，这时候又会产生新的down事件，那么这个时候就会再一次去寻找消费down事件的view。所以，**事件分发，是以事件序列为单位的** 。

因此下面的工作流程中都是指**down事件的分发** ，而不是ACTION_MOVE或ACTION_UP的分发。因为消费了down事件，意味着接下来的move和up事件都会给这个view处理，也就无所谓分发了。但同时注意事件序列是可以被viewGroup的onInterceptTouchEvent中断的，这些就属于其他的情况了。

细心的读者还会发现事件分发中包含了多点触控。在多点触控的情况下，ACTION_POINTER_DOWN与ACTION_DOWN的分发规则是不同的，具体可前往第二篇文章了解详细。ACTION_POINTER_DOWN在ACTION_DOWN的分发模型上稍作了一些修改而已，后面会详细解析，



## 工作流程模型

工作流程模型，本质上就是不同的控件对象，viewGroup和view之间事件分发方法的关系。需要注意的是，这里讨论的是viewGroup和view的默认方法实现，不涉及其他实现类如DecorView的重写方法。

下面用一段伪代码来表示三个事件分发方法之间的关系（ **这里再次强调，这里的事件分发模型分发的事件类型是ACTION_DOWN且都是默认的方法，没有经过重写，这点很重要** ）：

```java
public boolean dispatchTouchEvent(MotionEvent event){
    
    // 先判断是否拦截
    if (onInterceptTouchEvent()){
        // 如果拦截调用自身的onTouchEvent方法判断是否消费事件
        return onTouchEvent(event);
    }
    // 否则调用子view的分发方法判断是否处理事件
    if (childView.dispatchTouchEvent(event)){
        return true;
    }else{
        return onTouchEvent(event);
    }
}
```

这段代码非常好的展示了三个方法之间的关系：在viewGroup收到触摸事件时，会先去调用 `onInterceptTouchEvent` 方法判断是否拦截，如果拦截则调用自己的 `onTouchEvent` 方法处理事件，否则调用子view的 `dispatchTouchEvent` 方法来分发事件。因为子view也有可能是一个viewGroup，这样就形成了一个类似递归的关系。

这里我再补上view分发逻辑的简化伪代码：

```java
public boolean dispatchTouchEvent(MotionEvent event){
    // 先判断是否存在onTouchListener且返回值为true
    if (mOnTouchListener!=null && mOnTouchListener.onTouch(event)){
        // 如果成功消费则返回true
        return true;
    }else{
        // 否则调用onTouchEvent消费事件
        return onTouchEvent(event);
    }
}
```

view与viewGroup不同的是他不需要分发事件，所以也就没有必要拦截事件。view会先检查是否有onTouchListener且返回值是否为true，如果是true则直接返回，否则调用onTouchEvent方法来处理事件。

基于上述的关系，可以得到下面的工作流程图：

![](https://s3.ax1x.com/2021/01/24/sHey8I.png)

这里为了展示类递归关系使用了画了两个viewGroup，只需看中间一个即可，下面对这个图进行解析：

- viewGroup
  1. viewGroup的dispatchTouchEvent方法接收到事件消息，首先会去调用onInterceptTouchEvent判断是否拦截事件
     - 如果拦截，则调用自身的onTouchEvent方法
     - 如果不拦截则调用子view的dispatchTouchEvent方法
  2. 子view没有消费事件，那么会调用viewGroup本身的onTouchEvent
  3. 上面1、2步的处理结果为viewGroup的dispatchTouchEvent方法的处理结果，并返回给上一层的onTouchEvent处理
- view
  1. view的dispatchTouchEvent默认情况下会调用onTouchEvent来处理事件，返回true表示消费事件，返回false表示没有消费事件
  2. 第1步的结果就是dispatchTouchEvent方法的处理结果，成功消费则返回true，没有消费则返回false并交给上一层的onTouchEvent处理

可以看到整个工作流程就是一个“U”型结构，在不拦截的情况下，会一层层向下寻找消费事件的view。而如果当前view不处理事件，那么就一层层向上抛，寻找处理的viewGroup。

上述的工作流程模型并不是完整的，还有其他的特殊情况没有考虑。下面讨论几种特殊的情况：

##### 事件序列被中断

我们知道，当一个view接收了down事件之后，该触控点接下来的事件都会被这个view消费。但是，viewGroup是可以在中途掐断事件流的，因为每一个需要分发给子view的事件都需要经过拦截方法：`onInterceptTouchEvent` （当然，这里不讨论子view设置不拦截标志的情况）。那么当viewGroup掐断事件流之后，事件的走向又是如何的呢？参看下图：（**注意，这里不讨论多指操作的情况，仅讨论单指操作的move或up事件被viewGroup拦截的情况**）

![](https://i.loli.net/2021/01/24/rMBhWtyPluD4ZH7.png)

1. 当viewGroup拦截子view的move或up事件之后，会将当前事件改为cancel事件并发送给子view
2. 如果当前事件序列还未结束，那些接下来的事件都会交给viewGroup的ouTouchEvent处理
3. 此时不管是viewGroup还是view的onTouchEvent返回了false，那么将导致整个控件树的dispatchTouchEvent方法返回false
   - 秉承着一个事件序列只能给一个view消费的原则，如果一个view消耗了down事件却在接下来的move或up事件返回了false，那么此事件不会给上层的viewGroup处理，而是直接返回false。

##### 多点触控情况

上面讨论的所有情况，都是不包含多点触控情况的。多点触控的情况，在原有的事件分发流程上，新增了一些特殊情况。这里就不再画图，而是把一些特殊情况描述一下，读者了解一下就可以了。

默认情况下，viewGroup是支持多点触控的分发，但view是不支持多点触控的，需要自己去重写 `dispatchTouchEvent` 方法来支持多点触控。

多点触控的分发规则如下：

viewGroup在已有view接受了其他触点的down事件的情况下，另一个手指按下产生ACTION_POINTER_DOWN事件传递给viewGroup：

1. viewGroup会按照ACTION_DOWN的方式去分发ACTION_POINTER_DOWN事件
   - 如果子view消费该事件，那么和单点触控的流程一致
   - 如果子view未消费该事件，那么会交给上一个最后接收down事件的view去处理
2. viewGroup两个view接收了不同的down事件，那么拦截其中一个view的事件序列，viewGroup不会消费拦截的事件序列。换句话说，viewGroup和其中的view不能同时接收触摸事件。



## Activity的事件分发

细心的读者会发现，上述的工作流程并不涉及Activity。我们印象中的事件分发都是 `Activity -> Window -> ViewGroup` ，那么这是怎么回事？这一切，都是DecorView “惹的祸” 。

DecorView重写viewGroup的 `dispatchTouchEvent` 方法，当接收到触摸事件后，DecorView会首先把触摸对象传递给内部的callBack对象。没错，这个callBack对象就是Activity。加入Activity这个环节之后，分发的流程如下图所示：

[![](https://s3.ax1x.com/2021/01/24/sbEW0e.png)](https://imgchr.com/i/sbEW0e)

整体上和前面的流程没有多大的不同，Activity继承了Window.CallBack接口，所以也有dispatchTouchEvent和onTouchEvent方法。对上图做个简单的分析：

1. activity接收到触摸事件之后，会直接把触摸事件分发给viewGroup
2. 如果viewGroup的dispatchTouchEvent方法返回false，那么会调用Activity的onTouchEvent来处理事件
3. 第1、2步的处理结果就是activity的dispatchTouchEvent方法的处理结果，并返回给上层

上面的流程不仅适用于Activity，同样适用于Dialog等使用DecorView和callback模式的控件系统。



## 最后

到这里，事件分发的主要内容也就讲解完了。结合前两篇文章，相信读者对于事件分发有更高的认知。

纸上得来终觉浅，绝知此事要躬行。学了知识之后最重要的就是实践。下一篇文章将简单分析一下如何利用学习到的事件分发知识运用到实际开发中。

# 原创不易，你的点赞是我创作最大的动力，感谢阅读 ~



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

