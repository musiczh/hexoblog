---
title: Android之window机制token验证 	#标题
date: 2020/10/13 00:00:00 	#建立日期
summary: 					#文章摘要
tags: 						#标签
 - android 
 - window
categories:  				#分类
 - Android

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

很高兴遇见你~ 欢迎阅读我的文章

这篇文章讲解关于window token的问题，同时也是[Context机制](https://blog.csdn.net/weixin_43766753/article/details/109017196)和[Window机制](https://blog.csdn.net/weixin_43766753/article/details/108350589)这两篇文章的一个补充。如果你对Android的Window机制和Context机制目前位了解过，**强烈建议**你先阅读前面两篇文章，可以帮助理解整个源码的解析过程以及对token的理解。同时文章涉及到Activty启动流程源码，读者可先阅读[Activity启动流程](https://blog.csdn.net/weixin_43766753/article/details/107746968)这篇文章。文章涉及到这些方面的内容默认读者已经阅读且了解，不会对这方面的内容过多阐述，如果遇到一些内容不理解，可以找到对应的文章看一下。那么，我们开始吧。

当我们想要在屏幕上展示一个Dialog的时候，我们可能会在Activity的onCreate方法里这么写：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    val dialog = AlertDialog.Builder(this)
    dialog.run{
        title = "我是标题"
        setMessage("我是内容")
    }
    dialog.show()
}
```

他的构造参数需要一个context对象，但是这个context不能是ApplicationContext等其他context，只能是ActivityContext（当然没有ApplicationContext这个类，也没有ActivityContext这个类，这里这样写只是为了方便区分context类型，下同）。这样的代码运行时没问题的，如果我们使用Application传入会怎么样呢？

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    // 注意这里换成了ApplicationContext
    val dialog = AlertDialog.Builder(applicationContext)
    ...
}
```

运行一下：

<img src="https://s1.ax1x.com/2020/10/10/06ZcB6.png"  border="0" />

报错了，原因是`You need to use a Theme.AppCompat theme (or descendant) with this activity.`，那我们给他添加一个Theme：

```java
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    // 注意这里添加了主题
    val dialog = AlertDialog.Builder(applicationContext,R.style.AppTheme)
    ...
}
```

好了再次运行：

<img src="https://s1.ax1x.com/2020/10/10/06eZ8J.png"  border="0" />

嗯嗯？又崩溃了，原因是：`Unable to add window -- token null is not valid; is your activity running?`token为null？这个token是什么？为什么同样是context，使用activity没问题，用ApplicationContext就出问题了？他们之间有什么区别？那么这篇文章就围绕这个token来展开讨论一下。

> 文章采用思考问题的思路来展开讲述，我会根据我学习这部分内容时候的思考历程进行复盘。希望这种解决问题的思维可以帮助到你。
> 对token有一定了解的读者可以看到最后部分的整体流程把握，再选择想阅读的部分仔细阅读。

## 什么是token

首先我们看到报错是在`ViewRootImpl.java:907`，这个地方肯定有进行token判断，然后抛出异常，这样我们就能找到token了，那我们直接去这个地方看看。：

```java
ViewRootImpl.class(api29)
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    int res;
    ...
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                        mTempInsets);
    ...
    if (res < WindowManagerGlobal.ADD_OKAY) {
        ...
        switch (res) {
            case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
            case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                /*
                *	1
                */
                throw new WindowManager.BadTokenException(
                    "Unable to add window -- token " + attrs.token
                    + " is not valid; is your activity running?");    
                ...
        }
        ...
    }
    ...
}
```

我们看到代码就是在注释1的地方抛出了异常，是根据一个变量`res`来判断的，这个`res`来自方法`addToDisplay`，那么token的判断肯定在这个方法里面了，`res`只是一个 判断的结果，那么我们需要进到这个`addToDisplay`里去看一下。mWindowSession的类型是IWindowSession，他是一个接口，那他的实现类是什么？找不到实现类就无法知道他的具体代码。这里涉及到window机制的相关内容，简单讲一下：

> WindowManagerService是系统服务进程，应用进程跟window联系需要通过跨进程通信：AIDL，这里的IWindowSession只是一个Binder接口，他的具体实现类在系统服务进程的Session类。所以这里的逻辑就跳转到了Session类的`addToDisplay`方法中。关于window机制更加详细的内容，读者可以阅读[Android全面解析之Window机制](https://blog.csdn.net/weixin_43766753/article/details/108350589)这篇文章进一步了解，限于篇幅这里不过多讲解。

那我们继续到Session的方法中看一下：

```java
Session.class(api29)
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
   	final WindowManagerService mService; 
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
            Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
            InsetsState outInsetsState) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel,
                outInsetsState);
    }
}
```

可以看到，Session确实是继承自接口IWindowSession，因为WMS和Session都是运行在系统进程，所以不需要跨进程通信，直接调用WMS的方法：

```java
public int addWindow(Session session, IWindow client, int seq,
        LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
        InsetsState outInsetsState) {
   	...
    WindowState parentWindow = null;
    ...
	// 获取parentWindow
    parentWindow = windowForClientLocked(null, attrs.token, false);
    ...
    final boolean hasParent = parentWindow != null;
    // 获取token
    WindowToken token = displayContent.getWindowToken(
        hasParent ? parentWindow.mAttrs.token : attrs.token);
    ...
  	// 验证token
    if (token == null) {
    if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
          Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                           + attrs.token + ".  Aborting.");
            return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
        }
       ...//各种验证
    }
    ...
}
```

WMS的addWindow方法代码这么多怎么找到关键代码？还记得viewRootImpl在判断res是什么值的情况下抛出异常吗？没错是`WindowManagerGlobal.ADD_BAD_APP_TOKEN和WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN`，我们只需要找到其中一个就可以找到token的判断位置，从代码中可以看到，当token==null的时候，会进行各种判断，第一个返回的就是`WindowManagerGlobal.ADD_BAD_APP_TOKEN`，这样我们就顺利找到token的类型：**WindowToken**。那么根据我们这一路跟过来，终于找到token的类型了。再看一下这个类：

```java
class WindowToken extends WindowContainer<WindowState> {
    ...
    // The actual token.
    final IBinder token;
}
```

官方告诉我们里面的token变量才是真正的token，而这个token是IBinder对象。

好了到这里关于token是什么已经弄清楚了：

> - token是一个IBinder对象
> - 只有利用token才能成功添加dialog

那么接下来就有更多的问题需要思考了：

- Dialog在show过程中是如何拿到token并给到WMS验证的？
- 这个token在activity和application两者之间有什么不同？
- WMS怎么知道这个token是合法的，换句话说，WMS怎么验证token的？

## dialog如何获取到context的token的？

首先，我们解决第一个问题：Dialog在show过程中是如何拿到token并给到WMS验证的？

我们知道导致两种context（activity和application）弹出dialiog的不同结果，原因在于token的问题。那么在弹出Dialog的过程中，他是如何拿到context的token并给到WMS验证的？源码内容很多，我们需要先看一下token是封装在哪个参数被传输到了WMS,确定了参数我们的搜索范围就减小了，我们回到WMS的代码：

```java
parentWindow = windowForClientLocked(null, attrs.token, false);
WindowToken token = displayContent.getWindowToken(
        hasParent ? parentWindow.mAttrs.token : attrs.token);
```

我们可以看到token和一个`attrs.token`关系非常密切，而这个attrs从调用栈一路往回走到了viewRootImpl中：

```java
ViewRootImpl.class(api29)
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
   ...
}
```

可以看到这是一个WindowManager.LayoutParams类型的对象。那我们接下来需要从最开始`show()`开始，追踪这个token是如何被获取到的：

```java
Dialog.class(api30)
public void show() {
    ...
    WindowManager.LayoutParams l = mWindow.getAttributes();
    ...
    mWindowManager.addView(mDecor, l);
    ...
}
```

这里的`mWindow`和`mWindowManager`是什么？我们到Dialog的构造函数一看究竟：

```java
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    // 如果context没有主题，需要把context封装成ContextThemeWrapper
    if (createContextThemeWrapper) {
        if (themeResId == Resources.ID_NULL) {
            final TypedValue outValue = new TypedValue();
            context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
            themeResId = outValue.resourceId;
        }
        mContext = new ContextThemeWrapper(context, themeResId);
    } else {
        mContext = context;
    }
    // 初始化windowManager
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    // 初始化PhoneWindow
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    ...
    // 把windowManager和PhoneWindow联系起来
    w.setWindowManager(mWindowManager, null, null);
    ...
}
```

初始化的逻辑我们看重点就好：首先判断这是不是个有主题的context，如果不是需要设置主题并封装成一个ContextThemeWrapper对象，这也是为什么我们文章一开始使用application但是没有设置主题会抛异常。然后获取windowManager，注意，这里是重点，也是我当初看源码的时候忽略的地方。这里的context可能是Activity或者Application，他们的`getSystemService`返回的windowManager是一样的吗，看代码：

```java
Activity.class(api29)
public Object getSystemService(@ServiceName @NonNull String name) {
    if (getBaseContext() == null) {
        throw new IllegalStateException(
                "System services not available to Activities before onCreate()");
    }
    if (WINDOW_SERVICE.equals(name)) {
        // 返回的是自身的WindowManager
        return mWindowManager;
    } else if (SEARCH_SERVICE.equals(name)) {
        ensureSearchManager();
        return mSearchManager;
    }
    return super.getSystemService(name);
}

ContextImpl.class(api29)
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

**Activity返回的其实是自身的WindowManager，而Application是调用ContextImpl的方法，返回的是应用服务windowManager**。这两个有什么不同，我们暂时不知道，先留意着，再继续把源码看下去寻找答案。我们回到前面的方法，看到`mWindowManager.addView(mDecor, l);`我们知道一个PhoneWindow对应一个WindowManager，这里使用的WindowManager并不是Dialog自己创建的WindowManager，而是参数context的windowManager，也意味着并没有使用自己创建的PhoneWindow。Dialog创建PhoneWindow的目的是为了使用DecorView模板，我们可以看到addView的参数里并不是window而只是mDecor。

我们继续看代码，，同时要注意这个`l`参数，最终token就是封装在里面。`addView`方法最终会调用到了`WindowManagerGlobal`的`addView`方法，具体调用流程可以看我文章开头的文章：

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ...
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    }
	...
    ViewRootImpl root;
    ...
    root = new ViewRootImpl(view.getContext(), display);
	...
    try {
        root.setView(view, wparams, panelParentView);
    } 
    ...
}
```

这里我们只看WindowManager.LayoutParams参数，parentWindow是与windowManagerPhoneWindow，所以这里肯定不是null，进入到`adjustLayoutParamsForSubWindow`方法进行调整参数。最后调用ViewRootImpl的setView方法。到这里WindowManager.LayoutParams这个参数依旧没有被设置token，那么最大的可能性就是在`adjustLayoutParamsForSubWindow`方法中了，马上进去看看：

```java
Window.class(api29)
void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
    CharSequence curTitle = wp.getTitle();
    if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
            wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
        // 子窗口token获取逻辑
        if (wp.token == null) {
            View decor = peekDecorView();
            if (decor != null) {
                wp.token = decor.getWindowToken();
            }
        }
        ...
    } else if (wp.type >= WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
        // 系统窗口token获取逻辑
        ...
    } else {
        // 应用窗口token获取逻辑
        if (wp.token == null) {
            wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
        }
        ...
    }
    ...
}
```

终于看到了token的赋值了，这里分为三种情况：应用层窗口、子窗口和系统窗口，分别进行token赋值。

应用窗口直接获取的是与WindowManager对应的PhoneWindow的mAppToken，而子窗口是拿到DecorView的token，系统窗口属于比较特殊的窗口，使用Application也可以弹出，但是需要权限，这里不深入讨论。而这里的关键就是：**这个dialog是什么类型的窗口？以及windowManager对应的PhoneWindow中有没有token？**

而这个判断跟我们前面赋值的不同WindowManagerImpl有直接的关系。那么这里，就必须到Activity和Application创建WindowManager的过程一看究竟了。

## Activity与Application的WindowManager

首先我们看到Activity的window创建流程。这里需要对Activity的启动流程有一定的了解，有兴趣的读者可以阅读[Activity启动流程](https://blog.csdn.net/weixin_43766753/article/details/107746968)。追踪Activity的启动流程，最终会到ActivityThread的`performLaunchActivity`：

```java
ActivityThread.class(api29)
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
	// 最终会调用这个方法来创建window
    // 注意r.token参数
    activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback,
        r.assistToken);
    ...
}
```

这个方法调用了activity的attach方法来初始化window，同时我们看到参数里有了`r.token`这个参数，这个token最终会给到哪里，我们赶紧继续看下去：

```java
Activity.class(api29)
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    ...
	// 创建window
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ...
	// 创建windowManager
    // 注意token参数
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    mWindowManager = mWindow.getWindowManager();
    ...
}
```

attach方法里创建了PhoneWindow以及对应的WindowManager，再把创建的windowManager给到activity的mWindowManager属性。我们看到创建WindowManager的参数里有token，我们继续看下去：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated;
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

这里利用应用服务的windowManager给Activity创建了WindowManager，同时把token保存在了PhoneWindow内。到这里我们知道Activity的PhoneWindow是拥有token的。那么Application呢？

----

Application的创建具体流程可以看到这篇文章[contentProvider启动流程](https://blog.csdn.net/weixin_43766753/article/details/108110605)，因为contentProvider是伴随着应用的启动而启动的。最终Application的启动会来到ActivityThread的`handleBindApplication`方法：

```java
/frameworks/base/core/java/android/app/ActivityThread.java
private void handleBindApplication(AppBindData data) {
    ...
    // 获取到Application实例
    Application app;
    ...
    try {
        app = data.info.makeApplication(data.restrictedBackupMode, null);
        ...
    }
    ...
}
```

这里的`data.info`是LoadedApk对象，我们深入这个方法看一下：

```java
LoadedApk.java(api29)
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }
    ...
    Application app = null;
    ...
    try {
        java.lang.ClassLoader cl = getClassLoader();
        ...
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } 
    ...
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app);
        } 
        ...
    }
    ...
    return app;
}
```

构建Application的逻辑也不复杂。首先判断Application如果存在则直接返回。后面通过Instrumentation来创建Application对象，最后再通过Instrumentation来回调Application的onCreate方法，我们分别看一下这两个方法：

```java
Instrumentation.java(api29)
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    app.attach(context);
    return app;
}
public void callApplicationOnCreate(Application app) {
    app.onCreate();
}

Application.java
final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}
```

我们可以看到，直到Application回调onCreate方法，整个过程没有涉及token的赋值。Activity是在attach方法中传入了token参数，而这里Application并没有，所以Application并没有token，也没有初始化PhoneWindow和WindowManager。那么Application的getSystemService返回的是什么呢？Application调用的是ContextImpl的getSystemService方法，而这个方法返回的是应用服务的windowManager，Application本身并没有创建自己的PhoneWindow和WindowManager，所以也没有给PhoneWindow赋值token的过程。

因此，**Activity拥有自己PhoneWindow以及WindowManager，同时它的PhoneWindow拥有token；而Application并没有自己的PhoneWindow，他返回的WindowManager是应用服务windowManager，并没有赋值token的过程**。

那么到这里结论已经快要出来了，还差最后一步，我们回到赋值token的那个方法中：

```java
Window.class(api29)
void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
    if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
            wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
        // 子窗口token获取逻辑
        if (wp.token == null) {
            View decor = peekDecorView();
            if (decor != null) {
                wp.token = decor.getWindowToken();
            }
        }
        ...
    } else {
        // 应用窗口token获取逻辑
        if (wp.token == null) {
            wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
        }
        ...
    }
    ...
}
```

当我们使用Activity来添加dialog的时候，Activity本身是带有token的，Dialog是属于应用级窗口（至于为什么读者可以前往dialog的show方法中跟踪源码），他的window层级数是2，而Activity的界面是1，所以他会显示在Activity之上。而因为这里Dialog是应用级窗口，所以他最终就拿到了Activity的token。

但是如果使用的是Application，因为它内部并没有token，那么这里获取到的token就是null，后面到WMS也就会抛出异常了。而这也就是为什么使用Activity可以弹出Dialog而Application不可以的原因。因为受到了token的限制。

## WMS是如何验证token的

到这里我们已经知道。我们从WMS的token判断找到了token的类型以及token的载体：WindowManager.LayoutParams，然后我们再从dialog的创建流程追到了赋值token的时候会因为windowManager的不同而不同。因此我们再去查看了两者不同的windowManager，最终得到结论**Activity的PhoneWindow拥有token，而Application使用的是应用级服务windowManager，并没有token**。

那么此时还是会有疑问：

- token到底是在什么时候被创建的？
- WMS怎么知道我这个token是合法的？

虽然到目前我们已经弄清原因，但是知识却少了一块，秉着探索知识的好奇心我们继续研究下去。

我们从前面Activity的创建window过程知道token来自于`r.token`，这个`r`是ActivityRecord，是AMS启动Activity的时候传进来的Activity信息。那么要追踪这个token的创建就必须顺着这个`r`的传递路线一路回溯。同样这涉及到Activity的完整启动流程，我不会解释详细的调用栈情况，默认你清楚activity的启动流程，如果不清楚，可以先去阅读[Activity的启动流程](https://blog.csdn.net/weixin_43766753/article/details/107746968)。首先看到这个ActivityRecord是在哪里被创建的：

```java
/frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java/;
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client);
    // ClientTransactionHandler是ActivityThread实现的接口，具体逻辑回到ActivityThread
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

这样我们需要继续往前回溯，看看这个token是在哪里被获取的：

```java
/frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
    
public void execute(ClientTransaction transaction) {
    ...
    executeCallbacks(transaction);
    ...
}
public void executeCallbacks(ClientTransaction transaction) {
    ...
        final IBinder token = transaction.getActivityToken();
        item.execute(mTransactionHandler, token, mPendingActions);
    ...
}
```

可以看到我们的token在ClientTransaction对象获取到。ClientTransaction是AMS传来的一个事务，负责控制activity的启动，里面包含两个item，一个负责执行activity的create工作，一个负责activity的resume工作。那么这里我们就需要到ClientTransaction的创建过程一看究竟了。下面我们的逻辑就要进入系统进程了：

```java
ActivityStackSupervisor.class(api28)
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
    boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
            r.appToken);
    ...
}
```

这个方法创建了ClientTransaction，但是token并不是在这里被创建的，我们继续往上回溯（注意代码的api版本，不同版本的代码会不同）：

```java
ActivityStarter.java(api28)
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        SafeActivityOptions options,
        boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
        TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
    ...
  
    //记录得到的activity信息
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, checkedOptions, sourceRecord);
   ...
}
```

我们一路回溯,终于看到了ActivityRecord的创建，我们进去构造方法中看看有没有token相关的构造：

```java
ActivityRecord.class(api28)
ActivityRecord(... Intent _intent,...) {
    appToken = new Token(this, _intent);
    ...
}

static class Token extends IApplicationToken.Stub {
   ...
    Token(ActivityRecord activity, Intent intent) {
        weakActivity = new WeakReference<>(activity);
        name = intent.getComponent().flattenToShortString();
    }
    ...
}
```

可以看到确实这里进行了token创建。而这个token看接口就知道是个Binder对象，他持有ActivityRecord的弱引用，这样可以访问到activity的所有信息。到这里token的创建我们也找到了。那么WMS是怎么知道一个token是否合法呢？每个token创建后，会在后续发送到WMS ，WMS对token进行缓存，而后续对于应用发送来的token只需要在缓存拿出来匹配一下就知道是否合法了。那么WMS是怎么拿到token的？

-----

activity的启动流程后续会走到一个方法：`startActivityLocked`,这个方法在我前面的activity启动流程并没有讲到，因为它并不属于“主线”，但是他有一个非常重要的方法调用，如下：

```java
ActivityStack.class(api28)
void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
        boolean newTask, boolean keepCurTransition, ActivityOptions options) {
    ...
    r.createWindowContainer();
    ...
}
```

这个方法就把token送到了WMS 那里，我们继续看下去：

```java
ActivityRecord.class(api28)
void createWindowContainer() {
    ...
    // 注意参数有token，这个token就是之前初始化的token
    mWindowContainerController = new AppWindowContainerController(taskController, appToken,
            this, Integer.MAX_VALUE /* add on top */, info.screenOrientation, fullscreen,
            (info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0, info.configChanges,
            task.voiceSession != null, mLaunchTaskBehind, isAlwaysFocusable(),
            appInfo.targetSdkVersion, mRotationAnimationHint,
            ActivityManagerService.getInputDispatchingTimeoutLocked(this) * 1000000L);
	...
}
```

注意参数有token，这个token就是之前初始化的token，我们进入到他的构造方法看一下：

```java
AppWindowContainerController.class(api28)
public AppWindowContainerController(TaskWindowContainerController taskController,
        IApplicationToken token, AppWindowContainerListener listener, int index,
        int requestedOrientation, boolean fullscreen, boolean showForAllUsers, int configChanges,
        boolean voiceInteraction, boolean launchTaskBehind, boolean alwaysFocusable,
        int targetSdkVersion, int rotationAnimationHint, long inputDispatchingTimeoutNanos,
        WindowManagerService service) {
    ...
    synchronized(mWindowMap) {
        AppWindowToken atoken = mRoot.getAppWindowToken(mToken.asBinder());
       ...
        atoken = createAppWindow(mService, token, voiceInteraction, task.getDisplayContent(),
                inputDispatchingTimeoutNanos, fullscreen, showForAllUsers, targetSdkVersion,
                requestedOrientation, rotationAnimationHint, configChanges, launchTaskBehind,
                alwaysFocusable, this);
        ...
    }
}
```

还记得我们在一开始看WMS的时候他验证的是什么对象吗？WindowToken，而AppWindowToken是WindowToken的子类。那么我们继续追下去：

```java
AppWindowContainerController.class(api28)
AppWindowToken createAppWindow(WindowManagerService service, IApplicationToken token,
        boolean voiceInteraction, DisplayContent dc, long inputDispatchingTimeoutNanos,
        boolean fullscreen, boolean showForAllUsers, int targetSdk, int orientation,
        int rotationAnimationHint, int configChanges, boolean launchTaskBehind,
        boolean alwaysFocusable, AppWindowContainerController controller) {
    return new AppWindowToken(service, token, voiceInteraction, dc,
            inputDispatchingTimeoutNanos, fullscreen, showForAllUsers, targetSdk, orientation,
            rotationAnimationHint, configChanges, launchTaskBehind, alwaysFocusable,
            controller);
}
AppWindowToken(WindowManagerService service, IApplicationToken token, ...) {
    this(service, token, voiceInteraction, dc, fullscreen);
    ...
}

WindowToken.class
WindowToken(WindowManagerService service, IBinder _token, int type, boolean persistOnEmpty,
        DisplayContent dc, boolean ownerCanManageAppTokens, boolean roundedCornerOverlay) {
    token = _token;
    ...
    onDisplayChanged(dc);
}
```

createAppWindow方法调用了AppWindow的构造器，然后再调用了父类WindowToken的构造器，我们可以看到这里最终对token进行了缓存，并调用了一个方法，我们看看这个方法做了什么：

```java
WindowToken.class
void onDisplayChanged(DisplayContent dc) {
    dc.reParentWindowToken(this);
	...
}

DisplayContent.class(api28)
void reParentWindowToken(WindowToken token) {
    addWindowToken(token.token, token);
}
private void addWindowToken(IBinder binder, WindowToken token) {
    ...
    mTokenMap.put(binder, token);
    ...
}
```

mTokenMap 是一个 HashMap<IBinder, WindowToken> 对象，这里就可以保存一开始初始化的token以及后来创建的windowToken两者的关系。这里的逻辑其实已经在WMS中了，所以这个也是保存在WMS中。AMS和WMS都是运行在系统服务进程，因而他们之间可以直接调用方法，不存在跨进程通信。WMS就可以根据IBinder对象拿到windowToken进行信息比对了。至于怎么比对，代码位置在一开始的时候已经有涉及到，读者可自行去查看源码，这里就不讲了。

那么，到这里关于整个token的知识就全部走了一遍了，AMS怎么创建token，WMS怎么拿到token的流程也根据我们回溯的思路走了一遍。

## 整体流程把握

 前面根据我们思考问题的思维走完了整个token流程，但是似乎还是有点乱，那么这一部分，就把前面讲的东西整理一下，对token的知识有一个整体上的感知，同时也当时前面内容的总结。先来看整体图：

<img src="https://s1.ax1x.com/2020/10/13/0hgH1A.png"  border="0" width=500/>

1. token在创建ActivityRecord的时候一起被创建，他是一个IBinder对象，实现了接口IApplicationToken。
2. token创建后会发送到WMS，在WMS中封装成WindowToken，并存在一个HashMap<IBinder,WindowToken>。
3. token会随着ActivityRecord被发送到本地进程，ActivityThread根据AMS的指令执行Activity启动逻辑。
4. Activity启动的过程中会创建PhoneWindow和对应的WindowManager，同时把token存在PhoneWindow中。
5. 通过Activity的WindowManager添加view/弹出dialog时会把PhoneWindow中的token放在窗口LayoutParams中。
6. 通过viewRootImpl向WMS进行验证，WMS在LayoutParams拿到IBinder之后就可以在Map中获取WindowToken。
7. 根据获取的结果就可以判断该token的合法情况。

这就是整个token的运作流程了。而具体的源码和细节在上面已经解释完了，读者可自行选择重点部分再次阅读源码。


## 从源码设计看token

我在[Context机制](https://blog.csdn.net/weixin_43766753/article/details/109017196)一文中讲到，不同的context拥有不同的职责，系统对不同的context限制了不同的权利，让在对应情景下的组件只能做对应的事情。其中最明显的限制就是UI操作。

token看着是属于window机制的领域内容，其实是context的知识范畴。我们知道context一共有三种最终实现类：Activity、Application、Service，context是区分一个类是普通Java类还是android组件的关键。context拥有访问系统资源的权限，是各种组件访问系统的接口对象。但是，三种context，只有Activity允许有界面，而其他的两种是不能有界面的，也没必要有界面。**为了防止开发者乱用context造成混乱，那么必须对context的权限进行限制，这也就是token存在的意义**。拥有token的context可以创建界面、进行UI操作，而没有token的context如service、Application，是不允许添加view到屏幕上的（这里的view除了系统窗口）。

为什么说这不属于window机制的知识范畴？从[window机制](https://blog.csdn.net/weixin_43766753/article/details/108350589)中我们知道WMS控制每一个window，是通过viewRootImpl中的IWindowSession来进行通信的，token在这个过程中只充当了一个验证作用，且当PhoneWindow显示了DecorView之后，后续添加的View使用的token都是ViewRootImpl的IWindowSession对象。这表示当一个PhoneWindow可以显示界面后，那么对于后续其添加的view无需再次进行权限判断。因而，**token真正限制的，是context是否可以显示界面，而不是针对window**。

而我们了解完底层逻辑后，不是要去知道怎么绕过他的限制，动一些“大胆的想法”，而是要知道官方这么设计的目的。我们在开发的时候，也要针对不同职责的context来执行对应的事务，**不要使用Application或Service来做UI操作**。

## 总结

文章采用思考问题的思路来表述，通过源码分析，讲解了关于token的创建、传递、验证等内容。同时，token在源码设计上的思想进行了总结。

android体系中各种机制之间是互相联系，彼此连接构成一个完整的系统框架。token涉及到window机制和context机制，同时对activity的启动流程也要有一定的了解。阅读源码各种机制的源码，可以从多个维度来帮助我们对一个知识点的理解。同时阅读源码的过程中，不要局限在当前的模块内，思考不同机制之间的联系，系统为什么要这么设计，解决了什么问题，可以帮助我们从架构的角度去理解整个android源码设计。阅读源码切忌无目标乱看一波，要有明确的目标、验证什么问题，针对性寻找那一部分的源码，与问题无关的源码暂时忽略，不然会在源码的海洋里游着游着就溺亡了。

> 全文到此，感谢你的阅读
>
> 原创不易，觉得有帮助可以点赞收藏评论转发关注。
> 笔者才疏学浅，有任何错误欢迎评论区或私信交流。
> 如需转载请私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)