---
title: 	Android事件分发机制二：viewGroup与view对事件的处理		#标题
date: 2021/1/22 00:00:00 						#建立日期
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

在上一篇文章 [Android事件分发机制一：事件是如何到达activity的？](https://juejin.cn/post/6918272111152726024) 中，我们讨论了触摸信息从屏幕产生到发送给具体 的view处理的整体流程，这里先来简单回顾一下：

![整体流程](https://i.loli.net/2021/01/19/MnhFcAOg32rUJuN.png)

1. 触摸信息从手机触摸屏幕时产生，通过IMS和WMS发送到viewRootImpl
2. viewRootImpl把触摸信息传递给他所管理的view
3. view根据自身的逻辑对事件进行分发
4. 常见的如Activity布局的顶层viewGroup为DecorView，他对事件分发方法进行了重新，会优先回调windowCallBack也就是Activity的分发方法
5. 最后事件都会交给viewGroup去分发给子view

前面的分发步骤我们清楚了，那么viewGroup是如何对触摸事件进行分发的呢？View又是如何处理触摸信息的呢？正是本文要讨论的内容。

事件处理中涉及到的关键方法就是 `dispatchTouchEvent` ，不管是viewGroup还是view。在viewGroup中，`dispatchTouchEvent` 方法主要是把事件分发给子view，而在view中，`dispatchTouchEvent` 主要是处理消费事件。而主要的消费事件内容是在 `onTouchEvent` 方法中。下面讨论的是viewGroup与view的默认实现，而在自定义view中，通常会重写 `dispatchTouchEvent` 和 `onTouchEvent` 方法，例如DecorView等。

秉着逻辑先行源码后到的原则，本文虽然涉及到大量的源码，但会优先讲清楚流程，有时间的读者仍然建议阅读完整源码。

## 理解MotionEvent

事件分发中涉及到一个很重要的点：多点触控，这是在很多的文章中没有体现出来的。而要理解viewGroup如何处理多点触控，首先需要对触摸事件信息类：MotionEvent，有一定的认识。MotionEvent中承载了触摸事件的很多信息，理解它更有利于我们理解viewGroup的分发逻辑。所以，首先需要先理解MotionEvent。

触摸事件的基本类型有三种：

- ACTION_DOWN: 表示手指按下屏幕
- ACTION_MOVE: 手指在屏幕上滑动时，会产生一系列的MOVE事件
- ACTION_UP: 手指抬起，离开屏幕

**一个完整的触摸事件系列是：从ACTION_DOWN开始，到ACTION_UP结束** 。这其实很好理解，就是手指按下开始，手指抬起结束。

手指可能会在屏幕上滑动，那么中间会有大量的ACTION_MOVE事件，例如：ACTION_DOWN、ACTION_MOVE、ACTION_MOVE...、ACTION_UP。

这是正常的情况，而如果出现了一些异常的情况，事件序列被中断，那么会产生一个取消事件：

- ACTION_CANCEL：当出现异常情况事件序列被中断，会产生该类型事件

所以，完整的事件序列是：**从ACTION_DOWN开始，到ACTION_UP或者ACTION_CANCEL结束** 。当然，这是我们一个手指的情况，那么在多指操作的情况是怎么样的呢？这里需要引入另外的事件类型：

- ACTION_POINTER_DOWN: 当已经有一个手指按下的情况下，另一个手指按下会产生该事件
- ACTION_POINTER_UP: 多个手指同时按下的情况下，抬起其中一个手指会产生该事件

区别于ACTION_DOWN和ACTION_UP，使用另外两个事件类型来表示手指的按下与抬起，使得**ACTION_DOWN和ACTION_UP可以作为一个完整的事件序列的边界** 。

**同时，一个手指的事件序列，是从ACTION_DOWN/ACTION_POINTER_DOWN开始，到ACTION_UP/ACTION_POINTER_UP/ACTION_CANCEL结束。**

到这里先简单做个小结：

> 触摸事件的类型有：ACTION_DOWN、ACTION_MOVE、ACTION_UP、ACTION_POINTER_DOWN、ACTION_POINTER_UP，他们分别代表不同的场景。
>
> 一个完整的事件序列是从ACTION_DOWN开始，到ACTION_UP或者ACTION_CANCEL结束。
> **一个手指**的完整序列是从ACTION_DOWN/ACTION_POINTER_DOWN开始，到ACTION_UP/ACTION_POINTER_UP/ACTION_CANCEL结束。

---

第二，我们需要理解MotionEvent中所携带的信息。

假如现在屏幕上有两个手指按下，如下图：

![](https://i.loli.net/2021/01/20/b2MueonjGwZWaUg.png)

触摸点a先按下，而触摸点b**后**按下，那么自然而然就会产生两个事件：ACTION_DOWN和ACTION_POINTER_DOWN。那么是不是ACTION_DOWN事件就只包含有触摸点a的信息，而ACTION_POINTER_DOWN只包含触摸点b的信息呢？换句话说，这两个事件是不是会独立发出触摸事件？答案是：不是。

每一个触摸事件中，都包含有所有触控点的信息。例如上述的点b按下时产生的ACTION_POINTER_DOWN事件中，就包含了触摸点a和触摸点b的信息。那么他是如何区分这两个点的信息？我们又是如何知道ACTION_POINTER_DOWN这个事件类型是属于触摸点a还是触摸点b？

在MotionEvent对象内部，维护有一个数组。这个数组中的每一项对应不同的触摸点的信息，如下图：

![image.png](https://i.loli.net/2021/01/20/iqfQn36ZJwtWNV4.png)

数组下标称为触控点的索引，每个节点，拥有一个触控点的完整信息。这里要注意的是，一个触控点的索引并不是一成不变的，而是会随着触控点的数目变化而变化。例如当同时按下两个手指时，数组情况如下图：

![image.png](https://i.loli.net/2021/01/20/Iwkaytf6Yu8gp2V.png)

而当手指a抬起后，数组的情况变为下图：

![image.png](https://i.loli.net/2021/01/20/oWHX71JY932GgCu.png)

可以看到触控点b的索引改变了。所以**跟踪一个触控点必须是依靠一个触控点的id，而不是他的索引** 。

现在我们知道每一个MotionEvent内部都维护有所有触控点的信息，那么我们怎么知道这个事件是对应哪个触控点呢？这就需要看到MotionEvent的一个方法：`getAction` 。

这个方法返回一个整型变量，他的低1-8位表示该事件的类型，高9-16位表示触控点索引。我们只需要将这16位进行分离，就可以知道触控点的类型和所对应的触控点。同时，MotionEvent有两个获取触控点坐标的方法：`getX()/getY()` ，他们都需要传入一个触控点索引来表示获取哪个触控点的坐标信息。

同时还要注意的是，MOVE事件和CANCEL事件是没有包含触控点索引的，只有DOWN类型和UP类型的事件才包含触控点索引。这里是因为非DOWN/UP事件，不涉及到触控点的增加与删除。

这里我们再来小结一下：

> - 一个MotionEvent对象内部使用一个数组来维护所有触控点的信息
> - UP/DOWN类型的事件包含了触控点索引，可以根据该索引做出对应的操作
> - 触控点的索引是变化的，不能作为跟踪的依据，而必须依据触控点id

----

关于MotionEvent需要了解一个更加重要的点：事件分离。

首先需要知道事件分发的一个原则：**一个view消费了某一个触点的down事件后，该触点事件序列的后续事件，都由该view消费** 。这也比较符合我们的操作习惯。当我们按下一个控件后，只要我们的手指一直没有离开屏幕，那么我们希望这个手指滑动的信息都交给这个view来处理。换句话说，一个触控点的事件序列，只能给一个view消费。

经过前面的描述我们知道，一个事件是包含所有触摸点的信息的。当viewGroup在派发事件时，每个触摸点的信息就需要分开分别发送给感兴趣的view，这就是事件分离。

例如Button1接收了触摸点a的down事件，Button2接收了触摸点b的down事件，那么当一个MotionEvent对象到来时，需要将他里面的触摸点信息，把触摸点a的信息拆开发送给button1，把触摸点b的信息拆开发送给button2。如下图：

![事件分离](https://i.loli.net/2021/01/20/fxvCXwZhFcusr6N.png)

那么，可不可以不进行分离？当然可以。这样的话每次都把所有触控点的信息发送给子view。这可以通过FLAG_SPLIT_MOTION_EVENTS这个标志进行设置是否要进行分离。

小结一下：

> 一个触控点的序列一般情况下只给一个view处理，当一个view消费了一个触控点的down事件后，该触控点的事件序列后续事件都会交给他处理。
>
> 事件分离是把一个motionEvent中的触控点信息进行分离，只向子view发送其感兴趣的触控点信息。
>
> 我们可以通过设置FLAG_SPLIT_MOTION_EVENTS标志让viewGroup是否对事件进行分离

---

到这里关于MotionEvent的内容就讲得差不多，当然在分离的时候，还需要进行一定的调整，例如坐标轴的更改、事件类型的更改等等，放在后面讲，接下来看看ViewGroup是如何分发事件的。



## ViewGroup对于事件的分发

这一步可以说是事件分发中的重头戏了。不过在理解了上面的MotionEvent之后，对于ViewGroup的分发细节也就容易理解了。

整体来说，ViewGroup分发事件分为三个大部分，后面的内容也会围绕着三大部分展开：

1. 拦截事件：在一定情况下，viewGroup有权利选择拦截事件或者交给子view处理
2. 寻找接收事件序列的控件：每一个需要分发给子view的down事件都会先寻找是否有适合的子view，让子view来消费整个事件序列
3. 派发事件：把事件分发到感兴趣的子view中或自己处理

大体的流程是：每一个事件viewGroup会先判断是否要拦截，如果是down事件（这里的down事件表示ACTION_DOWN和ACTION_POINTER_DOWN，下同），还需要挨个遍历子view看看是否有子view消费了down事件，最后再把事件派发下去。

在开始解析之前，必须先了解一个关键对象：TouchTarget。

#### TouchTarget

前面我们讲到：一个触控点的序列一般情况下只给一个view处理，当一个view消费了一个触控点的down事件后，该触控点的事件序列后续事件都会交给他处理。对于viewGroup来说，他有很多个子view，如果不同的子view接受了不同的触控点的down事件，那么ViewGroup如何记录这些信息并精准把事件发送给对应的子view呢？答案就是：TouchTarget。

TouchTarget中维护了每个子view以及所对应的触控点id，这里的id可以不止一个。TouchTarget本身是个链表，每个节点记录了子view所对应的触控点id。在viewGroup中，该链表的链表头是mFirstTouchTarget，如果他为null，表示没有任何子view接收了down事件。

TouchTarget有个非常神奇的设计，他只使用一个整型变量来记录所有的触控id。整型变量中哪一个二进制位为1，则对应绑定该id的触控点。

例如 00000000 00000000 00000000 10001000，则表示绑定了id为3和id为7的两个触控点，因为第3位和第7位的二进制位是1。这里可以间接说明系统支持的最大多点触控数是32，当然实际上一般是8比较多。当要判断一个TouchTarget绑定了哪些id时，只需要通过一定的位操作即可，既提高了速度，也优化了空间占用。

当一个down事件来临时，viewGroup会为这个down事件寻找适合的子view，并为他们创建一个TouchTarget加入到链表中。而当一个up事件来临时，viewGroup会把对应的TouchTarget节点信息删除。那接下来，就直接看到viewGroup中的`dispatchTouchEvent` 是如何分发事件的。首先看到源码中的第一部分：事件拦截。

---

#### 事件拦截

这里的拦截分为两部分：安全拦截和逻辑拦截。

安全拦截是一直被忽略的一种情况。当一个控件a被另一个非全屏控件b遮挡住的时候，那么有可能被恶意软件操作发生危险。例如我们看到的界面是这样的：

![](https://i.loli.net/2021/01/20/TRQMUhovYAIJLuF.png)

但实际上，我们看到的这个按钮时不可点击的，实际上触摸事件会被分发到这个按钮后面的真正接收事件的按钮：

![](https://i.loli.net/2021/01/20/2q1hH3wMgkGUcrb.png)

然后我们就白给了。这个安全拦截行为由两个标志控制：

- FILTER_TOUCHES_WHEN_OBSCURED：这个标志可以手动给控件设置，表示被非全屏控件覆盖时，直接过滤掉所有触摸事件。
- FLAG_WINDOW_IS_OBSCURED：这个标志表示当前窗口被一个非全屏控件覆盖。

具体的源码如下:

```java
View.java api29
public boolean onFilterTouchEventForSecurity(MotionEvent event) {
    // 两个标志，前者表示当被覆盖时不处理；后者表示当前窗口是否被非全屏窗口覆盖
    if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
            && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
        // Window is obscured, drop this touch.
        return false;
    }
    return true;
}
```

第二种拦截是逻辑拦截。如果当前viewGroup中没有TouchTarget，而且这个事件不是down事件，这就意味着viewGroup自己消费了先前的down事件，那么这个事件就无须分发到子view必须自己消费，也就不需要拦截这种情况的事件。除此之外的事件都是需要分发到子view，那么viewGroup就可以对他们进行判断是否进行拦截。简单来说，**只有需要分发到子view的事件才需要拦截** 。

判断是否拦截主要依靠两个因素：FLAG_DISALLOW_INTERCEPT标志和 `onInterceptTouchEvent()` 方法。

1. 子view可以通过requestDisallowInterupt方法强制要求viewGroup不要拦截事件，viewGroup中会设置一个FLAG_DISALLOW_INTERCEPT标志表示不拦截事件。但是当前事件序列结束后，这个标志会被清除。如果需要的话需要再次调用requestDisallowInterupt方法进行设置。
2. 如果子view没有强制要求不拦截，那么会调用`onInterceptTouchEvent()` 方法判断是否需要拦截。onInterceptTouchEvent方法默认只对一种特殊情况作了拦截。一般情况下我们会重写这个方法来拦截事件：

```java
// 只对一种特殊情况做了拦截
// 鼠标左键点击了滑动块
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
            && ev.getAction() == MotionEvent.ACTION_DOWN
            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
            && isOnScrollbarThumb(ev.getX(), ev.getY())) {
        return true;
    }
    return false;
}
```



viewGroup的 `dispatchTouchEvent` 方法逻辑中对于事件拦截部分的源码分析如下：

```java
ViewGroup.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
        
    // 对遮盖状态进行过滤
    if (onFilterTouchEventForSecurity(ev)) {
        
        ...

        // 判断是否需要拦截
        final boolean intercepted;
        // down事件或者有target的非down事件则需要判断是否需要拦截
        // 否则不需要进行拦截判断，因为一定是交给自己处理
        if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
            // 此标志为子view通过requestDisallowInterupt方法设置
            // 禁止viewGroup拦截事件
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                // 调用onInterceptTouchEvent判断是否需要拦截
                intercepted = onInterceptTouchEvent(ev);
                // 恢复事件状态
                ev.setAction(action); 
            } else {
                intercepted = false;
            }
        } else {
            // 自己消费了down事件，那么后续的事件非down事件都是自己处理
            intercepted = true;
        }
        ...;
    }
    ...;
}
```



---

#### 寻找消费down事件的子控件

对于每一个down事件，不管是ACTION_DOWN还是ACTION_POINTER_DOWN，viewGroup都会优先在控件树中寻找合适的子控件来消费他。因为对于每一个down事件，标志着一个触控点的一个崭新的事件序列，viewGroup会尽自己的最大能力寻找合适的子控件。如果找不到合适的子控件，才会自己处理down事件。因为，消费了down事件，意味着接下来该触控点的事件序列事件都会交给该view消费，如果viewGroup拦截了事件，那么子view就无法接收到任何事件消息。

viewGroup寻找子控件的步骤也不复杂。首先viewGroup会为他的子控件构造一个控件列表，构造的顺序是view的绘制顺序的逆序，也就是一个view的z轴系数越高，显示高度越高，在列表的顺序就会越靠前。这其实比较好理解，显示越高的控件肯定是优先接收点击的。除了默认情况，我们也可以进行自定义列表顺序，这里就不展开了。

viewGroup会按顺序遍历整个列表，判断触控点的位置是否在该view的范围内、该view是否可以点击等，寻找合适的子view。如果找到合适的子view，则会把down事件分发给他，如果该view接收事件，则会为他创建一个TouchTarget，将该触控id和view进行绑定，之后该触控点的事件就可以直接分发给他了。

而如果没有一个控件适合，那么会默认选取TouchTarget链表的最新一个节点。也就是当我们多点触控时，两次手指按下，如果没有找到合适的子view，那么就被认为是和上一个手指点击的是同个view。因此，如果viewGroup当前有正在消费事件的子控件，那么viewGroup自己是不会消费down事件的。

接下来我们看看源码分析(代码有点长，需要慢慢分析理解）：

```java
ViewGroup.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
         
    // 对遮盖状态进行过滤
    if (onFilterTouchEventForSecurity(ev)) {
        
        // action的高9-16位表示索引值
        // 低1-8位表示事件类型
        // 只有down或者up事件才有索引值
        final int action = ev.getAction();
        // 获取到真正的事件类型
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        ...

        // 拦截内容的逻辑
        if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
            ...
        } 

        ...

        // 三个变量：
        // split表示是否需要对事件进行分裂，对应多点触摸事件
        // newTouchTarget 如果是down或pointer_down事件的新的绑定target
        // alreadyDispatchedToNewTouchTarget 表示事件是否已经分发给targetview了
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        
        // 如果没有取消和拦截进入分发
        if (!canceled && !intercepted) {
			...
			// down或pointer_down事件，表示新的手指按下了，需要寻找接收事件的view
            if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                
                // 多点触控会有不同的索引，获取索引号
                // 该索引位于MotionEvent中的一个数组，索引值就是数组下标值
                // 只有up或down事件才会携带索引值
                final int actionIndex = ev.getActionIndex(); 
                // 这个整型变量记录了TouchTarget中view所对应的触控点id
                // 触控点id的范围是0-31，整型变量中哪一个二进制位为1，则对应绑定该id的触控点
                // 例如 00000000 00000000 00000000 10001000
                // 则表示绑定了id为3和id为7的两个触控点
                // 这里根据是否需要分离，对触控点id进行记录，
                // 而如果不需要分离，则默认接收所有触控点的事件
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                    : TouchTarget.ALL_POINTER_IDS;

                // down事件表示该触控点事件序列是一个新的序列
                // 清除之前绑定到到该触控id的TouchTarget
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                // 如果子控件数目不为0而且还没绑定到新的id
                if (newTouchTarget == null && childrenCount != 0) {
                    // 使用触控点索引获取触控点位置
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // 从前到后创建view列表
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    // 判断是否是自定义view顺序
                    final boolean customOrder = preorderedList == null
                        && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    
                    // 遍历所有子控件
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // 从子控件列表中获取到子控件
                        final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);
                        
                        ...

                        // 检查该子view是否可以接受触摸事件和是否在点击的范围内
                        if (!child.canReceivePointerEvents()
                            || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        // 检查该子view是否在touchTarget链表中
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // 链表中已经存在该子view，说明这是一个多点触摸事件
                            // 即两次都触摸到同一个view上
                            // 将新的触控点id绑定到该TouchTarget上
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        // 找到合适的子view，把事件分发给他，看该子view是否消费了down事件
                        // 如果消费了，需要生成新的TouchTarget
                        // 如果没有消费，说明子view不接受该down事件，继续循环寻找合适的子控件
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // 保存该触控事件的相关信息
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 保存该view到target链表
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            // 标记已经分发给子view，退出循环
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        ...
                    }// 这里对应for (int i = childrenCount - 1; i >= 0; i--)
                    ...
                }// 这里对应判断：(newTouchTarget == null && childrenCount != 0)

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // 没有子view接收down事件，直接选择链表尾的view作为target
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
                
            }// 这里对应if (actionMasked == MotionEvent.ACTION_DOWN...)
        }// 这里对应if (!canceled && !intercepted)
        ...
    }// 这里对应if (onFilterTouchEventForSecurity(ev))
    ...
}
```



#### 派发事件

经过了拦截与寻找消费down事件的控件之后，无论前面的处理结果如何，最终都是需要将事件进行派发，不管是派发给自己还是子控件。这里派发的对象只有两个：viewGroup自身或TouchTarget。

经过了前面的寻找消费down事件子控件步骤，那么每个触控点都找到了消费自己事件序列的控件并绑定在了TouchTarget中；而如果没有找到合适的子控件，那么消费的对象就是viewGroup自己。因此派发事件的主要任务就是：**把不同触控点的信息分发给合适的viewGroup或touchTarget。** 

派发的逻辑需要结合前面MotionEvent和TouchTarget的内容。我们知道MotionEvent包含了当前屏幕所有触控点信息，而viewGroup的每个TouchTarget则包含了不同的view所感兴趣的触控点。
如果不需要进行事件分离，那么直接将当前的所有触控点的信息都发送给每个TouchTarget即可；
如果需要进行事件分离，那么会将MotionEvent中不同触控点的信息拆开分别创建新的MotionEvent，并发送给感兴趣的子控件；
如果TouchTarget链表为空，那么直接分发给viewGroup自己；所以touchTarget不为空的情况下，viewGroup自己是不会消费事件的，这也就意味着viewGroup和其中的view不会同时消费事件。

![事件分离派发事件](https://i.loli.net/2021/01/21/6Myj1AaF2XeRVq9.png)

上图展示了需要事件分离的情况下进行的事件分发。

在把原MotionEvent拆分成多个MotionEvent时，不仅需要把不同的触控点信息进行分离，还需要对坐标进行转换和改变事件类型：

- 我们接收到的触控点的位置信息并不是基于屏幕坐标系，而是基于当前view的坐标系。所以当viewGroup往子view分发事件时，需要把触控点的信息转换成对应view的坐标系。
- viewGroup收到的事件类型和子view收到的事件类型并不是完全一致的，在分发给子view的时候，viewGroup需要对事件类型进行修改，一般有以下情况需要修改：
  1.  viewGroup收到一个ACTION_POINTER_DOWN事件分发给一个子view，但是该子view前面没有收到其他的down事件，所以对于该view来说这是一个崭新的事件序列，所以需要把这个ACTION_POINTER_DOWN事件类型改为ACTION_DOWN再发送给子view。
  2. viewGroup收到一个ACTION_POINTER_DOWN或ACTION_POINTER_UP事件，假设这个事件类型对应触控点2，但是有一个子view他只对触控点1的事件序列感兴趣，那么在分离出触控点1的信息之后，还需要把事件类型改为ACTION_MOVE再分发给该子view。
- 注意，把原MotionEvent对象拆分为多个MotionEvent对象之后，触控点的索引也发生了改变，如果需要分发一个ACTION_POINTER_DOWN/UP事件给子view，那么需要注意更新触控点的索引值。

viewGroup中真正执行事件派发的关键方法是 `dispatchTransformedTouchEvent` ，该方法会完成关键的事件分发逻辑。源码分析如下：

```java
ViewGroup.java api29
// 该方法接收原MotionEvent事件、是否进行取消、目标子view、以及目标子view感兴趣的触控id
// 如果不是取消事件这个方法会把原MotionEvent中的触控点信息拆分出目标view感兴趣的触控点信息
// 如果是取消事件则不需要拆分直接发送取消事件即可
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // 如果是取消事件，那么不需要做其他额外的操作，直接派发事件即可，然后直接返回
    // 因为对于取消事件最重要的内容就是事件本身，无需对事件的内容进行设置
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    // oldPointerIdBits表示现在所有的触控id
    // desirePointerIdBits来自于该view所在的touchTarget，表示该view感兴趣的触控点id
    // 因为desirePointerIdBits有可能全是1，所以需要和oldPointerIdBits进行位与
    // 得到真正可接收的触控点信息
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // 控件处于不一致的状态。正在接受事件序列却没有一个触控点id符合
    if (newPointerIdBits == 0) {
        return false;
    }

    // 来自原始MotionEvent的新的MotionEvent，只包含目标感兴趣的触控点
    // 最终派发的是这个MotionEvent
    final MotionEvent transformedEvent;
    
    // 两者相等，表示该view接受所有的触控点的事件
    // 这个时候transformedEvent相当于原始MotionEvent的复制
    if (newPointerIdBits == oldPointerIdBits) {
        // 当目标控件不存在通过setScaleX()等方法进行的变换时，
        // 为了效率会将原始事件简单地进行控件位置与滚动量变换之后
        // 发送给目标的dispatchTouchEvent()方法并返回。
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);

                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        // 复制原始MotionEvent
        transformedEvent = MotionEvent.obtain(event);
    } else {
        // 如果两者不等，说明需要对事件进行拆分
        // 只生成目标感兴趣的触控点的信息
        // 这里分离事件包括了修改事件的类型、触控点索引等
        transformedEvent = event.split(newPointerIdBits);
    }

    // 对MotionEvent的坐标系，转换为目标控件的坐标系并进行分发
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        // 计算滚动量偏移
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        // 存在scale等变换，需要进行矩阵转换
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
		// 调用子view的方法进行分发
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // 分发完毕，回收MotionEvent
    transformedEvent.recycle();
    return handled;
}
```

好了，了解完上面的内容，来看看viewGroup的 `dispatchTouchEvent` 中派发事件的代码部分：

```java
ViewGroup.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
        
    // 对遮盖状态进行过滤
    if (onFilterTouchEventForSecurity(ev)) {
		...

		
        if (mFirstTouchTarget == null) {
            // 经过了前面的处理，到这里touchTarget依旧为null，说明没有找到处理down事件的子控件
            // 或者down事件被viewGroup本身消费了，所以该事件由viewGroup自己处理
            // 这里调用了dispatchTransformedTouchEvent方法来分发事件
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 已经有子view消费了down事件
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            // 遍历所有的TouchTarget并把事件分发下去
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    // 表示事件在前面已经处理了，不需要重复处理
                    handled = true;
                } else {
                    // 正常分发事件或者分发取消事件
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                    // 这里调用了dispatchTransformedTouchEvent方法来分发事件
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                                                      target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    // 如果发送了取消事件，则移除该target
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // 如果接收到取消获取up事件，说明事件序列结束
        // 直接删除所有的TouchTarget
        if (canceled
            || actionMasked == MotionEvent.ACTION_UP
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            // 清除记录的信息
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            // 如果仅仅只是一个PONITER_UP
            // 清除对应触控点的触摸信息
            removePointersFromTouchTargets(idBitsToRemove);
        }
        
    }// 这里对应if (onFilterTouchEventForSecurity(ev))

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```

#### 小结

到这里，viewGroup的事件分发源码就解析完成了，这里再来小结一下：

- 每一个触控点的事件序列，只能给一个view消费；如果一个view消费了一个触控点的down事件，那么该触控点的后续事件都会给他处理。
- 每一个事件到达viewGroup，如果需要分发到子view，那么viewGroup会新判断是否要拦截。
  - 当viewGroup的touchTarget!=null || 事件的类型为down 需要进行判断是否拦截；
  - 判断是否拦截受两个因素影响：onInterceptTouchEvent和FLAG_DISALLOW_INTERCEPT标志
- 如果该事件是down类型，那么需要遍历所有的子控件判断是否有子控件消费该down事件
  - 当有新的down事件被消费时，viewGroup会把该view和对应的触控点id绑定起来存储到touchTarget中
- 根据前面的处理情况，将事件派发到viewGroup自身或touchTarget中
  - 如果touchTarget==null，说明没有子控件消费了down事件，那么viewGroup自己处理事件
  - 否则将事件分离成多个MotionEvent，每个MotionEvent只包含对应view感兴趣的触控点的信息，并派发给对应的子view

viewGroup中的源码很多，但大体的逻辑也就这三大部分。理解好MotionEvent和TouchTarget的设计，那么理解viewGroup的事件分发源码也是手到擒来。上面的源码我省略了一些细节内容，下面附上完整的viewGroup分发代码。

```java
ViewGroup.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    // 一致性检验器，用于调试用途
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }
        
    // 辅助功能，用于辅助有障碍人群使用;
    // 如果这个事件是辅助功能事件，那么他会带有一个target view，要求事件必须分发给该view
    // 如果setTargetAccessibilityFocus(false)，表示取消辅助功能事件，按照常规的事件分发进行
    // 这里表示如果当前是目标target view，则取消标志，直接按照普通分发即可
    // 后面还有很多类似的代码，都是同样的道理
    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }   

    boolean handled = false;
    // 对遮盖状态进行过滤
    if (onFilterTouchEventForSecurity(ev)) {
        
        // action的高9-16位表示索引值
        // 低1-8位表示事件类型
        // 只有down或者up事件才有索引值
        final int action = ev.getAction();
        // 获取到真正的事件类型
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // ACTION_DOWN事件，表示这是一个全新的事件序列，会清除所有的touchTarget，重置所有状态
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // 判断是否需要拦截
        final boolean intercepted;
        // down事件或者有target的非down事件则需要判断是否需要拦截
        // 否则直接拦截自己处理
        if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
            // 此标志为子view通过requestDisallowInterupt方法设置
            // 禁止viewGroup拦截事件
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                // 调用onInterceptTouchEvent判断是否需要拦截
                intercepted = onInterceptTouchEvent(ev);
                // 恢复事件状态
                ev.setAction(action); 
            } else {
                intercepted = false;
            }
        } else {
            // 自己消费了down事件
            intercepted = true;
        }

        // 如果已经被拦截、或者已经有了目标view，取消辅助功能的target标志
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // 判断是否需要取消
        // 这里有很多种情况需要发送取消事件
        // 最常见的是viewGroup拦截了子view的ACTION_MOVE事件，导致事件序列中断
        // 那么需要发送cancel事件告知该view，让该view做一些状态恢复工作
        final boolean canceled = resetCancelNextUpFlag(this)
            || actionMasked == MotionEvent.ACTION_CANCEL;

        // 三个变量：
        // 是否需要对事件进行分裂，对应多点触摸事件
        // newTouchTarget 如果是down或pointer_down事件的新的绑定target
        // alreadyDispatchedToNewTouchTarget 是否已经分发给target view了
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        
        // 下面部分的代码是寻找消费down事件的子控件
        // 如果没有取消和拦截进入分发
        if (!canceled && !intercepted) {
			// 如果是辅助功能事件，我们会寻找他的target view来接收这个事件
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;
            
			// down或pointer_down事件，表示新的手指按下了，需要寻找接收事件的view
            if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                
                // 多点触控会有不同的索引，获取索引号
                // 该索引位于MotionEvent中的一个数组，索引值就是数组下标值
                // 只有up或down事件才会携带索引值
                final int actionIndex = ev.getActionIndex(); 
                
                // 这个整型变量记录了TouchTarget中view所对应的触控点id
                // 触控点id的范围是0-31，整型变量中哪一个二进制位为1，则对应绑定该id的触控点
                // 例如 00000000 00000000 00000000 10001000
                // 则表示绑定了id为3和id为7的两个触控点
                // 这里根据是否需要分离，对触控点id进行记录，
                // 而如果不需要分离，则默认接收所有触控点的事件
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                    : TouchTarget.ALL_POINTER_IDS;

                // 清除之前获取到该触控id的TouchTarget
                removePointersFromTouchTargets(idBitsToAssign);

                // 如果子控件的数量等于0，那么不需要进行遍历只能给viewGroup自己处理
                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    // 使用触控点索引获取触控点位置
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // 从前到后创建view列表
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    // 这一句判断是否是自定义view顺序
                    final boolean customOrder = preorderedList == null
                        && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    
                     // 遍历所有子控件
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // 获得真正的索引和子view
                        final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);

                        // 如果是辅助功能事件，则优先给对应的target先处理
                        // 如果该view不处理，再交给其他的view处理
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }

                        // 检查该子view是否可以接受触摸事件和是否在点击的范围内
                        if (!child.canReceivePointerEvents()
                            || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        // 检查该子view是否在touchTarget链表中
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // 链表中已经存在该子view，说明这是一个多点触摸事件
                            // 将新的触控点id绑定到该TouchTarget上
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }
						
                        // 设置取消标志
                        // 下一次再次调用这个方法就会返回true
                        resetCancelNextUpFlag(child);
                        
                        // 找到合适的子view，把事件分发给他，看该子view是否消费了down事件
                        // 如果消费了，需要生成新的TouchTarget
                        // 如果没有消费，说明子view不接受该down事件，继续循环寻找合适的子控件
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // 保存信息
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 保存该view到target链表
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            // 标记已经分发给子view，退出循环
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        // 辅助功能事件对应的targetView没有消费该事件，则继续分发给普通view
                        ev.setTargetAccessibilityFocus(false);
                        
                    }// 这里对应for (int i = childrenCount - 1; i >= 0; i--)
                    
                    if (preorderedList != null) preorderedList.clear();
                    
                }// 这里对应判断：(newTouchTarget == null && childrenCount != 0)

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // 没有子view接收down事件，直接选择链表尾的view作为target
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }// 这里对应if (actionMasked == MotionEvent.ACTION_DOWN...)
        }// 这里对应if (!canceled && !intercepted)

        if (mFirstTouchTarget == null) {
            // 经过了前面的处理，到这里touchTarget依旧为null，说明没有找到处理down事件的子控件
            // 或者down事件被viewGroup本身消费了，所以该事件由viewGroup自己处理
            // 这里调用了dispatchTransformedTouchEvent方法来分发事件
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 已经有子view消费了down事件
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            // 遍历所有的TouchTarget并把事件分发下去
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    // 表示事件在前面已经处理了，不需要重复处理
                    handled = true;
                } else {
                    // 正常分发事件或者分发取消事件
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                    // 这里调用了dispatchTransformedTouchEvent方法来分发事件
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                                                      target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    // 如果发送了取消事件，则移除该target
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // 如果接收到取消获取up事件，说明事件序列结束
        // 直接删除所有的TouchTarget
        if (canceled
            || actionMasked == MotionEvent.ACTION_UP
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            // 清除记录的信息
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            // 如果仅仅只是一个PONITER_UP
            // 清除对应触控点的触摸信息
            removePointersFromTouchTargets(idBitsToRemove);
        }
        
    }// 这里对应if (onFilterTouchEventForSecurity(ev))

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```



## View对于事件的分发

不管是viewGroup自己处理事件，还是view处理事件，如果没有被子类拦截（子类重写方法），最终都会调用到 `view.dispatchTouchEvent` 方法来处理事件。view处理事件的逻辑就比viewGroup简单多了，因为它不需要向下去分发事件，只需要自己处理。整体的逻辑如下：

1. 首先判断是否被其他非全屏view覆盖。这和上面viewGroup的安全性检查是一样的
2. 经过检查之后先检查是否有onTouchListener监听器，如果有则调用它
3. 如果第2步没有消费事件，那么会调用onTouchEvent方法来处理事件
   - 这个方法是view处理事件的核心，里面包含了点击、双击、长按等逻辑的处理需要重点关注。

我们先看到 `view.dispatchTouchEvent` 方法源码：

```java
View.java api29
public boolean dispatchTouchEvent(MotionEvent event) {
    // 首先处理辅助功能事件
    if (event.isTargetAccessibilityFocus()) {
        // 本控件没有获取到焦点，不处理事件
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // 获取到焦点，按照常规处理事件
        event.setTargetAccessibilityFocus(false);
    }

    // 表示是否消费事件
    boolean result = false;

    // 一致性检验器，检验事件是否一致
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    } 

    // 如果是down事件，停止嵌套滑动
    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        stopNestedScroll();
    }

    // 安全过滤，本窗口位于非全屏窗口之下时，可能会阻止控件处理触摸事件
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            // 如果事件为鼠标拖动滚动条
            result = true;
        }
        // 先调用onTouchListener监听器
        // 当我们设置onTouchEventListener之后，L
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        // 若onTouchListener没有消费事件，调用onTouchEvent方法
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    // 一致性检验
    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // 如果是事件序列终止事件或者没有消费down事件，终止嵌套滑动
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

源码内容不长，主要的逻辑内容上面已经讲了，其他的都是一些细节的处理。onTouchListener一般情况下我们是不会使用，那么接下来我们直接看到onTouchEvent方法。

onTouchEvent总体上就做一件事：**根据按下情况选择触发onClickListener或者onLongClickListener** ，也就是判断是单击还是长按事件，其他的源码都是实现细节。onTouchEvent方法正确处理每一个事件类型，来确保点击与长按监听器可以被准确地执行。理解onTouchEvent的源码之前，有几个重要的点需要先了解一下。

我们的操作模式有按键模式、触摸模式。按键模式对应的是外接键盘或者以前的老式键盘机，在按键模式下我们要点击一个按钮通常都是先使用方向光标选中一个button（也就是让该button获取到focus），然后再点击确认按下一个button。但是在触摸模式下，button却不需要获取焦点。**如果一个view在触摸模式下可以获取焦点，那么他将无法响应点击事件，也就是无法调用onClickListener监听器** ，例如EditText。

view辨别单击和长按的方法是**设置延时任务**，在源码中会看到很多的类似的代码，这里延时任务使用handler来实现。当一个down事件来临时，会添加一个延时任务到消息队列中。如果时间到还没有接收到up事件，说明这是个长按事件，那么就会调用onLongClickListener监听器，而如果在延时时间内收到了up事件，那么说明这是个单击事件，取消这个延时的任务，并调用onClickListener。判断是否是一个长按事件，调用的是 `checkForLongClick` 方法来设置延时任务：

```java
// 接收四个参数：
// delay:延时的时长；x、y: 触控点的位置；classification：长按类型分类
private void checkForLongClick(long delay, float x, float y, int classification) {
    // 只有是可以长按或者长按会显示工具提示的view才会创建延时任务
    if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE || (mViewFlags & TOOLTIP) == TOOLTIP) {
        // 标志还没触发长按
        // 如果延迟时间到，触发长按监听，这个变量 就会被设置为true
        // 那么当up事件到来时，就不会触摸单击监听，也就是onClickListener
        mHasPerformedLongPress = false;

        // 创建CheckForLongPress
        // 这是一个实现Runnable接口的类，run方法中回调了onLongClickListener
        if (mPendingCheckForLongPress == null) {
            mPendingCheckForLongPress = new CheckForLongPress();
        }
        // 设置参数
        mPendingCheckForLongPress.setAnchor(x, y);
        mPendingCheckForLongPress.rememberWindowAttachCount();
        mPendingCheckForLongPress.rememberPressedState();
        mPendingCheckForLongPress.setClassification(classification);
        // 使用handler发送延时任务
        postDelayed(mPendingCheckForLongPress, delay);
    }
}
```

上面这个方法的逻辑还是比较简单的，下面看看 `CheckForLongPress` 这个类:

```java
private final class CheckForLongPress implements Runnable {
...
    @Override
    public void run() {
        if ((mOriginalPressedState == isPressed()) && (mParent != null)
                && mOriginalWindowAttachCount == mWindowAttachCount) {
            recordGestureClassification(mClassification);
            // 在延时时间到之后，就会运行这个任务
            // 调用onLongClickListener监听器
            // 并设置mHasPerformedLongPress为true
            if (performLongClick(mX, mY)) {
                mHasPerformedLongPress = true;
            }
        }
    }
...
}
```

延迟时间结束后，就会运行 `CheckForLongPress` 对象，回调onLongClickListener，这样就表示这是一个长按的事件了。

另外，在默认的情况下，当我们按住一个view，然后手指滑动到该view所在的范围之外，那么系统会认为你对这个view已经不感兴趣，所以无法触发单击和长按事件。当然，很多时候并不是如此，这就需要具体的view来重写onTouchEvent逻辑了，但是view的默认实现是这样的逻辑。

好了，那么接下来就来看一下完整的 `view.onTouchEvent` 代码：

```java
View.java api29
public boolean onTouchEvent(MotionEvent event) {
    // 获取触控点坐标
    // 这里我们发现他是没有传入触控点索引的
    // 所以默认情况下view是只处理索引为0的触控点
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    // 判断是否是可点击的
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    // 一个被禁用的view如果被设置为clickable，那么他仍旧是可以消费事件的
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            // 如果是按下状态，取消按下状态
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // 返回是否可以消费事件
        return clickable;
    }
    
    // 如果设置了触摸事件代理你，那么直接调用代理来处理事件
    // 如果代理消费了事件则返回true
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    // 如果该控件是可点击的，或者长按会出现工具提示
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                // 如果是长按显示工具类标志，回调该方法
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                    handleTooltipUp();
                }
                // 如果是不可点击的view，同时会清除所有的标志，恢复状态
                if (!clickable) {
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
                }
                
                // 判断是否是按下状态
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // 如果可以获取焦点但是没有获得焦点，请求获取焦点
                    // 正常的触摸模式下是不需要获取焦点，例如我们的button
                    // 但是如果在按键模式下，需要先移动光标选中按钮，也就是获取focus
                    // 再点击确认触摸按钮事件
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // 确保用户看到按下状态
                        setPressed(true, x, y);
                    }

                    // 两个参数分别是：长按事件是否已经响应、是否忽略本次up事件
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // 这是一个单击事件，还没到达长按的时间，移除长按标志
                        removeLongPressCallback();

                        // 只有不能获取焦点的控件才能触摸click监听
                        if (!focusTaken) {
                            // 这里使用发送到消息队列的方式而不是立即执行onClickListener
                            // 原因在于可以在点击前触发一些其他视觉效果
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClickInternal();
                            }
                        }
                    }

                    // 取消按下状态
                    // 这里也是个post任务
                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }
                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // 如果发送到队列失败，则直接取消
                        mUnsetPressedState.run();
                    }

                    // 移除单击标志
                    removeTapCallback();
                }
                // 忽略下次up事件标志设置为false
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                // 输入设备源是否是可触摸屏幕
                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                }
                // 标志是否是长按
                mHasPerformedLongPress = false;

                // 如果是不可点击的view，说明是长按提示工具的view
                // 直接检查是否发生了长按
                if (!clickable) {
                    // 这个方法会发送一个延迟的任务
                    // 如果延迟时间到还是按下状态，那么就会回调onLongClickListener接口
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    break;
                }

                // 判断是否是鼠标右键或者手写笔的第一个按钮
                // 特殊处理直接返回
                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // 向上遍历view查看是否在一个可滑动的容器中
                boolean isInScrollingContainer = isInScrollingContainer();

                // 如果在一个可滑动的容器中，那么需要延迟一小会再响应反馈
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    // 利用消息队列来延迟检测一个单击事件，延迟时间是ViewConfiguration.getTapTimeout()
                    // 这个时间是100ms
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // 没有在可滑动的容器中，直接响应触摸反馈
                    // 设置按下状态为true
                    setPressed(true, x, y);
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                // 取消事件，恢复所有的状态
                if (clickable) {
                    setPressed(false);
                }
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                break;

            case MotionEvent.ACTION_MOVE:
                // 通知view和drawable热点改变
                // 暂时不知道什么意思
                if (clickable) {
                    drawableHotspotChanged(x, y);
                }

                final int motionClassification = event.getClassification();
                final boolean ambiguousGesture =
                        motionClassification == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE;
                int touchSlop = mTouchSlop;
                
                // view已经被设置了长按标志且目前的事件标志是模糊标志
                // 系统并不知道用户的意图，所以即使滑出了view的范围，并不会取消长按标志
                // 而是延长越界的误差范围和检查长按的时间
                // 因为这个时候系统并不知道你是想要长按还是要滑动，结果就是两种行为都没有响应
                // 由你接下来的行为决定
                if (ambiguousGesture && hasPendingLongPressCallback()) {
                    final float ambiguousMultiplier =
                            ViewConfiguration.getAmbiguousGestureMultiplier();
                    // 判断此时触控点的位置是否还在view的范围内
                    // touchSlop是一个小范围的误差，超出view位置slop距离依旧判定为在view范围内
                    if (!pointInView(x, y, touchSlop)) {
                       // 移除原来的长按标志
                        removeLongPressCallback();
                        // 延长等待时间，这里是原来长按等待的两倍
                        long delay = (long) (ViewConfiguration.getLongPressTimeout()
                                * ambiguousMultiplier);
                        // 减去已经等待的时间
                        delay -= event.getEventTime() - event.getDownTime();
                        // 添加新的长按标志
                        checkForLongClick(
                                delay,
                                x,
                                y,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    }
                    touchSlop *= ambiguousMultiplier;
                }

                // 判断此时触控点的位置是否还在view的范围内
                // touchSlop是一个小范围的误差，超出view位置slop距离依旧判定为在view范围内
                if (!pointInView(x, y, touchSlop)) {
                    // 如果已经超出范围，直接移除点击标志和长按标志，点击和长按事件均无法响应
                    removeTapCallback();
                    removeLongPressCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        // 取消按下标志
                        setPressed(false);
                    }
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                }

                final boolean deepPress =
                        motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
                // 表示用户在屏幕上用力按压，加快长按响应速度
                if (deepPress && hasPendingLongPressCallback()) {
                    // 移除原来的长按标志，直接响应长按事件
                    removeLongPressCallback();
                    checkForLongClick(
                            0 /* 延迟时间为0 */,
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
                }
                break;
        }

        return true;
    } // 对应if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) 

    return false;
}
```



## 最后

如果你能看到这里，说明你对于viewGroup和view的事件处理源码已经了如指掌了。（高兴之余不如给笔者点个赞？(: ~)

最后这里再来总结一下：

- 触摸事件，从屏幕产生后，经过系统服务的处理，最终会发送到viewRootImpl来进行分发；
- viewRootImpl会调用它所管理的view的 `dispatchTouchEvent` 方法来分发事件，那么这里就会分为两种情况：
  1. 如果是view，那么会直接处理事件
  2. 如果是viewGroup，那么会向下派发事件
- viewGroup会为每个触控点尽量寻找感兴趣的子view，最后再自己处理事件。viewGroup的任务就是把事件分发按照原则精准地分发给他子view。
  - 事件分发中一个非常重要的原则就是：一个触控点的事件序列，只能给一个view消费，除了特殊情况，如被viewGroup拦截。
  - viewGroup为了践行这个原则，touchTarget的设计是非常重要的；他将view与触控点进行绑定，让一个触控点的事件只会给一个view消费
- view的 `dispatchTouchEvent` 主要内容是处理事件。首先会调用onTouchListener，如果其没有处理则会调用onTouchEvent方法。
  - onTouchEvent的默认实现中的主要任务就是辨别单击与长按事件，并回调onClickListener与onLongClickListener



到此本文的内容就结束了，事件分发的整体流程回顾、学了事件分发有什么作用、高频面试题相关文章，将会在后续继续创作。

# 原创不易，你的点赞是我最大的动力。感谢阅读 ~



## 优秀文献

在学习过程中，以下相关资料给了我非常大的帮助，都是非常优秀的文章：

- 《深入理解android卷Ⅲ》：学习android系统必备，作者对于android系统的理解非常透彻，可以帮助我们认识到最本质的知识，而不是停留在表层。但对于新手可能会比较难以读懂。
- 《Android开发艺术探索》：进阶学习android必备，作者讲得比较通俗易懂。深度可能相对而言可能较浅，但对新手比较友好，例如笔者。
- [Android 触摸事件分发机制（三）View触摸事件分发机制](https://www.viseator.com/2017/11/02/android_view_event_3/) : 这篇文章采用拆分源码的思路来讲解源码，更好地吸收源码中的内容，笔者也是借鉴了他的写法来创作本文。文中对于源码的分析非常到位，值得一看。
- [安卓自定义View进阶-事件分发机制详解](https://www.gcssloop.com/customview/dispatch-touchevent-source) : 作者言语幽默，通俗易懂，不可多得的好文。
- [Android事件分发机制 详解攻略，您值得拥有](https://blog.csdn.net/carson_ho/article/details/54136311) : 著名博主carson_Ho的文章。特点是干货满满。全文无废话，只讲重要知识点，适合用来复习知识点。
- [Android事件分发机制](http://gityuan.com/2015/09/19/android-touch/) : gityuan大佬的博客，对于源码的研究都很深入。但对于一些源码细节并没有做过多的解释，有些地方难以理解。



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

