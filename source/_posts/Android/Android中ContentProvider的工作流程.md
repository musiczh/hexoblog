---
title: Android中ContentProvider的工作流程（基于api29）	#标题
date: 2020/8/19 00:00:00 	#建立日期
summary: ContentProvider作为四大组件之一，但是使用的频率确实很少，甚至有一些读者都没用过他，真是毫无存在感的四大组件。但是既然他能作为四大组件，说明他的重要性肯定不低。了解他背后的工作机制，有助于我们更好地了解ContentProvider。					#文章摘要
tags: 						#标签
 - android 
 - contentProvider
categories:  				#分类
 - Android
img:  						#文章卡片显示的图，使用图床

updated: 					#更新日期
author:  					#作者

top:						#boolean,文章是否置顶
cover: 						#文章是否加入轮播图
coverImg: 					#文章轮播图显示的图片
toc: true						#是否开启toc
mathjax: 					#是否开启数学公式支持

comments: true 				#开启评论
---

## 前言

你好 ~
我是一只修仙的猿，欢迎阅读我的文章。

ContentProvider作为四大组件之一，但是使用的频率确实很少，甚至有一些读者都没用过他，真是毫无存在感的四大组件。但是既然他能作为四大组件，说明他的重要性肯定不低，只是目前来说我们使用不到。

ContentProvider的作用是**跨进程共享数据**，他向我们屏蔽了底层的Binder操作，使得我们可以像查询本地的数据一样查询别的进程的数据，让跨进程的数据共享变得非常方便。如我们查询手机的通讯录、短信等，都是通过ContentProvider来实现的。

ContentProvider的使用方式也很简单，内容提供方只需要继承ContentProvider类，并实现对应的`onCreate,query,delete,insert,update,getType`方法即可。内容获取方只需要通过`Content.getContentResolver()`获取到ContentResolver对象然后通过uri直接调用对应的ContentProvider的方法就可以增删查改数据了。

这篇文章我们要讨论的不是如何使用ContentProvider，而是ContentProvider启动以及请求的流程。了解他背后的工作机制，有助于我们更好地了解ContentProvider。文章涉及到很多的源码，我以“思路先行，代码后讲”的思维来讲解这一过程。先对总体有个感知，这样不会在海量的源码中迷失，再对具体代码进行研究会有更深刻的理解。希望读者可以认真看完每一处的代码，我更加建议把一边看文章一边自己查看源码，这样可以加深理解。文章源码我加了非常多的注释，帮助理解每一处关键代码。文章涉及到跨进程通信，如果对此还没有学习的话，可能看起来一些步骤比较难以理解，建议先学一下跨进程通信AIDL，当然，文中我也会简单介绍一下。

> 笔者才疏学浅，有不同观点欢迎评论区或私信讨论。如需转载请留言告知。
> 另外欢迎阅读笔者的个人博客[一只修仙的猿的个人博客](https://qwerhuan.gitee.io/)，更精美的UI，拥有更好的阅读体验。

本文大纲：

<img src="https://s1.ax1x.com/2020/08/19/d1wN8S.png" alt="本文大纲" border="0" width=60%>

## 工作流程总体思路

ContentProvider的工作离不开AMS（ActivityManagerService），事实上，四大组件的工作流程都离不开AMS。我们在创建一个ContentProvider的时候，除了新建一个类继承并重写方法，还需要在AndroidManifest中进行注册，而AndroidManifest就是AMS进行处理的。AndroidManifest会在当前应用被创建时进行处理，因而ContentProvider是伴随着应用进程的启动而被创建的。下面我们看一张图了解一下ContentProvider的启动流程：

<img src="https://s1.ax1x.com/2020/08/19/d1B9m9.png" alt="ContentProvider启动流程" border="0" width=60%/>

ContentProvider是在应用进程被创建的时候被创建的，所以这个流程也是进程创建的流程。进程间的通信是通过Binder机制实现的。

- 应用进程创建后的代码入口是ActivityThread的main函数，然后他通过Binder机制把ApplicationThread发送给AMS进行缓存，以后AMS要访问该进程就是通过ApplicationThread来访问的。
- AMS做处理之后，通过ApplicationThread来让ActivityThread进行初始化。
- 在ActivityThread初始化中会创建ContextImp，Application和ContentProvider。
- 然后ActivityThread把ContentProvider发布到AMS中。以后如果有别的进程想要读取数据可以直接从AMS中获取到对应的ContentProvider。

这就是ContentProvider的启动流程了。当然涉及到跨进程通信，不可能直接以ContentProvider来通信，而是以IContentProvider来通信，同样也是利用AIDL技术，下面我会详细讲到。那么一个应用请求另一个应用的ContentProvider的流程是怎么样的呢？先看图：

<img src="https://s1.ax1x.com/2020/08/19/d1rYZj.png" alt="ContentProvider的请求流程" border="0" width=70%/>

好像看起来很复杂对吧？其实就是判断的情况比较多了。整体的思路就是：获取到对应URI的IContentProvider，然后跨进程访问数据，所以重点在获取IContentProvider。

- 首先会查询本地是否有和URI对应的IContentProvider，有的话直接使用，没有的话去AMS获取。
- 如果ContentProvider对应的进程已启动，那么AMS一般是有该IContentProvider的，原因在上面的启动有讲。但是如果应用尚未启动，那么需要AMS去启动该进程。
- 进程B启动后会自然把IContentProvider发布给AMS，AMS先把他缓存起来，然后再返回给进程A
- 进程A拿到IContentProvider之后，就是直接跨进程访问ContentProvider了。

可以看到整体的思路并不复杂，那么下面我会根据这些流程，按照源码详细走一遍。还是再强调一下，源码中我加了大量的注释，千万不要忽视源码中的注释。

## 源码讲解

### ContentProvider的启动流程

<img src="https://s1.ax1x.com/2020/08/19/d125lt.png" alt="ContentProvider的启动源码流程" border="0" width=70%/>

1. ContentProvider是在AndroidManifest中进行静态注册的，也就说明了他是在应用启动的时候被创建的。我们要了解他的启动流程，就必须了解应用的启动流程。这里不深入讲应用的启动流程，应用进程启动最终会调用ActivityThread的main方法，有兴趣的读者可以去深入了解一下。那么我们就可以定位到main方法中，看看这个方法做了什么事情。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java;
   public static void main(String[] args) {
       ...
       Looper.prepareMainLooper();
       ...;
       // 这里创建了ActivityThread对象，并紧接着调用了他的attach方法
       // 初始化是在attach中完成的，我们进入看一下
       ActivityThread thread = new ActivityThread();
       thread.attach(false, startSeq);
   	...;
       // 这里开启主线程的消息循环
       Looper.loop();
   }
   
   private void attach(boolean system, long startSeq) {
       // 这里判断是否是系统进程我们的应用自然不是
       if (!system) {
           ...
           //这个地方比较复杂，先说结论。下面再进行解释    
       	//ActivityManager.getService()获取到的对象是ActivityManagerService，简称AMS
       	//通过AMS来启动activity。AMS是全局唯一的，所有的活动启动都要经过他的验证，运行在独立的进程中
       	//所以这里是采用AIDL的方式进行跨进程通信，获取到的对象其实是一个IBinder接口
           final IActivityManager mgr = ActivityManager.getService();
           try {
               // 这里通过访问AMS把IApplicationThread发送给AMS
               mgr.attachApplication(mAppThread, startSeq);
           } catch (RemoteException ex) {
               throw ex.rethrowFromSystemServer();
           }
       }
   }
   ```

   >这里深入讲一下ActivityManager.getService()。
   >
   >```java
   >/frameworks/base/core/java/android/app/ActivityManager.java/;
   >// 这里可以看出来是个单例类，可以直接获取到单例
   >public static IActivityManager getService() {
   >	return IActivityManagerSingleton.get();
   >}
   >
   >private static final Singleton<IActivityManager> IActivityManagerSingleton =
   >new Singleton<IActivityManager>() {
   >    @Override
   >    protected IActivityManager create() {
   >    // 得到AMS的IBinder对象
   >    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
   >    // 转化成IActivityManager对象。远程服务实现了这个接口，所以可以直接调用这个
   >    // AMS代理对象的接口方法来请求AMS。这里采用的技术是AIDL
   >    final IActivityManager am = IActivityManager.Stub.asInterface(b);
   >    return am;
   >    }
   >};
   >```

   

2. AMS逻辑很多，但是我们只要注意到他最终会调用IApplicationThread的bindApplication方法回到本地进程进行初始化。AMS这里的处理包括了读取AndroidManifest，特别是ContentProvider的信息，后面会讲到。然后把初始化工作交给了ActivityThread去处理。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java/;
   public final void attachApplication(IApplicationThread thread, long startSeq) {
       synchronized (this) {
           ...
           // 直接跳转下个方法
           attachApplicationLocked(thread, callingPid, callingUid, startSeq);
           Binder.restoreCallingIdentity(origId);
       }
   }
   
   private final boolean attachApplicationLocked(IApplicationThread thread,
           int pid, int callingUid, long startSeq) {
       ...
       final ActiveInstrumentation instr2 = app.getActiveInstrumentation();
       if (app.isolatedEntryPoint != null) {
           ...
       } else if (instr2 != null) {
           // 这里又回到了正在启动初始化的进程。thread是IApplicationThread，
           // 和上面的IActivityManangerService一样，属于AIDL接口
           thread.bindApplication(processName, appInfo, providers,
                   instr2.mClass,
                   profilerInfo, instr2.mArguments,
                   instr2.mWatcher,
                   instr2.mUiAutomationConnection, testMode,
                   mBinderTransactionTrackingEnabled, enableTrackAllocation,
                   isRestrictedBackupMode || !normalMode, app.isPersistent(),
                   new Configuration(app.getWindowProcessController().getConfiguration()),
                   app.compat, getCommonServicesLocked(app.isolated),
                   mCoreSettingsObserver.getCoreSettingsLocked(),
                   buildSerial, autofillOptions, contentCaptureOptions);
       }
       ...
   }
   
   ```

   

3. ApplicationThread接收AMS的请求后，把数据封装成一个AppBindData类，然后调用Handle类H切换到主线程进行处理。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java/ApplicationThread.class/;
   public final void bindApplication(String processName, ApplicationInfo appInfo,
           List<ProviderInfo> providers, ComponentName instrumentationName,
           ProfilerInfo profilerInfo, Bundle instrumentationArgs,
           IInstrumentationWatcher instrumentationWatcher,
           IUiAutomationConnection instrumentationUiConnection, int debugMode,
           boolean enableBinderTracking, boolean trackAllocation,
           boolean isRestrictedBackupMode, boolean persistent, Configuration config,
           CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
           String buildSerial, AutofillOptions autofillOptions,
           ContentCaptureOptions contentCaptureOptions) {
       ...
   
       AppBindData data = new AppBindData();
       data.processName = processName;
       data.appInfo = appInfo;
       data.providers = providers;
       data.instrumentationName = instrumentationName;
       data.instrumentationArgs = instrumentationArgs;
       ...// 这里省略一堆的data初始化
           
       // 最终调用ActivityThread的sendMessage方法
       // 这里的sendMessage最后会给到ActivityThread的内部类H中，H是一个Handle
       // 所以我们接下里看看H是怎么处理事件的
       sendMessage(H.BIND_APPLICATION, data);
   }
   ```

   

4. H，即ActivityThread的内部类H，继承自Handle，收到信息后进行处理，调用ActivityThread的方法来初始化各种组件。包括Instrumentation、Application、ContentProvider等等。但是这里要注意的是：**ContentProvider的初始化是在其他两个组件的onCreate回调之前**。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java/H.class
   public void handleMessage(Message msg) {
       if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
       switch (msg.what) {
           case BIND_APPLICATION:
               Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
               // 这里把刚才疯转的data提取出来后，直接调用ActivityThread的handleBindApplication方法
               AppBindData data = (AppBindData)msg.obj;
               handleBindApplication(data);
               Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
               break;
               ...
       }
       ...
   }
   
   /frameworks/base/core/java/android/app/ActivityThread.java
   private void handleBindApplication(AppBindData data) {
       ...
       // 这里获取到ContextImpl
       final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
       ...
       // 利用类加载获取到Instrumentation实例
       try {
           final ClassLoader cl = instrContext.getClassLoader();
           mInstrumentation = (Instrumentation)
               cl.loadClass(data.instrumentationName.getClassName()).newInstance();
       }
       ...
       // 初始化Instrumentation
       final ComponentName component = new ComponentName(ii.packageName, ii.name);
       mInstrumentation.init(this, instrContext, appContext, component,
               data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
       ...
       // 获取到Application实例
       Application app;
       ...
       try {
           app = data.info.makeApplication(data.restrictedBackupMode, null);
           ...
       }
       ...
       // 初始化ContentProvider，终于初始化ContentProvider了
       // 下面我们深入看一下这个怎么进行初始化
       if (!data.restrictedBackupMode) {
           if (!ArrayUtils.isEmpty(data.providers)) {
               installContentProviders(app, data.providers);
           }
       }
      	// 回调Instrumentation的onCreate方法
       try {
           mInstrumentation.onCreate(data.instrumentationArgs);
       }
       ...
       // 利用Instrumentation回调Application的onCreate方法
       try {
           mInstrumentation.callApplicationOnCreate(app);
       }
       ...
   }
   ```

   

5. 这一步是遍历从AMS中拿来的ContentProvider信息列表，依次安装ContentProvider之后，最后把ContentProvider列表提交到AMS进行缓存。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java
   private void installContentProviders(
           Context context, List<ProviderInfo> providers) {
       final ArrayList<ContentProviderHolder> results = new ArrayList<>();
   
       // 这里遍历当前应用程序的ProviderInfo列表，这些信息是AMS读取AndroidManifest得出来的
       for (ProviderInfo cpi : providers) {
           if (DEBUG_PROVIDER) {
               StringBuilder buf = new StringBuilder(128);
               buf.append("Pub ");
               buf.append(cpi.authority);
               buf.append(": ");
               buf.append(cpi.name);
               Log.i(TAG, buf.toString());
           }
           // 这里进行安装ContentProvider
           ContentProviderHolder cph = installProvider(context, null, cpi,
                   false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
           if (cph != null) {
               cph.noReleaseNeeded = true;
               results.add(cph);
           }
       }
   
       try {
           // 最后把ContentProvider列表发布到AMS中
           // 目的是进行缓存，下次别的应用进程想要获取他的ContentProvider的时候
           // 可以直接在缓存中拿
           ActivityManager.getService().publishContentProviders(
               getApplicationThread(), results);
       } catch (RemoteException ex) {
           throw ex.rethrowFromSystemServer();
       }
   }
   ```

   

6. 这一步是新建ContentProvider实例，并回调onCreate进行初始化。到此整个contentProvider就初始化完成了。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java;
   private ContentProviderHolder installProvider(Context context,
           ContentProviderHolder holder, ProviderInfo info,
           boolean noisy, boolean noReleaseNeeded, boolean stable) {
       ContentProvider localProvider = null;
       IContentProvider provider;
       ...
       try {
           // 这里通过类加载器新建ContentProvider和IContentProvider
           // IContentProvider很明显，就是为了跨进程通信的接口
           final java.lang.ClassLoader cl = c.getClassLoader();
           LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
           if (packageInfo == null) {
               packageInfo = getSystemContext().mPackageInfo;
           }
           
           localProvider = packageInfo.getAppFactory()
                   .instantiateProvider(cl, info.name);
           // 注意这里的provider实际上是Transport对象，我会在下面进行补充。
           provider = localProvider.getIContentProvider();
           ...
           // 这里对ContentProvider进行初始化
           // 下面继续深入
           localProvider.attachInfo(c, info);
       }    
       ...
   }
   
   /frameworks/base/core/java/android/content/ContentProvider.java
   private void attachInfo(Context context, ProviderInfo info, boolean testing) {
       ...
       if (mContext == null) {
           ...
           // 这里会回调onCreate方法，这个onCreate方法是个抽象方法，需要我们重写
           // 到此我们的ContentProvider就安装完成了
           ContentProvider.this.onCreate();
       }
   }
   ```

   >我们都知道IContentProvider只是一个接口，如果我们需要知道他的工作流程，那么我们必须知道他的实现类是什么。进入`localProvider.getIContentProvider()`方法
   >
   >```java
   >/frameworks/base/core/java/android/content/ContentProvider.java
   >private Transport mTransport = new Transport();
   >public IContentProvider getIContentProvider() {
   >    // 我们可以看到返回的类型是Transport
   >    // Transport继承自ContentProviderNative，这是个抽象类
   >    // ContentProviderNative实现了IContentProvider接口
   >    return mTransport;
   >}
   >```

   

### contentProvider的请求流程

> 前情提示：ContentProvider的请求流程其实很简单，就是直接通过IContentProvider跨进程访问，难点在如何获取Cursor对象，但这不属于整体流程而属于细节流程了。所以这里主要展开讲如何获取到IContentProvider。下面的流程以目标ContentProvider所在进程尚未创建的情况进行讲解。

<img src="https://s1.ax1x.com/2020/08/19/d14kb6.png" alt="ContentProvider的请求源码流程" border="0" width=80%/>

1. 我们需要请求别的应用的ContentProvider的时候，首先要获取ContentResolver对象。通常我们都是直接通过Context.getResolver来获取其对象的，但是ContentResolver是一个抽象类，他的真正实现类是ApplicationContentResolver。我们来跟踪一下源码证明一下：

   ```java
   /frameworks/base/core/java/android/content/ContextWrapper.java;
   // 当我们在Activity中调用getContentResolver的时候其实是调用了ContextWrapper的方法
   // Activity是继承自ContextWrapper的。getContentResolver是Context的抽象方法，
   // ContentWrapper继承自Content，但是他运行了装饰者模式，本身没有真正实现Context，
   // 他的内部有一个真正的实例mBase，他是ContextImpl的实例，所以最终的真正实现是在ContextImpl中。
   public ContentResolver getContentResolver() {
       // 接着我们去看一下方法的具体实现
       return mBase.getContentResolver();
   }
   
   /frameworks/base/core/java/android/app/ContextImpl.java;
   private final ApplicationContentResolver mContentResolver;
   public ContentResolver getContentResolver() {
       // 可以看到这里返回的对象确实是ApplicationContentResolver
       // 那我们接下来就是看看这个类是如何完成请求的。
       return mContentResolver;
   } 
   ```

   >ContextWrapper中的mBase为什么是ContextImpl的实例呢？这个要回到ActivityThread创建Activity的时候创建的Context是什么类型的。最终可以看到就是ContextImpl这个类是具体的实现。
   >
   >```java
   >/frameworks/base/core/java/android/app/ActivityThread.java
   >
   >private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
   >// 在这里创建了Content，可以看到他的类型是ContentImpl
   >ContextImpl appContext = createBaseContextForActivity(r);
   >Activity activity = null;
   >...
   >// 在这里把appContext给到了activity。说明这个就是Activity里面的Context的具体实现
   >appContext.setOuterContext(activity);
   >activity.attach(appContext, this, getInstrumentation(), r.token,
   >       r.ident, app, r.intent, r.activityInfo, title, r.parent,
   >       r.embeddedID, r.lastNonConfigurationInstances, config,
   >       r.referrer, r.voiceInteractor, window, r.configCallback,
   >       r.assistToken);
   >...
   >}
   >```
   >关于Activity的详细启动流程，读者感兴趣可以阅读我的另一篇文章[Avtivity的启动流程](https://blog.csdn.net/weixin_43766753/article/details/107746968)

   

2. 这一部分是获取到对应uri的contentProvider的远程接口IContentProvider，然后直接调用远程contentProvider的方法来获取Cursor。这里获取到IContentProvider有很多种情况，具体如何获取Cursor我们不深入讲，后面我们讲一下IContentProvider获取的各种情况。

   ```java
   /frameworks/base/core/java/android/content/ContentResolver.java;
   // query调用的是ContentResolver的方法。虽然真正实现类是ApplicationContentResolver
   // 但contentProvider的增删查改方法并不是抽象方法
   public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
           @Nullable String[] projection, @Nullable Bundle queryArgs,
           @Nullable CancellationSignal cancellationSignal) {
       ...
       
       // 首先会获取IContentProvider对象
       // 这里的获取对象有三种情况：本地是否有缓存？AMS是否有缓存？ContentProvider的进程是否建立
       // 下面我们会简单展开讲一下。
       IContentProvider unstableProvider = acquireUnstableProvider(uri);
       ...
       try {
           ...
           try {
               // 这里通过IContentProvider对象通过跨进程获取到Cursor对象
               // 从上面的ContentProvider的安装知道这个接口的实例是Transport对象
               // query方法的真正实现是contentProviderNative，他通过调用对应的ContentProvider对象
               // 来得到相对应的Cursor对象进行返回
               // 下面我们就不深入讲Cursor是如何获取的了，因为涉及到的知识太多了，我们只简单了解流程。
               qCursor = unstableProvider.query(mPackageName, uri, projection,
                       queryArgs, remoteCancellationSignal);
           }
       }
       ...
   }
   ```

   

3. 这一步是从ContentResolver中获取IContentProvider，他会调用ActivityThread的方法来获取。

   ```java
   /frameworks/base/core/java/android/content/ContentResolver.java;
   public final IContentProvider acquireUnstableProvider(Uri uri) {
       if (!SCHEME_CONTENT.equals(uri.getScheme())) {
           return null;
       }
       String auth = uri.getAuthority();
       if (auth != null) {
           // 根据uri的host获取IContentProvider
           // 这个方法是个抽象方法，他的具体实现是ApplicationContentResolver
           // 他是ContentImpl的静态内部类
           return acquireUnstableProvider(mContext, uri.getAuthority());
       }
       return null;
   }
   
   /frameworks/base/core/java/android/app/ContextImpl.java/ApplicationContentResolver.java;
   protected IContentProvider acquireUnstableProvider(Context c, String auth) {
       // 这里直接通过ActivityThread来获取对象
       return mMainThread.acquireProvider(c,
               ContentProvider.getAuthorityWithoutUserId(auth),
               resolveUserIdFromAuthority(auth), false);
   }
   ```

   

4. 这里会先检查本地是否存在IContentProvider，存在则直接返回；否则要去AMS获取，然后安装在本地，下次就可以直接调用了返回了。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java;
   public final IContentProvider acquireProvider(
           Context c, String auth, int userId, boolean stable) {
       // 如果本地已经有了，就直接获取返回
       final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
       if (provider != null) {
           return provider;
       }
   	...
       ContentProviderHolder holder = null;
       try {
           synchronized (getGetProviderLock(auth, userId)) {
               // 如果本地没有的话，则会去AMS获取对象
               holder = ActivityManager.getService().getContentProvider(
                       getApplicationThread(), c.getOpPackageName(), auth, userId, stable);
           }
       } 
       ...
       // 安装contentProvider
       holder = installProvider(c, holder, holder.info,
               true /*noisy*/, holder.noReleaseNeeded, stable);
       return holder.provider;
   }
   ```

   

5. 这一步是从AMS获取IContentProvider。从上面的ContentProvider的启动我们可以知道，ContentProvider的创建伴随着进程的创建，且会把IContentProvider上传到AMS进行缓存。

   如果ContentProvider所在的进程创建了，那么AMS重肯定有该ContentProvider。当然不能排除意外情况，所以AMS会进程判断，如果进程已经启动了，但是却没有该IContentProvider，则会通知该进程去初始化，随后AMS就会得到对应的IContentProvider了。

   而如果ContentProvider所在的进程还没被创建，则会先去创建进程，而创建进程又伴随着ContentProvider的创建，这样AMS就可以拿到IContentProvider了，然后返回给调用端。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java/;
   public final ContentProviderHolder getContentProvider(
           IApplicationThread caller, String callingPackage, String name, int userId,
           boolean stable) {
       ...
       // 跳转方法
       return getContentProviderImpl(caller, name, null, callingUid, callingPackage,
               null, stable, userId);
   }
   
   private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
           String name, IBinder token, int callingUid, String callingPackage, String callingTag,
           boolean stable, int userId) {
       ...
       // 这里获取到ContentProvider的进程信息
       ProcessRecord proc = getProcessRecordLocked(
               cpi.processName, cpr.appInfo.uid, false);
       // 判断该进程是否已经创建了
       if (proc != null && proc.thread != null && !proc.killed) {
           ...
           // 如果进程已经创建了但是contentProvider却没有被创建，则会通知该进程初始化ContentProvider
           if (!proc.pubProviders.containsKey(cpi.name)) {
               ...
               proc.pubProviders.put(cpi.name, cpr);
               try {
                   proc.thread.scheduleInstallProvider(cpi);
               } 
           }
       } else {
           ...
           // 如果还没有创建进程则先启动进程，获取IContentProvider
           proc = startProcessLocked(cpi.processName,
                   cpr.appInfo, false, 0,
                   new HostingRecord("content provider",
                   new ComponentName(cpi.applicationInfo.packageName,
                           cpi.name)), false, false, false);
           ...
       }
       ...
   }
   
   ```



## 总结

通过这篇文章我们了解了ContentProvider的启动以及访问流程，重点在ContentProvider的创建时机以及他的缓存机制。希望通过文章能够对你有所帮助。

> 笔者才疏学浅，有不同观点欢迎评论区或私信讨论。如需转载请留言告知。
> 另外欢迎阅读笔者的个人博客[一只修仙的猿的个人博客](https://qwerhuan.gitee.io/)，更精美的UI，拥有更好的阅读体验。

## 参考文献

- 《Android开发艺术探索》
- 《Android进阶解密》