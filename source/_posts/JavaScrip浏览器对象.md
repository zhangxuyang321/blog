---
title: JavaScrip浏览器对象
date: 2017-04-05 16:49:08
tags: [javaScript,笔记]
categories: JavaScript
---
## window
> window即是全局作用域也是浏览器窗口

* window对象的innerWidth 和 innerHeight属性表示,浏览器窗口内部的宽高.(内部宽高指出去菜单栏,工具栏,边框等占位元素后,用于显示网页的净宽高)

* window对象的outerWidth和outerHeight表示浏览器窗口的整个宽高

## navigator(浏览器的信息)

<!-- more -->

常用属性:

navigator.appName: 浏览器名称

navigator.appVersion: 浏览器版本

navigator.language: 浏览器设置的语言

navigator.platform: 操作系统类型

navigator.userAgent: 浏览器设定的 'user-Agent' 字符串

**注意:** navigator的信息很容易被用户修改,所以读取的值不一定正确

## screen(屏幕信息)

常用属性:

screen.width: 屏幕宽度,单位像素

screen.height: 屏幕高度,单位像素

screen.colorDepth: 返回颜色数,如8,16,24

## location

> location 对象表示当前页面的URL信息

例

```javascript

	//一个完整的URL,可以用location.href获取.
	http://www.example.com:8080/path/index.html?a=1&b=2#TOP

	//获取各个部分
      location.protocol;  	// http
      location.host;       	// www.example
      location.port; 		// 8080
      location.pathname;    // '/path/index.html'
      location.search;	    //  '?a=1&b=2'
	  location.hash;	    // TOP

	  location.assign('新的URL地址');  //加载一个新的界面
      location.reload(); 	重新加载当前界面
```

## document

> document 对象表示当前界面.也是整个DOM树的根节点

要查找某个DOM树的某个节点,需要从document对象开始查找,常用根据ID或Tag Name

### Cookie

> Cookie是由服务器发送的key-value标示符. 因为http是没有状态的,但是服务器要区分是哪个用户发的请求,就可以用cookie来区分.cookie开可以用来保存网站的一些设置,例如页面显示的语言等.

JavaScript可以通过document.cookie,获取当前界面的cookie

服务器在设置Cookie时可以使用httpOnly,来防止JavaScript读取cookie

## history

> histroy对象保存了浏览器记录JavaScript可以调用history对象的back()或forward(),相当于点击了浏览器的前进和后退





> 本笔记根据 [廖雪峰老师的JavaScript](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000) 教程记录



