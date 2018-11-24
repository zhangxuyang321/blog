---
layout: posts
title: Activity启动流程
date: 2018-11-08 14:29:22
tags: [源码解析,笔记,android]
categories: 源码解析
---
## 概述
Activity作为Android的四大组件之一,使用最为频繁.得益于最近比较清闲,特地看了一遍Activity的启动流程.现在总结于此.Activity的启动流程源码真是一大片又一大片.在次特地感谢[YoungerHu的博客](https://www.jianshu.com/p/459d38ade254).本篇记录是从startActivity方法进行记录的



<!-- more -->



## Activity整体时序图如下

##具体方法调用整理

### 1. startActivity
	startActivity()是Activity中的方法,还有另一个重载的方法.因为简单就不贴源码了.startActivity最后调用的都是startActivityForResult方法.
> 我们以最简单的 startActivity(new Intent(xxActivity.calss)) 为例,不涉及参数传递,只看最直接的启动过程

### 2. startActivityForResult
 startActivityForResult也有很多重载方法,根据我们的例子实际调用的是
startActivityForResult(Intent intent, int requestCode,Bundle options),
由1可知,此时参数:

intent: 传递过来的Intent,携带着ComponentName(包含包名和要打开的Activity的类名,与Intent的一样全都实现了序列化)

requestCode: -1

options: null

在此基础上分析代码如下

```java
	public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
	//mParent在此一直为空,具体原因在下边解释a
        if (mParent == null) {  
		//options为null,所以我们不关心这个
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
              //此处调用了Instrumentation的execStartActivity方法
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,intent, requestCode, options);
            
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
           //省略无关代码
        } else {
                //省略无关代码
            }
        }
    }
```
#### 解释
(a) mParent为什么为空[点这里](https://blog.csdn.net/JustBeauty/article/details/78278958)

### execStartActivity
在Instrumentation中execStartActivity也有很多重载方法
实际调用的是execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target,Intent intent, int requestCode, Bundle options);此时参数:

who: 原Activity的Contenxt上下文实例

contextThread: 原Activity的ApplicationThread,ApplicationThread继承了IApplicationThread.Stub,可见ApplicationThread是IBinder的实现类(AIDL)

token: 原Activity传递过来的mToken

target: 原Activity实例

intent: 继续传递的Intent

requestCode: -1

options: null

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        //省略代码
        try {
		//省略代码

		// ActivityManager.getService()获取到IActivityManager实例,
		//IActivityManager在源码中是一个aidl文件,所以这里用到了AIDL进程通信,实际实现类是ActivityManagerService
		//调用此方法后,启动页进入到了框架层中
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
### ActivityManagerService.startActivity,startActivityAsUser
ActivityManagerService中的startActivity方法并没有做什么,然后直接调用startActivityAsUser,在原有参数的基础上加了一个userId参数.
userId参数是跟多用户有关的,在我们的Activity启动流程中不用关心.
startActivityAsUser方法对userId进行了处理后就直接调用startActivityMayWait方法了.

### startActivityMayWait
startActivityMayWait(IApplicationThread caller, int callingUid,String callingPackage, Intent intent, String resolvedType,IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,IBinder resultTo, String resultWho, int requestCode, int startFlags,ProfilerInfo profilerInfo, WaitResult outResult,Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,IActivityContainer iContainer, TaskRecord inTask, String reason)

此方法真是参数超多啊!一个一个看吧

caller: 原ApplicationThread

callingUid: -1

callingUid: 包名

intent: 传递过来的Intent

resolvedType: null

voiceSession: null

voiceInteractor: null

resultTo: 原Activity传递过来的mToken

resultWho: 从注释"package"来看应该是包名,原Activity标识

requestCode: -1

startFlags:0

profilerInfo: null

outResult: null

globalConfig: null

bOptions: null

ignoreTargetSecurity: false

userId: 传递过来的userId

iContainer: null

inTask: null

reason: startActivityAsUser

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask, String reason) {
        
        //省略代码

        // 备份Intent
        final Intent ephemeralIntent = new Intent(intent);
        // 拷贝一份intent,不在原有的Intent上做改动
        intent = new Intent(intent);
        //省略代码

        //解析Intent 
        ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
        //省略代码
        // resolveIntent和resolveActivity解析出目标Activity的信息
        ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
        //options为null 不用关心
        ActivityOptions options = ActivityOptions.fromBundle(bOptions);
        //iContainer为null,container也为null
        ActivityStackSupervisor.ActivityContainer container =
                (ActivityStackSupervisor.ActivityContainer)iContainer;
        synchronized (mService) {
            //省略代码
            if (aInfo != null &&
                    (aInfo.applicationInfo.privateFlags
                            & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
                // 检查一下我们是否已经有另一个不同的重载进程在运行。并非Activity启动主要流程
                if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                    final ProcessRecord heavy = mService.mHeavyWeightProcess;
                    if (heavy != null && (heavy.info.uid != aInfo.applicationInfo.uid
                            || !heavy.processName.equals(aInfo.processName))) {
                        //省略代码
                    }
                }
            }

            final ActivityRecord[] outRecord = new ActivityRecord[1];
            
            int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                    aInfo, rInfo, voiceSession, voiceInteractor,
                    resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                    options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                    inTask, reason);

            //省略代码
            return res;
        }
    }
```

















