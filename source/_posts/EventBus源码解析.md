---
title: EventBus源码解析
date: 2017-08-17 17:33:31
tags: [源码解析,笔记,android]
categories: 源码解析
---
## 介绍
EventBus在Android开发中运用很广泛,在观看其源码后受益颇深,现将源码笔记整理如下
## EventBus注册流程
### 注册部分整体流程图如下

* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register.png" width="70%" height="70%" />

<!-- more -->

### SubscriberMethodFinder类
#### findSubscriberMethods(Class<?> subscriberClass)
* subscriberClass 订阅者的Class对象

	<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%231.png" width="70%" height="70%" />

* 此方法主要是用来判断是否存在缓存,如果不存在缓存则调用findUsingInfo方法寻找订阅的方法集

#### findUsingInfo(Class<?> subscribeClass)
* subscriberClass 订阅者的Class对象

	<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%232.png" width="70%" height="70%" />

* 在此方法中,如果我们开启了索引,在编译时会生成订阅信息通过getSubscriberInfo()方法将订阅信息封装进FindState中.如果没有使用索引则调用findUsingReflectionInSingleClass()方法寻找订阅信息.无论调用哪个方法都会将信息封装进FindState中.

#### FindState类介绍
* FindSate是SubscriberMethodFinder类的静态内部类.
<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%233.png" width="70%" height="70%" />

* FindState的两级校验在findUsingReflectionInSingleClass中说明


#### findUsingReflectionInSingleClass(FindState findState)

* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%234.png" width="70%" height="70%" />
* FindState中的双重校验
* checkAdd()

>	在checkAdd方法中,anyMethodByEventType是以事件类型为key,订阅事件的方法为value的HashMap.一般来说我在订阅类的方法中对同一个事件的监听只有一个方法,所以基本上existing都是为null,然后返回.但是不排除在父类中也同样对事件监听或者本类中多个方法对通一个事件监听,所以还需要使用checkAddWithMethodSingnature()方法进行第二层校验
 <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%235.png" width="70%" height="70%" />

* checkAddWithMethodSingnature()

> 由checkAddWithMethodSingnature()方法可知,同一个类中可以有多个方法对同一事件监听
> <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%236.png" width="70%" height="70%" />

* 此时寻找订阅信息已经完成,又回到了findUsingInfo()方法中,并向下执行到getMethodsAndRelease(FindState findState)方法.在getMethodAndRelease()方法中将FindState中的subscriberMethods取出返回并将FindSate重置为初始状态.此时SubscriberMethodFinder类执行完毕又回到了EventBus类的register()方法中,然后将subscriberMethods遍历调用subscribe方法将订阅关系维护进EventBus对应的三个Map集合中.

### subscribe(Object subscriber,SubscriberMethod subscriberMethod)
* EventBus中三个订阅关系的HashMap
<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%237.png" width="70%" height="70%" />
* subscriptionsByEventType关系维护如下
<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%238.png" width="70%" height="70%" />
* typesBySubscriber维护如下
<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%239.png" width="70%" height="70%" />
* stickyEvents的维护是在发送粘性事件是存入的,如下
<img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/register%2310.png" width="70%" height="70%" />

## EventBus 事件发送
### 事件发送整体流程图如下
* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/post.png" width="70%" height="70%" />
### post(Object event)
* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/post%231.png" width="70%" height="70%" />

### postSingleEvent(Object event, PostingThreadState postingState)
* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/post%232.png" width="70%" height="70%" />

### postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass)
postSingleEventForEventType方法主要是根据事件类型的Class对象回去到对应的subscriptions,然后遍历调用postToSubscription()方法

### postToSubscription(Subscription subscription, Object event, boolean isMainThread);
postToSubscription方法主要是用来判断是否要切换线程的,isMainThread是标示post发送时所处的线程是否是主线程,subscription.subscriberMethod.threadMode标示接受事件希望所处的线程,通过比较,如果一致则调用invokeSubscriber()方法通过反射调用.如果不一致则通过对应的MainThreadPoster,BackgroundPoster和AsyncPoster来进行线程切换.

### invokeSubscriber(Subscription subscription, Object event)
* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/post%233.png" width="70%" height="70%" />

## 粘性事件的发送
粘性事件关系维护是在发送的时候,粘性事件的处理是在1.注册流程中的subscribe中,与正常事件的处理正好相反.这么做的原因是因为粘性事件发送的时候,还没有订阅者订阅此事件,所以无法发送.而订阅关系的发生最终会调用subscribe中,所以在此方法中处理粘性事件,代码如下:

* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/post%234.png" width="70%" height="70%" />
* <img src="http://zhangxy-blog.oss-cn-beijing.aliyuncs.com/post%235.png" width="70%" height="70%" />
* 粘性事件的处理最终会调用postToSubscription()方法,以后就跟普通事件一样处理了

## 注解相关传送门
* [Android注解入门](http://www.jianshu.com/p/9ca78aa4ab4d)
* [java注解处理器](https://race604.com/annotation-processing/)
* [鸿洋大神编写基于编译时注解项目](http://blog.csdn.net/lmj623565791/article/details/51931859)
* [自动生成代码三方库](https://github.com/square/javapoet)
* [javapoet使用解析](http://www.jianshu.com/p/95f12f72f69a)

