---
title: 'JavaScript基础-闭包,箭头函数和生成器'
date: 2017-02-03 15:46:54
tags: JavaScript基础笔记
categories: JavaScript
---
# 闭包
> 闭包简单来说即函数作为返回值(返回的函数并没有立即执行,而是在调用的时候才执行),它有两个用途

* 可以读取函数内部的变量
* 让这些变量的值始终保存在内存中(可能引起内存泄漏)

<!-- more -->

错误示例	

```javascript
function count() {
    var arr = [];
    for (var i=1; i<=3; i++) {
        arr.push(function () {
            return i * i;
        });
    }
    return arr;
}
var results = count();
var f1 = results[0];
var f2 = results[1];
var f3 = results[2];
f1(); // 16
f2(); // 16
f3(); // 16
```
结果全部为16!原因在于返回的函数引用了循环变量i,单他并没有执行,等到函数都返回时,引用的变量i=4,因此最终结果为16

图解:
![](http://okskqdic8.bkt.clouddn.com/bibao1.png)

**注:返回闭包时返回函数不要引用任何循环变量,或者后续变化的变量**

如果必须引用循环变量则需要再创建一个函数,用该函数绑定循环变量的值

正确示例:

```javascript
function count() {
    var arr = [];
    for (var i=1; i<=3; i++) {
        arr.push((function (n) {
            return function () {
                return n * n;
            }
        })(i));
    }
    return arr;
}
var results = count();
var f1 = results[0];
var f2 = results[1];
var f3 = results[2];
f1(); // 1
f2(); // 4
f3(); // 9
```

如果必须引用循环变量则需要再创建一个函数,用该函数绑定循环变量的值
# 箭头函数
> 箭头函数相当于匿名函数,但实际上与匿名函数相比有明显的区别:箭头函数中的this是词法作用域,由上下文确定.修复了this的指向问题

## 箭头函数的两种格式
* 只包含一个表达式: x=> x * x 相当于 function(x){ return x * x }

* 包含多条语句

例:

```javascript
x => {
    if (x > 0) {
        return x * x;
    }
    else {
        return - x * x;
    }
}
```
相当于

```javascript
function(x){
	if(x>0){
		return x * x;
	}else{
		retrurn - x * x;
	}
}
```

## 箭头函数需要注意的问题
### 多参数
如果参数不是一个需要用括号括起来

```javascript
// 两个参数:
(x, y) => x * x + y * y

// 无参数:
() => 3.14

// 可变参数:
(x, y, ...rest) => {
    var i, sum = x + y;
    for (i=0; i<rest.length; i++) {
        sum += rest[i];
    }
    return sum;
}
```
### 如果返回的是一个对象,也需要一个括号
错误:

```javascript
// SyntaxError:
x => { foo: x }
```
正确:

```javascript
// ok:
x => ({ foo: x })
```

# 生成器 generator
> generator 就是可以返回多次的函数

## 格式
generator由 **function*** 定义(注意多了一个*号),并且除了return语句,还可以用yield返回多次.

```javascript
function* foo(x) {
    yield x + 1;
    yield x + 2;
    return x + 3;
}
```
## generator调用方法
* 第一种方法不断调用generator对象的next()方法,next方法会执行generator的代码,然后每次遇到yield x 就会返回一个对象  {value: x, done : false/true},然后暂停.返回的value就是yield的返回值,done 表示这个generator是否执行结束,如果done 为true 表示generator执行结束,value是return返回的值.
* 第二种方法是直接使用for...of 循环迭代,这种方式不需要我们自己判断done

```javascript
for (var x of fib()) {
    console.log(x); // 依次输出0, 1, 1, 2, 3
}
//fib() 是一个generator函数
```

## generator作用
* 保存状态
* generator还有另一个巨大的好处，就是把异步回调代码变成“同步”代码

> 本笔记根据 [廖雪峰老师的JavaScript](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000) 教程记录