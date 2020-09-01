---


title: 最透彻的Android安卓Window全面解析	#标题
date: 2020/9/1 00:00:00 	#建立日期
summary: 带你认识最本质Android的window机制					#文章摘要
tags: 						#标签
 - android 
 - window
categories:  				#分类
 - Android
img:  						#文章卡片显示的图，使用图床

updated: 					#更新日期
author:  					#作者

top:						#boolean,文章是否置顶
cover: true 						#文章是否加入轮播图
coverImg: 					#文章轮播图显示的图片
toc: true						#是否开启toc
mathjax: 					#是否开启数学公式支持

comments: true 				#开启评论
---

## 前言

你好！
我是一只修仙的猿，欢迎阅读我的文章。

Window，读者可能更多的认识是windows系统的窗口。在windows系统上，我们可以多个窗口同时运行，每个窗口代表着一个应用程序。但在安卓上貌似并没有这个东西，但读者可以马上想到，不是有小窗口模式吗，像米UI最新的系统，不就是可以随意创建一个小窗口，然后两个应用同时操作？是的，那是属于android中，window的一种表现方式。但是手机屏幕终究不能和电脑相比，因为屏幕太小了，小到只能操作一款应用，多个窗口就显得非常不习惯，所以Android上关于窗口方面的知识读者可能接触不多。那window的意思就只是小米系统中那种小窗口吗？

当然不是。Android框架层意义上的window和我们认识的window其实是有点不一样的。我们日常最直观的，每个应用界面，都有一个应用级的window。再例如popupWindow、Toast、dialog、menu都是需要通过创建window来实现。所以其实window我们一直都见到，只是不知道那就是window。了解window的机制原理，可以更好地了解window，进而更好地了解android是怎么管理屏幕上的view。这样，当我们需要使用dialog或者popupWindow的时候，可以懂得他背后究竟做了什么，才能够更好的运用dialog、popupWindow等。

当然，到此如果你有很多的疑问，甚至质疑我的理论，那就希望你可以阅读完这一篇文章。我会从window是什么，有什么用，内部机制是什么，各种组件是如何创建window等等方面来阐述Android中的window。文章内容非常多，读者可自选章节阅读。强烈建议阅读【window的概念】【window与PhoneWindow的关系】【从Android架构看window】这三章，会对window的本质有更清晰的认识。

本文大纲：

<img src="https://s1.ax1x.com/2020/09/01/dxLpI1.png" alt="本文大纲" border="0" width=80%/>

> 笔者才疏学浅，有不同观点欢迎评论区或私信讨论。如需转载请留言告知。
> 另外欢迎阅读笔者的个人博客[一只修仙的猿的个人博客](https://qwerhuan.gitee.io/)，更精美的UI，拥有更好的阅读体验。



----



## window的概念

windowowindow，先假设如果没有window，会发生什么：

我们看到的界面ui是view，如我们的应用布局，更简单是一个button。假如屏幕上现在有一个Button，如图1，现在往屏幕中间添加一个TextView，那么最终的结果是图2，还是图3：

<img src="https://s1.ax1x.com/2020/08/30/dbxhY8.png" alt="示例图" border="0" width=60%/>

在上图的图2中，如果我要实现点击textView执行他的监听事件逻辑，点击不是textView的区域让textView消失，需要怎么实现呢？读者可能会说，我们可以在Activity中添加这部分的逻辑，那如果我们需要让一个悬浮窗在所有界面显示呢，如上文我讲到的小米悬浮窗，两个不用应用的view，怎么确定他们的显示次序？又例如我们需要弹出一个dialog来提示用户，怎么样可以让dialog永远处于最顶层呢，包括显示dialog期间应用弹出的如popupWindow必须显示在dialog的低下，但toast又必须显示在dialog上面。

很明显，我们的屏幕可以允许多个应用同时显示非常多的view，他们的显示次序或者说显示高度是不一样的，如果没有一个统一的管理者，那么每一家应用都想要显示在最顶层，那么屏幕上的view会非常乱，这时候急需一个管理者来统一管理view的显示以及接受触摸事件的逻辑，这个管理者，就是整个Android的window机制。

> **window机制就是为了解决屏幕上的view的显示逻辑问题。**

---

那什么是window，在Android的window机制中，每个view树都可以看成一个window。为什么不是每个view呢？因为view树中每个view的显示次序是固定的，例如我们的Activity布局，每一个控件的显示都是已经安排好的，对于window机制来说，属于“不可再分割的view”。

>什么是view树？例如你在布局中给Activity设置了一个布局xml，那么最顶层的布局如LinearLayout就是view树的根，他包含的所有view就都是该view树的节点，所以这个view树就对应一个window。
>
>举几个具体的例子：
>
>- 我们在添加dialog的时候，需要给他设置view，那么这个view他是不属于antivity的布局内的，是通过WindowManager添加到屏幕上的，不属于activity的view树内，所以这个dialog是一个独立的view树，所以他是一个window。
>- popupWindow他也对应一个window，因为它也是通过windowManager添加上去的，不属于Activity的view树。
>- 当我们使用使用windowManager在屏幕上添加的任何view都不属于Activity的布局view树，即使是只添加一个button。

view树（后面使用view代称，后面我说的view都是指view树）是window机制的操作单位，每一个view对应一个window，**view是window的存在形式，window是view的载体**，我们平时看到的应用界面、dialog、popupWindow以及上面描述的悬浮窗，都是**window的表现形式**。注意，我们看到的不是window，而是view。**window是view的管理者，同时也是view的载体。他是一个抽象的概念，本身并不存在，view是window的表现形式。**这里的不存在，指的是我们在屏幕上是看不到window的，他不像windows系统，如下图：

<img src="https://s1.ax1x.com/2020/08/26/dfibnK.png" alt="windows系统窗口" border="0" width=60%/>

有一个很明显的标志：看，我就是window。但在Android中我们是无法感知的，我们只能看到view无法看到window，window是控制view需要怎么显示的管理者。每个成功的男人背后都有一个女人，每个view背后都有一个window。

window本身并不存在，他只是一个概念。举个栗子：如班集体，就是一个概念，他的存在形式是这整个班的学生，当学生不存在那么这个班集体也就不存在。但是他的好处是得到了一个新的概念，我们可以以班为单位来安排活动。因他不存在，所以也很难从源码中找到他的痕迹，window机制的操作单位都是view，如果要说他在源码中的存在形式，笔者目前的认知就是在WindowManagerService中每一个view对应一个windowStatus。WindowManagerService是什么如果没了解过可以先忽略后面会讲到。读者可以慢慢思考一下这个抽象的概念，后面会慢慢深入讲源码帮助理解。

> - view是window的存在形式，window是view的载体
> - window是view的管理者，同时也是view的载体。他是一个抽象的概念，本身并不存在，view是window的表现形式

---

最后做一个总结：

> - window机制是为了解决屏幕上view的显示混乱问题，让所有view都按照秩序来显示，满足我们的开发需求
> - window是window机制中的操作单位，每个window对应一个view。
> - window本身并不存在，他是一个抽象的概念，他的存在形式是view，每个view的当前状态保存在WindowManagerService中的windowStatus

思考：Android中不是有一个抽象类叫做window还有一个PhoneWindow实现类吗，他们不就是window的存在形式，为什么说window是抽象不存在的？读者可自行思考，后面会讲到。



---



## Window的相关属性

在了解window的操作流程之前，先补充一下window的相关属性。

#### window的type属性

前面我们讲到window机制解决的一个问题就是view的显示次序问题，这个属性就决定了window的显示次序。window是有分类的，不同类别的显示高度范围不同，例如我把1-1000m高度称为低空，1001-2000m高度称为中空，2000以上称为高空。window也是一样按照高度范围进行分类，他也有一个变量Z-Order，决定了window的高度。window一共可分为三类：

- 应用程序窗口：应用程序窗口一般位于最底层，Z-Order在1-99
- 子窗口：子窗口一般是显示在应用窗口之上，Z-Order在1000-1999
- 系统级窗口：系统级窗口一般位于最顶层，不会被其他的window遮住，如Toast，Z-Order在2000-2999。**如果要弹出自定义系统级窗口需要动态申请权限**。

Z-Order越大，window越靠近用户，也就显示越高，高度高的window会覆盖高度低的window。

window的type属性就是Z-Order的值，我们可以给window的type属性赋值来决定window的高度。系统为我们三类window都预设了静态常量，如下（以下常用参数介绍转自参考文献第一篇文章）：

- 应用级window

  ```java
  // 应用程序 Window 的开始值
  public static final int FIRST_APPLICATION_WINDOW = 1;
  
  // 应用程序 Window 的基础值
  public static final int TYPE_BASE_APPLICATION = 1;
  
  // 普通的应用程序
  public static final int TYPE_APPLICATION = 2;
  
  // 特殊的应用程序窗口，当程序可以显示 Window 之前使用这个 Window 来显示一些东西
  public static final int TYPE_APPLICATION_STARTING = 3;
  
  // TYPE_APPLICATION 的变体，在应用程序显示之前，WindowManager 会等待这个 Window 绘制完毕
  public static final int TYPE_DRAWN_APPLICATION = 4;
  
  // 应用程序 Window 的结束值
  public static final int LAST_APPLICATION_WINDOW = 99;
  ```

- 子window

  ```java
  // 子 Window 类型的开始值
  public static final int FIRST_SUB_WINDOW = 1000;
  
  // 应用程序 Window 顶部的面板。这些 Window 出现在其附加 Window 的顶部。
  public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;
  
  // 用于显示媒体(如视频)的 Window。这些 Window 出现在其附加 Window 的后面。
  public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;
  
  // 应用程序 Window 顶部的子面板。这些 Window 出现在其附加 Window 和任何Window的顶部
  public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;
  
  // 当前Window的布局和顶级Window布局相同时，不能作为子代的容器
  public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;
  
  // 用显示媒体 Window 覆盖顶部的 Window， 这是系统隐藏的 API
  public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;
  
  // 子面板在应用程序Window的顶部，这些Window显示在其附加Window的顶部， 这是系统隐藏的 API
  public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;
  
  // 子 Window 类型的结束值
  public static final int LAST_SUB_WINDOW = 1999;
  ```

- 系统级window

  ```java
  // 系统Window类型的开始值
  public static final int FIRST_SYSTEM_WINDOW = 2000;
  
  // 系统状态栏，只能有一个状态栏，它被放置在屏幕的顶部，所有其他窗口都向下移动
  public static final int TYPE_STATUS_BAR = FIRST_SYSTEM_WINDOW;
  
  // 系统搜索窗口，只能有一个搜索栏，它被放置在屏幕的顶部
  public static final int TYPE_SEARCH_BAR = FIRST_SYSTEM_WINDOW+1;
  
  // 已经从系统中被移除，可以使用 TYPE_KEYGUARD_DIALOG 代替
  public static final int TYPE_KEYGUARD = FIRST_SYSTEM_WINDOW+4;
  
  // 系统对话框窗口
  public static final int TYPE_SYSTEM_DIALOG = FIRST_SYSTEM_WINDOW+8;
  
  // 锁屏时显示的对话框
  public static final int TYPE_KEYGUARD_DIALOG = FIRST_SYSTEM_WINDOW+9;
  
  // 输入法窗口，位于普通 UI 之上，应用程序可重新布局以免被此窗口覆盖
  public static final int TYPE_INPUT_METHOD = FIRST_SYSTEM_WINDOW+11;
  
  // 输入法对话框，显示于当前输入法窗口之上
  public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;
  
  // 墙纸
  public static final int TYPE_WALLPAPER = FIRST_SYSTEM_WINDOW+13;
  
  // 状态栏的滑动面板
  public static final int TYPE_STATUS_BAR_PANEL = FIRST_SYSTEM_WINDOW+14;
  
  // 应用程序叠加窗口显示在所有窗口之上
  public static final int TYPE_APPLICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 38;
  
  // 系统Window类型的结束值
  public static final int LAST_SYSTEM_WINDOW = 2999;
  ```

---

#### Window的flags参数

flag标志控制window的东西比较多，很多资料的描述是“控制window的显示”，但我觉得不够准确。flag控制的范围包括了：各种情景下的显示逻辑（锁屏，游戏等）还有触控事件的处理逻辑。控制显示确实是他的很大部分功能，但是并不是全部。下面看一下一些常用的flag，就知道flag的功能了（以下常用参数介绍转自参考文献第一篇文章）：

```java
// 当 Window 可见时允许锁屏
public static final int FLAG_ALLOW_LOCK_WHILE_SCREEN_ON = 0x00000001;

// Window 后面的内容都变暗
public static final int FLAG_DIM_BEHIND = 0x00000002;

// Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的
// Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL
public static final int FLAG_NOT_FOCUSABLE = 0x00000008;

// 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件
// Window 之外的 view 也是可以响应 touch 事件。
public static final int FLAG_NOT_TOUCH_MODAL  = 0x00000020;

// 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口。
public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;

// 只要 Window 可见时屏幕就会一直亮着
public static final int FLAG_KEEP_SCREEN_ON     = 0x00000080;

// 允许 Window 占满整个屏幕
public static final int FLAG_LAYOUT_IN_SCREEN   = 0x00000100;

// 允许 Window 超过屏幕之外
public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;

// 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示
public static final int FLAG_FULLSCREEN      = 0x00000400;

// 表示比FLAG_FULLSCREEN低一级，会显示状态栏
public static final int FLAG_FORCE_NOT_FULLSCREEN   = 0x00000800;

// 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件
public static final int FLAG_IGNORE_CHEEK_PRESSES    = 0x00008000;

// 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件。
public static final int FLAG_WATCH_OUTSIDE_TOUCH = 0x00040000;

@Deprecated
// 窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替
public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;

// 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，
// 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色。
public static final int FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS = 0x80000000;

// 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸
public static final int FLAG_SHOW_WALLPAPER = 0x00100000;
```

---

#### window的solfInputMode属性

这一部分就是当软件盘弹起来的时候，window的处理逻辑，这在日常中也经常遇到，如：我们在微信聊天的时候，点击输入框，当软键盘弹起来的时候输入框也会被顶上去。如果你不想被顶上去，也可以设置为被软键盘覆盖。下面介绍一下常见的属性（以下常见属性介绍选自参考文献第一篇文章）：

```java
// 没有指定状态，系统会选择一个合适的状态或者依赖于主题的配置
public static final int SOFT_INPUT_STATE_UNCHANGED = 1;

// 当用户进入该窗口时，隐藏软键盘
public static final int SOFT_INPUT_STATE_HIDDEN = 2;

// 当窗口获取焦点时，隐藏软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_HIDDEN = 3;

// 当用户进入窗口时，显示软键盘
public static final int SOFT_INPUT_STATE_VISIBLE = 4;

// 当窗口获取焦点时，显示软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_VISIBLE = 5;

// window会调整大小以适应软键盘窗口
public static final int SOFT_INPUT_MASK_ADJUST = 0xf0;

// 没有指定状态,系统会选择一个合适的状态或依赖于主题的设置
public static final int SOFT_INPUT_ADJUST_UNSPECIFIED = 0x00;

// 当软键盘弹出时，窗口会调整大小,例如点击一个EditView，整个layout都将平移可见且处于软件盘的上方
// 同样的该模式不能与SOFT_INPUT_ADJUST_PAN结合使用；
// 如果窗口的布局参数标志包含FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏。
public static final int SOFT_INPUT_ADJUST_RESIZE = 0x10;

// 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的,
// 例如有两个EditView的输入框，一个为Ev1，一个为Ev2，当你点击Ev1想要输入数据时，当前的Ev1的输入框会移到软键盘上方
// 该模式不能与SOFT_INPUT_ADJUST_RESIZE结合使用
public static final int SOFT_INPUT_ADJUST_PAN = 0x20;

// 将不会调整大小，直接覆盖在window上
public static final int SOFT_INPUT_ADJUST_NOTHING = 0x30;
```

---

#### window的其他属性

上面的三个属性是window比较重要也是比较复杂 的三个，除此之外还有几个日常经常使用的属性：

- x与y属性：指定window的位置
- alpha：window的透明度
- gravity：window在屏幕中的位置，使用的是Gravity类的常量
- format：window的像素点格式，值定义在PixelFormat中

----

#### 如何给window属性赋值

window属性的常量值大部分存储在WindowManager.LayoutParams类中，我们可以通过这个类来获得这些常量。当然还有Gravity类和PixelFormat类等。

一般情况下我们会通过以下方式来往屏幕中添加一个window：

```java
// 在Activity中调用
WindowManager.LayoutParams windowParams = new WindowManager.LayoutParams();
windParams.flags = WindowManager.LayoutParams.FLAG_FULLSCREEN;
TextView view = new TextView(this);
getWindowManager.addview(view,windowParams);
```

我们可以直接给WindowManager.LayoutParams对象设置属性。

第二种赋值方法是直接给window赋值，如

```java
getWindow().flags = WindowManager.LayoutParams.FLAG_FULLSCREEN;
```
除此之外，window的solfInputMode属性比较特殊，他可以直接在AndroidManifest中指定，如下：

```java
 <activity android:windowSoftInputMode="adjustNothing" />
```


---

最后总结一下：

> - window的重要属性有type、flags、solfInputMode、gravity等
> - 我们可以通过不同的方式给window属性赋值
> - 没必要去全部记下来，等遇到需求再去寻找对应的常量即可



-----



## window机制的关键类

从这里开始就要开始讲window的内部机制，首先我们需要了解一下window里面的那些关键的类和接口，当然首先我们要了解一下window的内部机制中有哪些角色：

#### window相关

window的实现类只有一个：PhoneWindow，他继承自Window抽象类。后面我们经常见到他。

#### WindowManager相关

顾名思义，windowManager就是window管理类。这一部分的关键类有windowManager，viewManager，windowManagerImpl，windowManagerGlobal。windowManager是一个接口，继承自viewManager。viewManager中包含了我们非常熟悉的三个接口：`addView,removeView,updateView`。
windowManagerImpl和PhoneWindow是成对出现的，前者负责管理后者。WindowManagerImpl是windowManager的实现类，但是他本身并没有真正实现逻辑，而是交给了WindowManagerGlobal。WindowManagerGlobal是全局单例，windowManagerImpl内部使用桥接模式，他是windowManager接口逻辑的真正实现

#### view相关

这里有个很关键的类：ViewRootImpl。每个view树都会有一个。当我使用windowManager的addView方法时，就会创建一个ViewRootImpl。ViewRootImpl的作用很关键：

- 负责连接view和window的桥梁事务
- 负责和WindowManagerService的联系
- 负责管理和绘制view树
- 输入事件的中转站

每个window都会有一个ViewRootImpl，viewRootImpl是负责绘制这个view树和window与view的桥梁，每个window都会有一个ViewRootImpl。如果这里对他的这些功能不太理解，没事，只要记住这个类就好了，后面会讲到。

#### WindowManagerService

这个是window的真正管理者，类似于AMS（ActivityManagerService）管理四大组件。所有的window创建最终都要经过windowManagerService。整个Android的window机制中，WMS绝对是核心，他决定了屏幕所有的window该如何显示如何处理点击事件。

---

#### 总结

好了，看了上面那么多的类，可能有点感觉眼花缭乱，画个图帮助理解一下：（注意里面绿色的window不是关键类，只是为了方便理解画进去了）

<img src="https://s1.ax1x.com/2020/08/26/dfGICn.png" alt="window内部关键类" border="0" width=70%/>

> PhoneWindow是窗口类，继承自抽象类Window，也是唯一子类。WindowManager是Window管理接口，继承自ViewManager，他的唯一实现类是WindowManagerImpl。WindowManagerImpl并没有真正实现windowManager接口逻辑，而是把逻辑转给了WindowManagerGlobal，WindowManagerGlobal是全局单例。Window和View的联系通过ViewRootImpl桥梁，同时ViewRootImpl还负责管理view树、绘制view树、和WMS通信。WMS即WindowManagerService，是Window的真正管理类。

了解完上面的一些关键类，可能读者对于他们的功能还是一头雾水，没关系，下面我将通过源码来讲解，很快你就可以理解了。到时候可以再次回头来看一下，就更加融会贯通了。



---



## Window的添加过程

通过理解源码之后，可以对之前的理论理解更加的透彻，同时也是对上一小节的讲解。window的添加过程，指的是我们通过WindowManagerImpl的addView方法来添加window的过程。window的存在形式是view也可以从这个方法名字看出来，添加window即为添加view。

想要添加一个window，我们知道首先得有view和WindowManager.LayoutParams对象，才能去创建一个window，这是我们常见的代码：

```java
Button button = new Button(this);
WindowManager.LayoutParams windowParams = new WindowManager.LayoutParams();
// 这里对windowParam进行初始化
windowParam.addFlags...
// 获得应用PhoneWindow的WindowManager对象进行添加window
getWindowManager.addView(button,windowParams);
```

然后接下来我们进入addView方法中看看。我们知道这个windowManager的实现类是WindowManagerImpl，上面讲过，进入他的addView方法看一看：

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

可以发现他把逻辑直接交给mGlobal去处理了。这个mGlobal是啥？有印象的读者就会知道他是WindowManagerGlobal，是一个全局单例，所以这里可以看到WindowManagerGlobal确实是WindowManager接口的具体逻辑实现，这里运用的是桥接模式。那我们进WindowManagerGlobal的方法看一下：

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    // 首先判断参数是否合法
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }
    
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    // 如果不是子窗口，会对其做参数的调整
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
    
	synchronized (mLock) {
        ...
        // 这里新建了一个viewRootImpl，并设置参数
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);

        // 添加到windowManagerGlobal的三个重要list中，后面会讲到
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // 最后通过viewRootImpl来添加window
        try {
            root.setView(view, wparams, panelParentView);
        } 
        ...
    }  
}
```

代码有点长，盲猜读者你并没有看（嘻嘻），一步步看不复杂的。

- 首先对参数的合法性进行检查
- 然后判断该窗口是不是子窗口，如果是的话需要对窗口进行调整，这个好理解，子窗口要跟随父窗口的特性。
- 接着新建viewRootImpl对象，并把view、viewRootImpl、params三个对象添加到三个list中进行保存
- 最后通过viewRootImpl来进行添加

> 补充一点关于WindowManagerGlobal中的三个list，他们分别是：
>
> ```java
> private final ArrayList<View> mViews = new ArrayList<View>();
> private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
> private final ArrayList<WindowManager.LayoutParams> mParams =
>         new ArrayList<WindowManager.LayoutParams>();
> ```
>
> 每一个window所对应的这三个对象都会保存在这里，之后对window的一些操作就可以直接来这里取对象了。当window被删除的时候，这些对象也会被从list中移除。

可以看到添加的window的逻辑就交给ViewRootImpl了。viewRootImpl是window和view之间的桥梁，viewRootImpl可以处理两边的对象，然后联结起来。下面看一下viewRootImpl怎么处理：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        ...
        try {
            mOrigWindowType = mWindowAttributes.type;
            mAttachInfo.mRecomputeGlobalAttributes = true;
            collectViewAttributes();
            // 这里调用了windowSession的方法，调用wms的方法，把添加window的逻辑交给wms
            res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                    mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                    mTempInsets);
            setFrame(mTmpFrame);
        } 
        ...
    }
}
```

viewRootImpl的逻辑很多，重要的就是调用了mWindowSession的方法调用了WMS的方法。这个mWindowSession很重要重点讲一下。

> mWindowSession是一个IWindowSession对象，看到这个命名很快地可以像到这里用了AIDL跨进程通信。IWindowSession是一个IBinder接口，他的具体实现类在WindowManagerService，本地的mWindowSession只是一个Binder对象，通过这个mWindowSession就可以直接调用WMS的方法进行跨进程通信。
>
> 那这个mWindowSession是从哪里来的呢？我们到viewRootImpl的构造器方法中看一下：
>
> ```java
> public ViewRootImpl(Context context, Display display) {
>    	...
>     	mWindowSession = WindowManagerGlobal.getWindowSession();
>     	...
> }
> ```
>
> 可以看到这个session对象是来自WindowManagerGlobal。再深入看一下：
>
> ```java
> public static IWindowSession getWindowSession() {
>     synchronized (WindowManagerGlobal.class) {
>         if (sWindowSession == null) {
>             try {
>                 ...
>                 sWindowSession = windowManager.openSession(
>                         new IWindowSessionCallback.Stub() {
>                             ...
>                         });
>             } 
>             ...
>         }
>         return sWindowSession;
>     }
>    }
>    ```
>    
>    这熟悉的代码格式，可以看出来这个session是一个单例，也就是**整个应用的所有viewRootImpl的windowSession都是同一个，也就是一个应用只有一个windowSession**。对于wms而言，他是服务于多个应用的，如果说每个viewRootImpl整一个session，那他的任务就太重了。WMS的对象单位是应用，他**在内部给每个应用session分配了一些数据结构如list，用于保存每个应用的window以及对应的viewRootImpl**。当需要操作view的时候，通过session直接找到viewRootImpl就可以操作了。

后面的逻辑就交给WMS去处理了，WMS就会创建window，然后结合参数计算window的高度等等，最后使用viewRootImpl进行绘制。这后面的代码逻辑就不讲了，这是深入到WMS的内容，再讲进去就太复杂了（笔者也还没读懂WMS）。读源码的目的是了解整个系统的本质与工作流程，对系统整体的感知，而不用太深入代码细节，Android系统那么多的代码，如果深入进去会出不来的，所以点到为止就好了。

我们知道windowManager接口是继承viewManager接口的，viewManager还有另外两个接口：removeView、updateView。这里就不讲了，有兴趣的读者可以自己去阅读源码。讲添加流程主要是为了理解window系统的运作，对内部的流程感知，以便于更好的理解window。

最后老样子，做个总结：

> window的添加过程是通过PhoneWindow对应的WindowManagerImpl来添加window，内部会调用WindowManagerGlobal来实现。WindowManagerGlobal会使用viewRootImpl来进行跨进程通信让WMS执行创建window的业务。
>
> 每个应用都有一个windowSession，用于负责和WMS的通信，如ApplicationThread与AMS的通信。



----



## window与PhoneWindow的关系

> 解释一下标题，window是指window机制中window这个概念，而PhoneWindow是指PhoneWindow这个类。后面我在讲的时候，如果是指类，我会在后面加个‘类’字。如window是指window概念，window类是指window这个抽象类。读者不要混淆。

还记得我在讲window的概念的时候留了一个思考吗？

> 思考：Android中不是有一个抽象类叫做window还有一个PhoneWindow实现类吗，他们不就是window的存在形式，为什么说window是抽象不存在的

这里我再抛出几个问题：

- 有一些资料认为PhoneWindow就是window，是view容器，负责管理容器内的view，windowManagerImpl可以往里面添加view，如上面我们讲过的addView方法。但是，同时它又说每个window对应一个viewRootImpl，但却没解释为什么每次addView都会新建一个viewRootImpl，前后发送矛盾。
- 有一些资料也是认为PhoneWindow是window，但是他说addView方法不是添加view而是添加window，同时拿这个方法的名字作为论据证明view就是window，但是他没解释为什么在使用addView方法创建window的过程却没有创建PhoneWindow对象。

---

我们一步步来看。我们首先来看一下源码中对于window抽象类的注释：

```java
 Abstract base class for a top-level window look and behavior policy.  An
 instance of this class should be used as the top-level view added to the
 window manager. It provides standard UI policies such as a background, title
 area, default key processing, etc.
     
顶层窗口外观和行为策略的抽象基类。此类的实例应用作添加到窗口管理器的顶层视图。
它提供标准的UI策略，如背景、标题区域、默认键处理等。
```

大概意思就是：这个类是顶级窗口的抽象基类，顶级窗口必须继承他，他负责窗口的外观如背景、标题、默认按键处理等。这个类的实例被添加到windowManager中，让windowManager对他进行管理。PhoneWindow是一个top-level window（顶级窗口），他被添加到顶级窗口管理器的顶层视图，其他的window，都需要添加到这个顶层视图中，所以更准确的来说，**PhoneWindow并不是view容器，而是window容器。**

每一个PhoneWindow都有一个WindowManagerImpl对象，他们是成对出现的。当我们调用windowManagerImpl来添加window的时候，其实就是往PhoneWindow中添加window。WindowManagerImpl是利用应用服务中的windowManagerImpl来进行创建的，看似每个PhoneWindow都有一个WindowManagerImpl，但是他的内部其实是同个WindowManagerImpl（后面在讲Activity的window创建流程会讲到）。

**PhoneWindow不是Android的window机制中的window概念，他只是在应用端的一个window容器**。那PhoneWindow的存在意义是什么？

---

每一个view树，无论你点击哪个控件，都可以追溯到根部，有一个统一管理事件的中心：viewRootImpl，他负责把事件统一发送到合理的地方。但是如果是不同的view树，就无法追溯到统一根源，无法统一处理事件。Activity可能有很多的view树，例如popupWindow，menu，那这个时候怎么让他们有一个统一的处理事件根源呢？没错就是PhoneWindow。所有通过PhoneWindow对应的WindowManagerImpl添加的window都要接受PhoneWindow的管理，我们可以通过WindowManagerImpl往PhoneWindow中添加子window。这样所有的点击事件等就可以直接交给PhoneWindow，然后PhoneWindow再把事件交给Activity统一处理。这样，Activity就可以把属于自己应用的window当成一棵“view树”，可以管理到所有属于他的window。如下图：

<img src="https://s1.ax1x.com/2020/08/26/dfE5VK.png" alt="PhoneWindow关系图" border="0" width=60%/>

同时，Activity可以给PhoneWindow设置属性，他会按照一定的逻辑给PhoneWindow中的window设置属性。还记得谷歌的官方给window类注释的最后一句话吗：`它提供标准的UI策略，如背景、标题区域、默认键处理等。`。给PhoneWindow设置的属性很多，最常见的有背景、软键盘模式、状态栏区域的延伸等等，他的内部会配合DecorView来实现我们日常的一些开发需求（后面会讲到DecorView），封装成API供我们使用。PhoneWindow的层级结构如下：

<img src="https://s1.ax1x.com/2020/08/31/dOqQ3Q.png" alt="dOqQ3Q.png" border="0" width=30%/>

PhoneWindow利用DecorView，实现给contentView添加背景，设置标题区域等等功能。而**真正的window是没有背景、标题栏等这一说的**。真正的window只是一个抽象的概念，他本身并不存在，PhoneWindow是利用DecorView才实现了这些功能。读者需要区分好这两者的关系。

这里我就我不深入讲PhoneWindow了，我们侧重点是讲PhoneWindow与window的关系，有兴趣的读者可以自行研究一下。

----

总结一下：

> - PhoneWindow本身不是window，他是一个window容器，统一管理其中的window
> - windowManagerImpl并不是管理window的类，而是管理PhoneWindow的类
> - PhoneWindow的作用是协作Activity等需要管理多个window的工具类，让这些window拥有统一的事件处理源，
> - PhoneWindow可以配合DecorView可以给其中的window按照一定的逻辑提供标准的UI策略



---



## 常见组件的window创建流程

上面讲的是通过windowManagerImpl创建window的过程，我们通过前面的讲解了解到，WindowManagerImpl是管理PhoneWindow的，他们是同时出现的。因而有两种创建window的方式：

- 已经存在PhoneWindow，直接通过WindowManagerImpl创建window
- PhoneWindow尚未存在，先创建PhoneWindow，再利用windowManagerImpl来创建window

当我们在Activity中使用getWindowManager方法获取到的就是应用的PhoneWindow对应的WindowManagerImpl。下面来讲一下不同的组件是如何创建window的，

#### Activity

如果有阅读过Activity的启动流程的读者，会知道Activity的启动最后来到了ActivityThread的`handleLaunchActivity`这个方法。

> 关于Activity的启动流程，我写过一篇文章，有兴趣的读者可以点击下方链接前往：
>
> [Activity启动流程详解（基于api28）](https://blog.csdn.net/weixin_43766753/article/details/107746968)

至于为什么是这个方法这里就不讲了，有兴趣的读者可以去看上面的文章。我们直接来看这个方法的代码：

```java
public void handleLaunchActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...;
    // 这里对WindowManagerGlobal进行初始化
    WindowManagerGlobal.initialize();

   	// 启动Activity并回调activity的onCreate方法
    final Activity a = performLaunchActivity(r, customIntent);
    ...
}


private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    try {
        // 这里创建Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
		...
        if (activity != null) {
            ...
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            // 这里将window作为参数传到activity的attach方法中
            // 一般情况下这里window==null
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);  
            ...
            // 最后这里回调Activity的onCreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
        }
    
    ...
}
```

`handleLaunchActivity`的代码中首先对WindowManagerGlobal进行初始化，然后调用了`performLaunchActivity`方法。代码很多，这里只截取了重要部分。首先会创建Application对象，然后再调用Activity的attach方法，把window作为参数传进去，最后回调activity的onCreate方法。所以这里最有可能创建window的方法就是Activity的`attach`方法了。我们进去看一下：

```java
final void attach(...,Context context,Window window, ...) {
    ...;
 	// 这里新建PhoneWindow对象，并对window进行初始化
	mWindow = new PhoneWindow(this, window, activityConfigCallback);
    // Activity实现window的callBack接口，把自己设置给window
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);    
    ...
    // 这里初始化window的WindowManager对象
	mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);        
}
```

同样只截取了重要代码，attach方法参数非常多，我只留下了window相关的参数。在这方法里首先利用传进来的window创建了PhoneWindow。Activity实现window的callBack接口，可以把自己设置给window当观察者。当window发生变化的时候可以通知activity。然后再创建WindowManager和PhoneWindow绑定在一起，这样我们就可以通过windowManager操作PhoneWindow了。（这里不是setWindowManager吗，windowManager是什么时候创建的？）我们进去`setWindowManager`方法看一下：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    // 这里创建了windowManager
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

这个方法里首先会获取到应用服务的WindowManager（实现类也是WindowManagerImpl），然后通过这个应用服务的WindowManager创建了新的windowManager。

> 从这里可以看到是利用系统服务的windowManager来创建新的windowManagerImpl，因而这个应用所有的WindowManagerImpl都是同个内核windowManager，而创建出来的仅仅是包了个壳。

这样PhoneWindow和WindowManagerImpl就绑定在一起了。Activity可以通过WindowManagerImpl来操作PhoneWindow。

---

到这里Activity的PhoneWindow和WindowManagerImpl对象就创建完成了，接下来是如何把Activity的布局文件设置给PhoneWindow。在上面我讲到调用Activity的attach方法之后，会回调Activity的onCreate方法，在onCreate方法我们会调用`setContentView`来设置布局，如下：

```java
public void setContentView(View view, ViewGroup.LayoutParams params) {
    getWindow().setContentView(view, params);
    initWindowDecorActionBar();
}
```

这里的getWindow就是获取到我们上面创建的PhoneWindow对象。我们继续看下去：

```java
// 注意他有多个重载的方法，要选择参数对应的方法
public void setContentView(int layoutResID) {
    // 创建DecorView
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // 这里根据布局id加载布局
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        // 回调activity的方法
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

同样我们只看重点代码：

- 首先看decorView创建了没有，没有的话创建DecorView
- 把布局加载到DecorView中
- 回调Activity的callBack方法

> 这里补充一下什么是DecorView。DecorView是在PhoneWindow中预设好的一个布局，这个布局长这样：
>
> <img src="https://s1.ax1x.com/2020/08/31/dLc7ZR.png" alt="decorView" border="0" width=20%/>
>
> 他是一个垂直排列的布局，上面是ActionBar，下面是ContentView，他是一个FrameLayout。我们的Activity布局就加载到ContentView里进行显示。所以Decorview是Activity布局最顶层的viewGroup。

然后我们看一下怎么初始化DercorView的：

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        // 这里创建了DecorView
        mDecor = generateDecor(-1);
        ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        // 对DecorView进行初始化，得到ContentView
        mContentParent = generateLayout(mDecor);
        ...
    }
}
```

`installDecor`方法中主要是新建一个DecorView对象，然后加载预设好的布局对DecorView进行初始化，（预设好的布局就是上面讲述的布局）并获取到这个预设布局的ContentView。好了然后我们再回到window的setContentView方法中，初始化了DecorView之后，把Activity布局加载到DecorView的ContentView中如下代码：

```java
// 注意他有多个重载的方法，要选择参数对应的方法
public void setContentView(int layoutResID) {
    ...
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // 这里根据布局id加载布局
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    ...
   	mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        // 回调activity的方法
        cb.onContentChanged();
    }
}
```

所以可以看到Activitiy的布局确实是添加到DecorView的ContentView中，这也是为什么onCreate中使用的是setContentView而不是setView。最后会回调Activity的方法告诉Activity，DecorView已经创建并初始化完成了。

---

到这里DecorView创建完成了，但还缺少了最重要的一步：**把DecorView**作为window添加到屏幕上。从前面的介绍我们知道添加window需要用到WindowManagerImpl的addView方法。这一步是在ActivityThread的`handleResumeActivity`方法中被执行：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    // 调用Activity的onResume方法
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...
    // 让decorView显示到屏幕上
	if (r.activity.mVisibleFromClient) {
        r.activity.makeVisible();
  	}
```

这一步方法有两个重点：回调onResume方法，把decorView添加到屏幕上。我们看一下`makeVisible`方法做了什么：

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

是不是非常熟悉？直接调用WindowManagerImpl的addView方法来吧decorView添加到屏幕上，至此，我们的Activity界面就会显示在屏幕上了。

----

好了，这部分很长，最后来总结一下：

> - 从Activity的启动流程可以得到Activity创建Window的过程
> - 创建PhoneWindow -> 创建WindowManager -> 创建decorView -> 利用windowManager把DecorView显示到屏幕上
> - 回调onResume方法的时候，DecorView还没有被添加到屏幕，所以当onResume被回调，指的是屏幕即将到显示，而不是已经显示

---

#### PopupWindow

popupWindow日常使用的也比较多，最常见的需求是弹一个菜单出来等。popupWindow也是利用windowManager来往屏幕上添加window，但，popupWindow是依附于activity而存在的，当Activity未运行时，是无法弹出popupWindow的，通过源码可以知道，当调用onResume方法的时候，其实后续还有很多事情在做，这个时候Activity也是尚未完全启动，所以popupWindow不能在onCreate、onStart、onResume方法中弹出。

弹出popupWindow的过程分为两个：创建view；通过windowManager添加window。首先看到PopupWindow的构造方法：

```java
public PopupWindow(View contentView, int width, int height, boolean focusable) {
    if (contentView != null) {
        mContext = contentView.getContext();
        mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
    }

    setContentView(contentView);
    setWidth(width);
    setHeight(height);
    setFocusable(focusable);
}
```

他有多个重载方法，但最终都会调用到这个有四个参数的方法。主要是前面的得到context和根据context获得WindowManager。

---

然后我们看到他的显示方法。显示方法有两个：`showAtLocation`和`showAsDropDown`。主要是处理显示的位置不同，其他都是相似的。我们看到第一个方法：

```java
public void showAtLocation(View parent, int gravity, int x, int y) {
    mParentRootView = new WeakReference<>(parent.getRootView());
    showAtLocation(parent.getWindowToken(), gravity, x, y);
}
```

逻辑很简单，父view的根布局存储了起来，然后调用另外的重载方法：

```java
public void showAtLocation(IBinder token, int gravity, int x, int y) {
    // 如果contentView是空直接返回
    if (isShowing() || mContentView == null) {
        return;
    }

    TransitionManager.endTransitions(mDecorView);
    detachFromAnchor();
    mIsShowing = true;
    mIsDropdown = false;
    mGravity = gravity;
	// 得到WindowManager.LayoutParams对象
    final WindowManager.LayoutParams p = createPopupLayoutParams(token);
    // 做一些准备工作
    preparePopup(p);

    p.x = x;
    p.y = y;
	// 执行popupWindow显示工作
    invokePopup(p);
}
```

这个方法的逻辑主要有：

- 判断contentView是否为空或者是否呀进行显示
- 做一些准备工作
- 行popupWindow显示工作

---

这里我们看一下他的准备工作做了什么：

```java
private void preparePopup(WindowManager.LayoutParams p) {
    ...
        
    if (mBackground != null) {
        mBackgroundView = createBackgroundView(mContentView);
        mBackgroundView.setBackground(mBackground);
    } else {
        mBackgroundView = mContentView;
    }
	// 创建了DecorView
    // 注意，这里的DecorView并不是我们之前讲的DecorView，而是他的内部类：PopupDecorView
    mDecorView = createDecorView(mBackgroundView);
    mDecorView.setIsRootNamespace(true);

    ...
}
```

接下来再看他的显示工作：

```java
private void invokePopup(WindowManager.LayoutParams p) {
    ...
   	// 调用windowManager添加window
    mWindowManager.addView(decorView, p);

    ...
}
```

到这里popupWindow就会被添加到屏幕上了。

---

最后总结一下：

> - 根据参数构建popupDecorView
> - 把popupDecorView添加到屏幕上

---

#### Dialog

dialog和popupWindow最大的区别是：dialog会新建一个PhoneWindow。前面我们了解到，PhoneWindow的作用就是为了统一管理所有属于自身的window，所以dialog的设计就是一个和Activity独立开来的window，他会直接显示在Activity所有view的上面，这也是dialog和popupWindow的本质区别。

dialog的创建过程和popupWindow是大同小异的：创建PhoneWindow，初始化DecorView，添加DecorView。我这里就简单讲解一下。首先看到他的构造方法：

```java
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    ...
    // 获取windowManager
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
	// 构造PhoneWindow
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    // 初始化PhoneWindow
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setOnWindowSwipeDismissedCallback(() -> {
        if (mCancelable) {
            cancel();
        }
    });
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);
    mListenersHandler = new ListenersHandler(this);
}
```

这里和前面的Activity创建过程非常像。接下来看到他的show方法：

```java
public void show() {
   ...
    // 回调onStart方法，获取前面初始化好的decorview
    onStart();
    mDecor = mWindow.getDecorView();
    ...
    WindowManager.LayoutParams l = mWindow.getAttributes();
    ...
    // 利用windowManager来添加window    
    mWindowManager.addView(mDecor, l);
    ...
    mShowing = true;
    sendShowMessage();
}
```

几乎和Activity的添加流程一模一样，这里我也不多讲解了。

---

总结一下：

> - dialog和popupWindow不同，dialog创建了新的PhoneWindow，他是独立在Activity之外的window
> - popupWindow需要依赖于activity，这是两者的本质区别
> - dialog会直接显示在Activity上面，而popUpWindow则不会，后来的新添加view会覆盖在popupWindow上
> - dialog的创建流程和activity几乎是一样的



---



## 从Android架构角度看Window

前面我们介绍过关于PhoneWindow和window之间的关系，了解到PhoneWindow其实不是Window，只是一个window容器。不知读者有没想过一个问题，为什么谷歌要建一个不是window但却名字是window的类？是故意要迷惑我们吗？要了解这个问题，我们先来回顾一下整个android的window机制结构。

首先从WindowManagerService开始，我们知道WMS是window的最终管理者，在WMS中为每一个应用持有一个session，关于session前面我们讲过，每个应用都是全局单例，负责和WMS通信的binder对象。WMS为每个window都建立了一个windowStatus对象，同一个应用的window使用同个session进行跨进程通信，结构大概如下：

<img src="https://s1.ax1x.com/2020/08/31/dX1X7j.png" alt="WMS结构" border="0" width=40%/>

而负责与WMS通信的，是viewRootImpl。前面我们讲过每个view树即为一个window，viewRootImpl负责和WMS进行通信，同时也负责view的绘制。如果把上面的图画仔细一点就是：

<img src="https://s1.ax1x.com/2020/08/31/dXGkk9.png" alt="更详细的结构图" border="0" width=60%/>

图中每一个windowStatus对应一个viewRootImpl，WMS通过viewRootImpl来控制view。这也就是window机制的管理结构。当我们需要添加window的时候，最终的逻辑实现是WindowManagerGlobal，他的内部使用自己的session创建一个viewRootImpl，然后向WMS申请添加window，结构图大概如下：

<img src="https://s1.ax1x.com/2020/09/01/dxRCX6.png" alt="window的添加结构" border="0" width=60%/>

windowManagerGlobal使用自己的IWindowSession创建viewRootImpl，这个IWindowSession是全局单例。viewRootImpl和WMS申请创建window，然后WMS允许之后，再通知viewRootImpl绘制view，同时WMS通过windowStatus存储了viewRootImpl的相关信息，这样如果WMS需要修改view，直接通过viewRootImpl就可以修改view了。

---

从上面的描述中可以发现我全程没有提及到PhoneWindow和WindowManagerImpl。这是因为**他们不属于window机制内的类，而是封装于window机制之上的框架**。假设如果没有PhoneWindow和WindowManager我们该如何添加一个window？首先需要调用WindowGlobal获取session，再创建viewRootImpl，再访问wms，然后再利用viewRootImpl绘制view，是不是很复杂，而这仅仅只是整体的步骤。而WindowManagerImpl正是这个功能。他内部拥有WindowManagerGlobal的单例，然后帮助我们完成了这一系列的步骤。同时，**windowManagerImpl也是只有一个实例，其他的windowManagerImpl都是建立在windowManagerImpl单例上**。这一点在前面有通过源码介绍到。

另外，上面我讲到PhoneWindow并不是window而是一个window容器，那为什么他不要命名为windowContainer呢？PhoneWindow这个类是谷歌给window机制进行更上一层的封装。PhoneWindow内部拥有一个DecorView，我们的布局view都是添加到decorView中的，因为我们可以通过给decorView设置背景，宽高度，标题栏，按键反馈等等，来间接给我们的布局view设置。当我们通过windowManagrImpl添加window的时候，都会把创建的window和PhoneWindow联系起来，让PhoneWindow可以统一处理通过windowManagerImpl添加的window，这点上面有讲到。这样一来，PhoneWindow的存在，**向开发者屏蔽真正的window，暴露给开发者一个“存在的”window**。我们可以**认为**PhoneWindow就是一个window，window是view容器。当我们需要在屏幕上添加view的时候，只需要获得应用window对应的windowManagerImpl，然后直接调用addView方法添加view即可。这里也可以解释为什么windowManager的接口方法是addView而不是addWindow，一个是window确实是以view的存在形式没错，二是为了向开发者屏蔽真正的window，让我们以为是在往window中添加view。画个图如下：

<img src="https://s1.ax1x.com/2020/08/31/dOq4vd.png" alt="window整体结构" border="0" width=60%/>

黄色部分输于谷歌提供给开发者的window框架，而绿色是真正的window机制结构。通过谷歌的框架我们可以很方便地管理屏幕上的view，而不须了解底层究竟是如何工作的。PhoneWindow的存在，更是让window的“可见性”得到了实现，让window变成了一个“view容器”。

---

好了最后来总结一下：

> - Android内部的window机制与谷歌暴露给我们的api是不一样的，谷歌封装的目的是为了让我们更好地使用window。
> - dialog、popupWindow等框架更是对具体场景进行更进一步的封装。
> - 我们在了解window机制的时候，需要跳过应用层，看到window的本质，才能更好地帮助我们理解window。
> - 在android的其他地方也是一样，利用封装向开发者屏蔽底层逻辑，让我们更好地运用。但如果我们需要了解他的机制的时候，就需要绕过这层封装，看到本质。



----



## 总结

全文到这里，就基本结束了。下面先总结一下我这篇文章说了什么：

> - 详述了什么是window
> - 对window的各种参数进行讲解
> - 讲解window机制内的关键类
> - 从源码讲解window的添加流程以及各大组件的window添加流程
> - 详解了PhoneWindow与window的关系，谈了关于谷歌的封装思想

文中最重要的一点就是认识window的本质，区分好window和view之间的关系。为什么window可以设置背景设置标题栏设置宽高？那是因为有PhoneWindow的存在。读者需要区分好window与PhoneWindow的关系。

笔者在写这篇文章的时候，对于各节的安排是比较犹豫的：如果先讲概念，没有源码流程的讲解很难懂；先讲源码流程，没有概念的认知很难读懂源码。最终还是决定了先讲window的真正概念，先让读者有个整体上的感知。

文章很长，笔者对于window想要讲的都在这篇文章中。

希望文章对你有帮助。

> 笔者才疏学浅，有不同观点欢迎评论区或私信讨论。如需转载请留言告知。
> 另外欢迎阅读笔者的个人博客[一只修仙的猿的个人博客](https://qwerhuan.gitee.io/)，更精美的UI，拥有更好的阅读体验。



----



## 参考文献

- [直面底层：探索 Android 中的 Window](https://mp.weixin.qq.com/s/1j1lsciZFnmh5y_1_CUD8g)
- [直面底层：WindowManager 视图绑定以及体系结构](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650832409&idx=2&sn=a9f16b2134b0a4e7f34a315d6704a401&chksm=80b7aa87b7c02391036ad82825a4b9cbf68483e6d22cb8fc88385076bb485d2e23475ed015eb&scene=21#wechat_redirect)
- [奇妙的Window之旅](https://mp.weixin.qq.com/s/gl5qaHrbXKrvz257QJmQnw)
- [Android进阶必备，Window机制探索](https://mp.weixin.qq.com/s/IrwjQqlDoLp3xQZbthncVg)

《Android开发艺术探索》

《Android进阶解密》



