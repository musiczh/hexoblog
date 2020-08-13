---
title: Android广播Broadcast的注册与广播过程详解（基于api29） 	#标题
date: 2020/4/21 00:00:00 	#建立日期
summary: 					#文章摘要
tags: 						#标签
 - android 
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

#### 注册广播流程

1.  首先直接调用到ContextWrapper的registerReceiver方法，Activity是继承自ContextWrapper的。registerReceiver是Context的抽象方法，ContentWrapper继承自Content，但ContextWrapper本身并没有真正实现了Context的抽象方法，而是使用了装饰者模式。Context的真正实现是他内部的一个变量：mBase。这是ContextImpl的实例，所以最终的真正实现是在ContextImpl中。

   ```java
   /frameworks/base/core/java/android/content/ContextWrapper.java;
   public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
       // 直接调用mBase的方法。mBase是ContextImpl的实例，也是Context的真正实现。
       return mBase.registerReceiver(receiver, filter);
   }
   ```

   >这里补充一下为什么mBase是ContextImpl的实例。如果你阅读过Activity的启动流程（[点击前往Activity的启动流程文章](https://blog.csdn.net/weixin_43766753/article/details/107746968)），可以知道是在ActivityThread创建Activity的时候，会给Activity初始化Context。下面看一下这部分的源码：
   >
   >```java
   >/frameworks/base/core/java/android/app/ActivityThread.java
   >
   >// ActivityThread调用此方法来启动Activity
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

   

2. 这一步是ContextImpl的逻辑。主要的任务就是构建IIntentReceiver并请求AMS来注册广播接收器。这里涉及到两个知识点：如何请求AMS，为什么不能直接把BroadcastReceiver传递给AMS。

   第一个问题采用的是跨进程通信技术AIDL（Android Interface Define Language）安卓接口定义语言，主要用于跨进程通信，这里就不详细展开，不然就篇幅过大。读者可自行了解。

   第二个问题，是因为广播涉及跨进程通信，而BoradcastReceiver本身并不支持跨进程，所以要进行封装一下。这个问题会在这一步的源码下面进行补充说明。

   后面的逻辑就交给AMS去处理了。

   ```java
   /frameworks/base/core/java/android/app/ContextImpl.java;
   
   // 直接跳转
   public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
       return registerReceiver(receiver, filter, null, null);
   }
   
   // 再跳转
   public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
           String broadcastPermission, Handler scheduler) {
       return registerReceiverInternal(receiver, getUserId(),
               filter, broadcastPermission, scheduler, getOuterContext(), 0);
   }
   
   // 此方法主要是获取IIntentReceiver对象
   // 为什么要获取IIntentReceiver上面已经有讲了。
   // 并调用AMS在本地代理对象IActivityManagerService的方法来注册广播接收器
   private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
           IntentFilter filter, String broadcastPermission,
           Handler scheduler, Context context, int flags) {
       IIntentReceiver rd = null;
       if (receiver != null) {
           // 判断mPackageInfo和context是否为空，选择不同的方式获得IIntentReceiver
           // 注意这里的scheduler是mMainThread.getHandler()
           // 他是ActivityThread的内部类H，这里先不管有什么用，注意一下就好，后面会讲到。
           if (mPackageInfo != null && context != null) {
               if (scheduler == null) {
                   scheduler = mMainThread.getHandler();
               }
               rd = mPackageInfo.getReceiverDispatcher(
                   receiver, context, scheduler,
                   mMainThread.getInstrumentation(), true);
           } else {
               if (scheduler == null) {
                   scheduler = mMainThread.getHandler();
               }
               rd = new LoadedApk.ReceiverDispatcher(
                       receiver, context, scheduler, null, true).getIIntentReceiver();
           }
       }
       try {
           // ActivityManager.getService()获取到的对象是AMS在本地的代理对象
           // 通过代理对象可以直接跨进程访问AMS
           // 调用IActivityManagerService的方法来启动
           final Intent intent = ActivityManager.getService().registerReceiver(
                   mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                   broadcastPermission, userId, flags);
           ...
       } 
       ...
   }
   
   ```

   > 这里深入讲一下IIntentReceiver。IIntentReceiver是一个接口，他的真正实现是InnerReceiver。InnerReceiver是ReceiverDispatcher的静态内部类。ReceiverDispatcher负责维护BroadcastReceiver和IIntentReceiver之间的通信。远程AMS调用InnerReceiver的方法，ReceiverDispatcher把逻辑转到BroadcastReceiver中。
   >
   > 有读过Service启动流程（[点击前往Service启动流程](https://blog.csdn.net/weixin_43766753/article/details/107881248)）的读者应该知道Service的绑定也是采用了类似的设计。可能你觉得有点绕，下面画一个图帮助你理解一下：
   >
   > <img src="https://s1.ax1x.com/2020/08/11/aOuFpV.png" alt="IIntentReceiver各类的关系.png" border="0" width=60%/>
   >
   > Map是LoadedApk类中维持的一个HashMap，key是BroadcastReceiver，value是ReceiverDispatcher。

   

3. 这一步在AMS中的处理主要分为两个：注册粘性广播和注册普通广播。由于粘性广播在android5.0已经被废弃，所以这里只讲注册普通广播。AMS的工作主要是把传进来的广播接收器存起来。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;
   
   public Intent registerReceiver(IApplicationThread caller, String callerPackage,
           IIntentReceiver receiver, IntentFilter filter, String permission, int userId,
           int flags) {
       ...;
       synchronized (this) {
           // 获取ReceiverList，如果不存在就创建一个
           ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
           if (rl == null) {
               rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                       userId, receiver);
               ...
               mRegisteredReceivers.put(receiver.asBinder(), rl);
           } 
           ...
           // 创建BroadcastFilter并存起来
           BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                   permission, callingUid, userId, instantApp, visibleToInstantApps);
           if (rl.containsFilter(filter)) {
               ...
           } else {
               rl.add(bf);
               ...
           
               mReceiverResolver.addFilter(bf);
           }
       }
       ...
   }
   ```

   

#### 发送广播和接收流程

1. 这里同样是先调用ContextWrapper的方法，然后再调用ContextImp的方法，原因和前面一样，不再赘述，主要看ContextImp的实现。ContextImp的逻辑也很简单，简单处理后直接调用AMS的方法。

   ```java
   /frameworks/base/core/java/android/content/ContextWrapper.java;
   public void sendBroadcast(Intent intent) {
       mBase.sendBroadcast(intent);
   }
   
   /frameworks/base/core/java/android/app/ContextImpl.java;
   public void sendBroadcast(Intent intent) {
       ...
       try {
           intent.prepareToLeaveProcess(this);
           // 直接调用AMS的方法进行广播
           ActivityManager.getService().broadcastIntent(
                   mMainThread.getApplicationThread(), intent, resolvedType, null,
                   Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                   getUserId());
       } catch (RemoteException e) {
           throw e.rethrowFromSystemServer();
       }
   }
   ```

   

2. 这一步主要是检查广播是否合法，然后获取信息后跳转下个方法。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;
   
   public final int broadcastIntent(IApplicationThread caller,
           Intent intent, String resolvedType, IIntentReceiver resultTo,
           int resultCode, String resultData, Bundle resultExtras,
           String[] requiredPermissions, int appOp, Bundle bOptions,
           boolean serialized, boolean sticky, int userId) {
       ...
       synchronized(this) {
           // 验证广播是否合法
           intent = verifyBroadcastLocked(intent);
   		// 获得各种信息
           final ProcessRecord callerApp = getRecordForAppLocked(caller);
           final int callingPid = Binder.getCallingPid();
           final int callingUid = Binder.getCallingUid();
   
           final long origId = Binder.clearCallingIdentity();
           try {
               // 跳转下个方法
               return broadcastIntentLocked(callerApp,
                       callerApp != null ? callerApp.info.packageName : null,
                       intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                       requiredPermissions, appOp, bOptions, serialized, sticky,
                       callingPid, callingUid, callingUid, callingPid, userId);
           }
           ...
       } 
   }
   ```

   >这里看一下验证广播合法性的方法：
   >
   >```java
   >final Intent verifyBroadcastLocked(Intent intent) {
   >    // 检查Intent不为null且有文件修饰符
   >    if (intent != null && intent.hasFileDescriptors() == true) {
   >        throw new IllegalArgumentException("File descriptors passed in Intent");
   >    }
   >
   >    // 获取Intent的Flag
   >    int flags = intent.getFlags();
   >
   >    // 当系统还没有启动时，判断这个广播是否有权限
   >    if (!mProcessesReady) {
   >        if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT) != 0) {
   >        } else if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
   >            Slog.e(TAG, "Attempt to launch receivers of broadcast intent " + intent
   >                    + " before boot completion");
   >            throw new IllegalStateException("Cannot broadcast before boot completed");
   >        }
   >    }
   >    ...
   >}
   >```

   

3. 这一步的方法代码非常多。主要完成的任务是：根据intentFilter筛选出符合条件的receiver，并按照优先级进行排序后存到队列中。并调用方法进行广播。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;
   final int broadcastIntentLocked(ProcessRecord callerApp,
           String callerPackage, Intent intent, String resolvedType,
           IIntentReceiver resultTo, int resultCode, String resultData,
           Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
           boolean ordered, boolean sticky, int callingPid, int callingUid, int realCallingUid,
           int realCallingPid, int userId, boolean allowBackgroundActivityStarts) {
       ...
        // 默认不发送给已经停止的app
   	intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
       ...;
       // 获取广播接收器队列
       final BroadcastQueue queue = broadcastQueueForIntent(intent);
       BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
               callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
               requiredPermissions, appOp, brOptions, registeredReceivers, resultTo,
               resultCode, resultData, resultExtras, ordered, sticky, false, userId,
               allowBackgroundActivityStarts, timeoutExempt);    
       ...
       if (!replaced) {
           // 调用方法进行广播
           queue.enqueueParallelBroadcastLocked(r);
           queue.scheduleBroadcastsLocked();
       }
       
   }
   ```

   