---
layout: posts
title: Activity启动流程
date: 2018-11-08 14:29:22
tags: [源码解析,笔记,android]
categories: 源码解析
---


# 概述

Activity作为Android的四大组件之一,使用最为频繁.得益于最近比较清闲,特地看了一遍Activity的启动流程.现在总结于此.Activity的启动流程源码真是一大片又一大片.在次特地感谢[YoungerHu的博客](https://www.jianshu.com/p/459d38ade254).本篇记录是从startActivity方法进行记录的



<!-- more -->



# Activity整体时序图如下

<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/activity_start.png" width="90%" height="80%" />





# 具体方法调用整理



## 1. startActivity

	startActivity()是Activity中的方法,还有另一个重载的方法.因为简单就不贴源码了.startActivity最后调用的都是startActivityForResult方法.
> 我们以最简单的 startActivity(new Intent(xxActivity.calss)) 为例,不涉及参数传递,只看最直接的启动过程

## 2. startActivityForResult

 startActivityForResult也有很多重载方法,根据我们的例子实际调用的是
startActivityForResult(Intent intent, int requestCode,Bundle options),

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

##  3.execStartActivity

在Instrumentation中execStartActivity也有很多重载方法
实际调用的是execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target,Intent intent, int requestCode, Bundle options);

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
## 4.ActivityManagerService.startActivity,startActivityAsUser

ActivityManagerService中的startActivity方法并没有做什么,然后直接调用startActivityAsUser,在原有参数的基础上加了一个userId参数.
userId参数是跟多用户有关的,在我们的Activity启动流程中不用关心.
startActivityAsUser方法对userId进行了处理后就直接调用startActivityMayWait方法了.

## 5.startActivityMayWait


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



startActivityMayWait方法解析了目标activity并对一些状态进行了检验,然后调用startActivityLocked方法,继续启动Activity.让我们来看此方法.



##  6.startActivityLocked



``` java
int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
            TaskRecord inTask, String reason) {

        //省略代码
       
        //单独抽取了一个方法,继续启动activity
        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                container, inTask);

        //省略代码

        return mLastStartActivityResult;
    }
```

startActivityLocked方法很简单,并没有做太多处理,然后继续调用了ActivityStarter的startActivity方法



##  7.ActivityStarter.startActivity

``` java
// 不要直接调用这个方法。使用{@link # startactivitylock}代替。 
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
            TaskRecord inTask) {
    
        
        // 省略代码,检测用户UId,是否为同一用户,检测权限等

        doPendingActivityLaunchesLocked(false);
		//此处调用的是10.startActivity方法,与7方法参数不同
        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
                options, inTask, outActivity);
    }
```



## 8.doPendingActivityLaunchesLocked

``` java
/**
mPendingActivityLaunches中记录着所有将要启动的Activity.由于在startActivityLocked的过程时App切换功能被禁止, 也就是不运行切换Activity, 那么此时便会把相应的Activity加入到mPendingActivityLaunches队列. 该队列的成员在执行完doPendingActivityLaunchesLocked便会清空.
启动mPendingActivityLaunches中所有的Activity, 由于doResume = false, 那么这些activtity并不会进入resume状态,而是设置delayedResume = true, 会延迟resume.）
*/
final void doPendingActivityLaunchesLocked(boolean doResume) {
        while (!mPendingActivityLaunches.isEmpty()) {
            final PendingActivityLaunch pal = mPendingActivityLaunches.remove(0);
            final boolean resume = doResume && mPendingActivityLaunches.isEmpty();
            try {
                //此处调用的是10.startActivity方法,与7方法参数不同
                startActivity(pal.r, pal.sourceRecord, null, null, pal.startFlags, resume, null,
                        null, null /*outRecords*/);
            } catch (Exception e) {
                Slog.e(TAG, "Exception during pending activity launch pal=" + pal, e);
                pal.sendErrorResult(e.getMessage());
            }
        }
    }
```



##  9. ActivityStarter.startActivity

``` java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
        //省略代码

        //继续启动Activity
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
        //省略代码

        //Activity启动以后活动处理
        postStartActivityProcessing(r, result, mSupervisor.getLastStack().mStackId,  mSourceRecord,
                mTargetStack);

        return result;
    }
```



## 10.startActivityUnchecked

``` java
    // 注意:此方法只能从{@link startActivity}调用
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {

        //设置初始状态
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);
        //处理Activity加入任务栈的方式,是加入原有任务栈,还是新的任务栈
        computeLaunchingTaskFlags();
        computeSourceStack();
        mIntent.setFlags(mLaunchFlags);

        //重用Activity
        ActivityRecord reusedActivity = getReusableIntentActivity();

        //省略代码

        //处理Activity的singleTop和SingleTask启动模式
        final ActivityStack topStack = mSupervisor.mFocusedStack;
        final ActivityRecord topFocused = topStack.topActivity();
        final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart = top != null && mStartActivity.resultTo == null
                && top.realActivity.equals(mStartActivity.realActivity)
                && top.userId == mStartActivity.userId
                && top.app != null && top.app.thread != null
                && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || mLaunchSingleTop || mLaunchSingleTask);
        if (dontStart) {
            //省略代码
            if (mDoResume) {
                mSupervisor.resumeFocusedStackTopActivityLocked();
            }
            //省略代码
            return START_DELIVERED_TO_TOP;
        }
        //省略代码

        //将Activity放入任务栈中,并且创建window等
        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);

        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                //省略代码
            } else {
                //省略代码
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else {
            mTargetStack.addRecentActivityLocked(mStartActivity);
        }
       
        //省略代码

        return START_SUCCESS;
    }
```



此方法中mTargetStack.startActivityLocked方法将Activity加入到了任务栈中,但此时Activity还没有显示出来.在继续调用resumeFocusedStackTopActivityLocked方法后,Activity才可见.



## 11.resumeFocusedStackTopActivityLocked

```java
	boolean resumeFocusedStackTopActivityLocked() {
        return resumeFocusedStackTopActivityLocked(null, null, null);
    }
    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        if (targetStack != null && isFocusedStack(targetStack)) {
            //继续调用
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || r.state != RESUMED) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.state == RESUMED) {
            // Kick off any lingering app transitions form the MoveTaskToFront operation.
            mFocusedStack.executeAppTransition(targetOptions);
        }
        return false;
    }
```

## 12.resumeTopActivityUncheckedLocked

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            mStackSupervisor.inResumeTopActivity = true;
            //继续调用
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        
        mStackSupervisor.checkReadyForSleepLocked();

        return result;
    }
```



## 13.resumeTopActivityInnerLocked

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        
        //省略代码

        //在启动新的Activity之前先,将当前的Activity onPause
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            //startPausingLocked 中断当前显示的Activity,最主要的操作是与应用进程的ApplicationThread进行			//AIDL通信，调用其schedulePauseActivity函数.
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }

         //省略代码
        
        mStackSupervisor.startSpecificActivityLocked(next, true, true);

         //省略代码
        return true;
    }
```

此方法很复杂,代码行数超多.很难找到其中的关键代码. startSpecificActivityLocked方法在Android8.0源码的2633行. 继续来看此方法



## 14. startSpecificActivityLocked

```java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.getStack().setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                //省略代码

                //看到这方法名,终于感觉到了曙光,真正启动Activity的方法
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
        }
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```



## 15.realStartActivityLocked

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {

        //省略代码

        //app.thread是IApplicationThread,此处通过AIDL调用应用层ApplicationThread的
        //scheduleLaunchActivity方法
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info,    
                mergedConfiguration.getGlobalConfiguration(),
                mergedConfiguration.getOverrideConfiguration(), r.compat,
                r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                r.persistentState, results, newIntents, !andResume,
                mService.isNextTransitionForward(), profilerInfo);
           
        //省略代码
        return true;
    }
```

现在又回到了应用层离Activity显示已经不远了.ApplicationThread是ActivityThread的内部类,在ActivityThread的685行.我们继续来看scheduleLaunchActivity



## 16.scheduleLaunchActivity

```java

@Override
 public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
     	CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

        updateProcessState(procState, false);

        //对新启动的Activity进行参数赋值,并通过Handler继续下一步操作
        ActivityClientRecord r = new ActivityClientRecord();
        r.token = token;
      	r.ident = ident;
        r.intent = intent;
        r.referrer = referrer;
        r.voiceInteractor = voiceInteractor;
        r.activityInfo = info;
        r.compatInfo = compatInfo;
       	r.state = state;
        r.persistentState = persistentState;
        r.pendingResults = pendingResults;
        r.pendingIntents = pendingNewIntents;
        r.startsNotResumed = notResumed;
        r.isForward = isForward;
	    r.profilerInfo = profilerInfo;
        r.overrideConfig = overrideConfig;
        updatePendingConfiguration(curConfig);
        sendMessage(H.LAUNCH_ACTIVITY, r);
}
```



## 17. Handler

```java
private class H extends Handler {
        public static final int LAUNCH_ACTIVITY  = 100;
        
        //省略代码

        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    //处理Activity
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;

                //省略代码
            }

        }
                
    }


private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        
        //省略代码

        Activity a = performLaunchActivity(r, customIntent);

        //省略代码
    }
```



## 18.performLaunchActivity

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

        //省略代码

        //获取要启动的Activity的包名类名等信息
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        //创建Context上下文实例
        ContextImpl appContext = createBaseContextForActivity(r);
        
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //创建Activity实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            
            //省略代码
        } catch (Exception e) {
            //省略代码
        }

        try {
            //创建Application,如果已经创建则直接返回
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            //省略代码

            if (activity != null) {
                //省略代码

                //获取window对象
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                //调用Attach方法,初始化Activity实例的一些数据
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                 //省略代码
               
                //启动Activity,调用callActivityOnCreate,在callActivityOnCreate方法中调用了Activity的
                //onCreate方法
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                //调用performStart后,调用mInstrumentation.callActivityOnStart(this);
                //即onStart方法
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                //省略其他生命周期
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```

在执行了以上主要的调用后,Activity就显示到屏幕上,至于具体的创建Context,Application等方法就不在此记录了,有兴趣的可以自己去看看.