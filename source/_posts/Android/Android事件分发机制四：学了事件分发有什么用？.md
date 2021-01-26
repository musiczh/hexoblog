---
title: 	Android事件分发机制四：学了事件分发有什么用？		#标题
date: 2021/1/25 00:00:00 						#建立日期
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

> “ 学了事件分发，影响我CV大法吗？”
> “ 影响我陪女朋友的时间”
> “ ..... ”

## 前言

Android事件分发机制已经来到第四篇了，在前三篇中：

- [Android事件分发机制一：事件是如何到达activity的？](https://juejin.cn/post/6918272111152726024) : 从window机制出发分析了事件分发的整体流程，以及事件分发的真正起点
- [Android事件分发机制二：viewGroup与view对事件的处理](https://juejin.cn/post/6920883974952714247) : 源码分析了viewGroup和view是如何分发事件的
- [Android事件分发机制三：事件分发工作流程](https://juejin.cn/post/6921238915143696392) : 分析了触摸事件在控件树中的分发流程模型

那么关于事件分发的知识在上面三篇文章也就分析地差不多了，接下来就分析一下学了之后该如何使运用到实际开发中，简单阐述一下笔者的思考。

Android中的view一般由两个重要的部分组成：绘制和触摸反馈。如何精准地针对用户的操作给出正确的反馈，是我们学事件分发最重要的目标。

运用事件分发一般有两个场景：给view设置监听器和自定义view。接下来就针对这两方面展开分析，最后再给出笔者的一些思考与总结。



## 监听器

触摸事件监听器可以说是我们接触Android事件体系的第一步。监听器通常有：

- OnClickListener ： 单击事件监听器
- OnLongClickListener ： 长按事件监听器
- OnTouchListener ： 触摸事件监听器

这些是我们使用得最频繁的监听器，他们之间的关系是：

```java
if (mOnTouchListener!=null && mTouchListener.onTouch(event)){
    return true;
}else{
    if (单击事件){
        mOnClickListener.onClick(view);
    }else if(长按事件){
        mOnLongClickListener.onLongClick(view);
    }
}
```

上面的伪代码可以很明显地发现：**onTouchListener是直接把MotionEvent对象直接接管给自己处理且会最先调用，而其他的两个监听器是view判断点击类型之后再分别调用** 。

除此之外，另一个很常见的监听器是双击监听器，但这种监听器并不是view默认支持的，需要我们自己去实现。双击监听器的实现思路可以参考view实现长按监听器的思路来实现：

> 当我们接收到点击事件时，可以发送一个单击延时任务。如果在延迟时间到还没收到另一个点击事件，那么这就是一个单击事件；如果在延迟时间内收到另一个点击事件，说明这是一个双击事件，并取消延时任务。

我们可以实现 `view.OnClickListener` 接口来完成以上逻辑，核心代码如下：

```kotlin
// 实现onClickListener接口
class MyClickListener() : View.OnClickListener{
    private var isClicking = false
    private var singleClickListener : View.OnClickListener? = null
    private var doubleClickListener : View.OnClickListener? = null
    private var delayTime = 1000L
    private var clickCallBack : Runnable? = null
    private var handler : Handler = Handler(Looper.getMainLooper())

    override fun onClick(v: View?) {
        // 创建一个单击延迟任务，延迟时间到了之后触发单击事件
        clickCallBack = clickCallBack?: Runnable {
            singleClickListener?.onClick(v)
            isClicking = false
        }
		// 如果已经点击过一次，在延迟时间内再次接受到点击
        // 意味着这是个双击事件
        if (isClicking){
            // 移除延迟任务，回调双击监听器
            handler.removeCallbacks(clickCallBack!!)
            doubleClickListener?.onClick(v)
            isClicking = false
        }else{
            // 第一次点击，发送延迟任务
            isClicking = true
            handler.postDelayed(clickCallBack!!,delayTime)
        }
    }
...
}
```

代码中实现了创建了一个 `View.OnclickListener` 接口实现类，并在类型实现单击和双击的逻辑判断。我们可以如下使用这个类：

```kotlin
val c = MyClickListener()
// 设置单击监听事件
c.setSingleClickListener(View.OnClickListener {
    Log.d(TAG, "button: 单击事件")
})
// 设置双击监听事件
c.setDoubleClickListener(View.OnClickListener {
    Log.d(TAG, "button: 双击事件")
})
// 把监听器设置给按钮
button.setOnClickListener(c)
```

这样就实现了按钮的双击监听了。

其他类型的监听器如：三击、双击长按等等，都可以基于这种思路来实现监听器接口。



## 自定义view

在自定义view中，我们可以更加灵活地运用事件分发来解决实际的需求。举几个例子：

滑动嵌套问题：外层是viewPager，里层是recyclerView，要实现左右滑动切换viewPager，上下滑动recyclerView。这也就是著名的滑动冲突问题。类似的还有外层viewPager，里层也是可以左右滑动的recyclerView。
实时触摸反馈问题：如设计一个按钮，要让他按下的时候缩小降低高度，放开的时候恢复到原来的大小和高度，而且如果在一个可滑动的容器中，按下之后滑动不会触发点击事件而是把事件交给外层可滑动容器。

我们可以发现，基本上都是基于实际的开发需求来灵活运用事件分发。具体到代码实现，都是围绕着三个关键方法展开： `dispatchTouchEvent` 、 `onIntercepterTouchEvent` 、 `onTouchEvent` 。这三个方法在view和viewGroup中已经有了默认的实现，而我们需要基于默认实现来完成我们的需求。下面看看几种常见的场景如何实现。

#### 实现方块按下缩小

我们先来看看具体的实现效果：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa8881368b054bbf8ffbd6a480d777b9~tplv-k3u1fbpfcp-watermark.image)

方块按下后，会缩小高度变低透明度增加，释放又会恢复。

这个需求可以通过结合属性动画来实现。按钮块本身有高度、有圆角，我们可以考虑继承cardView来实现，重写cardView的dispatchTouchEvent方法，在按下的时候，也就是接收到down事件的时候缩小，在接收到up和cancel事件的时候恢复。**注意，这里可能会忽视cancel事件，导致按钮块的状态无法恢复，一定要加以考虑cancel事件** 。然后来看下代码实现：

```java
public class NewCardView extends CardView {

    //点击事件到来的时候进行判断处理
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        // 获取事件类型
        int actionMarked = ev.getActionMasked();
        // 根据时间类型判断调用哪个方法来展示动画
        switch (actionMarked){
            case MotionEvent.ACTION_DOWN :{
                clickEvent();
                break;
            }
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                upEvent();
                break;
            default: break;
        }
        // 最后回调默认的事件分发方法即可
        return super.dispatchTouchEvent(ev);
    }

    //手指按下的时候触发的事件;大小高度变小，透明度减少
    private void clickEvent(){
        setCardElevation(4);
        AnimatorSet set = new AnimatorSet();
        set.playTogether(
                ObjectAnimator.ofFloat(this,"scaleX",1,0.97f),
                ObjectAnimator.ofFloat(this,"scaleY",1,0.97f),
                ObjectAnimator.ofFloat(this,"alpha",1,0.9f)
        );
        set.setDuration(100).start();
    }

    //手指抬起的时候触发的事件；大小高度恢复，透明度恢复
    private void upEvent(){
        setCardElevation(getCardElevation());
        AnimatorSet set = new AnimatorSet();
        set.playTogether(
                ObjectAnimator.ofFloat(this,"scaleX",0.97f,1),
                ObjectAnimator.ofFloat(this,"scaleY",0.97f,1),
                ObjectAnimator.ofFloat(this,"alpha",0.9f,1)
        );
        set.setDuration(100).start();
    }
}
```

动画方面的内容就不分析了，不属于本文的范畴。可以看到我们只是给cardView设置了动画效果，监听事件我们可以设置给cardView内部的ImageView或者直接设置给CardView。需要注意的是，如果设置给cardView需要重写cardView的 `intercepTouchEvent` 方法永远返回true，防止事件被子view消费而无法触发监听事件。

####  解决滑动冲突

滑动冲突是事件分发运用最频繁的场景，也是一个重难点（敲黑板，考试要考的）。滑动冲突的基本场景有以下三种：

![](https://i.loli.net/2021/01/25/WD978KROTIYVNG4.png)

- 情况一：内外view的滑动方向不同，例如viewPager嵌套ListView
- 情况二：内外view滑动方向相同，例如viewPager嵌套水平滑动的recyclerView
- 情况三：情况一和情况二的组合

解决这类问题一般有两个步骤：确定最终实现效果、代码实现。

滑动冲突的解决需要结合具体的实现需求，而不是一套解决方案可以解决一切的滑动冲突问题，这不现实。因此在解决这类问题时，需要先确定好最终的实现效果，然后再根据这个效果去思考代码实现。这里主要讨论情况一和情况二，情况三类同。

##### 情况一

情况一是内外滑动方向不一致。这种情况的通用解决方案就是：根据手指滑动直线与水平线的角度来判断是左右滑动还是上下滑动：

![](https://i.loli.net/2021/01/25/TY3PeqfyGU9LkIx.png)

如果这个角度小于45度，可以认为是在左右滑动，如果大于45度，则认为是上下滑动。那么现在确定好解决方案，接下来就思考如何代码实现。

滑动角度可以通过两个连续的MotionEvent对象的坐标计算出来，之后我们再根据角度的情况选择把事件交给外部容器还是内部view。这里根据事件处理的位置可分为**内部拦截法和外部拦截法** 。

- 外部拦截法：在viewGroup中判断滑动的角度，如果符合自身滑动方向消费则拦截事件
- 内部拦截法：在内部view中判断滑动的角度，如果是符合自身滑动方向则继续消费事件，否则请求外部viewGroup拦截事件处理

从实现的复杂度来看，外部拦截法会更加优秀，不需要里外view去配合，只需要viewGroup自身做好事件拦截处理即可。两者的区别就在于主动权在谁的手上。如果view需要做更多的判断可以采用内部拦截法，而一般情况下采用外部拦截法会更加简单。

接下来思考一下这两种方法的代码实现。

---

外部拦截法中，重点在于是否拦截事件，那么我们的重心就放在了 `onInterceptTouchEvent` 方法中。在这个方法中计算滑动角度并判断是否要进行拦截。这里以ScrollView为例子(外部是垂直滑动，内部是水平滑动），代码如下：

```java
public class MyScrollView extends ScrollView {
    // 记录上一次事件的坐标
    float lastX = 0;
    float lastY = 0;

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        int actionMasked = ev.getActionMasked();
        // 不能拦截down事件，否则子view永远无法获取到事件
        // 不能拦截up事件，否则子view的点击事件无法被触发
        if (actionMasked == MotionEvent.ACTION_DOWN || actionMasked == MotionEvent.ACTION_UP){
            lastX = ev.getX();
            lastY = ev.getY();
            return false;
        }   

        // 获取斜率并判断
        float x = ev.getX();
        float y = ev.getY();
        return Math.abs(lastX - x) < Math.abs(lastY - y);
    }
}
```

代码的实现思路很简单，记录两次触控点的位置，然后计算出斜率来判断是垂直还是水平滑动。代码中有个需要注意的点：viewGroup不能拦截up事件和down事件。如果拦截了down事件那么子view将永远接收不到事件信息；如果拦截了up事件那么子view将永远无法触发点击事件。

上面的代码是事件分发的核心代码，更加具体的代码还需要根据实际需求去完善细节，但整体的逻辑是不变的。

---

内部拦截法的思路和外部拦截的思路很像，只是判断的位置放到了内部view中。内部拦截法意味着内部view必须要有控制事件流走向的能力，才能对事件进行处理。这里就运用到了内部view一个重要的方法： `requestDisallowInterceptTouchEvent` 。

这个方法可以强制外层viewGroup不拦截事件。因此，我们可以让viewGroup默认拦截除了down事件以外的所有事件。当子view需要处理事件时，只需要调用此方法即可获取事件；而当想要把事件交给viewGroup处理时，那么只需要取消这个标志，外层viewGroup就会拦截所有事件。从而达到内部view控制事件流走向的目的。

代码实现需要分两步走，首先是设置外部viewGroup拦截除了down事件以外的所有事件（这里用viewPager和ListView来进行代码演示）：

```java
public class MyViewPager extends ViewPager {
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.getActionMasked()==MotionEvent.ACTION_DOWN){
            return false;
        }
        return true;
    }
}
```

接下来需要重写内部view的dispatchTouchEvent方法：

```java
public class MyListView extends ListView {
    float lastX = 0;
    float lastY = 0;

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        int actionMarked = ev.getActionMasked();
        switch (actionMarked){
            // down事件，必须请求不拦截，否则拿不到move事件无法进行判断
            case MotionEvent.ACTION_DOWN:{
                requestDisallowInterceptTouchEvent(true);
                break;
            }
            // move事件，进行判断是否处理事件
            case MotionEvent.ACTION_MOVE:{
                float x = ev.getX();
                float y = ev.getY();
                // 如果滑动角度大于90度自己处理事件
                if (Math.abs(lastY-y)<Math.abs(lastX-x)){
                    requestDisallowInterceptTouchEvent(false);
                }
                break;
            }
            default:break;
        }
        // 保存本次触控点的坐标
        lastX = ev.getX();
        lastY = ev.getY();
        // 调用ListView的dispatchTouchEvent方法来处理事件
        return super.dispatchTouchEvent(ev);
    }
}
```

两种方法的代码思路基本一致，但是内部拦截法会更加复杂一点，所以在一般的情况下，还是使用外部拦截法较好。

到这里已经解决了情况一的滑动冲突解决方案，接下来看看情况二的滑动冲突如何解决。

##### 情况二

第二种情况是里外容器的滑动方向是一致的，这种情况的主流解决方法有两种，一种是外容器先滑动，外容器滑动到边界之后再滑动内部view，例如京东app（注意向下滑动时的情况）：

![](https://i.loli.net/2021/01/25/KDFxY2cTuGkZ6wM.gif)

第二种情况的内部view先滑动，等内部view滑动到边界之后再滑动外部viewGroup,例如饿了么app（注意向下滑动时的情况）：

![](https://i.loli.net/2021/01/25/qb9khe2FLcjgK5U.gif)

这两种方案没有孰好孰坏，而是需要根据具体的业务需求来确定具体的解决方案。下面就上述的第二种方案展开分析，第一种方案类同。

首先分析一下具体的效果：外层viewGroup与内层view的滑动方向是一致的，都是垂直滑动或水平滑动；向上滑动时，先滑动viewGroup到顶部，再滑动内部view；向下滑动时，先滑动内部view到顶部后再滑动外层viewGroup。

这里我们采用外部拦截法来实现。首先我们先确定好我们的布局：

![image.png](https://i.loli.net/2021/01/26/pAqZnSs4xXyB2iP.png)

最外层是一个ScrollView，内部首先是一个LinearLayout，因为ScrollView只能有一个view。内部顶部是一个LinearLayout可以放置头部布局，下面是一个ListView。现在需要确定ScrollView的拦截规则：

1. 当ScrollView没有滑动到底部时，直接给ScrollView处理
2. 当ScrollView滑动到底部时：
   - 如果LinearLayout没有滑动到顶部，则交给ListView处理
   - 如果LinearLayout滑动到顶部：
     - 如果是向上滑动则交给listView处理
     - 如果是向下滑动则交给ScrollView处理

接下来就可以确定我们的代码了：

```java
public class MyScrollView extends ScrollView {
	...
    float lastY = 0;
    boolean isScrollToBottom = false;
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        boolean intercept = false;
        int actionMarked = ev.getActionMasked();
        switch (actionMarked){
            case MotionEvent.ACTION_DOWN:
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:{
                // 这三种事件默认不拦截，必须给子view处理
                break;
            }
            case MotionEvent.ACTION_MOVE:{
                LinearLayout layout = (LinearLayout) getChildAt(0);
                ListView listView = (ListView)layout.getChildAt(1);
                // 如果没有滑动到底部，由ScrollView处理，进行拦截
                if (!isScrollToBottom){
                    intercept = true;
                    // 如果滑动到底部且listView还没滑动到顶部，不拦截
                }else if (!ifTop(listView)){
                    intercept = false;
                }else{
                    // 否则判断是否是向下滑
                    intercept = ev.getY() > lastY;
                }
                break;
            }
            default:break;
        }
        // 最后记录位置信息
        lastY = ev.getY();
        // 调用父类的拦截方法，ScrollView需要做一些处理，不然可能会造成无法滑动
        super.onInterceptTouchEvent(ev);
        return intercept;
    }
    ...
}

```

代码中我还增加了如果listView下面有view的情况，判断是否滑动到底部。判断listView滑动情况和scrollView滑动情况的代码如下：

```java
{
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        // 设置滑动监听
        setOnScrollChangeListener((v, scrollX, scrollY, oldScrollX, oldScrollY) -> {
            ViewGroup viewGroup = (ViewGroup)v;
            isScrollToBottom = v.getHeight() + scrollY >= viewGroup.getChildAt(0).getHeight();
        });
    }
}
// 判断listView是否到达顶部
private boolean ifTop(ListView listView){
    if (listView.getFirstVisiblePosition()==0){
        View view = listView.getChildAt(0);
        return view != null && view.getTop() >= 0;
    }
    return false;
}
```

最终的实现效果如下图：

![](https://i.loli.net/2021/01/26/6BsC3AGZb4jQDuq.gif)

这样就简单地解决一个滑动冲突了。但是要注意的是，在实际问题中，往往有更加复杂的细节需要处理。而上述只是把解决滑动冲突的一个思想分析了一下，具体到业务上，还需要去细心打磨代码才行。有兴趣可以去看看NeatedScrollView是如何解决滑动冲突的源码。

## 最后

事件分发作为Android的基础知识储备可谓是非常重要。不能说学了事件分发，就可以直接一飞冲天。而是掌握了事件分发之后，面对一些具体的需求，就有了一定的思路去处理。或者在了解一些框架的源码的时候，懂得他这些代码是什么意思。

学习事件分发的过程中，深入研究了很多的源码，有一些小伙伴觉得没必要。实际开发中也就用到那三个主要的方法，了解一个主要的流程就足够了。我想说：确实是这样；但没有研究背后的原理，就只能知其然而不知其所以然。当遇到一些异常的情况时，就无法从源码的角度去分析结果的bug。学习源码的过程中，也是与设计android系统的作者的一种交流。倘若现在没有事件分发机制，那么我该如何去解决触摸信息的分发问题？学习的过程就是在思考android系统作者给出的解决方案。而掌握原理之后，对于事件分发的问题，稍加思考和分析，也就手到擒来了。正所谓：

> 只有打败10级的敌人，才能掌控9级的敌人。

希望文章对你有帮助。

# 要不留下个小小的点赞鼓励一下作者？



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

