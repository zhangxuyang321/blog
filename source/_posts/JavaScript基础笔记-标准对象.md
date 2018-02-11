---
title: JavaScript基础笔记-标准对象
date: 2017-02-09 10:25:20
tags: JavaScript基础笔记
categories: JavaScript
---
本节记录了 

* 标准对象
* Date

***
## 标准对象
Javascript中一切都是对象,为了区分对象类型,我们用 **typeof** 操作符获取对象类型,它总是返回一个字符串:

<!-- more -->

```javascript
typeof 123; // 'number'
typeof NaN; // 'number'
typeof 'str'; // 'string'
typeof true; // 'boolean'
typeof undefined; // 'undefined'
typeof Math.abs; // 'function'
typeof null; // 'object'
typeof []; // 'object'
typeof {}; // 'object'
```
**特别注意 null 的类型是 Object,Array的类型也是 Object,如果我们用typeof将无法区分出null, array 和通常意义的object--{}**

## 包装对象
包装对象用new创建

```JavaScript
var n = new Number(123); // 123,生成了新的包装类型
var b = new Boolean(true); // true,生成了新的包装类型
var s = new String('str'); // 'str',生成了新的包装类型
```
虽然看上去一模一样,但是经过以上包装后,他们的类型全都变成了object!包装对象和原始值通过===比较会返回false

```JavaScript
typeof new Number(123); // 'object'
new Number(123) === 123; // false

typeof new Boolean(true); // 'object'
new Boolean(true) === true; // false

typeof new String('str'); // 'object'
new String('str') === 'str'; // false
```
>另外如果我们在使用Number、Boolean和String时，没有写new会发生什么情况？
此时，Number()、Boolean和String()被当做普通函数，把任何类型的数据转换为number、boolean和string类型（注意不是其包装类型）：

```JavaScript
var n = Number('123'); // 123，相当于parseInt()或parseFloat()
typeof n; // 'number'
var b = Boolean('true'); // true
typeof b; // 'boolean'
var b2 = Boolean('false'); // true! 'false'字符串转换结果为true！因为它是非空字符串！
var b3 = Boolean(''); // false
var s = String(123.45); // '123.45'
typeof s; // 'string'
```
需要遵守的规则

* 不要使用new Number(), new Boolean(), new String()创建包装对象
* 用parseInt()或parseFloat来转换任意类型到number
* 用String()来转任意类型到string,或者直接调用某个对象的toString()方法 
* 通常不必把任意类型转换为boolean再判断,因为可以直接写if(myVar){...}
* typeof可以判断出number, boolean,string,function和undefined
* 判断array 要使用 Array.isArray(arr);
* 判断null 请使用myVar===null
* 判断某个全局变量是否存在 typeof window.myVar === 'undefined'
* 函数内部判断某个变量是否存在用 type myVar === 'undefined'

**特别注意null和undefined没有toString()方法. number对象调用toString()报错需要按一下方式处理**

```JavaScript
123.toString(); // SyntaxError
123..toString(); // '123', 注意是两个点！
(123).toString(); // '123'
```
## Date
> Date对象用来表示时间和日期

获取当前系统时间

```javascript
var now = new Date();
now; // Thu Feb 09 2017 15:00:16 GMT+0800 (CST)
now.getFullYear(); // 2017, 年份
now.getMonth(); // 1, 月份, 注意月份是0~11, 1 表示2月;
now.getDate(); // 9, 表示9号
now.getDay(); // 4, 表示星期四
now.getHours(); // 15点, 24小时制
now.getMinutes(); // 18分
now.getSeconds(); // 58秒
now.getMilliseconds(); // 625 毫秒
now.getTime(); // 1486624909839 时间戳

//创建一个指定日期的对象
//第一种
var d1 = new Date(2017, 3, 21, 08, 15, 30, 123);
console.log(d1); // Fri Apr 21 2017 08:15:30 GMT+0800 (CST)
//第二种
var d2 = Date.parse('2017-02-09T15:40:30.666+08:00')
console.log(d2); // 1486626030666
//时间戳转换Date
var d3= new Date(1486626030666);
console.log(d3); // Thu Feb 09 2017 15:40:30 GMT+0800 (CST)
```
**注意当前时间是浏览器从本机操作系统获取的时间,所以不一定准确,因为用户可自己随意设置**

### 时区
> Date对象表示的时间都是按浏览器所在的时区显示的,不过我们既可以显示本地时间,也可以显示调整后的时间

```JavaScript
var d = new Date(1486626030666);
d.toLocaleString(); // 2017/2/9 下午3:40:30, 显示格式与操作系统设置有关
d.toUTCString(); // Thu, 09 Feb 2017 07:40:30 GMT,与本地时间相差8个小时
``` 
#### 关于时区转换
只要我们通过时间戳,就无需担心时间转换

```JavaScript
if(Date.now){
	alert(Date.now()); // 老版IE没有now()方法
}else {
	alert(new Date().getTime());
}
```

> 本笔记根据 [廖雪峰老师的JavaScript](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000) 教程记录
