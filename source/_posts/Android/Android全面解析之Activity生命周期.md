---
title: 	Android全面解析之Activity生命周期		#标题
date: 2020/11/8 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - android
 - 生命周期
					
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

很高兴遇见你~ 欢迎阅读我的文章。

关于Activity生命周期的文章，网络上真的很多，有很多的博客也都讲得相当不错，可见Activity的重要性是非常高的。事实上，我猜测每个android开发者接触的第一个android组件都是Activity。我们从新建第一个Activity开始，运行了代码，看到模拟机上显示了一个MainActivity标题和一行HolleWorld，从此打开Android世界的大门。

本篇文章讲解的重点是Activity的生命周期，在文章的最后也会涉及Activity的设计。不同于其他的博客设计，文章采用系统化的讲解，关于Activity生命周期的相关知识基本都会涉及到。

- 文章第一部分讲解关于Activity状态的认知；
- 第二部分全面讲解activity生命周期回调方法；
- 第三部分是分析不同情景下的生命周期回调顺序：
- 第四部分是原码分析；
- 最后一部分是从更高的角度来思考activity以及生命周期。

那么，我们开始吧。

## 生命状态概述

Activity是一个很重要、很复杂的组件，他的启动不像我们平时直接new一个对象就完事了，他需要经历一系列的初始化。例如"刚创建状态"，“后台状态”，“可见状态”等等。当我们在界面之间进行切换的时候，activity也会在多种状态之间进行切换，例如可见或者不可见状态、前台或者后台状态。**当Activity在不同的状态之间切换时，会回调不同的生命周期方法。我们可以重写这一些方法，当进入不同的状态的时候，执行对应的逻辑**。

在ActivityLifecycleItem`抽象类中定义了9种状态。这个抽象类有很多的子类，是AMS管理Activity生命周期的事务类。（其实就像一个圣旨，AMS丢给应用程序，那么应用程序就必须执行这个圣旨）Activity主要使用其中6个（这里的6个是笔者在源码中明确看到调用setState来设置状态，其他的三种并未看到调用setState方法来设置状态，所以这里主要讲这6种），如下：

```java
// Activity刚被创建时
public static final int ON_CREATE = 1;
// 执行完转到前台的最后准备工作
public static final int ON_START = 2;
// 执行完即将与用户交互的最后准备工作
// 此时该activity位于前台
public static final int ON_RESUME = 3;
// 用户离开，activity进入后台
public static final int ON_PAUSE = 4;
// activity不可见
public static final int ON_STOP = 5;
// 执行完被销毁前最后的准备工作
public static final int ON_DESTROY = 6;

```

状态之间的跳转不是随意的，例如不能从`ON_CREATE`直接跳转到`ON_PAUSE`状态，状态之间的跳转收到AMS的管理。当Activity在这些状态之间切换的时候，就会回调对应的生命周期。这里的状态看着很不好理解，笔者画了个图帮助理解一下：

<img src="https://s1.ax1x.com/2020/11/06/BhdKpQ.png" width=700 border="0" />

这里按照「可交互」「可见」「可存在」三个维度来区分Activity的生命状态。可交互则为是否可以与用户操作；可见则为是否显示在屏幕上；可存在，则为该activity是否被系统杀死或者调用了finish方法。箭头的上方为进入对应状态会调用的方法。这里就先不展开讲每个状态之间的切换，主要是让读者可以更好地理解activity的状态与状态切换。

> 注意，这里使用的三个维度并不是非常严谨的，是结合总体的显示规则来进行区分的。
>
> 在谷歌的官方文档中对于`onStart`方法是这样描述的：`onStart()` 调用使 Activity 对用户可见，因为应用会为 Activity 进入前台并支持互动做准备。这也符合我们上面的维度的区分。而当activity进入ON_PAUSE状态的时候，Activity是可能依旧可见的，但是不可交互。如操作另一个应用的悬浮窗口的时候，当前应用的activity会进入ON_PAUSE状态。
>
> 但是！在activity启动的流程中，直到onResume方法被调用，界面依旧是不可见的。这点在后面的源码分析再详细解释。所以这里的状态区分维度，仅仅只是总体上的一种区分，可以这么认为，但细节上并不是非常严谨的。需要读者注意一下。

**生命周期的一个重要作用就是让activity在不同状态之间切换的时候，可以执行对应的逻辑**。举个栗子。我们在界面A使用了相机资源，当我们切换到下个界面B的时候，那么界面A就必须释放相机资源，这样才不会导致界面B无法使用相机；而当我们切回界面A的时候，又希望界面A继续保持拥有相机资源的状态；那么我们就需要在界面不可见的时候释放相机资源，而在界面恢复的时候再次获取相机资源。每个Activity一般情况下可以认为是一个界面或者说，一个屏幕。当我们在界面之间进行导航切换的时候，其实就是在切换Activity。当界面在不同状态之间进行切换的时候，也就是Activity状态的切换，就会回调activity相关的方法。例如当界面不可见的时候会回调`onStop`方法，恢复的时候会回调`onReStart`方法等。

**在合适的生命周期做合适的工作会让app变得更加有鲁棒性**。避免当用户跳转别的app的时候发生崩溃、内存泄露、当用户切回来的时候失去进度、当用户旋转屏幕的时候失去进度或者崩溃等等。这些都需要我们对生命周期有一定的认知，才能在具体的场景下做出最正确的选择。

这一部分概述并没有展开讲生命周期，而是需要重点理解**状态与状态之间的切换，生命周期的回调就发生在不同的状态之间的切换**。我们学习生命周期的一个重要目的就是**能够在对应的业务场景下做合适的工作**，例如资源的申请、释放、存储、恢复，让app更加具有鲁棒性。



## 重要生命周期解析

关于Activity重要的生命周期回调方法谷歌官方有了一张非常重要的流程图，可以说是人人皆知。我在这张图上加上了一些常用的回调方法。这些方法严格上并不算Activity的生命周期，因为并没有涉及到状态的切换，但却在Activity的整个生命历程中发挥了非常大的作用，也是很重要的回调方法。方法很多，我们先看图，再一个个地解释。看具体方法解释的时候一定要结合下面这张图以及上一部分概述的图一起理解。

<img src="https://s1.ax1x.com/2020/11/06/Bh7sbt.png" width=600 border="0" />

#### 主要生命周期

首先我们先看到最重要的七个生命周期，这七个生命周期是严格意义上的生命周期，他符合**状态切换**这个关键定义。这部分内容建议结合概述部分的图一起理解。（onRestart并不涉及状态变换，但因为执行完他之后会马上执行onStart，所以也放在一起讲）

- onCreate：当Activity创建实例完成，并调用attach方法赋值PhoneWindow、ContextImpl等属性之后，调用此方法。该方法在整个Activity生命周期内只会调用一次。调用该方法后Activity进入`ON_CREATE`状态。

  > 该方法是我们使用最频繁的一个回调方法。
  >
  > 我们需要在这个方法中初始化基础组件和视图。如viewmodel，textview。同时必须在该方法中调用setContentView来给activity设置布局。
  >
  > 这个方法接收一个参数，该参数保留之前状态的数据。如果是第一次启动，则该参数为空。该参数来自onSaveInstanceState存储的数据。只有当activity暂时销毁并且预期一定会被重新创建的时候才会被回调，如屏幕旋转、后台应用被销毁等

  

- onStart：当Activity准备进入前台时会调用此方法。调用后Activity会进入`ON_START`状态。

  > 要注意理解这里的前台的意思。虽然谷歌文档中表示调用该方法之后activity可见，如下图：
  >
  > <img src="https://s1.ax1x.com/2020/11/06/Bh0Uy9.png" width=700 border="0" />
  >
  > 但是我们前面讲到，**前台并不意味着Activity可见，只是表示activity处于活跃状态。**这也是谷歌文档里让我比较迷惑的地方之一。（事实上谷歌文档有挺多地方写的缺乏严谨，可能考虑到易懂性，就牺牲了一点严谨性吧）。
  >
  > 前台activity一般只有一个，所以这也意味着**其他的activity都进入后台了**。这里的前后台需要结合activity返回栈来理解，后续笔者再写一篇关于返回栈的。
  >
  > **这个方法一般用于从别的activity切回来本activity的时候调用。**

  

- onResume：当Activity准备与用户交互的时候调用。调用之后Activity进入`ON_RESUME`状态。

  > 注意，这个方法一直被认为是activity一定可见，且准备好与用户交互的状态。但事实并不**一直**是这样。如果在`onReume`方法中弹出popupWindow你会收获一个异常：`token is null`，表示界面尚没有被添加到屏幕上。
  >
  > 但是，这种情况只出现在第一次启动activity的时候。当activity启动后decorview就已经拥有token了，再次在`onReume`方法中弹出popupWindow就不会出现问题了。
  >
  > 因此，**在onResume调用的时候activity是否可见要区分是否是第一次创建activity**。
  >
  > onStart方法是后台与前台的区分，而这个方法是是否可交互的区分。使用场景最多是在当弹出别的activity的窗口时，原activity就会进入`ON_PAUSE`状态，但是仍然可见；当再次回到原activity的时候，就会回调onResume方法了。

  

- onPause：当前activity窗口失去焦点的时候，会调用此方法。调用后activity进入`ON_PAUSE`状态，并进入后台。

  > 这个方法一般在另一个activity要进入前台前被调用。只有当前activity进入后台，其他的activity才能进入前台。所以，**该方法不能做重量级的操作，不然则会引用界面切换卡顿**。
  >
  > 一般的使用场景为界面进入后台时的轻量级资源释放。
  >
  > 最好理解这个状态就是弹出另一个activity的窗口的时候。因为前台activity只能有一个，所以当前可交互的activity变成另一个activity后，原activity就必须调用onPause方法进入`ON_PAUSE`状态；但是！！仍然是可见的，只是无法进行交互。这里也可以更好地体会前台可交互与可见性的区别。

  

- onStop：当activity不可见的时候进行调用。调用后activity进入`ON_STOP`状态。

  > 这里的不可见是严谨意义上的不可见。
  >
  > 当activity不可交互时会回调onPause方法并进入`ON_PAUSE`状态。但如果进入的是另一个全屏的activity而不是小窗口，那么当新的activity界面显示出来的时候，原Activity才会进入`ON_STOP`状态，并回调onStop方法。同时，activity第一创建的时候，界面是在onResume方法之后才显示出来，所以onStop方法会在新activity的onResume方法回调之后再被回调。
  >
  > 注意，被启动的activity并不会等待onStop执行完毕之后再显示。因而如果onStop方法里做一些比较耗时的操作也不会导致被启动的activity启动延迟。
  >
  > onStop方法的目的就是做**资源释放操作**。因为是在另一个activity显示之后再被回调，所以这里可以做一些相对重量级的资源释放操作，如中断网络请求、断开数据库连接、释放相机资源等。
  >
  > 如果一个应用的全部activity都处于`ON_STOP`状态，那么这个应用是很有可能被系统杀死的。而如果一个`ON_STOP`状态的activity被系统回收的话，系统会保留该activity中view的相关信息到bundle中，下一次恢复的时候，可以在onCreate或者onRestoreInstanceState中进行恢复。

  

- onRestart ：当从另一个activity切回到该activity的时候会调用。调用该方法后会立即调用onStart方法，之后activity进入`ON_START`状态。

  > 这个方法一般在activity从`ON_STOP`状态被重新启动的时候会调用。执行该方法后会立即执行onStart方法，然后Activity进入`ON_START`状态，进入前台。

  

- onDestroy：当activity被系统杀死或者调用finish方法之后，会回调该方法。调用该方法之后activity进入`ON_DESTROY`状态。

  > 这个方法是activity在被销毁前回调的最后一个方法。我们需要在这个方法中释放所有的资源，防止造成内存泄漏问题。
  >
  > 回调该方法后的activity就等待被系统回收了。如果再次打开该activity需要从onCreate开始执行，重新创建activity。

那到这里七个最关键的生命周期方法就讲完了。需要读者注意的是，在概述一图中，我们使用了三个维度来进行区分不同类型的状态，但是很明显，同一类型的状态并不是等价的。如`ON_START`状态表示activity进入前台，而`ON_PAUSE`状态却表示activity进入后台。这可能也是为什么谷歌要区分出on_start和on_pause两个状态，他们代表并不是一致的状态。

这七个生命周期回调方法是最重要的七个生命周期回调方法，需要读者好好理解每个回调方法设计到的activity的状态转换。而理解了状态转换后，也就可以写出更加强壮的代码了。

#### 其他生命周期回调方法

- onActivityResult

这个方法也很常见，他需要结合`startActivityForResult`一起使用。

使用的场景是：启动一个activity，并期望在该activity结束的时候返回数据。

当启动的activity结束的时候，返回原activity，原activity就会回调onActivityResult方法了。**该方法执行在其他所有的生命周期方法前**。关于onActivityResult如何使用这里就不展开了，我们主要介绍生命周期。

- onSaveInstanceState/onRestoreInstanceState

这两个方法，主要用于在Activity被意外杀死的情况下进行界面数据存储与恢复。什么叫意外杀死呢？

如果你主动点击返回键、调用finish方法、从多任务列表清除后台应用等等，这些操作表示用户想要完整得退出activity，那么就没有必要保留界面数据了，所以也不会调用这两个方法。而当应用被系统意外杀死，或者系统配置更改导致的activity销毁，这个时候当用户返回activity时，期望界面的数据还在，则会通过回调onSaveInstanceState方法来保存界面数据，而在activity重新创建并运行的时候调用onRestoreInstanceState方法来恢复数据。事实上，onRestoreInstanceState方法的参数和onCreate方法的参数是一致的，只是他们两个方法回调的时机不同。因此，判断是否执行的关键因素就是**用户是否期望返回该activity时界面数据仍然存在**。

这里需要注意几个点：

1. 不同android版本下，onSaveInstanceState方法的调用时机是不同的。目前笔者的源码是api30，在官方注释中可以看到这么一句话：

   ```java
   /*If called, this method will occur after {@link #onStop} for applications
    * targeting platforms starting with {@link android.os.Build.VERSION_CODES#P}.
    * For applications targeting earlier platform versions this method will occur
    * before {@link #onStop} and there are no guarantees about whether it will
    * occur before or after {@link #onPause}.
    */
   ```

   翻译过来意思就是，在api28及以上版本onSaveInstanceState是在onStop之后调用的，但是在低版本中，他是在onStop之前被调用的，且与onPause之间的顺序是不确定的。

2. 当activity进入后台的时候，onSaveInstanceState方法则会被调用，而不是异常情况下才会调用onSaveInstanceState方法，因为并不确定在后台时，activity是否会被系统杀死，所以以最保险的方法，先保存数据。当确实是因为异常情况被杀死时，返回activity用户期望界面需要恢复数据，才会调用onRestoreInstanceState来恢复数据。但是，activity直接按返回键或者调用finish方法直接结束Activity的时候，是不会回调onSaveInstanceState方法，因为非常明确下一次返回该activity用户期望的是一个干净界面的新activity。

3. onSaveInstanceState不能做重量级的数据存储。onSaveInstanceState存储数据的原理是把数据序列化到磁盘中，如果存储的数据过于庞大，会导致界面卡顿，掉帧等情况出现。

4. 正常情况下，每个view都会重写这两个方法，当activity的这两个方法被调用的时候，会向上委托window去调用顶层viewGroup的这两个方法；而viewGroup会递归调用子view的onSaveInstanceState/onRestoreInstanceState方法，这样所有的view状态就都被恢复了。

关于界面数据恢复这里也不展开细讲了，有兴趣的读者可以自行深入研究。

- onPostCreate

这个方法其实和onPostResume是一样的，同样的还有onContextChange方法。这三个方法都是不常用的，这里也点出其中一个来统一讲一下。

onPostCreate方法发生在onRestoreInstanceState之后，onResume之前，他代表着界面数据已经完全恢复，就差显示出来与用户交互了。在onStart方法被调用时这些操作尚未完成。

onPostResume是在Resume方法被完全执行之后的回调。

onContentChange是在setContentView之后的回调。

这些方法都不常用，仅做了解。如果真的遇到了具体的业务需求，也可以拿出来用一下。

- onNewIntent

这个方法涉及到的场景也是重复启动，但是与onRestart方法被调用的场景是不同的。

我们知道activity是有多种启动模式的，其中singleInstance、singleTop、singleTask都保证了在一定情况下的单例状态。如singleTop，如果我们启动一个正处于栈顶且启动模式为singleTop的activity，那么他并不会在创建一个activity实例，而是会回调该activity的onNewIntent方法。该方法接收一个intent参数，该参数就是新的启动Intent实例。

其他的情况如singleTask、singleInstance，当遇到这种强制单例情况时，都会回调onNewIntent方法。关于启动模式这里也不展开，后续笔者可能会再出一期文章讲启动模式。

## 场景生命周期流程

这一部分主要是讲解在一些场景下，生命周期方法的回调顺序。对于当个Activity而言，上述流程图已经展示了各种情况下的生命周期回调顺序了。但是，当启动另一个activity的时候，到底是onStop先执行，还是被启动的onStart先执行呢？这些就变得比较难以确定。

验证生命周期回调顺序最好的方法就是写demo，通过日志打印，可以很明显地观察到生命周期的回调顺序。当然，查看源码也是一个不错的方法，但是需要对系统源码有一定的认识，我们还是选择简单粗暴的方法。

#### 正常启动与结束

> onCreate -> onStart -> onResume -> onPause -> onStop -> onDestroy

这种情况的生命周期比较好理解，就是常规的启动与结束，也不会涉及到第二个activity。最后看日志打印：

<img src="https://s1.ax1x.com/2020/11/08/BI7vnJ.png" width=350 border="0" />

#### Activity切换

> Activity1:onPause
> Activity2:onCreate -> onStart -> onResume
> Activity1:onStop

当切换到另一个activity的时候，本activity会先调用onPause方法，进入后台；被启动的activity依次调用三个回调方法后准备与用户交互；这时原activity再调用onStop方法变得不可见，最后被启动的activity才会显示出来。

理解这个生命周期顺序只需要记住两个点：前后台、是否可见。onPause调用之后，activity会进入后台。而前台交互的activity只能有一个，所以原activity必须先进入后台后，目标activity才能启动并进入前台。onStop调用之后activity变得不可见，因而只有在目标activity即将要与用户交互的时候，需要进行显示了，原Activity才会调用onStop方法进入不可见状态。

当从Activity2回退到Activity1的时候，流程也是类似的，只是Activity1会在其他生命周期之前执行一次onRestart，跟前面的流程是类似的。读者可以看一下下面的日志打印，这里就不再赘述了。

下面看一下切换到另一个activity的生命周期日志打印：

<img src="https://s1.ax1x.com/2020/11/08/BosH81.png" width=400 border="0" />

这里我们看到最后回调了onSaveInstanceState方法，前面我们讲到了，当activity进入后台的时候，会调用该方法来保存数据。因为并不知道在后台时activity是否会被系统杀死。下面再看一下从activity2返回的时候，生命周期的日志打印：

<img src="https://s1.ax1x.com/2020/11/08/BoshbF.png" width=400 border="0" />

#### 屏幕旋转

> running -> onPause -> onStop -> onSaveInstanceState -> onDestroy
>
> onCreate -> onStart -> onRestoreInstanceState -> onResume 

当因资源配置改变时，activity会销毁重建，最常见的就是屏幕旋转。这个时候属于异常情况的Activity生命结束。因而，在销毁的时候，会调用onSaveInstanceState来保存数据，在重新创建新的activity的时候，会调用onRestoreInstanceState来恢复数据。

看一下日志打印：

<img src="https://s1.ax1x.com/2020/11/08/Bog6dU.png" width=400 border="0" />

#### 后台应用被系统杀死

>  onDestroy
>
> onCreate -> onStart -> onRestoreInstanceState -> onResume 

这个流程跟上面的资源配置更改是很像的，只是每个activity不可见的时候，会回调onSaveInstanceState提前保存数据，那么在被后台杀死的时候，就不需要再次保存数据了。

#### 具有返回值的启动

> onActivityResult -> onRestart -> onResume 

这里主要针对使用startActivityForResult方法启动另一个activity，当该activity销毁并返回时，原activity的onActivityResult方法的执行时机。大部分流程和activity切换是一样的。但在返回原Activity时，onActivityResult方法会在其他所有的生命周期方法执行前被执行。看一下日志打印：

<img src="https://s1.ax1x.com/2020/11/08/BoRYCQ.png" width=400 border="0" />

#### 重复启动

> onPause -> onNewIntent -> onResume

这个流程是比较容易在学习生命周期的时候被忽略的。前面已经有讲到了关于onNewIntent的相关介绍，这里就不再赘述。主要是记得如果当前activity正处于栈顶，那么会先回调onPause之后再回调onNewIntent。关于启动模式以及返回栈的设计这里就不展开讲了，记住生命周期就可以了。看一下日志打印：

<img src="https://s1.ax1x.com/2020/11/08/BoRhb6.png" width=300 border="0" />



## 从源码看生命周期

到这里关于生命周期的一些应用知识就已经讲得差不多了，这一部分是深入源码，去探究生命周期在源码中是如何实现的。这样对生命周期会有更加深刻的理解，同时可以更加了解android系统的源码设计。

由于生命周期方法很多，笔者不可能一一讲解，这样篇幅就太大了且没有意义。这一部分的内容一共分为两个部分：第一部分是概述一下ActivityThread中关于每个生命周期的调用方法，这样大家就懂得如何去寻找对应的源码来研究；第二部分是拿onResume这个方法来举例讲解，同时解释为什么在第一次启动时，当onResume被调用时界面依然不可见。

#### 从ActivityThread看生命周期

我们都知道，Activity的启动是受AMS调配的，那具体的调配方式是如何的呢？

通过[Handler机制](https://blog.csdn.net/weixin_43766753/article/details/108968666)一文我们知道，android的程序执行是使用handler机制来实现消息驱动型的。AMS想要控制Activity的生命周期，就必须不断地向主线程发送message；而程序想要执行AMS的命令，就必须handle这些message执行逻辑，两端配合，才能达到这种效率。

打个比方，领导要吩咐下属去工作，他肯定不会把工作的具体流程都给下属，而只是会发个命令，如：给明天的演讲做个ppt，给我预约个下星期的飞机等等。那么下属，就必须根据这些命令来执行指定的逻辑。所以，在android程序，肯定有一系列的逻辑，来分别执行来自AMS的“命令”。这就是ActivityThread中的一系列handlexxx方法。给个我在vs code中的搜索图感受一下：

<img src="https://s1.ax1x.com/2020/11/08/Bo4Yd0.png" width=200 border="0" />

当然，应用程序不止收到AMS的管理，同样的还有WMS、PMS等等系统服务。系统服务是运行在系统服务进程的，当系统服务需要控制应用程序的时候，会通过Binder跨进程通信把消息发送给应用程序。应用程序的Binder线程会通过handler把消息发送给主线程去执行。因而，从这里也可以看出，当应用程序刚被创建的时候，必须初始化的有主线程、binder线程、主线程handler、以及提前编写了命令的执行逻辑的类ActivityThread。光说不画假解释，画个图感受一下：

<img src="https://i.loli.net/2020/11/08/8OU6hRH7cnqDYg5.png" width=400>

回到我们的生命周期主题。关于生命周期命令的执行方法主要有：

```java
handleLaunchActivity;
handleStartActivity;
handleResumeActivity;
handlePauseActivity;
handleStopActivity;
handleDestroyActivity;
```

具体的方法当然不止这么多，只是列出一些比较常用的。这些方法都在ActivityThread中。ActivityThread每个应用程序有且只有一个，他是系统服务“命令”的执行者。

了解了AMS如何调配之后，那么他们的执行顺序如何确定呢？AMS是先发送handleStartActivity命令呢，还是先发送handleResumeActivity？这里就需要我们对Activity的启动流程有一定的认知，感兴趣读者可以点击[Activity启动流程](https://blog.csdn.net/weixin_43766753/article/details/107746968)前往学习，这里就不展开了。

最后再延伸一下，那，ActivityThread可不可以自己决定执行逻辑，而不理会AMS的命令呢？答案肯定是no。你想啊，如果在公司里，你没有老板的同意 ，你能动用公司的资源吗？回到Android系统也是一样的，没有AMS的授权，应用程序是无法得到系统资源的，所以AMS就保证了每一个程序都必须符合一定的规范。关于这方面的设计，读者感兴趣可以阅读[context机制](https://blog.csdn.net/weixin_43766753/article/details/109017196)这篇文章了解一下,重新认识一下context。

好了，扯得有点远，我们回到主题。下面呢就不展开去跟踪整个流程了，而是定位到具体的handle方法去看看具体执行了什么逻辑，对源码流程感性去的读者可以自行研究，限于篇幅这里就不展开了。下面主要介绍`handleResumeActivity`方法。

#### 解析onRsume源码

根据我们前面的学习，`handleResumeActivity`肯定是在`handleLaunchActivity`和`handleStartActivity`之后被执行的。我们直接来看源码：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...;
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        ...
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
    }
    ...
}
```

代码我截取了两个非常重要的部分。`performResumeActivity`最终会执行`onResume`方法；`activity.makeVisible();`是真正让界面显示在屏幕个上的方法，我们看一下`makeVisible()`:

```java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

如果尚未添加到屏幕上，那么会调用windowManager的addView方法来添加，之后，activity界面才真正显示在屏幕上。回应之前的问题：为什么在onResume弹出popupWindow会抛异常而弹出dialog却不会？原因就是这个时候activity的界面尚未添加到屏幕上，而popupWindow需要依附于父界面，这个时候弹出就会抛出`token is null`异常了。而Dialog属于应用层级窗口，不需要依附于任何窗口，所以dialog在onCreate方法中弹出都是没有问题的。为了验证我们的判断，我在生命周期中打印decorView的windowToken，当decorView被添加到屏幕上后，就会被赋值token了，看日志打印：

<img src="https://i.loli.net/2020/11/08/sPzkaO5GApVIhZK.png" width=600>

可以看到，直到onPostResume方法执行，界面依旧没有显示在屏幕上。而直到onWindowFocusChange被执行时，界面才是真正显示在屏幕上了。

好了，让我们再回到一开始的源码，深入performResumeActivity方法中看看，在哪里执行了onResume方法：

```java
public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
        String reason) {
    ...
    try {
        ...
        if (r.pendingIntents != null) {
            // 判断是否需要执行newIntent方法
            deliverNewIntents(r, r.pendingIntents);
            r.pendingIntents = null;
        }
        if (r.pendingResults != null) {
            // 判断是否需要执行onActivityResult方法
            deliverResults(r, r.pendingResults, reason);
            r.pendingResults = null;
        }
        // 回调onResume方法
        r.activity.performResume(r.startsNotResumed, reason);

        r.state = null;
        r.persistentState = null;
        // 设置状态
        r.setState(ON_RESUME);

        reportTopResumedActivityChanged(r, r.isTopResumedActivity, "topWhenResuming");
    } 
    ...
}
```

这个方法的重点就是，先判断是否是需要执行onNewIntent或者onActivityResult的场景，如果没有则执行调用`performResume`方法，我们深入`performResume`看一下：

```java
final void performResume(boolean followedByPause, String reason) {
    dispatchActivityPreResumed();
    performRestart(true /* start */, reason);
	...
    mInstrumentation.callActivityOnResume(this);
    ...
    onPostResume();
   	...
}

public void callActivityOnResume(Activity activity) {
    activity.mResumed = true;
    activity.onResume();
    ...
}
```

同样只看重点。首先会调用performRestart方法，这个方法内部会判断是否需要执行onRestart方法和onStart方法，如果是从别的activity返回这里是肯定要执行的。然后使用Instrumentation来回调Activity的onResume方法。当onResume回调完成后，会再调用onPostResume()方法。

那么到这里关于handleResumeActivity的方法就讲完了，为什么在onResume甚至onPostResume方法被回调的时候界面尚未显示，也有了更加深刻的认识。具体的代码逻辑非常多，而关于生命周期的代码我只挑了重点来讲，其他的源码，感兴趣的读者可以自行去查阅源码。笔者这里更多的是充当一个抛砖引玉的效果。要从源码中学习到知识，就必须自己手动去阅读源码，跟着文章看完事实上收获是不大的。

## 从系统设计看Activity与其生命周期

在笔者认为，每一个知识，都是在具体的场景下为了解决具体的问题，通过权衡各种条件设计出来的。在学习了每一个知识之后，笔者总是喜欢反过来，思考一下这一块知识的底层设计思想是什么，他是需要解决什么问题，权衡了什么条件。通过不断思考来从一个更高的角度来看待每一个知识点。

要理解生命周期的设计，首先需要理解Activity本身。想一下，如果没有Activity，那么我们该如何编写程序？有没有忽然反应到，没有了activity，我们的程序竟无处下手？因为这涉及到Activity的一个最大的作用：**`Activity` 类是 Android 应用的关键组件，而 Activity 的启动和组合方式则是该平台应用模型的基本组成部分**。

相信很多读者都写过c语言、java或者其他语言的课程设计，我们的程序入口就是`main`函数。从main函数开始，根据用户的输入来进入不同的功能模块，如更改信息模块、查阅模块等等。**以功能模块为基本组成部分的应用模型**是我们最初的程序设计模型。而android程序，我们会说这个程序有几个**界面**。我们更关注的是界面之间的跳转，而不是功能模块之间的跳转。我们在设计程序的时候，我们会说这个界面有什么功能，那个界面有什么功能，多个界面之间如何协调。对于用户来说，他们感知的也是一个个独立的界面。当我们通过通讯软件调用邮箱app的发送邮件界面时，我们喜欢看到的只是发送邮件的界面，而不需要看到收件箱、登录注册等界面。以功能模块为应用模型的设计从一个主功能模块入口，然后通过用户的输入去调用不同的功能模块。与其类似，android程序也有一个主界面，通过这个主界面，接受用户的操作去调用其他的界面。组成android程序的，是一个个的界面，而每一个界面，对应一个Activity。因此，**Activity是android平台应用模型的基本组成成分**。

功能模块的应用模型从main方法进入主功能模块，而android程序从ActivityThread的main方法开始，接收AMS的调度启动“LaunchActivity”，也就是我们在AndroidManifest中配置为main的activity，当应用启动的时候，就会首先打开这个activity。那么第一个界面被打开，其他的界面就根据用户的操作来依次跳转了。

那如何做到每个界面之间彼此解耦、各自的显示不发生混乱、界面之间的跳转有条不紊等等？这些工作，官方都帮我们做好了。Activity就是在这个设计思想下开发出来的。当我们在Activity上开发的时候，就已经沿用了他的这种设计思想。当我们开发一个app的时候，最开始要考虑的，是界面如何设计。设计好界面之后，就是考虑如何开发每个界面了。那我们如何自定义好每一个界面？如何根据我们的需求去设计每个界面的功能？Activity并没有main方法，我们的代码该写在哪里被执行？答案就是：**生命周期回调方法**。

到这里，你应该可以理解为什么启动activity并不是一句new就可以解决的吧？Activity承担的责任非常多，需要初始化的逻辑也非常多。当Activity被启动，他会根据自身的启动情况，来回调不同的生命周期方法。其中，**承担初始化整个界面已经各个功能组件的初始化任务**的就是onCreate方法。他有点类似于我们功能模块的入口函数，在这里我们通过setContentView来设计我们界面的布局，通过setOnClickListenner来给每个view设置监听等等。在MVVM设计模式中，还需要初始化viewModel、绑定数据等等。这是生命周期的第一个非常重要的意义所在。

而当界面的显示、退出，我们需要为之申请或者释放资源。如我上文举的相机例子，我在微信的扫一扫申请了相机权限，如果进入后台的时候没有释放资源，那么打开系统相机就无法使用了，资源被占领了。因此，生命周期的另一个重要的作用就是：**做好资源的申请与释放，避免内存泄露**。

其他生命周期的作用，如界面数据恢复、界面切换逻辑处理等等就不再赘述了，前面已经都有涉及到。

这一部分的重点就是理解**android应用程序是以Activity为基本组成部分的应用模型**这个点。当界面的启动以及不同界面之间进行切换的时候，也就可以更加感知生命周期的作用了。

## 最后

关于Activity生命周期的内容，在一篇文章讲完整是不可能的。当对他研究地越深，涉及到内容就会越多。每个知识点就是像是瓜藤架上的一个瓜，如果单纯摘瓜，那就是一个瓜；如果抓着藤蔓往外拔，那么整个架子都会被扯出来。这篇文章也当是抛砖引玉，在讲生命周期相关的知识讲完之后，提供给读者一个思考的思路。

希望文章对你有帮助。



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

