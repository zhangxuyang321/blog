---
title: JavaScript基础笔记-高级函数
date: 2017-01-17 17:23:42
tags: JavaScript基础笔记
categories: JavaScript
---
# 高阶函数
 一个函数的参数为一个函数的时候,这种函数就叫做高级函数

* map()		
* reduce()
* filter()
* sort()

***
<!-- more -->
## map()
> map(f(x))方法定义在JavaScript的Array中,数组调用map()方法可根据传入的函数得到一个新的数组.可以利用map改变数组或元素的类型

``` javascript
	{
		function pow(x) {
   		 return x * x;	
		}
		var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
		arr.map(pow); // [1, 4, 9, 16, 25, 36, 49, 64, 81]
	}
```

## reduce()
> reduce(f(x,y))也是Array中的方法,这个函数必须接受两个参数的函数,reduce(f(x,y))函数会首先将数组中的前两个元素传入f(x,y)得到计算结果后,会将这个结果与数组中的下一个元素继续作为f(x,y)的参数传入,以此类推知道最后获取一个结果(感觉有点类似递归)

``` javascript
	var arr = [1, 3, 5, 7, 9];
	arr.reduce(function (x, y) {
      return x + y;
	}); // 25
```
## filter()
> filter(f(x))用于过滤数组,会将数组的元素依次传入f(x)中作为参数,然后根据f(x)返回true还是false来决定保留还是丢弃元素

``` javascript
	var arr = [1, 2, 4, 5, 6, 9, 10, 15];
	var r = arr.filter(function (x) {
    		return x % 2 !== 0;
	});
	r; // [1, 5, 9, 15]
```

## sort()
> 排序算法是程序中常用的算法,其核心是比较两个元素的大小.如果是数字则直接比较,但是比较字符串或者比较对象,直接比较数学的大小是没有意义的,因此比较的过程需要通过函数抽象出来.通常规定,对于两个元素x和y,如果我们认为x>y,则返回1,如果认为x==y,则返回0,如果x<y,则返回-1.

		JavaScript的Array的sort()方法是根据ASCII码进行排序的
		无法理解的结果:
		[10, 20, 1, 2].sort(); // [1, 10, 2, 20]
*这是因为Array的sort()方法默认吧所有的元素转换成String再按照ASCII码的循序进行排序*
		
解决办法:
	
		sort()方法是一个高阶函数,它可以接收一个比较函数来实现自定义排序即sort(fun(x,y))
例

```Javascript
	var arr = [10, 20, 1, 2];
	arr.sort(function (x, y) {
    	if (x < y) {
       	 return -1;
   	 }
    	if (x > y) {
      		 return 1;
   	 }
    	return 0;
	}); // [1, 2, 10, 20]
```

注:sort()方法会直接对Array修改,最后返回的结果仍是当前的Array


> 本笔记根据 [廖雪峰老师的JavaScript](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000) 教程记录