---
title: Android广播Broadcast的注册与广播过程详解（基于api29） 	#标题
date: 2020/8/17 00:00:00 	#建立日期
summary: 作为四大组件之一，广播的地位不言而喻。而了解其源码流程，可以更好地帮助我们了解他的工作流程，进而更好地了解Broadcast这个组件。这样在开发中，就会更加有自信去写代码，出现了bug也能更快地定位与解决					#文章摘要
tags: 						#标签
 - android 
 - 广播
categories:  				#分类
 - Android
img:  						#文章卡片显示的图，使用图床

updated: 					#更新日期
author:  					#作者

top:						#boolean,文章是否置顶
cover: true						#文章是否加入轮播图
coverImg: 					#文章轮播图显示的图片
toc: true						#是否开启toc
mathjax: 					#是否开启数学公式支持

comments: true 				#开启评论
---

## 前言

你好！
我是一只修仙的猿，欢迎阅读我的文章。

本篇文章的内容是讲解Broadcast的注册与发送的源码流程，不涉及Broadcast的使用。如果还没了解过广播如何使用的读者可以先去了解一下，但我相信不会的，因为广播在开发是很经常使用的。作为四大组件之一，广播的地位不言而喻。而了解其源码流程，可以更好地帮助我们了解他的工作流程，进而更好地了解Broadcast这个组件。这样在开发中，就会更加有自信去写代码，出现了bug也能更快地定位与解决。

> 本篇文章涉及大量的代码讲解，因而会显得非常枯燥无味。笔者加了大量的代码注释以及流程图来帮助读者理解。注释是非常重要的，代码外的讲解仅仅只是整体流程的把握，注释才是重点细节。还愿读者可以耐心阅读。

文章我以先把握整体流程，再到细节源码讲解的思路讲解。读者要先对整体流程有个把握，知道整体的方向再去阅读源码，这样不会在源码中丢失了方向，也不会不知道哪个地方是重点。读者亦可以边阅读边查看源码，加深印象。

> 笔者才疏学浅，有不同观点欢迎评论区或私信讨论。如需转载私信告知即可。
> 另外欢迎阅读笔者的个人博客[一只修仙的猿的个人博客](https://qwerhuan.gitee.io/)，更精美的UI，拥有更好的阅读体验。

本文大纲：

<img src="https://s1.ax1x.com/2020/08/17/dmjFYD.png" alt="本文大纲.png" border="0" width=60%/>



## Broadcast注册流程

### 整体流程概述

<img src="https://s1.ax1x.com/2020/08/17/dmvAEV.png" alt="broadcast注册整体流程" border="0" width=40%/>

注册流程相对来说比较简单，这里涉及到两个进程：

- Activity进程，即注册广播所在进程。
- AMS进程，AMS即ActivityManagerService，该进程为AMS所在进程。

注册的流程为Activity进程把广播接收器发送给AMS，AMS把广播接收器缓存在内部。流程比较简单。不同的进程当然就涉及到跨进程通信，API29源码使用的是AIDL（Android Interface Define Language），如果不熟悉的读者可以先去了解一下，有帮助对源码的理解。当然我也会在源码讲解中适当做解释，但不会深入讲，因为本文的主题是Broadcast工作流程讲解。

### 源码讲解

> 源码流程图
>
> <img src="https://s1.ax1x.com/2020/08/17/dnSOMQ.png" alt="Broadcast注册源码流程.png" border="0" width=80%/>

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
    >    ...
    >    // 在这里创建了Content，可以看到他的类型是ContentImpl
    >    ContextImpl appContext = createBaseContextForActivity(r);
    >    
    >    ...
    >    // 在这里把appContext给到了activity。说明这个就是Activity里面的Context的具体实现
    >    appContext.setOuterContext(activity);
    >    activity.attach(appContext, this, getInstrumentation(), r.token,
    >      r.ident, app, r.intent, r.activityInfo, title, r.parent,
    >      r.embeddedID, r.lastNonConfigurationInstances, config,
    >      r.referrer, r.voiceInteractor, window, r.configCallback,
    >      r.assistToken);
    >    ...
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
           if (mPackageInfo != null && context != null) {
               if (scheduler == null) {
                   // 注意这里的scheduler是mMainThread.getHandler()
           	    // 他是ActivityThread的内部类H，这里先不管有什么用，注意一下就好，后面会讲到。
                   scheduler = mMainThread.getHandler();
               }
               rd = mPackageInfo.getReceiverDispatcher(
                   receiver, context, scheduler,
                   mMainThread.getInstrumentation(), true);
           } else {
               ...
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

   

## Broadcast广播流程

### 整体流程概述

广播有普通广播、有序广播和粘性广播三种。本文只讲普通广播的发送流程。

<img src="https://s1.ax1x.com/2020/08/17/dnkELD.png" alt="广播发送的整体流程" border="0" width=60%/>

广播的发送涉及三个进程：

- Activity进程，即发出广播的进程
- AMS进程，上面讲过
- 广播接收器所在进程

大体的流程是：

- Activity把广播发送到AMS中
- AMS首先检测广播是否合法，然后根据IntentFilter规则，把所有符合条件的广播接收器整理成一个队列
- 依次遍历队列中的广播接收器，判断是否拥有权限
- 把广播发送到广播接收器所在进程，回调广播的onReceive方法

### 源码讲解

> 第一步是从Activity所在进程，即发出广播的进程到AMS
>
> <img src="https://s1.ax1x.com/2020/08/17/dnpTm9.png" alt="广播发送流程图-1" border="0" width=80%/>



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

   

   >下面这一部分的源码是AMS对广播的处理
   >
   ><img src="https://s1.ax1x.com/2020/08/17/dni1ht.png" alt="广播发送流程图-2" border="0" />

   

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
   >// 检查Intent不为null且有文件修饰符
   >if (intent != null && intent.hasFileDescriptors() == true) {
   >   throw new IllegalArgumentException("File descriptors passed in Intent");
   >}
   >
   >// 获取Intent的Flag
   >int flags = intent.getFlags();
   >
   >// 当系统还没有启动时，判断这个广播是否有权限
   >if (!mProcessesReady) {
   >   if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT) != 0) {
   >   } else if ((flags&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
   >       Slog.e(TAG, "Attempt to launch receivers of broadcast intent " + intent
   >                   + " before boot completion");
   >           throw new IllegalStateException("Cannot broadcast before boot completed");
   >       }
   >   }
   >   ...
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
           queue.enqueueParallelBroadcastLocked(r);
           // 调用方法进行广播
           queue.scheduleBroadcastsLocked();
       }
       
   }
   ```

   

4. BroadcastQueue通过Handle来执行广播事务。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java;
   final BroadcastHandler mHandler;
   public void scheduleBroadcastsLocked() {
       ...
   	// 判断是否正在执行
       if (mBroadcastsScheduled) {
           return;
       }
       // 通过handle来执行事务。BroadcastHandler是BroadcastQueue的一个内部类
       mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
       // 设置标志位表示正在执行
       mBroadcastsScheduled = true;
   }
   
   /frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java/BroadcastHandler.class;
   public void handleMessage(Message msg) {
       switch (msg.what) {
           case BROADCAST_INTENT_MSG: {
               ...
               // 继续跳转
               processNextBroadcast(true);
           } break;
           ...
       }
   }
   ```

   

5. 这一步BroadcastQueue会把列表中的无序与有序广播遍历执行进行广播。有序广播就不深入讲了，感兴趣的读者可自行去阅读源码。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java;
   final void processNextBroadcast(boolean fromMsg) {
       synchronized (mService) {
           // 继续跳转
           processNextBroadcastLocked(fromMsg, false);
       }
   }
   // 此方法会处理有序广播和无序广播，这里只讲解如何分发无序广播
   final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
           BroadcastRecord r;
   		...
           // 表示从Handle发送过来的消息已经处理了
           if (fromMsg) {
               mBroadcastsScheduled = false;
           }
   
           // 遍历无序广播列表，然后发送广播
           while (mParallelBroadcasts.size() > 0) {
               // 获取到无序广播
               r = mParallelBroadcasts.remove(0);
               ...
               
               for (int i=0; i<N; i++) {
                   Object target = r.receivers.get(i);
                   ...
                   // 这里进行发送广播
                   deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
               }
               addBroadcastToHistoryLocked(r);
               if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Done with parallel broadcast ["
                       + mQueueName + "] " + r);
           }
   }
   
   ```

   

6. 这一部分的源码也非常多，主要的逻辑就是进行权限判断，如果通过权限判断，那么就会去调用广播接收器所在进程的ApplicationThread的方法进行广播。下面的逻辑就到了广播接收器所在进程了。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java;
   private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
           BroadcastFilter filter, boolean ordered, int index) {
       ...
       try {
           if (DEBUG_BROADCAST_LIGHT) Slog.i(TAG_BROADCAST,
                   "Delivering to " + filter + " : " + r);
           if (filter.receiverList.app != null && filter.receiverList.app.inFullBackup) {
              ...
           } else {
               ...
               // 最后执行此方法进行广播
               performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                       new Intent(r.intent), r.resultCode, r.resultData,
                       r.resultExtras, r.ordered, r.initialSticky, r.userId);
               ...
           }
           if (ordered) {
               r.state = BroadcastRecord.CALL_DONE_RECEIVE;
           }
       }
       ...
   }
   
   void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
           Intent intent, int resultCode, String data, Bundle extras,
           boolean ordered, boolean sticky, int sendingUser)
           throws RemoteException {
       // app是ProcessRecord，表示广播接收器所在进程信息，app.thread是IApplicationThread
       // 这是ApplicationThread给AMS的Binder接口，通过他可以跨进程访问接收器所在进程
       // 这里主要就是判断进程以及IApplicationThread是否存在，直接调用ApplicationThread的方法。
       // 下面的逻辑就到了广播接收器的进程了。
       if (app != null) {
           if (app.thread != null) {
               try {
                   app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                           data, extras, ordered, sticky, sendingUser, app.getReportedProcState());
               } 
               ...
           } 
           ...
       } else {
           receiver.performReceive(intent, resultCode, data, extras, ordered,
                   sticky, sendingUser);
       }
   }
   ```

   

   >这一部分是广播接收器所在进程的处理
   >
   ><img src="https://s1.ax1x.com/2020/08/17/dnFV5n.png" alt="广播发送流程图-3" border="0" />

   

7. 这一步来到ApplicationThread，会直接调用InnerReceiver对请求进行处理，InnerReceiver会调用ReceiverDispatcher进行处理，最终会调用ActivityThread的方法，利用H切换逻辑到主线程进行处理。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java/ApplicationThread.class;
   public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
           int resultCode, String dataStr, Bundle extras, boolean ordered,
           boolean sticky, int sendingUser, int processState) throws RemoteException {
       updateProcessState(processState, false);
       // 还记得IIntentReceiver吗？这是一个ALDL接口，他的实现类是InnerReceiver
       // 忘记的话可以回到上面的内容看一下.下面跳转到他的方法
       receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
               sticky, sendingUser);
   }
   
   /frameworks/base/core/java/android/app/LoadedApk.java/ReceiverDispatcher.java/InnerReceiver.java;
   public void performReceive(Intent intent, int resultCode, String data,
           Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
       ...
       if (rd != null) {
           // 这里的rd就是ReceiverDispatcher
           rd.performReceive(intent, resultCode, data, extras,
                             ordered, sticky, sendingUser);
       }    
       ...
       
   }
   
   /frameworks/base/core/java/android/app/LoadedApk.java/ReceiverDispatcher.java;
   public void performReceive(Intent intent, int resultCode, String data,
           Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
       // 这里把广播信息进行封装
       final Args args = new Args(intent, resultCode, data, extras, ordered,
               sticky, sendingUser);
       ...;
       // 这里的mActivityThread是ActivityThread的Handle内部类H，
       // 负责把逻辑切换到主线程进行事务处理
       // 这个Handle在我们实例化ReceiverDispatcher的时候会传进来，不知你是否还有印象
       // 下面我们就直接看这个Runable是什么
       if (intent == null || !mActivityThread.post(args.getRunnable())) {
           if (mRegistered && ordered) {
               IActivityManager mgr = ActivityManager.getService();
               if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                       "Finishing sync broadcast to " + mReceiver);
               args.sendFinished(mgr);
           }
       }
   }
   
   ```

   

8. 最后一步是在主线程中执行的，由上一步已经知道使用H类进行切线程。这一步是看给到H的Runable是怎么样的。这一步会在里面回调receiver的onReceive方法。到这里流程就走完了。

   ```java
   /frameworks/base/core/java/android/app/LoadedApk.java/ReceiverDispatcher.java/Args.class;
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

   

## 总结

文章整理了Android广播的注册与发送的源码流程，并简析了过程中涉及到的一些细节。相信读者现在对于广播的工作流程已经有了一定的理解了。

> 全文到此，希望对你有帮助.
> 笔者才疏学浅，有不同观点欢迎评论区或私信讨论。如需转载私信告知即可。
> 另外欢迎阅读笔者的个人博客[一只修仙的猿的个人博客](https://qwerhuan.gitee.io/)，更精美的UI，拥有更好的阅读体验。

## 参考文献

- 《Android进阶解密》
- 《Android开发艺术探索》