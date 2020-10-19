---


title: Anroid全面解析之Context机制 	#标题
date: 2020/10/11 00:00:00 	#建立日期
tags: 						#标签
 - android 
 - context
 - android全面解析
categories:  				#分类
 - Android

updated: 					#更新日期
keywords:				#关键词
description:				#文章描述
top_img:					#文章顶部照片
comments: true				#是否显示评论模块
cover:						#文章缩略图
toc: true							#是否显示toc
toc_number: true			#是否显示toc_number
auto_open: true				#是否自动打开toc
copyright: true					#显示文章版权模块
copyright_author: 一只修仙的猿		#文章版权作者
copyright_author_href: 			#文章版权作者链接
copyright_url:						#文章版权文章链接
copyright_info:						#文章版权声明文字
mathjax:
katex:
aplayer:
highlight_shrink: true       #代码框是否打开

---

## 前言

很高兴遇见你~ 欢迎阅读我的文章。

在文章[Android全面解析之由浅及深Handler消息机制](https://blog.csdn.net/weixin_43766753/article/details/108968666)中讨论到，Handler可以：

> 避免我们自己去手动写 死循环和输入阻塞 来不断获取用户的输入以及避免线程直接结束，而是采用事务驱动型设计，使用Handler消息机制，让AMS可以控制整个程序的运行逻辑。

这是关于android程序在设计上更加重要的一部分，不太了解的读者可以前往阅读了解一下。而当我们知道android程序的程序是通过main方法跑起来的，然后通过handler机制来控制程序的运行，那么四大组件和普通的Java类到底有什么区别？为什么同样是Java类，而ActivityThread、Activity等等这些类就显得那么特殊呢？我们的代码、写的布局是通过什么路径使用系统资源把界面展示在屏幕上的？这一切就涉及到我们今天的主角：Context。

## 什么是Context

回想一下最初学习Android开发的时候，第一用到context是什么时候？如果你跟我一样是通过郭霖的《第一行代码》来入门android，那么一般是Toast。Toast的常规用法是：

```kotlin
Toast.makeText(this, "我是toast", Toast.LENGTH_SHORT).show()
```

当初也不知道什么是Context，只知道他需要一个context类型，把activity对象传进去即可。从此context贯穿在我开发过程的方方面面，但我始终不知道这个context到底有什么用？为什么要这个对象？我们首先来看官方对于Context类的注释：

```java
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
public abstract class Context {...}
```

`关于应用程序环境的全局信息的接口。 这是一个抽象类，它的实现是由Android系统提供。 它允许访问特定应用的资源和类，以及向上调用应用程序级的操作，如启动活动，广播和接收Intent等`。

可以看到Context最重要的作用就是获取全局消息、访问系统资源、调用应用程序级的操作。可能对于这些作用没什么印象，想一下，如果没有context，我们如何做到以下操作：

- 弹出一个toast
- 启动一个activity
- 获取程序布局文件、drawable文件等
- 访问数据库

这些平时看似简单的操作，一旦失去了context将无法执行。这些行为都有一个共同点：**需要与系统交汇**。四大组件为什么配为组件，而我们的写的就只能叫做一个普通的Java类，正是因为context的这些功能让四大组件有了不一样的能力。简单来说，context是：

> 应用程序和系统之间的桥梁，应用程序访问系统各种资源的接口。

我们一般使用context最多的是两种情景：直接调用context的方法和调用接口时需要context参数。这些行为都意味着我们需要访问系统相关的资源。

那context是从哪里来的？AMS！AMS是系统级进程，拥有访问系统级操作的权利，应用程序的启动受AMS的调控，在程序启动的过程中，AMS会把一个“凭证”通过跨进程通信给到我们的应用程序，我们的程序会把这个“凭证”封装成context，并提供一系列的接口，这样我们的程序也就可以很方便地访问系统资源了。这样的好处是：

> 系统可以对应用程序级的操作进行调控，限制各种情景下的权限，同时也可以防止恶意攻击。

如Application类的context和Activity的context权利是不一样的，生命周期也不一样。对于想要操作系统攻击用户的程序也进行了阻止，没有获得允许的Java类没有任何权利，而Activity开放给用户也只有部分有限的权利。而我们开发者获取context的路径，也只有从activity、application等组件获取。

因而，什么是Context？Context是应用程序与系统之间沟通的桥梁，是应用程序访问系统资源的接口，同时也是系统给应用程序的一张“权限凭证”。有了context，一个Java类才可以被称之为组件。



## Context家族

上一部分我们了解什么是context以及context的重要性，这一部分就来了解一下context在源码中的子类继承情况。先看一个图：

<img src="https://s1.ax1x.com/2020/10/10/06GYuD.png" border="0" width="500"/>

最顶层是`Context抽象类`，他定义了一系列与系统交汇的接口。`ContextWrapper`继承自Context，但是并**没有真正**实现Context中的接口，而是把接口的实现都托管给`ContextImpl`，ContextImpl是Context接口的真正实现者，从AMS拿来的“凭证”也是封装到了ContextImpl中，然后赋值给ContextWrapper，这里运用到了一种模式：**装饰者模式**。`Application`和`Service`都继承自ContextWrapper，那么他们也就拥有Context的接口方法且本身即是context，方便开发者的使用。`Activity`比较特殊，因为它是有界面的，所以他需要一个主题：Theme，`ContextThemeWrapper`在ContextWrapper的基础上增加与主题相关的操作。

这样的设计有这样的优点：

- Activity等可以更加方便地使用context，可以把自身当成context来使用，遇到需要context的接口直接把自身传进去即可。
- 运用装饰者模式，向外屏蔽ContextImpl的内部逻辑，同时当需要更改ContextImpl的逻辑实现，ContextWrapper的逻辑几乎不需要更改。
- 更方便地扩展不同情景下的逻辑。如service和activity，情景不同，需要的接口方法也不同，但是与系统交互的接口是相同的，使用装饰者模式可以拓展出很多的功能，同时只需要把ContextImpl对象赋值进去即可。



## context的分类

前面讲到Context的家族体系时，了解到他的最终实现类有：Application、Activity、Service，ContextImpl被前三者持有，是Context接口的真正实现。那么这里讨论一下这三者有什么不同，和使用时需要注意的问题。

#### Application

Application是全局Context，整个应用程序只有一个，他可以访问到应用程序的包信息等资源信息。获取Application的方法一般有两个：

```kotlin
context.getApplicationContext()
activity.getApplication()
```

通过context和activity都可以获取到Application，那这两个方法有什么区别？没有区别。我们可以打印来看一下：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    Log.d("一只修仙的猿", "application：$application")
    Log.d("一只修仙的猿", "applicationContext：$applicationContext")
}
```

<img src="https://s1.ax1x.com/2020/10/10/06NOOK.png"  border="0" width="700"/>

可以看到确实是同个对象。但为什么要提供两个一样作用的方法？`getApplication()`方法更加直观，但是只能在activity中调用。`getApplicationContext()`适用范围更广，任意一个context对象皆可以调用此方法。

Application类的Context的特点是生命周期长，在整个应用程序运行的期间他都会存在。同时我们可以自定义Application，并在里面做一些全局的初始化操作，或者写一个静态的context供给全局获取，不需要在方法中传入context。如：

```kotlin
class MyApplication : Application(){
    // 全局context
    companion object{
        lateinit var context: Context
    }
    override fun onCreate() {
        super.onCreate()
        // 做全局初始化操作
        RetrofitManager.init(this)
        context = this
    }
}
```

这样我们就可以在应用启动的时候对一些组件进行初始化，同时可以通过MyApplication.context来获取Application对象。

但是！！！请不要把Application当成工具类使用。由于Application获取的便利性，有开发者会在Application中编写一些工具方法，全局获取使用，这样是不行的。自定义Application的目的是在程序启动的时候做全局初始化工作，而不能拿来取代工具类，这严重违背谷歌设计Application的原则，也违背Java代码规范的单一职责原则。

#### 四大组件

Activity继承自ContextThemeWrapper，是一个拥有主题的context对象。**Activity常用于与UI有关的操作**，如添加window等。常规使用可以直接用**activity.this**。

Service继承自ContextWrapper，也可以和Activity一样直接使用service.this来使用context。和activity不同的是，Service没有界面，所以也不需要主题。

ContextProvider使用的是Application的context，Broadcast使用的是activity的context，这两点在后面会进行源码分析。

#### BaseContext

嗯？baseContext是什么？把这个拿出来单独讲，细心的读者可能会发现activity中有一个方法：`getBaseContext`。这个是ContextWrapper中的mBase对象，也就是ContextImpl，也是context接口的真正逻辑实现。

#### context的使用问题

使用context最重要的问题之一是**注意内存泄露**。不同的context的生命周期不同，Application是在应用存在的期间会一直存在，而Activity是会随着界面的销毁而销毁，如果当我们的代码长时间持有了activity的context，如静态引用或者单例类，那么会导致activity无法被释放。如下面的代码：

```kotlin
object MyClass {
    lateinit var mContext : Context
    fun showToast(context : Context){
        mContext = context
    }
}
```

单例类在应用持续的时间都会一直存在，这样context也就会被一直被持有，activity无法被回收，导致内存泄露。

那，我们就都换成Application不就可以了，如下：

```kotlin
object MyClass {
    lateinit var mContext : Context
    fun showToast(context : Context){
        mContext = context.applicationContext
    }
}
```

答案是：不可以。什么时候可以使用Application？**不涉及UI以及启动Activity操作。**Activity的context是拥有主题属性的，如果使用Application来操作UI，那么会丢失自定义的主题，采用系统默认的主题。同时，有些UI操作只有Activity可以执行，如弹出dialog，这涉及到window的token问题，我在这篇文章[token验证](https://blog.csdn.net/weixin_43766753/article/details/109060496)进行了详细的解答，有兴趣的读者可以去阅读一下。这也是官方对于context不同权限的设计，**没有界面的context，就不应该有操作界面的权利**。使用Application启动的Activity必须指定task以及标记为singleTask，因为Application是没有任务栈的，需要重新开一个新的任务栈。因此，**我们需要根据不同context的不同职责来执行不同的任务**。



## Context的创建过程

经过上面的讨论，读者对于context在心中有了一定的理解。但始终觉得少点什么：activity是什么时候被创建的，他的contextImpl是如何被赋值的？Application呢？为什么说ContextProvider的context是Application，Broadcast的context是Activity？contextImpl又是如何被创建的？解决这些疑惑，就必须阅读源码了。阅读源码的好处非常多，上面我的讲述，都是基于我阅读源码之后的理解，而“一千个观众有一千个哈姆雷特”，阅读源码可以**形成自己对整个机制自己的思考和理解**，同时可以让自己对context那些知识真正落实到代码上，**增强自己对知识的自信心**。当别人和你意见不同的时候，你可以拍拍胸脯说：我看过源码，这个地方就是这样。是不是非常自信且傲娇？

然而阅读源码不是越多越好，而是**把握整体的流程之后阅读关键源码**，不要深入源码堆中无法自拔。例如我觉得activity的contextImpl是在Activity创建的过程中被赋值的，那么我就会去找activity的启动流程源码，然后只看和context有关的部分。提高效率的同时，还可以切中我们学习的点。下面的源码阅读我们给出整体流程，然后重点理解关键代码，其他的源码读者可自行下载源码去跟踪阅读一下。

#### Application

Application应用级别的context，是在应用被创建的时候被创建的，是第一个被创建的context，也是最后一个被销毁的context。因而追踪Application的创建需要从应用程序的启动流程看起。应用启动的源码流程如下（简化版）：

<img src="https://s1.ax1x.com/2020/10/11/0c5r3n.png"  border="0" width=500/>

应用程序从ActivityThread的main方法开始执行，从[Handler消息机制](https://blog.csdn.net/weixin_43766753/article/details/108968666)中我们知道main方法主要是开启线程的Looper以及handler，然后由AMS向主线程发送message控制应用的启动过程。因而我们可以把目标锁定在图中的最后一个方法：`handleBindApplication`，Application最有可能在这里被创建：

```java
ActivityThread.class (api29)

private void handleBindApplication(AppBindData data) {
    ...
	// 创建LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ...
    Application app;
    ...
    try {
		// 创建Application
        app = data.info.makeApplication(data.restrictedBackupMode, null);
        ...
    }
    try {
        ...
		// 回调Application的onCreate方法
        mInstrumentation.callApplicationOnCreate(app);
    }
    ...
}
```

`handleBindApplication`的参数AppBindData是AMS给应用程序的启动信息，其中就包含了“权限凭证”——ApplicationInfo等。LoadedApk就是通过这些对象来创建获取对系统资源的访问权限，然后通过LoadApk来创建ContextImpl以及Application。

这里我们只关注和context创建有关的逻辑，前面启动程序的源码以及AMS如何处理，这里就不讲了，读者有兴趣可以读[ContextProvider启动流程](https://blog.csdn.net/weixin_43766753/article/details/108110605)这篇文章，其中对ContextProvider的启动过程就有对上述源码进行追踪详解。

那么接下来我们继续关注Application是如何创建的：

```java
LoadeApk.class(api29)
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    // 如果application已经存在则直接返回
    if (mApplication != null) {
        return mApplication;
    }
	...
    Application app = null;
    String appClass = mApplicationInfo.className;
    ...
    try {
        java.lang.ClassLoader cl = getClassLoader();
       ...
		// 创建ContextImpl
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
		// 利用类加载器加载我们在AndroidMenifest指定的Application类
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        // 把Application的引用给comtextImpl，这样contextImpl也可以很方便地访问Application
        appContext.setOuterContext(app);
    } 
    ...
    mActivityThread.mAllApplications.add(app);
   	// 把app设置为mApplication，当我们调用context.getApplicationContext就是获取这个对象
    mApplication = app;

    if (instrumentation != null) {
        try {
			// 回调Application的onCreate方法
            instrumentation.callApplicationOnCreate(app);
        } 
        ...
    }
 	...
    return app;
}
```

代码的逻辑也不复杂，首先判断LoadedApk对象中的mApplication是否存在，否则创建ContextImpl，再利用类加载器和contextImpl创建Application，最后把Application对象赋值给LoadedApk的mApplication，再回调Application的onCreate方法。我们先来看一下contextImpl是如何创建的：

```java
ContextImpl.class(api29)
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo,
        String opPackageName) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
            null, opPackageName);
    context.setResources(packageInfo.getResources());
    return context;
}
```

这里直接new了一个ContextImpl，同时给ContextImpl赋值访问系统资源相关的“权限”对象——ActivityThread，LoadedApk等。让我们再回到Application的创建过程。我们可以猜测，在`newApplication`包含的逻辑肯定有：利用反射创建Application，再把contextImpl赋值给Application。原因是每个人自定义的Application类不同，需要利用反射来创建对象，其次Application中的mBase属性是对ContextImpl的引用。看源码：

```java
Instrumentation.class(api29)
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    app.attach(context);
    return app;
}

Application.class(api29)
final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}

ContextWrapper.class(api29)
Context mBase;    
protected void attachBaseContext(Context base) {
    if (mBase != null) {
        throw new IllegalStateException("Base context already set");
    }
    mBase = base;
}    
```

结果非常符合我们的猜测，先创建Application对象，再把ContextImpl通过Application的attach方法赋值给Application。然后Application的attach方法调用了ContextWrapper的attachBaseContext方法，因为Application也是继承自ContextWrapper。这样，就把ContextImpl赋值给Application的mBase属性了。

再回到前面的逻辑，创建了Application之后需要回调onCreate方法：

```java
Instrumentation.class(api29)
public void callApplicationOnCreate(Application app) {
    app.onCreate();
}
```

简单粗暴，直接回调。到这里，Application的创建以及context的创建流程就走完了。但是需要注意的是，**全局初始化需要在onCreate中进行，而不要在Application的构造器中执行**。从代码中我们可以看到ContextImpl是在Application被创建之后再赋值的。

#### Activity

Activity的context也是在Activity创建的过程中被创建的，这个就涉及到Activity的启动流程，这里涉及到三个流程：应用程序请求AMS，AMS处理请求，应用程序响应Activity创建事务：

<img src="https://s1.ax1x.com/2020/08/02/aYQwF0.png"  border="0" width = 800/>

依然，我们专注于Activity的创建流程，其他的读者可阅读[Activity启动流程](https://blog.csdn.net/weixin_43766753/article/details/107746968)这篇文章了解。和Application一样，Activity的创建时由AMS来控制的，AMS向应用程序进程发送消息来执行具体的启动逻辑。最后会执行到`handleLaunchActivity`这个方法：

```java
ActivityThread.class(api29)
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
    final Activity a = performLaunchActivity(r, customIntent);
	...
   return a;
}
```

最终的就是中间这句代码，进入看源码：

```java
ActivityThread.class(api29)
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
	// 创建Activity的ContextImpl
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        // 利用类加载创建activity实例
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        ...
    }
    try {
		// 创建Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
		...
        if (activity != null) {
            ...
			// 把activity设置给context，这样context也可以访问到activity了
            appContext.setOuterContext(activity);
            // 调用activity的attach方法把contextImpl设置给activity
            activity.attach(appContext, this, getInstrumentation(), r.token,
                            r.ident, app, r.intent, r.activityInfo, title, r.parent,
                            r.embeddedID, r.lastNonConfigurationInstances, config,
                            r.referrer, r.voiceInteractor, window, r.configCallback,
                            r.assistToken);

            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                // 设置主题
                activity.setTheme(theme);
            }
            ...
			// 回调onCreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
        }
        ...
    }
	...
    return activity;
}
```

代码的逻辑不是很复杂，首先创建Activity的ContextImpl，利用类加载创建activity实例，然后再通过LoadedApk创建Application，这个方法在前面我们讲过，如果Application已经创建会直接返回已经创建的对象。然后把activity设置给context，这样context也可以访问到activity了。这里要注意，前面讲到使用Activity的context会造成内存泄露，那么可不可以用Activity的contextImpl对象呢？答案是不可以，因为**ContextImpl也会持有Activity的引用**，需要特别注意一下。随后再调用activity的attach方法把contextImpl设置给activity。后面是设置主题和回调onCreate方法，我们就不深入了，主要看看attach方法：

```java
Activity.class(api29)
final void attach(Context context,...) {
    attachBaseContext(context);
 	...   
}
```

这里省略了大量的代码，只保留关键一句：`attachBaseContext`，是不是很熟悉？调用ContextWrapper的方法来给mBase属性赋值，和前面Application是一样的，就不再赘述。

#### Service

依然只关注关键代码流程，先看Service的启动流程图：

<img src="https://s1.ax1x.com/2020/08/07/afbYYq.png"  border="0" width=600/>

Service的创建过程也是受AMS的控制，同样我们看到创建Service的那一步，最终会调用到`handleCreateService`这个方法：

```java
private void handleCreateService(CreateServiceData data) {
    ...
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
    } 
    ...
    try {
        ...
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        service.onCreate();
        mServices.put(data.token, service);
        ...
    } 
    ...
}
```

Service的逻辑就相对简单了，同样创建service实例，再创建contextImpl，最后把contextImpl通过Service的attach方法赋值给mBase属性，最后回调Service的onCreate方法。过程和上面的很像，这里就不再深入讲了，感兴趣的读者可自行去阅读源码，也可以阅读[Android中Service的启动与绑定过程详解（基于api29）](https://blog.csdn.net/weixin_43766753/article/details/107881248)这篇文章了解Service的详细内容。

#### Broadcast

Broadcast和上面的组件不同，他并不是继承自Context，所以他的Context是需要通过上述三者来给予。我们一般使用广播的context是在接受器中，如：

```kotlin
class MyClass :BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        TODO("use context")
    }
}
```

那么onReceive的context对象是从哪里来的呢？同样我们先看广播接收器的注册流程：

<img src="https://s1.ax1x.com/2020/08/17/dnSOMQ.png" alt="Broadcast注册源码流程.png" border="0" width=600/>

同样，详细的广播相关工作流程可以阅读[Android广播Broadcast的注册与广播源码过程详解（基于api29）](https://blog.csdn.net/weixin_43766753/article/details/108066203)这篇文章了解。因为在创建Receiver的时候并没有传入context，所以我们需要追踪他的注册流程，看看在哪里获取了context。我们先看到ContextImpl的`registerReceiver`方法：

```java
ContextImpl.class(api29)
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {
    // 注意参数
    return registerReceiverInternal(receiver, getUserId(),
            filter, broadcastPermission, scheduler, getOuterContext(), 0);
}
```

registerReceiver方法最终会来到这个重载方法，我们可以注意到，这里有个`getOuterContext`，这个是什么？还记得Activity的context创建过程吗？这个方法获取的就是activity本身。我们继续看下去：

```java
ContextImpl.class(api29)
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context, int flags) {
    IIntentReceiver rd = null;
    if (receiver != null) {
        if (mPackageInfo != null && context != null) {
            ...
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        }
        ...
    }
    ...
}
```

这里利用context创建了ReceiverDispatcher，我们继续深入看：

```java
LoadedApk.class(api29)
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
        Context context, Handler handler,
        Instrumentation instrumentation, boolean registered) {
    synchronized (mReceivers) {
        LoadedApk.ReceiverDispatcher rd = null;
        ...
        if (rd == null) {
            rd = new ReceiverDispatcher(r, context, handler,
                    instrumentation, registered);
            ...
        }
        ...
    }
}

ReceiverDispatcher.class(api29)
ReceiverDispatcher(..., Context context,...) {
    ...
    mContext = context;
    ...
}
```

这里确实把receiver和context创建了ReceiverDispatcher，嗯?怎么没有给Receiver?其实这涉及到广播的内部设计结构。Receiver是没有跨进程通信能力的，而广播需要AMS的调控，所以必须有一个可以跟AMS沟通的对象，这个对象是InnerReceiver，而ReceiverDispatcher就是负责维护他们两个的联系，如下图：

<img src="https://s1.ax1x.com/2020/10/11/0gkmW9.png" border="0" width=300/>

而onReceive方法也是由ReceiverDispatcher回调的，最后我们再看到回调onReceive的那部分代码：

```java
ReceiverDispatcher.java/Args.class;
public final Runnable getRunnable() {
    return () -> {
        ...;
        try {
            ...;
            // 可以看到这里回调了receiver的方法，这样整个接收广播的流程就走完了。
            receiver.onReceive(mContext, intent);
        }
    }
}
```

Args是Receiver的内部类，mContext就是在创建ReceiverDispatcher时传入的对象，到这里我们就知道这个对象确实是Activity了。

但是，，**不一定每个都是Activity**。在源码中我们知道是通过`getOuterContext`来获取context，如果是通过别的context注册广播，那么对应的对象也就不同了，只是我们一般都是在Activity中创建广播，所以这个context一般是activity对象。

#### ContentProvider

ContextProvider我们用的就比较少了，内容提供器主要是用于应用间内容共享的。虽然ContentProvider是由系统创建的，但是他本身并不属于Context家族体系内，所以他的context也是从其他获取的。老样子，先看ContentProvider的创建流程：

<img src="https://s1.ax1x.com/2020/10/11/0gYQXV.png"  border="0" width=500/>

咦？这不是Application创建的流程图吗？是的，ContentProvider是伴随着应用启动被创建的，来看一张更加详细的流程图：

<img src=https://s1.ax1x.com/2020/08/19/d125lt.png width=600>

我们把目光聚集到ContentProvider的创建上，也就是`installContentProviders`方法。同样，详细的ContentProvider工作流程可以访问[Android中ContentProvider的启动与请求源码流程详解（基于api29）](https://blog.csdn.net/weixin_43766753/article/details/108110605)这篇文章。`installContentProviders`是在handleBindApplication中被调用的，我们看到调用这个方法的地方：

```java
private void handleBindApplication(AppBindData data) {
    try {
        // 创建Application
        app = data.info.makeApplication(data.restrictedBackupMode, null);
		...
        if (!data.restrictedBackupMode) {
            if (!ArrayUtils.isEmpty(data.providers)) {
                // 安装ContentProvider
                installContentProviders(app, data.providers);
        }
    }    
}
```

可以看到这里传入了application对象，我们继续看下去：

```java
private void installContentProviders(
        Context context, List<ProviderInfo> providers) {
    final ArrayList<ContentProviderHolder> results = new ArrayList<>();
    for (ProviderInfo cpi : providers) {
        ...
        ContentProviderHolder cph = installProvider(context, null, cpi,
                false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
        ...
    }
...
}
```

这里调用了installProvider，继续往下看：

```java
private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    ContentProvider localProvider = null;
    IContentProvider provider;
    if (holder == null || holder.provider == null) {
        ...
		// 这里c最终是由context构造的
        Context c = null;
        ApplicationInfo ai = info.applicationInfo;
        if (context.getPackageName().equals(ai.packageName)) {
            c = context;
        }
        ...
        try {
            // 创建ContentProvider
            final java.lang.ClassLoader cl = c.getClassLoader();
            LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
            ...
            localProvider = packageInfo.getAppFactory()
                    .instantiateProvider(cl, info.name);
            provider = localProvider.getIContentProvider();
            ...
			// 把context设置给ContentProvider
            localProvider.attachInfo(c, info);
        } 
        ...
    } 
    ...
}
```

这里最重要的一行代码是`localProvider.attachInfo(c, info);`，在这里把context设置给了ContentProvider，我们再深入一点看看：

```java
ContentProvider.class(api29)
public void attachInfo(Context context, ProviderInfo info) {
    attachInfo(context, info, false);
}
private void attachInfo(Context context, ProviderInfo info, boolean testing) {
    ...
    if (mContext == null) {
        mContext = context;
        ...
    }
    ...
}
```

这里确实把context赋值给了ContentProvider的内部变量mContext，这样ContentProvider就可以使用Context了。而这个context正是一开始传进来的Application。



## 从源码设计角度看Context

到这里关于Context的知识也讲得差不多了。研究Framework层知识，不能只停留在他是什么，有什么作用即可。Framework层他是一个整体，构成了android这个庞大的体系，还需要看Context，在其中扮演着什么样的角色，解决了什么样的问题。在[window机制](https://blog.csdn.net/weixin_43766753/article/details/108350589)中我讲到window的存在是为了解决屏幕上view的显示逻辑与触摸反馈问题，在[Hanlder机制](https://blog.csdn.net/weixin_43766753/article/details/108968666)中我写到整个android程序都是基于Handler机制来驱动执行的，而Context呢？

Android系统是一个完整的生态，他搭建了一个环境，让各种程序可以运行在上面。而任何一个程序，想要运行在这个环境上，必须得到系统的允许，也就是**软件安装**。安卓与电脑不同的是，他不是任意一个程序就可以直接访问到系统的资源。我们在window上可以写一个java程序，然后直接开启一个文件流就可以读取和修改文件了。而Android没这么简单，他**任意一个程序的运行都必须经过系统的调控**。也就是，即时程序获得允许（安装在手机上了)，程序本身要运行，还得是系统来控制程序运行，程序无法自发地执行在Android环境中。我们通过源码可以知道程序的main方法，仅仅只是开启了线程的Looper循环，而后续的一切，都必须等待AMS来控制。

那应用程序自己硬要执行可不可以？可以，但是没卵用。想要获得系统资源，如启动四大组件、读取布局文件、读写数据库、调用系统柜摄像头等等，都必须要通过Context，而context必须要通过AMS来获取。这就区分了一个程序是一个普通的Java程序，还是android程序。

Context承受的两大重要职责是：身份权限、程序访问系统的接口。一个Java类，如果没有context那么就是一个普通的Java类，而当他获得context那么他就可以称之为一个组件了，因为它获得了访问系统的权限，他不再是一个普通的身份，是属于android“公民”了。而“公民”并不是无法无天，系统也可以通过context来封装以及限制程序的权限。要想弹出一个通知，你必须通过这个api，用户关闭你的通知权限，你就别想通过第二条路来弹出通知了。同时 程序也无需知道底层到底是如何实现，只管调用api即可。四大组件为何称为四大组件，因为他们生来就有了context，特别是activity和service，包括Application。而我们写的一切程序，都必须间接或者直接从其中获取context。

总而言之，context就是负责区分android内外程序的一个机制，限制程序访问系统资源的权限。



## 总结

文章从什么是context开始介绍，再针对context的不同子类进行解析，最后结合源码深入地讲解了context的创建过程。最后再谈了我对context的设计理解。

关于context想说的就已经说完了。虽然这些内容日常很少用得到，但是非常有助于我们对Android整个系统框架的理解。而当我们对系统有更加深入的理解后，写出来的程序也就会更加健壮。

希望文章对你有帮助。

> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者能力有限，有任何想法欢迎评论区交流指正。
> 如需转载请私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)