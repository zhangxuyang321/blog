---
title: Android保活记录
date: 2017-03-31 16:41:43
tags: [android,保活,笔记]
categories: 笔记
---

## 介绍
因为实际业务需要实时接收推送,所以应用保活也成为了必要功能,我承认保活真的是耍流氓~

## 灰色保活

> 灰色保活,目前应用最广泛的手段.利用Android系统漏洞来启动一个前台的Service,区别在于它不会在通知栏出现通知,用户基本没有察觉

<!-- more -->

```java
	private final static int KEEP_ID = 123456; 
	if (Build.VERSION.SDK_INT < 18) {
            startForeground(KEEP_ID, new Notification());//API < 18 ，此方法能有效隐藏Notification上的图标
        }else{
            Intent keep = new Intent(this, KeepInnerService.class); 
            startService(keep);
            startForeground(KEEP_ID, new Notification());
        }
```

KeepInnerService

```java
public class KeepInnerService extends Service {
    private final static int KEEP_ID = -1001;
    public KeepInnerService() {
    }
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        startForeground(KEEP_ID, new Notification());
        stopForeground(true);
        stopSelf();
        return super.onStartCommand(intent, flags, startId);
    }
}
```

## 1像素保活(QQ神技)

### 创建一个Activity

```java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        PkApplication.addActivity(this);
        PkApplication.keppLiveInstance = this;
        Window window = getWindow();
        window.setGravity(Gravity.LEFT|Gravity.TOP);
        WindowManager.LayoutParams attributes = window.getAttributes();
        attributes.x=0;
        attributes.y=0;
        attributes.height=0;
        attributes.width=0;
        window.setAttributes(attributes);
    }
```

### 设置清单文件

```xml
<activity
            android:name=".weight.activitys.KeepLiveActivity"
            android:configChanges="keyboardHidden|orientation|screenSize|navigation|keyboard"
            android:excludeFromRecents="true"
            android:exported="false"
            android:finishOnTaskLaunch="false"
            android:launchMode="singleInstance"
            android:process=":live"
            android:theme="@style/keepLive" />
```

#### android:configChanges(未验证)

1. 不设置此属性, Activity切换横竖屏时会重新调用各个生命周期,切横屏会执行一次,切竖屏会执行两次

2. 设置android:configChanges="orientation",调用各个生命周期,切横竖屏都是一次

3. 设置Activity的android:configChanges="orientation|keyboardHidden"时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法

#### android:excludeFromRecents
是否显示在最近打开的Activity列表里

#### android:exported
是否允许activity被其他程序调用
#### android:finishOnTaskLaunch
当用户重启这个任务的时候是否关闭已打开的Activity
#### android:process
一个Activity运行时所在的进程名,所有组件运行在默认的进程中,这个进程跟包名一直. process属性能够为所有组件设定一个新的默认值.但任何组件都可以覆盖这个默认值,允许你将你的程序放在多进程中使用.如果这个属性被":"开头命名,当这个Activity运行时,一个新的专属于这个程序的进程将被创建.如果以小写字母开头,这个Activity将运行在全局的进程中

### 主题设置

```xml
<style name="keepLive">
        <item name="android:windowBackground">@android:color/transparent</item> //背景
        <item name="android:windowFrame">@null</item> //边框
        <item name="android:windowNoTitle">true</item> //有无标题
        <item name="android:windowIsFloating">true</item> //是否浮现在Activity之上
        <item name="android:windowIsTranslucent">true</item> //半透明
        <item name="android:windowContentOverlay">@null</item> //窗口内容放置的Drawable资源
        <item name="android:backgroundDimEnabled">false</item> // 模糊
        <item name="android:windowAnimationStyle">@null</item> //Dialog 进入、退出动画
        <item name="android:windowDisablePreview">true</item> //禁用默认启动窗口
        <item name="android:windowNoDisplay">false</item> //当前启动的窗口不可见
    </style>
```

### 监听锁屏,解锁启动 KeepLiveActivity
随着Android版本的升高,目前静态监听屏幕锁屏,解锁已不可用,只能动态注册

```java
private BroadcastReceiver screenReciver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if(action.equals(Intent.ACTION_SCREEN_OFF)){
                Intent keep = new Intent(getApplicationContext(),KeepLiveActivity.class);
                startActivity(keep);
            }

            if (action.equals(Intent.ACTION_USER_PRESENT)){
                if(null!=PkApplication.keppLiveInstance){
                    PkApplication.keppLiveInstance.finish();
                }
            }
        }
    };

private void initScreenBroadcastReceiver(){
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Intent.ACTION_SCREEN_OFF);
        intentFilter.addAction(Intent.ACTION_USER_PRESENT);
        registerReceiver(screenReciver,intentFilter);
    }

注意
@Override
    protected void onDestroy() {
        super.onDestroy();
        unregisterReceiver(screenReciver); //记得卸载监听
        }
```

注:
	此方法使用后出现了,在APP活动可见情况下锁屏,解锁后出现APP退出,可能是android:launchMode="singleInstance"原因,但是当APP点击HOME键以后再锁屏解锁则不受影响,目前还不清楚原因

解决办法:
	在接受到锁屏通知前,先调用HOME功能

```java
	Intent home = new Intent(Intent.ACTION_MAIN);
                home.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                home.addCategory(Intent.CATEGORY_HOME);
                startActivity(home);
```

## adb 命令
进入adb  shell 模式

### 打印指定包名的进程中所有Service信息
dumpsys activity services PackageName
### 查看进程优先级

* ps | grep PackageName  获取指定包名的进程信息,
	如果报 grep: can't execute: Permission denied可以先ps 打印所有的进程信息,复制到文本中,通过包名查找找到对应的进程ID

<img src="http://okskqdic8.bkt.clouddn.com/jincheng.png" width="50%" height="50%" />

* cat /proc/进程ID/oom_adj

<img src="http://okskqdic8.bkt.clouddn.com/youxianji.png" width="50%" height="50%" />

### oom_adj概念
* 进程的oom_adj越大,表示进程的优先级越低,越容易被回收;越少,表示进程优先级越高
* 普通APP的oom_adj>=0,系统进程的oom_adj才可能<0
* APP退出到后台所有的进程优先级都会降低. 其中UI进程降低的最为明显,因为它占用的内存资源最多.



## Tanks
[保活攻略](http://www.jianshu.com/p/63aafe3c12af)

[Android属性控件大全](http://www.cnblogs.com/fly-fish/p/4872289.html)

[Android系统主题属性](http://blog.csdn.net/babybear1224/article/details/50055113)

[configChanges属性](http://blog.csdn.net/zhaokaiqiang1992/article/details/19921703)

