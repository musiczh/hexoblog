---
title: Android中Service的启动与绑定过程详解（基于api29）  						#标题
date: 2020/8/8 00:00:00 						#建立日期
updated: 					#更新日期
tags:	#标签
 - android 
 - service
					
categories:	#分类
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

前面我写到一个文章是关于Activity启动流程的[点击链接前往](https://qwerhuan.gitee.io/2020/08/01/android/activity-qi-dong-liu-cheng-xiang-jie-ji-yu-api28/)。这一篇的内容是关于Service的启动和绑定的流程详解。Service和Activity一样，都是受AMS和ActivityThread的管理，所以在启动流程上两者有一些相似。不同的是Service有绑定的流程，相对比较复杂一点，结合Service的生命周期来理解，也不是很复杂。

了解启动源码的好处是整个Service对于你来说已经是透明的了，我们所做的每一个操作，心里都很清楚他的背后发生了什么。这是一种自信，也是一种能力，一种区别于入门与高级工程师的能力。

文章涉及到很多的源码，我以“思路先行，代码后讲”的思维来讲解这一过程。先对总体有个感知，再对具体代码进行研究会有更深刻的理解。希望可以认真看完每一处的代码。我更加建议把一边看文章一边自己查看源码，这样可以加深理解。文章涉及到跨进程通信，如果对此还没有学习的话，可能看起来一些步骤比较难以理解，建议先学一下跨进程通信AIDL。

> 代码中我加了非常多的注释来帮助大家理解每个重点的内容，请一定要看代码内容。代码外的讲解仅仅只是总体流程的概述。
>
> 理解源码需要对Service的生命周期有一个完整的认识，可以帮助理解源码中的内容。在下面的解析中我也会穿插生命周期的一些补充，希望可以帮助大家理解。

**本文内容大纲**：

<img src="https://s1.ax1x.com/2020/08/12/av8XNT.png" alt="本文大纲.png" border="0" width=70%/>

## 总体工作流程思路概述

这一部分主要为大家有个整体的流程感知，不会在源码中丢失自己。分为两个部分：启动流程与绑定流程。

### 直接启动流程概述

#### 概述

直接启动，也就是我们在Activity中调用startService来启动一个Service的方式。

#### 图解

<img src="https://s1.ax1x.com/2020/08/07/afbYYq.png" alt="Service启动整体流程概述.png" border="0" width=70%/>



#### 简析

启动的步骤：

1. Activity向AMS，即ActivityManagerService请求启动Service。
2. AMS判断Service所在的进程是否已经创建，注意Service和Activity是可以同个进程的。
3. 如果还没创建则通过Zygote进程fork一个进程。
4. AMS请求Service所在进程的ActivityThread创建Service和启动，并回调Service的onCreate方法。
5. Service创建完成之后，AMS再次请求ActivityThread进行相关操作并回调onStartCommand方法。

过程中onCreate和onStartCommand是在两次跨进程通信中完成的，需要注意一下。总共最多涉及到四个进程。



### 绑定流程概述 

#### 概述

绑定服务即我们在Activity中调用bindService来绑定服务。绑定的过程相对直接启动来说比较复杂一点，因为涉及到一些生命周期，需要进行一些条件判断。

#### 图解

> 图有点复杂，配合下面的解析一起看

<img src="https://s1.ax1x.com/2020/08/12/avlXND.png" alt="avlXND.png" border="0" width=60%/>

#### 简析

介绍一下步骤：

1. Activity请求AMS绑定Service。
2. AMS判断Service是否已经启动了。如果还没启动会先去启动Service。
3. AMS调用Service的onBind方法获取Service的IBinder对象。
4. Service把自己的IBinder对象发布到AMS中保存，这样下次绑定就不需要再调用onBind了。
5. 把拿到的IBinder对象返回给Activity。
6. Activity可以直接通过这个IBinder对象访问Service。

这里是主体的流程。绑定的过程涉及到很多情况的判断，如：进程是否创建，是否已经绑定过，是否已经调用了onUnBind方法等。后面的源码讲解会讲到。这里我只放整体的流程。



## 源码讲解

这一部分内容枯燥，但我把源码尽量提取相关的代码，其他的用省略号进行省略，减少无用的代码干扰。源码中的重点我用了很多的注释解析，**一定要看注释**，是非常重要的一部分。

### 直接启动流程源码讲解

#### 一、Activity访问AMS的过程

##### 图解

<img src="https://s1.ax1x.com/2020/08/08/a52D58.png" alt="Service启动流程-1.png" border="0" width=60%/>

##### 源码解析

1. 我们在Activity中调用startService方法，其实是调用ContentWrapper的startService，因为Activity是继承自ContextWrapper的。startService是Context的抽象方法，ContentWrapper继承自Content，但是他运行了装饰者模式，本身没有真正实现Context，他的内部有一个真正的实例mBase，他是ContextImpl的实例，所以最终的真正实现是在ContextImpl中。

   ```java
   /frameworks/base/core/java/android/content/ContextWrapper.java
   
   // 这里的mBase是startService的真正实现。他是从哪里来的？从activit的创建开始看
   Context mBase;
   public ComponentName startService(Intent service) {
       return mBase.startService(service);
   }
   ```

   >想要知道什么时候得到Context，要回到ActivityThread创建Activity的时候创建的Context。最终可以看到就是ContextImpl这个类是具体的实现。
   >
   >```java
   >/frameworks/base/core/java/android/app/ActivityThread.java
   >
   >private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
   >    // 在这里创建了Content，可以看到他的类型是ContentImpl
   >    ContextImpl appContext = createBaseContextForActivity(r);
   >    Activity activity = null;
   >    ...
   >    // 在这里把appContext给到了activity。说明这个就是Activity里面的Context的具体实现
   >    appContext.setOuterContext(activity);
   >    activity.attach(appContext, this, getInstrumentation(), r.token,
   >            r.ident, app, r.intent, r.activityInfo, title, r.parent,
   >            r.embeddedID, r.lastNonConfigurationInstances, config,
   >            r.referrer, r.voiceInteractor, window, r.configCallback,
   >            r.assistToken);
   >    ...
   >}
   >```
   >

   

2. ContextImpl的处理逻辑是直接调用AMS在本地的代理对象IActivityManagerService来请求AMS。

   ```java
   /frameworks/base/core/java/android/app/ContextImpl.java;
   
   // 直接跳转下个方法
   public ComponentName startService(Intent service) {
       warnIfCallingFromSystemProcess();
       return startServiceCommon(service, false, mUser);
   }
   
   private ComponentName startServiceCommon(Intent service, boolean requireForeground,
       UserHandle user) {
       // 如果了解过activity启动流程看到这个ActivityManager.getService()应该很熟悉
       // 这里直接调用AMS在本地的IBinder接口，进行跨进程通信。不了解的可以去看一下我的
       // 关于解析activity启动流程的文章。链接在上头。
       ComponentName cn = ActivityManager.getService().startService(
       mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                   getContentResolver()), requireForeground,
                   getOpPackageName(), user.getIdentifier());
       
   }
   ```

   >这里再次简单介绍一下这个IActivityManager。这一步是通过AIDL技术进行跨进行通信。拿到AMS的代理对象，把启动任务交给了AMS。
   >
   >```java
   >/frameworks/base/core/java/android/app/ActivityManager.java/;
   >//单例类
   >public static IActivityManager getService() {
   >	return IActivityManagerSingleton.get();
   >}
   >
   >private static final Singleton<IActivityManager> IActivityManagerSingleton =
   >new Singleton<IActivityManager>() {
   >
   >protected IActivityManager create() {
   >    //得到AMS的IBinder接口
   >    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
   >    //转化成IActivityManager对象。远程服务实现了这个接口，所以可以直接调用这个
   >    //AMS代理对象的接口方法来请求AMS。这里采用的技术是AIDL
   >    final IActivityManager am = IActivityManager.Stub.asInterface(b);
   >    return am;
   >    }
   >};
   >```



#### 二、AMS处理服务启动请求的过程

##### 图解

<img src="https://s1.ax1x.com/2020/08/08/a5RQMj.png" alt="Service启动流程-2.png" border="0" width=70%/>

##### 源码解析

1. AMS调用ActivityServices进行处理。有点类似activity启动中的调用ActivityStarter进行启动。把业务分给他的小弟。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;
   public ComponentName startService(IApplicationThread caller, Intent service,
           String resolvedType, boolean requireForeground, String callingPackage, int userId)
           throws TransactionTooLargeException {
       final ActiveServices mServices;
       ...
       // 这里的mServices是ActivityServices，跳转->
       res = mServices.startServiceLocked(caller, service,
                           resolvedType, callingPid, callingUid,
                           requireForeground, callingPackage, userId);
       ...
   }
   ```

   

2. ActivityServices类是整个服务创建过程的主要逻辑类。基本上AMS接收到请求后就把事务直接交给ActivityServices来处理了，所以整个流程都在这里进行。ActivityServices主要是获取Service的信息ServiceRecord，然后判断各种信息如权限，进程是否创建等。然后再创建一个StartItem，这个StartItem是后续执行onStartCommand的事务。这里要注意一下他。

   ```java
/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   // 方法中涉及到mAm就是AMS
   final ActivityManagerService mAm;
   
   // 直接跳下个方法
   ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
           int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
           throws TransactionTooLargeException {
       // 跳转下个方法
       return startServiceLocked(caller, service, resolvedType, callingPid, callingUid, fgRequired,
               callingPackage, userId, false);
   }
   
   // 这一步主要是获取ServiceRecord。也就是服务的各种信息，来检查服务的各个资源是否已经完备
   // 同时会新建一个StartItem，这个Item是后面负责回调Service的onStartCommand事务
   ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
           int callingPid, int callingUid, boolean fgRequired, String callingPackage,
           final int userId, boolean allowBackgroundActivityStarts)
           throws TransactionTooLargeException {
       ...
       // 这里会去查找ServiceRecord，和ActivityRecord差不多，描述Service的信息。
       // 如果没有找到则会调用PackageManagerService去获取service信息
       // 并封装到ServiceLookResult中返回
       ServiceLookupResult res =
       retrieveServiceLocked(service, null, resolvedType, callingPackage,
               callingPid, callingUid, userId, true, callerFg, false, false);
       if (res == null) {
           return null;
       }
       if (res.record == null) {
           return new ComponentName("!", res.permission != null
                   ? res.permission : "private to package");
       }
   	// 这里从ServiceLookResult中得到Record信息
       ServiceRecord r = res.record;    
       
       // 这里新建一个StartItem，注意他的第二个参数是false，这个参数名是taskRemove
       r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                   service, neededGrants, callingUid));
       ...
       // 继续跳转
       ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
   }
   ```
   
3. 这一步主要是检查Service所在进程是否已经启动。如果没有则先启动进程。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
           boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
       ...
       // 继续跳转
       String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
       if (error != null) {
           return new ComponentName("!!", error);
       }
       ...
   }
   
   // 此方法主要判断服务所在的进程是否已经启动，如果还没启动那么就去创建进程再启动服务；
   // 第一次绑定或者直接启动都会调用此方法
   // 进程启动完了就调用realStartServiceLocked下一步
   private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
           boolean whileRestarting, boolean permissionsReviewRequired)
           throws TransactionTooLargeException {
       ...
       // 这里获取进程名字。在启动的时候可以指定进程，默认是当前进程
       final String procName = r.processName;
       HostingRecord hostingRecord = new HostingRecord("service", r.instanceName);    
       ProcessRecord app;
       if (!isolated) {
           // 根据进程名字获取ProcessRecord
           app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
           ...
           // 这里判断进程是否为空。如果进程已经启动那就直接转注释1；否则转注释2
           // 创建进程的流程这里不讲，所以后面的逻辑直接进入启动service
           if (app != null && app.thread != null) {
              ...
               // 1
               // 启动服务
               realStartServiceLocked(r, app, execInFg);
               return null;
           }
           ...
       }
       // 如果进程为空，那么执行注释2进行创建进程，再去启动服务
       if (app == null && !permissionsReviewRequired) {
           // 2
           if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                   hostingRecord, false, isolated, false)) == null) {
               ...
               bringDownServiceLocked(r);
           }
           ...
       }
       ...
   }
   ```

   

4. 真正启动Service。分为两步：创建Service，回调Service的onStartCommand生命周期。是两次不同的跨进程通信。下面先讲如何创建，第四点会讲onStartCommand的调用流程。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   private final void realStartServiceLocked(ServiceRecord r,
           ProcessRecord app, boolean execInFg) throws RemoteException {
       
       // app.thread就是ApplicationThread在AMS的代理接口IApplicationThread
       // 这里就把逻辑切换到service所在进程去进行创建了
       // 关于这方面的内容，我在上一篇文章有描述，这里就不再赘述。
       app.thread.scheduleCreateService(r, r.serviceInfo,
                       mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                       app.getReportedProcState());
       
       ...
       // 这一步保证如果是直接启动不是绑定，但是r.pendingStarts.size() == 0
       // 也就是没有startItem，那么会创建一个，保证onStartCommand被回调
       if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
           r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                   null, null, 0));
       }    
       // 这个负责处理Item事务。前面我们在startServiceLocked方法里讲到添加了一个Item
       // 此Item就在这里被处理。这一部分内容在Service被create之后讲，下面先讲Service的创建
       sendServiceArgsLocked(r, execInFg, true);
   }   
   ```

   

#### 三、ActivityThread创建服务

##### 图解

<img src="https://s1.ax1x.com/2020/08/08/a5RUWF.png" alt="Service启动流程-3.png" border="0" width=70%/>

##### 源码解析

1. 这一步主要的工作就是Service所在进程接收AMS的请求，把数据封装起来后，调用ActivityThread的sendMessage方法把逻辑切到主线程去执行

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java/ApplicationThread.class
   
   // 这里把Service的信息封装成了CreateServiceData，然后交给ActivityThread去处理
   // 这里的sendMessage是ActivityThread的一个方法
   // ApplicationThread是ActivityThread的一个内部类
   public final void scheduleCreateService(IBinder token,
           ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
       updateProcessState(processState, false);
      	// 封装信息
       CreateServiceData s = new CreateServiceData();
       s.token = token;
       s.info = info;
       s.compatInfo = compatInfo;
       // 跳转
       sendMessage(H.CREATE_SERVICE, s);
   }
   
   /frameworks/base/core/java/android/app/ActivityThread.java;
   private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
       ...
       // 这里的mH也是ActivityThread的一个内部类H，是一个Handle.负责把逻辑切换到主线程来运行
       // 根据上述的参数，查看Handle的处理
       mH.sendMessage(msg);
   }
   ```

   

2. 这一步是Handle处理请求，直接调用ActivityThread的方法来执行事务

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java/H.class;
   public void handleMessage(Message msg) {
       ...
               if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
               switch (msg.what) {
                       ...
                   case CREATE_SERVICE:
                       Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj))); 
                       // 把service信息取出来，调用方法进行跳转
                       handleCreateService((CreateServiceData)msg.obj);
                       Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                       break;           
                       ...
               }
       ...
   }
   ```

   

3. Activity处理事务，创建Service，并初始化Service等

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java;
   private void handleCreateService(CreateServiceData data) {
       
       // 获取LodeApk，是apk描述类，主要用于获取类加载器等
       LoadedApk packageInfo = getPackageInfoNoCheck(
               data.info.applicationInfo, data.compatInfo);
       Service service = null;
       try {
           // 获取类加载器，并调用工厂类进行实例化
           // getAppFactory是一个工厂类，使用类加载器来实例化android四大组件
           java.lang.ClassLoader cl = packageInfo.getClassLoader();
           service = packageInfo.getAppFactory()
                   .instantiateService(cl, data.info.name, data.intent);
       } 
       ...
      
       try {
           if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
   		// 创建上下文Context，可以看到这个和activity的实例化对象是一致的
           // 都是contextImpl
           ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
           context.setOuterContext(service);
   
           Application app = packageInfo.makeApplication(false, mInstrumentation);
           // 初始化并回调onCreate方法
           service.attach(context, this, data.info.name, data.token, app,
                   ActivityManager.getService());
           service.onCreate();
           // 这里把服务放到ActivityThread的Map中进行存储
           mServices.put(data.token, service);
           ...
       }
       ...
   }  
   ```



#### 四、AMS处理onStartCommand相关事务

##### 图解

##### 源码解析

1. 不知道大家是否还有印象，前面讲到说在```realStartServiceLocked```方法中有一个`sendServiceArgsLocked`方法，用来处理onStartCommand事务逻辑的。

   > 这里的讲解要依赖于前面的流程，因为创建Service事务和onStartCommand事务两者在AMS中是并行进行的，只有在最终执行的过程中才是顺序进行的。遇到不理解的地方可以回去看一下上面的流程。

   最初的事务的创建在`startServiceLocked`中。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
           int callingPid, int callingUid, boolean fgRequired, String callingPackage,
           final int userId, boolean allowBackgroundActivityStarts)
           throws TransactionTooLargeException {
       
       // 这里新建一个StartItem，注意他的第二个参数是false，这个参数名是taskRemove
       r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                   service, neededGrants, callingUid));
       ...
       // 继续跳转
       ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
   }
   ```

   >这里详细讲一下。`r`是`ServiceRecord`，`pendingStarts`是一个`List`：
   >
   >```final ArrayList<StartItem> pendingStarts = new ArrayList<StartItem>();```
   >
   >泛型是`StartItem`。下面我们看一下这个类：
   >
   >```java
   >/frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java/StartItem.java;
   >// 此类是ServiceRecord的静态内部类
   >// 代码虽然不多，但是我们只关心我们关心的代码
   >static class StartItem {
   >    final ServiceRecord sr;
   >    // 注意这个属性，后面在Service进程执行的时候会用到这个进行判断
   >    final boolean taskRemoved;
   >    final int id;
   >    final int callingId;
   >    final Intent intent;
   >    final NeededUriGrants neededGrants;
   >    ...
   >
   >    StartItem(ServiceRecord _sr, boolean _taskRemoved, int _id, Intent _intent,
   >            NeededUriGrants _neededGrants, int _callingId) {
   >        sr = _sr;
   >        // 可以看到第二个属性进行赋值，从上面的代码可以看出来这里是false。（记住他）
   >        taskRemoved = _taskRemoved;
   >        id = _id;
   >        intent = _intent;
   >        neededGrants = _neededGrants;
   >        callingId = _callingId;
   >    }
   >```

   

2. 然后中间经过`bringUpServiceLocked`方法，负责创建进程，前面讲到这里就不赘述了。直接到`realStartServiceLocked`方法。onStartCommand的事务是在`sendServiceArgsLocked`被执行的，可以看到创建和执行onStartCommand事务确实是两个不同的跨进程通信，且是有先后顺序的。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   private final void realStartServiceLocked(ServiceRecord r,
           ProcessRecord app, boolean execInFg) throws RemoteException {
       
       // app.thread就是ApplicationThread在AMS的代理接口IApplicationThread
       // 这里就把逻辑切换到service所在进程去进行创建了
       // 关于这方面的内容，我在上一篇文章有描述，这里就不再赘述。
       app.thread.scheduleCreateService(r, r.serviceInfo,
                       mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                       app.getReportedProcState());
       
       ...
       // 这一步保证如果是直接启动不是绑定，但是r.pendingStarts.size() == 0
       // 也就是没有startItem，那么会创建一个，保证onStartCommand被回调
       // 这里也可以看到第二个参数是false，也就是taskRemove参数是false。
       if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
           r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                   null, null, 0));
       }    
       // 这个负责处理Item事务。前面我们在startServiceLocked方法里讲到添加了一个Item
       // 此Item就在这里被处理。这一部分内容在Service被create之后讲，下面先讲Service的创建
       sendServiceArgsLocked(r, execInFg, true);
   }
   ```

   

3. 这一步就是取出之前新建的StartItem，然后逐一执行。注意，Service绑定的流程也会走到这里，但是他并没有startItem，所以会直接返回。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   
   private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
           boolean oomAdjusted) throws TransactionTooLargeException {
       // 这里判断是否有startItem要执行。没有的话直接返回。
       // 因为绑定的流程也是会调用bringUpServiceLocked方法来启动Service，但是并不会
       // 回调onStartCommand方法，绑定流程中并没有添加startItem。后面讲到绑定流程就知道了。
       final int N = r.pendingStarts.size();
       if (N == 0) {
           return;
       }
       
       ArrayList<ServiceStartArgs> args = new ArrayList<>();
       while (r.pendingStarts.size() > 0) {
           // 取出startItem
           ServiceRecord.StartItem si = r.pendingStarts.remove(0);    
           ...
           // 然后保存在list中。看到item的taskRemoved变量被传进来了。前面我们知道他是false。
           args.add(new ServiceStartArgs(si.taskRemoved, si.id, flags, si.intent));
       }
       
       // 把args封装成一个新的对象ParceledListSlice，然后调用ApplicationThread的方法
       ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
       slice.setInlineCountLimit(4);
       Exception caughtException = null;
       try {
           // 跳转
           r.app.thread.scheduleServiceArgs(r, slice);
       }   
       ...
   }
   ```

   

4. 逻辑回到Service所在进程。这个时候取出内容封装成一个ServiceArgsData对象然后由Handle交给ActivityThread处理。

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java/ApplicationThread.class
   public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
       // 这里直接取出args
       List<ServiceStartArgs> list = args.getList();
   
       // 使用循环对每个事物进行执行
       for (int i = 0; i < list.size(); i++) {
           ServiceStartArgs ssa = list.get(i);
           ServiceArgsData s = new ServiceArgsData();
           s.token = token;
           // 这里的taskRemoved就是从最开始的startItem传进来的，他的值是false，上面已经讲过了。
           s.taskRemoved = ssa.taskRemoved;
           s.startId = ssa.startId;
           s.flags = ssa.flags;
           s.args = ssa.args;
   
           // 封装成一个ServiceArgsData后进行处理
           // 熟悉的逻辑，使用H进行且线程处理。我们就直接看到最终的ActivityThread的执行逻辑。
           // 中间的流程上面讲过是一样的，这里不再赘述。
           sendMessage(H.SERVICE_ARGS, s);
       }
   }
   ```

   

5. ActivityThread是最终的事务执行者。回调service的onStartCommand生命周期。

   ```java
   
   private void handleServiceArgs(ServiceArgsData data) {
       Service s = mServices.get(data.token);
       if (s != null) {
           try {
               ...
               // 之前一直让你们注意的taskRemoved变量这个时候就用到了。
               // 还记得他的值吗？没错就是false，这里就开始回调onStartCommand方法了。
               // 到这里的Service直接启动流程就讲完了。
               if (!data.taskRemoved) {
                   res = s.onStartCommand(data.args, data.flags, data.startId);
               } else {
                   s.onTaskRemoved(data.args);
                   res = Service.START_TASK_REMOVED_COMPLETE;
               }
               QueuedWork.waitToFinish();
               try {
                   // 通知AMS执行事务完成
                   ActivityManager.getService().serviceDoneExecuting(
                           data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
               }
               ...
           }
   ```

   

### 绑定启动流程源码讲解

> 绑定的过程中涉及到很多的分支。本文选择的是第一次绑定的流程进行讲解。其他的情况都是类似的，读者可自行查看源码。

#### 一、ContentWrapper请求AMS绑定服务

##### 图解

<img src="https://s1.ax1x.com/2020/08/08/a5hcF0.png" alt="Service绑定流程-1.png" border="0" width=70%/>

##### 源码解析

和直接启动一样，首先会来到ContextWrapper中进行处理，然后请求AMS进行创建。

```java
/frameworks/base/core/java/android/content/ContextWrapper.java
// 从上文可以知道这里的mBase是ContextImpl
Context mBase;
public boolean bindService(Intent service, ServiceConnection conn,
                           int flags) {
    // 第一个参数就是Service.class;第二个参数是一个ServiceConnection，用来回调返回IBinder对象
    // 最后一个是flag参数。直接跳转ContextImp的方法。
    return mBase.bindService(service, conn, flags);
}


/frameworks/base/core/java/android/app/ContextImpl.java;
public boolean bindService(Intent service, ServiceConnection conn, int flags) {
    warnIfCallingFromSystemProcess();
    // 继续跳转方法
    return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
            getUser());
}


private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
        String instanceName, Handler handler, Executor executor, UserHandle user) {
    // IServiceConnection是一个IBinder接口
    // 创建接口对象来支持跨进程
    IServiceConnection sd;
    ...
    if (mPackageInfo != null) {
        if (executor != null) {
            // 1
            // 这里获得了IServiceConnection接口。这里的mPackageInfo是LoadedApk类对象。
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
        } else {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        }
    }
    ...
    // 熟悉的味道，调用AMS的方法来进行处理，后面的逻辑进入AMS
    int res = ActivityManager.getService().bindIsolatedService(
    mMainThread.getApplicationThread(), getActivityToken(), service,
    service.resolveTypeIfNeeded(getContentResolver()),
    sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
}
```

>这里我们深入了解一下IServiceConnection。ServiceConnection并不支持跨进程通信，所以需要实现IServiceConnection来支持跨进程。
>
>从上面的注释1我们可以知道代码在LoadedApk类中，我们深入来看一下代码：
>
>```java
>/frameworks/base/core/java/android/app/LoadedApk.java;
>public final IServiceConnection getServiceDispatcher(ServiceConnection c,
>Context context, Executor executor, int flags) {
>// 跳转
>return getServiceDispatcherCommon(c, context, null, executor, flags);
>}
>
>// mServices是一个Map，里面是一个context对应一个map
>private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mServices
>= new ArrayMap<>();
>
>private IServiceConnection getServiceDispatcherCommon(ServiceConnection c,
>Context context, Handler handler, Executor executor, int flags) { 
>synchronized (mServices) {
>  // 这里可以看出来ServiceDispatcher确实是LoadedApk的内部类
>  LoadedApk.ServiceDispatcher sd = null;
>  // 先去找是否已经存在ServiceConnection
>  ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
>       ...
>      // 找不到则新建一个
>      if (sd == null) {
>      if (executor != null) {
>          sd = new ServiceDispatcher(c, context, executor, flags);
>      } else {
>          sd = new ServiceDispatcher(c, context, handler, flags);
>          }
>      if (DEBUG) Slog.d(TAG, "Creating new dispatcher " + sd + " for conn " + c);
>      if (map == null) {
>               map = new ArrayMap<>();
>              mServices.put(context, map);
>           }
>           map.put(c, sd);
>       } else {
>           sd.validate(context, handler, executor);
>       }
>       // 最后返回IServiceConnection对象
>       return sd.getIServiceConnection();
>    } 	
>}
>```
>IServiceConnection的具体实现是InnerConnection，是ServiceDispatcher的静态内部类，而ServiceDispatcher是LoadedApk的静态内部类。ServiceDispacher内部维护有ServiceConnection，IServiceConnection，起着联系两者之间的作用。
>
>画个图帮助理解：
>
><img src="https://s1.ax1x.com/2020/08/06/ag83kj.png" alt="ag83kj.png" border="0" width=60%/>
>
>在LoadedApk的内部维护有一个Map，这个Map的key是context，value是Map<ServiceConnection,ServiceDispatcher>。也就是一个应用只有一个LoadedApk，但是一个应用有多个context，每个context可能有多个连接，也就是多个connection，每个connection就对应一个ServiceDispatcher。ServiceDispatcher负责维护ServiceConnection和IServiceConnection之间的关系 。



#### 二、AMS处理绑定逻辑

##### 图解

<img src="https://s1.ax1x.com/2020/08/08/a5535t.png" alt="Service绑定流程-2.png" border="0" width=70%/>

##### 源码解析

1. AMS接收请求简单处理后交给ActivityServices去处理

```java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;

public int bindIsolatedService(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, IServiceConnection connection, int flags, String instanceName,
        String callingPackage, int userId) throws TransactionTooLargeException {
    ...
    // 还是一样使用ActivityServices来进行处理
    synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, instanceName, callingPackage, userId);
        }
    
}
```



2. 这一部分要结合服务的生命周期来理解。

>1. 绑定的时候肯定会先创建服务，如果还没创建的话。
>2. 调用onBind方法返回IBinder对象，但是可能该服务的onBind已经被调用过了，那么不会再调用一次，会把之前的代理对象直接返回。
>3. 这一步是怎么控制的？Service的代理对象要先到AMS中，AMS会先缓存下来，如果其他的客户端需要绑定，会先去查看是否已经存在该IBinder了。如果存在，那么还要判断他的onUnBind方法是不是已经调用过了。
>4. 如果调用过onUnBind，那么下一次绑定需要判断前面的onUnBind的返回值。如果是true，则会调用onReBind，如果是false，则**不会调用任何生命周期，且以后的任何绑定都不会再调用生命周期了，直到服务重建**。（这里不理解没关系，最后我会补充解绑的源码讲解给大家）
>5. 如果IBinder对象不存在，也就是首次绑定，那么会调用onBind方法进行获取对象。这里涉及到service的生命周期简单讲述一下，下面两个图帮助大家理解。理解了生命周期可以方便我们阅读源码。
>
><img src="https://s1.ax1x.com/2020/08/07/aRsE6S.png" alt="服务绑定生命周期" border="0" width=30%/>
>
><img src="https://s1.ax1x.com/2020/08/06/aRtaxs.png" alt="服务同时绑定和启动生命周期.png" border="0" width=50%/>
>
>

这一部分源码的内容也是围绕着生命周期来展开,这一部分源码很重要，涉及到service生命中周期的实现。

```java
/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;


int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, final IServiceConnection connection, int flags,
        String instanceName, String callingPackage, final int userId)
        throws TransactionTooLargeException {
    ...
    // 这里获得ProcessRecord，也就是进程的描述类
    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);  
    ...
        
    //获取Service描述类ServiceRecord
    ServiceLookupResult res =
        retrieveServiceLocked(service, instanceName, resolvedType, callingPackage,
                Binder.getCallingPid(), Binder.getCallingUid(), userId, true,
                callerFg, isBindExternal, allowInstant);
    ...
    ServiceRecord s = res.record;
    
   	...
    
    /**
    * 解释几个重要的类
    * ConnectionRecord:应用程序进程和service之间的一次通信信息
    * IntentBindRecord： 绑定service的Intent的信息
    * AppBindRecord：维护应用程序进程和service之间的关联，包括应用进程ProcessRecord，
    * 	服务ServiceRecord，通信信息ConnectionRecord，Intent信息IntentBindRecord
    */
    // 这里获取到AppBindRecord对象，并创建ConnectionRecord
    // 后面会讲到，记得留意一下
    AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
    ConnectionRecord c = new ConnectionRecord(b, activity,
            connection, flags, clientLabel, clientIntent,
            callerApp.uid, callerApp.processName, callingPackage);
    
    // 判断标志是否创建，和上面的直接启动服务类似，调用此方法进行启动。这里就不赘述
    if ((flags&Context.BIND_AUTO_CREATE) != 0) {
        s.lastActivity = SystemClock.uptimeMillis();
        if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                permissionsReviewRequired) != null) {
            return 0;
        }
    }
    ...
        
    // s是ServiceRecord，app是ProcessRecord，表示service已经创建
    // b.intent.received表示当前应用程序进程是否已经收到IBinder对象
    // 总体意思：服务是否已经建立且当前应用程序进程收到IBinder对象
    // 是：直接返回IBinder对象，否，转注释2
    if (s.app != null && b.intent.received) {
       
        try {
            // 如果已经有这个IBinder对象了，那么就直接返回
            c.conn.connected(s.name, b.intent.binder, false);
        } catch (Exception e) {
            Slog.w(TAG, "Failure sending service " + s.shortInstanceName
                    + " to connection " + c.conn.asBinder()
                    + " (in " + c.binding.client.processName + ")", e);
        }

        // 下面注释1和注释2的区别就是最后一个参数
        // 区别在于是否要调用onRebind方法。从服务的生命周期了解到绑定的时候第一次需要调用onBind
        // 如果onUnbind返回true，那么不会调用onBind而是调用onRebind。其他情况不会调用生命周期
        // 只是纯粹的绑定
        // 方法，判断是要执行onBind还是reBind方法
        if (b.intent.apps.size() == 1 && b.intent.doRebind) {
            // 1
            requestServiceBindingLocked(s, b.intent, callerFg, true);
        }
    } else if (!b.intent.requested) {
        // 2
        requestServiceBindingLocked(s, b.intent, callerFg, false);
    }
    
}
```



4. 这一步深入看看第一次绑定的时候怎么实现的。了解了第一次如何执行的，其他的也就自然明白了。这一步是和reBind使用的是同个方法，重点是在参数不同。但是道理和ReBind是差不多的，都是跨进程到Service所在进程进行回调生命周期方法。不同的是第一次是调用onBind方法，而Rebind是调用onRebind方法。且第一次回调还需要回来AMS进行记录。后面会详细讲到。先看这一部分源码：

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   
   // 首先看到requestServiceBindingLocked，在AMS还没有IBinder对象的时候会先调用
   // 这个方法去获取Service的IBinder对象，然后再回调客户端的ServiceConnection。
   private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
           boolean execInFg, boolean rebind) throws TransactionTooLargeException {
       
       // i.requested表示是否已经绑定过了，rebind表示是不是要进行rebind，这些都和上面的
       // 条件判断一一对应。第一次绑定和重绑这里(!i.requested || rebind)都是等于true
       // i.app.size()这里一般不会<=0.在下面的引用中有详细讲
       if ((!i.requested || rebind) && i.apps.size() > 0) {
           try {
               bumpServiceExecutingLocked(r, execInFg, "bind");
               r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
               // 这里就很熟悉了，通过ApplicationThread来知性逻辑
               // ApplicationThread的所有schedule开头的方法，最后都是会通过H切换到主线程去执行
               // 后面的逻辑就切换到service的所在线程中了，AMS要去获取IBinder
               r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                       r.app.getReportedProcState());
               if (!rebind) {
                   i.requested = true;
               }
               i.hasBound = true;
               i.doRebind = false;
           }
           ...
       }
       ...
   }
   
   ```

   > i.apps.size() > 0 ，这里的 `i`是IntentBindRecord，上面我们介绍到他的作用是记录每一个请求Intent的信息，因为不同的进程，可能使用同个Intent进行请求绑定的。看看IntentBindRecord的代码：
   >
   > ```java
   > /frameworks/base/services/core/java/com/android/server/am/IntentBindRecord.java;
   > 
   > // 通过官方的注释我们也可以理解了
   > // 它里面包含了对应的ServiceRecord，还有根据进程记录对应的AppBindRecord
   > // 同个Intent可能给不同的进程服务，所以要把不同的进程的信息记录下来
   > final class IntentBindRecord {
   >  /** The running service. */
   >  final ServiceRecord service;
   >  /** The intent that is bound.*/
   >  final Intent.FilterComparison intent; // 
   >  /** All apps that have bound to this Intent. */
   >  final ArrayMap<ProcessRecord, AppBindRecord> apps
   >          = new ArrayMap<ProcessRecord, AppBindRecord>();
   >  ...
   > }
   > ```
   >
   > AppBindRecord前面也介绍了是用于记录intent，service，connection还有process信息的。也就是一切的绑定信息这里都有。在上面有一个方法代码是```AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);```，来看看他是怎么实现的：
   >
   > ```java
   > // 这里的S是ServiceRecord类型
   > /frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java;
   > final ArrayMap<Intent.FilterComparison, IntentBindRecord> bindings
   >      = new ArrayMap<Intent.FilterComparison, IntentBindRecord>();
   > public AppBindRecord retrieveAppBindingLocked(Intent intent,
   >      ProcessRecord app) {
   >  // 判断绑定的intent是否已经存在对应的IntentBindRecord
   >  Intent.FilterComparison filter = new Intent.FilterComparison(intent);
   >  IntentBindRecord i = bindings.get(filter);
   >  if (i == null) {
   >      // 如果没有则创建一个，并存储起来
   >      i = new IntentBindRecord(this, filter);
   >      bindings.put(filter, i);
   >  }
   >  // 判断该IntentBindRecord是否有启动的进程的绑定信息AppBindRecord
   >  AppBindRecord a = i.apps.get(app);
   >  if (a != null) {
   >      return a;
   >  }
   >  // 没有就创建一个，并放到IntentBindRecord中
   >  a = new AppBindRecord(this, i, app);
   >  i.apps.put(app, a);
   >  return a;
   > }
   > ```
   >
   > 分析到这里就可以发现`i.apps.size()`肯定是大于0了。
   >
   > 不知道你们会不会被各种Record给绕晕了，这里我整理个图帮助你们理解（只涉及到我上面讲的内容）：
   >
> <img src="https://s1.ax1x.com/2020/08/06/a2PhaF.png" alt="各种Record的关系.png" border="0" width=60%/>
   >
   > 
   >




#### 三、Service进程调用onBind进行绑定

##### 图解

<img src="https://s1.ax1x.com/2020/08/08/a5Izm4.png" alt="Service绑定流程-3.png" border="0" width=70%/>

##### 源码解析

这一步是ActivityThread处理绑定任务。主要是回调生命周期，然后得到服务的IBinder对象，然后交给AMS。如果是reBind则不需要，只需要回调生命周期。

```java
/frameworks/base/core/java/android/app/ActivityThread.java/ApplicationThread.class;
// 和上面的类似，直接到H的handleMessage方法
public final void scheduleBindService(IBinder token, Intent intent,
        boolean rebind, int processState) {
    updateProcessState(processState, false);
    BindServiceData s = new BindServiceData();
    s.token = token;
    s.intent = intent;
    s.rebind = rebind;

    if (DEBUG_SERVICE)
        Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
    sendMessage(H.BIND_SERVICE, s);
}


/frameworks/base/core/java/android/app/ActivityThread.java/H.class;
public void handleMessage(Message msg) {
    ...
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                    ...
                case BIND_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                    // 跳转ActivityThread的方法
                    handleBindService((BindServiceData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;                        
                    ...
            }
    ...
}


/frameworks/base/core/java/android/app/ActivityThread.java;
private void handleBindService(BindServiceData data) {
    // 得到Service实例
    // 这里的实例是在直接启动的时候存进来的，没有印象的读者可以再回去看一下启动流程
    Service s = mServices.get(data.token);
    if (DEBUG_SERVICE)
        Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
    if (s != null) {
        try {
            data.intent.setExtrasClassLoader(s.getClassLoader());
            data.intent.prepareToEnterProcess();
            try {
                // 这里判断是重绑还是第一次绑定。
                // 第一次绑定需要把IBinder对象返回给AMS，然后让AMS把IBinder对象回调给
                // 调用端的ServiceConnection
                if (!data.rebind) {
                    IBinder binder = s.onBind(data.intent);
                    ActivityManager.getService().publishService(
                            data.token, data.intent, binder);
                } else {
                    s.onRebind(data.intent);
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                }
            } 
    ...
}
    
```



#### 四、AMS缓存onBind返回的IBinder对象，并把该对象回调到绑定端

##### 图解

<img src="https://s1.ax1x.com/2020/08/08/a5TqMT.png" alt="Service绑定流程-4.png" border="0" width=70%/>

##### 源码解析

1. 这一步回到AMS主要就是把拿到的IBinder存储一下，下次就不用再调用onBind了，直接返回即可。然后回调ServiceConnection的方法。

 ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;
   
   // AMS还是老样子把任务交给ActivityServices去处理
   public void publishService(IBinder token, Intent intent, IBinder service) {
       // Refuse possible leaked file descriptors
       if (intent != null && intent.hasFileDescriptors() == true) {
           throw new IllegalArgumentException("File descriptors passed in Intent");
       }
   
       synchronized(this) {
           if (!(token instanceof ServiceRecord)) {
               throw new IllegalArgumentException("Invalid service token");
           }
           // 跳转ActivityServices的方法
           mServices.publishServiceLocked((ServiceRecord)token, intent, service);
       }
   }
   
   /frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;
   void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
       if (r != null) {
           Intent.FilterComparison filter
                   = new Intent.FilterComparison(intent);
           IntentBindRecord b = r.bindings.get(filter);
           if (b != null && !b.received) {
               b.binder = service;
               b.requested = true;
               // 这里设置为true，还记得上面的条件判断逻辑吗？
               // 对的，下次就可以直接返回了，不需要再去ActivityThread执行onBind方法拿IBinder对象了
               b.received = true;
               ...
                   try {
                       // 这里拿到IBinder对象之后就执行回调了
                       // 下面我们深入看看怎么回调的。我们知道这个回调的具体实现是InnerConnection
                       // 前面已经介绍了不知道你是否还有印象。接下来我们就去InnerConnection看看
                       c.conn.connected(r.name, service, false);
                   } catch (Exception e) {
                       Slog.w(TAG, "Failure sending service " + r.shortInstanceName
                              + " to connection " + c.conn.asBinder()
                              + " (in " + c.binding.client.processName + ")", e);
                   }
                ...
            
       }
   }
   
 ```

   

2. 这一步主要是InnerConnection收到AMS的请求，然后把请求交给ServiceDispatcher去处理，ServiceDispatcher再封装成一个Runnable对象给ActivityThread的H在主线程中执行，最后的真正执行方法是ServiceDispatcher.doConnect()。最终ActivityThread会回调ServiceConnection的方法。

   ```java
   /frameworks/base/core/java/android/app/LoadedApk.java/ServiceDispatcher.java/InnerConnection.java;
   public void connected(ComponentName name, IBinder service, boolean dead)
       throws RemoteException {
       LoadedApk.ServiceDispatcher sd = mDispatcher.get();
       if (sd != null) {
           // 和ApplicationThread熟悉的味道，直接调用外部的方法，继续跳转
           sd.connected(name, service, dead);
       }
   }
      
   /frameworks/base/core/java/android/app/LoadedApk.java/ServiceDispatcher.java;
   // 我们在调用的时候并没有传入Executor，mActivityThread就是ActivityThread中的mH
   // 而post方法很明显是Handle的post方法，也就是他的内部类H去执行。
   // 所以这里就把任务丢给ActivityThread在主线程中执行了。
   // 这个参数在我们调用bindService会作为参数传入，不知你是否还有印象。
   // 接下来我们去看看要执行的这个任务是什么
   public void connected(ComponentName name, IBinder service, boolean dead) {
       if (mActivityExecutor != null) {
           mActivityExecutor.execute(new RunConnection(name, service, 0, dead));
       } else if (mActivityThread != null) {
           mActivityThread.post(new RunConnection(name, service, 0, dead));
       } else {
           doConnected(name, service, dead);
       }
   }  
      /frameworks/base/core/java/android/app/LoadedApk.java/ServiceDispatcher.java/RunConnection.class;
   private final class RunConnection implements Runnable {
       ...
           // 上面实例化的时候参数是0，所以执行doConnect方法
           public void run() {
           if (mCommand == 0) {
               doConnected(mName, mService, mDead);
           } else if (mCommand == 1) {
               doDeath(mName, mService);
           }
       }
       ...
   }
      
   /frameworks/base/core/java/android/app/LoadedApk.java/ServiceDispatcher.java;
   public void doConnected(ComponentName name, IBinder service, boolean dead) {
       // 这里有两个连接信息，一个是老的，另一个是新的。
       ServiceDispatcher.ConnectionInfo old;
       ServiceDispatcher.ConnectionInfo info;
       synchronized (this) {
           ...
               // 获取是否有已经有连接信息且IBinder!=null
               old = mActiveConnections.get(name);
           if (old != null && old.binder == service) {
               // 已经拥有这个IBinder了，不需要做任何操作
               return;
           }
   
           if (service != null) {
               // A new service is being connected... set it all up.
               info = new ConnectionInfo();
               info.binder = service;
               info.deathMonitor = new DeathMonitor(name, service);
               try {
                   // 监听service的生命周期，如果服务挂了就会通知
                   service.linkToDeath(info.deathMonitor, 0);
                   mActiveConnections.put(name, info);
               }
               ...
           }
           ...
       }
       // 如果连接信息存在且IBinder==null,说明出现了意外服务被杀死了，回调onServiceDisconnected。
       if (old != null) {
           mConnection.onServiceDisconnected(name);
       }
       // 这个dead参数由AMS传入
       if (dead) {
           mConnection.onBindingDied(name);
       }
       // 正常情况下直接回调onServiceConnected
       if (service != null) {
           mConnection.onServiceConnected(name, service);
       } else {
           // The binding machinery worked, but the remote returned null from onBind().
           // 官方注释翻译过来就是绑定正常工作但是服务的onBind方法返回了null
           mConnection.onNullBinding(name);
       }       
   }
   ```

   

## 总结

我们通过总体流程概述+源码解析的方式讲了Service的两种启动流程，详细解析了启动过程中的源码细节。可以看到Service的启动和Activity的启动是很像的，都是需要经过AMS的协助完成。

文章对于一些分支的流程没有深入讲，如onStartCommand方法何时被调用，reBind的流程是如何进行的等等。感兴趣的读者可以自行阅读源码。

希望文章对你有收获。



## 参考文献

《Android开发艺术探索》

《Android进阶解密》

[Service 的绑定原理](https://mp.weixin.qq.com/s?__biz=MzIzOTkwMDY5Nw==&mid=2247485354&idx=1&sn=8a45854571e5f748b3e7660f12c12b5e&chksm=e92246dcde55cfca27947bdaacfe3d9653699f97c1b5c3713af8dae8b9a47bf647dcb3f53755&scene=126&sessionid=1597213277&key=fdd054e9602c88a65a3a9e08a8a17cb1355d012fa208f797e516bfadceb22c49c7d121f196769a6465533ac3ce1da62af5a02d1cb4d23f6278965e9a34aae149afd7e2720e9e656db4aac94a36af66ca&ascene=1&uin=MTk1NjkyODg0NA%3D%3D&devicetype=Windows+10+x64&version=62090529&lang=zh_CN&exportkey=A0lyD%2FUBC7W4V5kTfJInsV0%3D&pass_ticket=boVLRIVa96sgQ7QyXPX7HDBNAP%2BMhCwkQRd4atoSMJjgk2jj5QbhbgHi6b2w4GUn)

[Service 详解](https://mp.weixin.qq.com/s?__biz=MzIzOTkwMDY5Nw==&mid=2247485146&idx=1&sn=590440e3aac46c9cc88dfb18a80c5cba&chksm=e92247acde55cebaf74a3bc17ab570dcd954f3ce14c87e57f19d0e0ce9219d39fa04cfa77743&scene=21#wechat_redirect)