---
title: Activity启动流程详解（基于api28） 	#标题
date: 2020/8/1 00:00:00 	#建立日期
summary: 					#文章摘要
tags: 						#标签
 - android 
 - activity
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

### 前言

Activity作为Android四大组件之一，他的启动绝对没有那么简单。这里涉及到了系统服务进程，启动过程细节很多，这里我只展示主体流程。activity的启动流程随着版本的更替，代码细节一直在进行更改，每次都会有很大的修改，如android5.0 android8.0。我这里的版本是基于android api28，也是目前我可以查得到的最新源码了。事实上大题的流程是相同的，掌握了一个版本，其他的版本通过源码也可以很快地掌握。

因为涉及到不同的进程之间的通信：系统服务进程和本地进程，在最新版本的android使用的是AIDL来跨进程通信。所以需要对AIDL有一定的了解，会帮助理解整个启动流程。

> 源码部分的讲解涉及到很多的代码讲解，可能会有一点不适，但还是建议看完源码。源码的关键代码处我都会加上注释，方便理解。
>
> 代码不会过分关注细节，只注重整体流程。想知道具体细节可以去查看源码。每份代码所在的路径我都会在代码前面标注出来，各位可以去查看相对应的源码。
>
> 每部分源码前我都会放流程图，一定要配合流程图食用，不然可能会乱。

### 整体流程概述

这一部分侧重于对整个启动流程的概述，在心中有大体的概念，这样可以帮助对下面具体细节流程的理解。

#### 普通Activity的创建

普通Activity创建也就是平常我们在代码中采用```startActivity(Intent intent)```方法来创建Activity的方式。总体流程如下图：

<img src="https://s1.ax1x.com/2020/08/02/aYQwF0.png" alt="aYQwF0.png" border="0" />

启动过程设计到两个进程：本地进程和系统服务进程。本地进程也就是我们的应用所在进程，系统服务进程为所有应用共用的服务进程。整体思路是：

1. activity向Instrumentation请求创建

2. Instrumentation通过AMS在本地进程的IBinder接口，访问AMS，这里采用的跨进程技术是AIDL。

3. 然后AMS进程一系列的工作，如判断该activity是否存在，启动模式是什么，有没有进行注册等等。

4. 通过ClientLifeCycleManager，利用本地进程在系统服务进程的IBinder接口直接访问本地ActivityThread。

   > ApplicationThread是ActivityThread的内部类，IApplicationThread是在远程服务端的Binder接口

5. ApplicationThread接收到服务端的事务后，把事务直接转交给ActivityThread处理。

6. ActivityThread通过Instrumentation利用类加载器进行创建实例，同时利用Instrumentation回调activity的生命中周期

这里涉及到了两个进程，本地进程主要负责创建activity以及回调生命周期，服务进程主要判断该activity是否合法，是否需要创建activity栈等等。进程之间就涉及到了进程通信：AIDL。（如果不熟悉可以先去了解一下，但可以简单理解为接口回调即可）

下面介绍几个关键类：

- Instrumentation是activity与外界联系的类（不是activity本身的统称外界，相对activity而言），activity通过Instrumentation来请求创建，ActivityThread通过Instrumentation来创建activity和调用activity的生命周期。

- ActivityThread，每个应用程序唯一一个实例，负责对Activity创建的管理，而ApplicationThread只是应用程序和服务端进程通信的类而已，只负责通信，把AMS的任务交给ActivityThread。

- AMS，全称ActivityManagerService，负责统筹服务端对activity创建的流程。

  其他的类，后面的源码解析会详解。

#### 根Activity的创建

根Activity也就是我们点击桌面图标的时候，应用程序第一个activity启动的流程。这里我侧重讲解多个进程之间的关系，下面的源码也不会讲细节，只讲解普通activity的创建流程。这里也相当于一个补充。先看整体流程图：

<img src="https://s1.ax1x.com/2020/08/02/aYGZVJ.png" alt="aYGZVJ.png" border="0" width=60%/>

主要涉及四个进程：

- Launcher进程，也就是桌面进程
- 系统服务进程，AMS所在进程
- Zygote进程，负责创建进程
- 应用程序进程，也就是即将要启动的进程

主要流程：

1. Launcher进程请求AMS创建activity
2. AMS请求Zygote创建进程。
3. Zygote通过fork自己来创建进程。并通知AMS创建完成。
4. AMS通知应用进程创建根Activity。

和普通Activity的创建很像，主要多了创建进程这一步。

### 源码讲解

#### Activity请求AMS的过程

##### 流程图

<img src="https://s1.ax1x.com/2020/08/01/aGokDO.png" alt="aGokDO.png" border="0" width=80%/>

##### 源码

1. 系统通过调用Launcher的startActivitySafely方法来启动应用程序。Launcher是一个类，负责启动根Activity。

   >  这一步是根Activity启动才有的流程，普通启动是没有的，放在这里是作为一点补充而已

   

   ```java
   packages/apps/Launcher3/src/com/android/launcher3/Launcher.java/;
   public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
       	//这里调用了父类的方法，继续查看父类的方法实现
           boolean success = super.startActivitySafely(v, intent, item);
           ...
           return success;
       }
       
   ```

   ```java
   packages/apps/Launcher3/src/com/android/launcher3/BaseDraggingActivity.java/;
   public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
           ...
           // Prepare intent
           //设置标志singleTask，意味着在新的栈打开
           intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
           if (v != null) {
               intent.setSourceBounds(getViewBounds(v));
           }
           try {
               boolean isShortcut = Utilities.ATLEAST_MARSHMALLOW
                       && (item instanceof ShortcutInfo)
                       && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
                       || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
                       && !((ShortcutInfo) item).isPromise();
               //下面注释1和注释2都是直接采用startActivity进行启动。注释1会做一些设置
               //BaseDraggingActivity是继承自BaseActivity，而BaseActivity是继承自Activity
               //所以直接就跳转到了Activity的startActivity逻辑。
               if (isShortcut) {
                   // Shortcuts need some special checks due to legacy reasons.
                   startShortcutIntentSafely(intent, optsBundle, item);//1
               } else if (user == null || user.equals(Process.myUserHandle())) {
                   // Could be launching some bookkeeping activity
                   startActivity(intent, optsBundle);//2
               } else {
                   LauncherAppsCompat.getInstance(this).startActivityForProfile(
                           intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
               }
               
              ...
           } 
       	...
           return false;
       }
   ```

   

2. Activity通过Instrumentation来启动Activity

   ```java
   /frameworks/base/core/java/android/app/Activity.java/;
   public void startActivity(Intent intent, @Nullable Bundle options) {
       	//最终都会跳转到startActivityForResult这个方法
           if (options != null) {
               startActivityForResult(intent, -1, options);
           } else {
               // Note we want to go through this call for compatibility with
               // applications that may have overridden the method.
               startActivityForResult(intent, -1);
           }
       }
   
   public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
               @Nullable Bundle options) {
       	//mParent是指activityGroup，现在已经采用Fragment代替，这里会一直是null
       	//下一步会通过mInstrumentation.execStartActivity进行启动
           if (mParent == null) {
               options = transferSpringboardActivityOptions(options);
               Instrumentation.ActivityResult ar =
                   mInstrumentation.execStartActivity(
                       this, mMainThread.getApplicationThread(), mToken, this,
                       intent, requestCode, options);//1
               if (ar != null) {
                   mMainThread.sendActivityResult(
                       mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                       ar.getResultData());
               }
               ...
           }
       ...
   }
   ```

3. Instrumentation请求AMS进行启动。该类的作用是监控应用程序和系统的交互。到此为止，任务就交给了AMS了，AMS进行一系列处理后，会通过本地的接口IActivityManager来进行回调启动activity。

   ```java
   /frameworks/base/core/java/android/app/Instrumentation.java/;
   public ActivityResult execStartActivity(
               Context who, IBinder contextThread, IBinder token, Activity target,
               Intent intent, int requestCode, Bundle options) {
       ...
       //这个地方比较复杂，先说结论。下面再进行解释    
       //ActivityManager.getService()获取到的对象是ActivityManagerService，简称AMS
       //通过AMS来启动activity。AMS是全局唯一的，所有的活动启动都要经过他的验证，运行在独立的进程中
       //所以这里是采用AIDL的方式进行跨进程通信，获取到的对象其实是一个IBinder接口
           
       //注释2是进行检查启动结果，如果异常则抛出，如没有注册。
       try {
               intent.migrateExtraStreamToClipData();
               intent.prepareToLeaveProcess(who);
               int result = ActivityManager.getService()
                   .startActivity(whoThread, who.getBasePackageName(), intent,
                           intent.resolveTypeIfNeeded(who.getContentResolver()),
                           token, target != null ? target.mEmbeddedID : null,
                           requestCode, 0, null, options);//1
               checkStartActivityResult(result, intent);//2
           } catch (RemoteException e) {
               throw new RuntimeException("Failure from system", e);
           }
           return null;
       
   }
   ```

   >这一步是通过AIDL技术进行跨进行通信。拿到AMS的代理对象，把启动任务交给了AMS。
   >
   >```java
   >/frameworks/base/core/java/android/app/ActivityManager.java/;
   >//单例类
   >public static IActivityManager getService() {
   >return IActivityManagerSingleton.get();
   >}
   >
   >private static final Singleton<IActivityManager> IActivityManagerSingleton =
   >new Singleton<IActivityManager>() {
   >@Override
   >protected IActivityManager create() {
   >//得到AMS的IBinder接口
   >final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
   >//转化成IActivityManager对象。远程服务实现了这个接口，所以可以直接调用这个
   >//AMS代理对象的接口方法来请求AMS。这里采用的技术是AIDL
   >final IActivityManager am = IActivityManager.Stub.asInterface(b);
   >return am;
   >}
   >};
   >```

   

#### AMS处理请求的过程

##### 流程图

<img src="https://s1.ax1x.com/2020/08/01/aGoya4.png" alt="aGoya4.png" border="0" width=80%/>

##### 源码

1. 接下来看AMS的实现逻辑。AMS这部分的源码是通过ActivityStartController来创建一个ActivityStarter，然后把逻辑都交给ActivityStarter去执行。ActivityStarter是android 7.0加入的类。

   ```java
    /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java/;
   //跳转到startActivityAsUser
   //注意最后多了一个参数UserHandle.getCallingUserId()，表示调用者权限
   public final int startActivity(IApplicationThread caller, String callingPackage,
           Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
           int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
       return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
               resultWho, requestCode, startFlags, profilerInfo, bOptions,
               UserHandle.getCallingUserId());
   }
   
   public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
               Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
               int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
               boolean validateIncomingUser) {
           enforceNotIsolatedCaller("startActivity");
   
           userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                   Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");
   
           // TODO: Switch to user app stacks here.
       	//这里通过ActivityStartController获取到ActivityStarter，通过ActivityStarter来
       	//执行启动任务。这里就把任务逻辑给到了AcitivityStarter
           return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                   .setCaller(caller)
                   .setCallingPackage(callingPackage)
                   .setResolvedType(resolvedType)
                   .setResultTo(resultTo)
                   .setResultWho(resultWho)
                   .setRequestCode(requestCode)
                   .setStartFlags(startFlags)
                   .setProfilerInfo(profilerInfo)
                   .setActivityOptions(bOptions)
                   .setMayWait(userId)
                   .execute();
   
       }
   ```

   >ActivityStartController获取ActivityStarter
   >
   >```java
   >/frameworks/base/services/core/java/com/android/server/am/ActivityStartController.java/;
   >//获取到ActivityStarter对象。这个对象仅使用一次，当他的execute被执行后，该对象作废
   >ActivityStarter obtainStarter(Intent intent, String reason) {
   >return mFactory.obtain().setIntent(intent).setReason(reason);
   >}
   >```

2. 这部分主要是ActivityStarter的源码内容，涉及到的源码非常多。AMS把整个启动逻辑都丢给ActivityStarter去处理了。这里主要做启动前处理，创建进程等等。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java/;
   //这里需要做启动预处理，执行startActivityMayWait方法
   int execute() {
           try {
              	...
               if (mRequest.mayWait) {
                   return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                           mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                           mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                           mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                           mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                           mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                           mRequest.inTask, mRequest.reason,
                           mRequest.allowPendingRemoteAnimationRegistryLookup);
               }
               ...
           } 
       	...
       }
   
   //启动预处理
   private int startActivityMayWait(IApplicationThread caller, int callingUid,
               String callingPackage, Intent intent, String resolvedType,
               IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
               IBinder resultTo, String resultWho, int requestCode, int startFlags,
               ProfilerInfo profilerInfo, WaitResult outResult,
               Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
               int userId, TaskRecord inTask, String reason,
               boolean allowPendingRemoteAnimationRegistryLookup) {
           ...
   		//跳转startActivity
            final ActivityRecord[] outRecord = new ActivityRecord[1];
        	int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                   voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                   callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                   ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                   allowPendingRemoteAnimationRegistryLookup);
   }
   
   //记录启动进程和activity的信息
   private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
           String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
           IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
           IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
           String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
           SafeActivityOptions options,
           boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
           TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
       ...
       //得到Launcher进程    
       ProcessRecord callerApp = null;
       if (caller != null) {
           callerApp = mService.getRecordForAppLocked(caller);
           ...
       }
       ...
       //记录得到的activity信息
       ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
               callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
               resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
               mSupervisor, checkedOptions, sourceRecord);
       if (outActivity != null) {
           outActivity[0] = r;
       }
       ...
       mController.doPendingActivityLaunches(false);
   	//继续跳转
       return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
               true /* doResume */, checkedOptions, inTask, outActivity);
   }
   
   //跳转startActivityUnchecked
   private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
               IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
               int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
               ActivityRecord[] outActivity) {
       int result = START_CANCELED;
       try {
           mService.mWindowManager.deferSurfaceLayout();
           //跳转
           result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                   startFlags, doResume, options, inTask, outActivity);
       } 
       ...
       return result;
   }
   
   //主要做与栈相关的逻辑处理，并跳转到ActivityStackSupervisor进行处理
   private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
           IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
           int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
           ActivityRecord[] outActivity) {
       ...
       int result = START_SUCCESS;
       //这里和我们最初在Launcher设置的标志FLAG_ACTIVITY_NEW_TASK相关，会创建一个新栈
       if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
               && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
           newTask = true;
           result = setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
       }
       ...
       if (mDoResume) {
           final ActivityRecord topTaskActivity =
               mStartActivity.getTask().topRunningActivityLocked();
           if (!mTargetStack.isFocusable()
               || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                   && mStartActivity != topTaskActivity)) {
               ...
           } else {
               if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                   mTargetStack.moveToFront("startActivityUnchecked");
               }
               //跳转到ActivityStackSupervisor进行处理
               mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                                                               mOptions);
           }
       }
   }
   ```

3. ActivityStackSupervisor主要负责做activity栈的相关工作，会结合ActivityStack来进行工作。主要判断activity的状态，是否处于栈顶或处于停止状态等

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java/;
   
   boolean resumeFocusedStackTopActivityLocked(
           ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
   	...
       //判断要启动的activity是不是出于停止状态或者Resume状态
       final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
       if (r == null || !r.isState(RESUMED)) {
           mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
       } else if (r.isState(RESUMED)) {
           // Kick off any lingering app transitions form the MoveTaskToFront operation.
           mFocusedStack.executeAppTransition(targetOptions);
       }
       return false;
   }
   ```

4. ActivityStack主要处理activity在栈中的状态

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityStack.java/;
   //跳转resumeTopActivityInnerLocked
   boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
       if (mStackSupervisor.inResumeTopActivity) {
           // Don't even start recursing.
           return false;
       }
       boolean result = false;
       try {
           // Protect against recursion.
           mStackSupervisor.inResumeTopActivity = true;
           //跳转resumeTopActivityInnerLocked
           result = resumeTopActivityInnerLocked(prev, options);
   	...
       } finally {
           mStackSupervisor.inResumeTopActivity = false;
       }
       return result;
   }
   
   //跳转到StackSupervisor.startSpecificActivityLocked，注释1
   @GuardedBy("mService")
   private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
       ...
       if (next.app != null && next.app.thread != null) {
           ...
       } else {
           ...
           mStackSupervisor.startSpecificActivityLocked(next, true, true);//1
       }     
       if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
       return true;
   }
   ```

5. 这里又回到了ActivityStackSupervisor，判断进程是否已经创建，未创建抛出异常。然后创建事务交回给本地执行。这里的事务很关键，Activity执行的工作就是这个事务，事务的内容是里面的item，所以要注意下面的两个item。

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java/;
   
   void startSpecificActivityLocked(ActivityRecord r,
           boolean andResume, boolean checkConfig) {
       //得到即将启动的activity所在的进程
       ProcessRecord app = mService.getProcessRecordLocked(r.processName,
               r.info.applicationInfo.uid, true);
       getLaunchTimeTracker().setLaunchTime(r);
   
       //判断该进程是否已经启动,跳转realStartActivityLocked，真正启动活动
       if (app != null && app.thread != null) {
           try {
               if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                       || !"android".equals(r.info.packageName)) {
                   app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                           mService.mProcessStats);
               }
               realStartActivityLocked(r, app, andResume, checkConfig);//1
               return;
           } 
           ...
       }
       mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
               "activity", r.intent.getComponent(), false, false, true);
   }
   
   
   //主要创建事务交给本地执行
   final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
       boolean andResume, boolean checkConfig) throws RemoteException {
       ...
       //创建启动activity的事务ClientTransaction对象
       // Create activity launch transaction.
       final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
               r.appToken);
       // 添加LaunchActivityItem，该item的内容是创建activity
       clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
               System.identityHashCode(r), r.info,
               // TODO: Have this take the merged configuration instead of separate global
               // and override configs.
               mergedConfiguration.getGlobalConfiguration(),
               mergedConfiguration.getOverrideConfiguration(), r.compat,
               r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
               r.persistentState, results, newIntents, mService.isNextTransitionForward(),
               profilerInfo));
   
       // Set desired final state.
       //添加执行Resume事务ResumeActivityItem,后续会在本地被执行
       final ActivityLifecycleItem lifecycleItem;
       if (andResume) {
           lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
       } else {
           lifecycleItem = PauseActivityItem.obtain();
       }
       clientTransaction.setLifecycleStateRequest(lifecycleItem);
   
       // 通过ClientLifecycleManager来启动事务
       // 这里的mService就是AMS
       // 记住上面两个item：LaunchActivityItem和ResumeActivityItem，这是事务的执行单位
       // Schedule transaction.
       mService.getLifecycleManager().scheduleTransaction(clientTransaction);
   }
       
   ```

   >通过AMS获取ClientLifecycleManager
   >
   >```java
   >/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java/;
   >//通过AMS获取ClientLifecycleManager
   >ClientLifecycleManager getLifecycleManager() {
   >return mLifecycleManager;
   >}
   >```

6. ClientLifecycleManager是事务管理类，负责执行事务

   ```java
   /frameworks/base/services/core/java/com/android/server/am/ClientLifecycleManager.java
   void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
       final IApplicationThread client = transaction.getClient();
       //执行事务
       transaction.schedule();
       if (!(client instanceof Binder)) {
           transaction.recycle();
       }
   }
   ```

7. 把事务交给本地ActivityThread执行。这里通过本地ApplicationThread在服务端的接口IApplicationThread来进行跨进程通信。后面的逻辑就回到了应用程序进程了。

   ```java
   /frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java/;
   
   //这里的IApplicationThread是要启动进程的IBinder接口
   //ApplicationThread是ActivityThread的内部类，IApplicationThread是IBinder代理接口
   //这里将逻辑转到本地来执行
   private IApplicationThread mClient;
   public void schedule() throws RemoteException {
       mClient.scheduleTransaction(this);
   }
   ```

   

#### ActivityThread创建Activity的过程

##### 流程图

<img src="https://s1.ax1x.com/2020/08/01/aGoBrT.png" alt="aGoBrT.png" border="0" width=80%/>

##### 源码

1. IApplicationThread接口的本地实现类ActivityThread的内部类ApplicationThread

   ```java
   /frameworks/base/core/java/android/app/ActivityThread.java/ApplicationThread.class/;
   //跳转到ActivityThread的方法实现
   public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
       ActivityThread.this.scheduleTransaction(transaction);
   }
   ```
   
12. ActivityThread执行事务。ActivityThread是继承ClientTransactionHandler，scheduleTransaction的具体实现是在ClientTransactionHandler实现的。这里的主要内容是把事务发送给ActivityThread的内部类H去执行。H是一个Handle，通过这个Handle来切到主线程执行逻辑。

    ```java
    /frameworks/base/core/java/android/app/ClientTransactionHandler.java
    void scheduleTransaction(ClientTransaction transaction) {
        //事务预处理
        transaction.preExecute(this);
        //这里很明显可以利用Handle机制切换线程，下面看看这个方法的实现
        //该方法的具体实现是在ActivityThread，是ClientTransactionHandler的抽象方法
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
    
    /frameworks/base/core/java/android/app/ActivityThread.java/;
    final H mH = new H();
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        //利用Handle进行切换。mH是H这个类的实例
        mH.sendMessage(msg);
    }
    
    ```

13. H对事务进行处理。调用事务池来处理事务

    ```java
    /frameworks/base/core/java/android/app/ActivityThread.java/H.class
    //调用事务池对事务进行处理
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            ...
            case EXECUTE_TRANSACTION:
                final ClientTransaction transaction = (ClientTransaction) msg.obj;
                //调用事务池对事务进行处理
                mTransactionExecutor.execute(transaction);
                if (isSystem()) {
                    transaction.recycle();
                }
                // TODO(lifecycler): Recycle locally scheduled transactions.
                break;
                ...
        }
        ...
    }
    ```

14. 事务池对事务进行处理。事务池会把事务中的两个item拿出来分别执行。这两个事务就是上面我讲的两个Item。对应不同的初始化工作。

    ```java
    /frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
        
    public void execute(ClientTransaction transaction) {
        final IBinder token = transaction.getActivityToken();
        log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);
    
        //执行事务
        //这两个事务就是当时在ActivityStackSupervisor中添加的两个事件（第8步）
        //注释1执行activity的创建，注释2执行activity的窗口等等并调用onStart和onResume方法
        //后面主要深入注释1的流程
        executeCallbacks(transaction);//1
        executeLifecycleState(transaction);//2
        mPendingActions.clear();
        log("End resolving transaction");
    }
    
    public void executeCallbacks(ClientTransaction transaction) {
        ...
            //执行事务
            //这里的item就是当初添加的Item，还记得是哪个吗？
           	// 对了就是LaunchActivityItem
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
        ...
    }
    
    private void executeLifecycleState(ClientTransaction transaction) {
        ...
        // 和上面的一样，执行事务中的item，item类型是ResumeActivityItem
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
    
    ```

15. LaunchActivityItem调用ActivityThread执行创建逻辑。

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

16. ActivityThread执行Activity的创建。主要利用Instrumentation来创建activity和回调activity的生命周期，并创建activity的上下文和app上下文（如果还没创建的话）。

    ```java
    /frameworks/base/core/java/android/app/ActivityThread.java/;
    
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        ...
            // 跳转到performLaunchActivity
            final Activity a = performLaunchActivity(r, customIntent);
        ...
    }
    
    //使用Instrumentation去创建activity回调生命周期
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        	//获取ActivityInfo，用户存储代码、AndroidManifes信息。
        	ActivityInfo aInfo = r.activityInfo;
            if (r.packageInfo == null) {
                //获取apk描述类
                r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                        Context.CONTEXT_INCLUDE_CODE);
            }
        
        	// 获取activity的包名类型信息
        	ComponentName component = r.intent.getComponent();
            if (component == null) {
                component = r.intent.resolveActivity(
                    mInitialApplication.getPackageManager());
                r.intent.setComponent(component);
            }
        ...
            // 创建context上下文
            ContextImpl appContext = createBaseContextForActivity(r);
       		// 创建activity
            Activity activity = null;
            try {
                java.lang.ClassLoader cl = appContext.getClassLoader();
                // 通过Instrumentation来创建活动
                activity = mInstrumentation.newActivity(
                        cl, component.getClassName(), r.intent);
                StrictMode.incrementExpectedActivityCount(activity.getClass());
                r.intent.setExtrasClassLoader(cl);
                r.intent.prepareToEnterProcess();
                if (r.state != null) {
                    r.state.setClassLoader(cl);
                }
            }
        ...
            try {
                // 根据包名创建Application，如果已经创建则不会重复创建
                Application app = r.packageInfo.makeApplication(false, mInstrumentation);
                ...
                // 为Activity添加window
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
                }
    	...
            // 通过Instrumentation回调Activity的onCreate方法
            ctivity.mCalled = false;
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
    }    
    ```
    
    >这里深入来看一下onCreate什么时候被调用
    >
    >```java
    >/frameworks/base/core/java/android/app/Instrumentation.java/;
    >public void callActivityOnCreate(Activity activity, Bundle icicle,
    >        PersistableBundle persistentState) {
    >    prePerformCreate(activity);
    >    // 调用了activity的performCreate方法
    >    activity.performCreate(icicle, persistentState);
    >    postPerformCreate(activity);
    >}
    >
    >/frameworks/base/core/java/android/app/Activity.java/;
    >final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    >    mCanEnterPictureInPicture = true;
    >    restoreHasCurrentPermissionRequest(icicle);
    >    // 这里就回调了onCreate方法了
    >    if (persistentState != null) {
    >        onCreate(icicle, persistentState);
    >    } else {
    >        onCreate(icicle);
    >    }
    >    ...
    >}
    >```
    
7. Instrumentation通过类加载器来创建activity实例

   ```java
   /frameworks/base/core/java/android/app/Instrumentation.java/;
   
   public Activity newActivity(ClassLoader cl, String className,
           Intent intent)
           throws InstantiationException, IllegalAccessException,
           ClassNotFoundException {
       String pkg = intent != null && intent.getComponent() != null
               ? intent.getComponent().getPackageName() : null;
       // 利用AppComponentFactory进行实例化
       return getFactory(pkg).instantiateActivity(cl, className, intent);
   }
   ```

8. 最后一步，通过AppComponentFactory工厂来创建实例。

   ```java
   /frameworks/support/compat/src/main/java/androidx/core/app/AppComponentFactory.java
   //其实就相当于直接返回instantiateActivityCompat
   public final Activity instantiateActivity(ClassLoader cl, String className, Intent intent)
       throws InstantiationException, IllegalAccessException, ClassNotFoundException {
       return checkCompatWrapper(instantiateActivityCompat(cl, className, intent));
   }
       
   //泛型方法
   static <T> T checkCompatWrapper(T obj) {
           if (obj instanceof CompatWrapped) {
               T wrapper = (T) ((CompatWrapped) obj).getWrapper();
               if (wrapper != null) {
                   return wrapper;
               }
           }
           return obj;
       }  
   //终于到了尽头了。利用类加载器来进行实例化。到此activity的启动就告一段落了。
   public @NonNull Activity instantiateActivityCompat(@NonNull ClassLoader cl,
           @NonNull String className, @Nullable Intent intent)
           throws InstantiationException, IllegalAccessException, ClassNotFoundException {
       try {
           return (Activity) cl.loadClass(className).getDeclaredConstructor().newInstance();
       } catch (InvocationTargetException | NoSuchMethodException e) {
           throw new RuntimeException("Couldn't call constructor", e);
       }
   }
   ```


### 小结

上文通过整体流程和代码详解解析了一个activity启动时的整体流程。不知道读者们会不会有个疑问：了解这些有什么用呢？日常又用不到。当走完这整个流程的时候，你会发现你对于android又深入了解了很多了，面对开发的时候，内心也会更加有自信心，出现的一些bug，可能别人要解决好久，而你，很快就可以解决。另外，这一部分内容在插件化也有很大的使用，也是学习插件化必学的知识。

好了讲了这么多，希望以上对你有帮助。有疑问可以评论区或者私信交流一下。另外，博主属于android新手，如有不对之处还望指正。

### 参考文献
 - 《Android进阶解密》
 - 《Android开发艺术探索》
 - [Android进阶（四）：Activity启动过程(最详细&最简单)](https://www.jianshu.com/p/7d0d548ebbb4)