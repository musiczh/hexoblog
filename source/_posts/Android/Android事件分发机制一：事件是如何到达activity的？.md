---
title: 	Android事件分发机制一：事件是如何到达activity的？		#标题
date: 2021/1/19 00:00:00 						#建立日期
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

> 事件分发，真的一定从Activity开始吗？

## 前言

很高兴遇见你~

事件分发，android中一个老生常谈的话题了。前阵子去面试一家企业，他里面有一道笔试题问到事件分发的流程，正确答案是选择：Activity->window->view，基本的流程我们也都知道是从Activity开始分发。

当时我选择完之后，我就开始思考，那事件是怎么到达Activity的？如果了解过window机制的读者会知道，事件分发也是window机制的一部分，而Activity不属于window机制内，那么触摸事件应该是从Window开始才对，怎么是从Activity开始的呢？

抱着这些疑问，我重新学习了事件分发，结合之前的window机制内容，对于事件分发的理解又有了新的认知。这篇文章就是要回答事件是如何到达Activity的这个问题。

你以为我接下来要开始讲源码、系统底层了？不不不，本文不讲这些。我们要探究的是，一个触摸信息从系统底层产生之后，一步步到达Activity进行分发的整体流程。而关于系统底层的逻辑，不在本文的讨论范围内。

本文是笔者android触摸事件系列文章的开篇，主要的内容是分析触摸事件传递的路径。不会纠结于源码与底层，而是把触摸事件来源的大体流程呈现出来，便于对事件分发体系有个更加完整的理解。



## 管理单位：window

android的view管理是以window为单位的，每个window对应一个view树。这里管理涉及到view的绘制以及事件分发等。Window机制不仅管理着view的显示，也负责view的事件分发。关于window的本质，可以阅读笔者的另一篇文章[window机制](https://juejin.cn/post/6888688477714841608)。研究事件分发的来源，则必须对于window机制有一定的了解。

所以，首先要了解一个概念：view树。

我们的应用布局，一般是有多层viewGroup和view的嵌套，如下图：

![image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5301713e67a34cab8887cf5f00bd45e0~tplv-k3u1fbpfcp-zoom-1.image)

而他们对应的结构关系如下图所示

![image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/754f43ce75e841229cfc44392fdaa3a5~tplv-k3u1fbpfcp-zoom-1.image)

此时，我们就可以称该布局是以一个LinearLayout为根的一棵view树。LinearLayout可以直接访问FrameLayout和RelativeLayout，因为他们都是LinearLayout的子view，同样的RelativeLayout可以直接访问Button。

每一棵view树都有一个根，叫做**ViewRootImpl** ，他负责管理这整一棵view树的绘制、事件分发等。

我们的应用界面一般会有多个view树，我们的activity布局就是一个view树、其他应用的悬浮窗也是一个view树、dialog界面也是一个view树、我们使用windowManager添加的view也是一个view树等等。最简单的view树可以只有一个view。

 android中view的绘制和事件分发，都是以view树为单位。**每一棵view树，则为一个window** 。系统服务WindowManagerService，管理界面的显示就是以window为单位，也可以说是以view树为单位。而view树是由viewRootImpl来负责管理的，所以可以说，wms（WindowManagerService的简写）管理的是viewRootImpl。如下图：

![window机制](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af6d0eeb6fe5488690a8a6bed48d95bd~tplv-k3u1fbpfcp-zoom-1.image)

- wms是运行在系统服务进程的，负责管理所有应用的window。应用程序与wms的通信必须通过Binder进行跨进程通信。
- 每个viewRootImpl在wms中都有一个windowState对应，wms可以通过windowState找到对应的viewRootImpl进行管理。

了解window机制的一个重要原因是：**事件分发并不是由Activity驱动的，而是由系统服务驱动viewRootImpl来进行分发** ，甚至可以说，在框架层角度，和Activity没有任何关系。这将有助于我们对事件分发的本质理解。

那么触摸信息是如何一步步到达viewRootImpl？为什么说viewRootImpl是事件分发的起点？viewRootImpl如何对触摸信息进行分发处理的？这是我们接下来要讨论的。



## 触摸信息是如何到达viewRootImpl的？

我们都知道的是，在我们手指触摸屏幕时，即产生了触摸信息。这个触摸信息由屏幕这个硬件产生，被系统底层驱动获取，交给Android的输入系统服务：InputManagerService，也就是IMS。

IMS会对这个触摸信息进行处理，通过WMS找到要分发的window，随后发送给对应的viewRootImpl。所以发送触摸信息的并不是WMS，WMS提供的是window的相关信息。

这一部分涉及到系统底层的逻辑，不是本文的重点，感兴趣的读者推荐阅读gityuan博主的文章[Input系统-事件处理全过程](http://gityuan.com/2016/12/31/input-ipc/)。这里不展开讲解。大体的过程如下图：

![事件是如何到达viewRootImpl](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1497277cd15b4230af087535f01078ef~tplv-k3u1fbpfcp-zoom-1.image)

当viewRootImpl接收到触摸信息时，也正是应用程序进程事件分发的开始。



## viewRootImpl是如何分发事件的？

前面我们讲到，viewRootImpl管理一棵view树，view树的最外层是viewGroup,	而viewGroup继承于view。因此整一棵view树，从外部可以看做一个view。viewRootImpl接收到触摸信息之后，经过处理之后，封装成MotionEvent对象发送给他所管理的view，由view自己进行分发。

前面我们讲到，view树的根节点可以是一个viewGroup，也可以是一个单独的view，因此，这里的派发就会有两种不同的方式：直接给view进行处理 or viewGroup进行事件分发。viewGroup继承自view，view中有一个方法用于分发事件：`dispatchTouchEvent` 。子类可重写该方法来实现自己的分发逻辑，ViewGroup重写了该方法。

我们的应用布局界面或者dialog的布局界面，顶层的viewGroup为DecorView，因此会调用DecorView的 `dispatchTouchEvent` 方法进行分发。DecorView重写了该方法，逻辑比较简单，仅仅做了一个判断：

```java
DecorView.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

1. 如果window callBack对象不为空，则调用callBack对象的分发方法进行分发
2. 如果window callBack对象为空，则调用父类ViewGroup的事件分发方法进行分发

这里的windowCallBack是一个接口，他里面包含了一些window变化的回调方法，其中就有 `dispatchTouchEvent` ，也就是事件分发方法。

Activity实现了Window.CallBack接口，并在创建布局的时候，把自己设置给了DecorView，因此在Activity的布局界面中，DecorView会把事件分发给Activity进行处理。同理，在Dialog的布局界面中，会分发给实现了callBack接口的Dialog。

而如果顶层的viewGroup不是DecorView，那么对调用对应view的`dispatchTouchEvent`方法进行分发。例如，顶层的view是一个Button，那么会直接调用Button的 `dispatchTouchEvent` 方法；如果顶层viewGroup子类没有重写 `dispatchTouchEvent` 方法，那么会直接调用ViewGroup默认的 `dispatchTouchEvent` 方法。

整体的流程如下图：

![viewRootImpl对事件的分发流程](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce4115c10663451ab8add7741175279c~tplv-k3u1fbpfcp-watermark.image)

1. viewRootImpl会直接调用管理的view的 `dispatchTouchEvent` 方法，根据具体的view的类型，调用具体的方法。
2. view树的根view可能是一个view，也可能是一个viewGroup，view会直接处理事件，而viewGroup则会进行分发。
3. DecorView重写了 `dispatchTouchEvent` 方法，会先判断是否存在callBack，优先调用callBack的方法，也就是把事件传递给了Activity。
4. 其他的viewGroup子类会根据自身的逻辑进行事件分发。

因此，触摸事件一定是从Activity开始的吗？不是,Activity只是其中的一种情况，只有Activity自己负责的那一棵view树，才一定会到达activity，而其他的window，则不会经过Activity。触摸事件是从viewRootImpl开始，而不是Activity。



## 控件对于事件的分发

到这里，我们知道触摸事件是先发送到viewRootImpl，然后由viewRootImpl调用其所管理的view的方法进行事件分发。按照正常的流程，view会按照控件树向下去分发。而事件却到了activity、dialog，就是因为DecorView这个“叛徒”的存在。

前面讲到，DecorView和其他的viewGroup很不一样，他有一个windowCallBack，会优先把触摸事件发送给callBack，从而导致触摸事件脱离了控件树。那么，这些callBack是如何处理触摸事件的？触摸事件又是如何再一次回到控件树进行分发的呢？

了解具体的分发之前，需要先来了解一个类：PhoneWindow。

PhoneWindow继承自抽象类Window，但是，他本身并不是一个window，而是一个窗口功能辅助类。我们知道，一个view树，或者说控件树，就是一个window。PhoneWindow内部维护着一个控件树和一些window参数，这个控件树的根view，就是DecorView。他们和Activity的关系如下图：

![phonewindow](https://i.loli.net/2021/01/19/hUD5RqS8Ilyjrbu.png)

我们的Activity通过直接持有PhoneWindow实例从而来管理这个控件树。DecorView可以认为是一个界面模板，他的布局大概如下：

![decorView](https://i.loli.net/2021/01/19/38fBVbsdM7NHaYX.png)

我们的Activity布局，就被添加到内容栏中，属于DecorView控件树的一部分。这样Activity可以通过PhoneWindow，间接管理自身的界面，把window相关的操作都托管给PhoneWindow，减轻自身负担。

PhoneWindow并不是Activity专属的，其他如Dialog也是自己创建了一个PhoneWindow。PhoneWindow仅仅只是作为一个窗口功能辅助类，帮助控件更好地创建与管理界面。

前面讲到，DecorView接收到事件之后，会调用windowCallBack的方法进行事件分发，我们先来看看Activity是如何分发的：

#### Activity

我们首先看到Activity对于callBack接口方法的实现：

```java
Activity.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    // down事件，回调onUserInteraction方法
    // 这个方法是个空实现，给开发者去重写
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    // getWindow返回的就是PhoneWindow实例
    // 直接调用PhoneWindow的方法
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    // 如果前面分发过程中事件没有被处理，那么调用Activity自身的方法对事件进行处理
    return onTouchEvent(ev);
}
```

可以看到Activity对于事件的分发逻辑还是比较简单的，直接调用PhoneWindow的方法进行分发。如果事件没有被处理，那么自己处理这个事件。接下来看看PhoneWindow如何处理：

```java
PhoneWindow.java api29
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

这里的mDecor就是PhoneWindow内部维护的DecorView了，简单粗暴，直接调用DecorView的方法进行分发。看到DecorView的方法：

```java
DecorView.java api29
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

好家伙，DecorView对于事件也是没有做任何处理，直接调用父类的方法进行分发。DecorView继承自FrameLayout，但是FrameLayout并没有重写 `dispatchTouchEvent` 方法，所以调用的就是viewGroup类的方法了。所以到这里，事件就交给viewGroup去分发给控件树了。

我们来回顾一下：DecorView交给Activity处理，Activity直接交给PhoneWindow处理，PhoneWindow直接交给其内部的DecorView处理，而DecorView则直接调用父类ViewGroup的方法进行分发，ViewGroup则会按照具体的逻辑分发到整个控件树中感兴趣的子控件。关于ViewGroup如何进行分发的内容，在后续的文章继续分析。

从DecorView开始，绕了一圈，又回到控件树进行分发了。接下来看看Dialog是如何分发的：

#### Dialog

直接看到Dialog中的 `diapatchTouchEvent` 代码：

```java
Dialog.java api29
public boolean dispatchTouchEvent(@NonNull MotionEvent ev) {
    if (mWindow.superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

这里的mWindow，就是Dialog内部维护的PhoneWindow实例，接下去的逻辑就和Activity的流程一样了，这里不再赘述。

而如果没有使用DecorView作为模板的窗口，流程就会和上述不一致了，例如PopupWindow：

#### PopupWindow

PopupWindow他的根View是 `PopupDecorView` ，而不是 `DecorView` 。虽然他的名字带有DecorView，但是却和DecorView一点关系都没有，他是直接继承于FrameLayout。我们看到他的事件分发方法：

```java
PopupWindow.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (mTouchInterceptor != null && mTouchInterceptor.onTouch(this, ev)) {
        return true;
    }
    return super.dispatchTouchEvent(ev);
}
```

`mTouchInterceptor` 是一个拦截器，我们可以手动给PopupWindow设置拦截器。时间会优先交给拦截器处理，如果没有拦截器或拦截器没有消费事件，那么才会交给viewGroup去进行分发。



## 总结

最后我们对整个流程进行一次回顾：

![整体流程](https://i.loli.net/2021/01/19/MnhFcAOg32rUJuN.png)

1. IMS从系统底层接收到事件之后，会从WMS中获取window信息，并将事件信息发送给对应的viewRootImpl
2. viewRootImpl接收到事件信息，封装成motionEvent对象后，发送给管理的view
3. view会根据自身的类型，对事件进行分发还是自己处理
4. 顶层viewGroup一般是DecorView，DecorView会根据自身callBack的情况，选择调用callBack或者调用父类ViewGroup的方法
5. 而不管顶层viewGroup的类型如何，最终都会到达ViewGroup对事件进行分发。

到这里，虽然触摸事件的“去脉”我们还不清楚，但是他的“来龙”就已经非常清晰了。

本文的主要内容是讲事件的来源，但事件分发的来源远没有这么简单，源码的细节有非常多的内容值得我们去学习，而本文只是把整体的流程抽了出来。而其他关于系统底层的内容，对于有兴趣读者推荐一系列书籍：《深入理解androidⅠⅡⅢ》。

下一篇文章，则是着重分析viewGroup是如何把一个个的事件准确无误地分发给他的子view。

感谢阅读。



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

