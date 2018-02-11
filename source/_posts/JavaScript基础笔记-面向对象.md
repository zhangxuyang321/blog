---
title: JavaScript基础笔记-面向对象
date: 2017-02-17 15:57:01
tags: JavaScript基础笔记
categories: JavaScript
---
## 面向对象
> Javascript的面向对象不区分类和示例的概念,而是通过原型(prototype)来实现的

原型是指当我们想要创建 **xiaoming** 这个具体学生时,我们并没有 **Student** 类型可用,但是此时正好有这么一个对象:

<!-- more -->

```javascript
var robot = {
    name: 'Robot',
    height: 1.6,
    run: function () {
        console.log(this.name + ' is running...');
    }
};
```
robot对象有名字,有身高,很像小明,所以就根据它来创建小明

```javascript
var robot = {
name: 'Robot',
height: 1.4,
run: function(){
	console.log(this.name+'is running........');
}
}
function createStudent(name) {
    // 基于robot原型创建一个新对象:
    var s = Object.create(Student);
    // 初始化新对象:
    s.name = name;
    return s;
}
var xiaoming = createStudent('小明');
xiaoming.run(); // 小明 is running...
```
## 创建对象

>  JavaScript对每个创建的对象都会设置一个原型,指向他的原型对象.当我们用**obj.xxx**访问一个对象的属性时,JS引擎会现在当前对象上查找该属性,如果没找到就到其原型对象上找,如果还没找到就一直上溯到Object.prototype对象,最后如果还是没有找到,就返回undefined.

### 构造函数
除了直接使用**{.......}**创建一个对象外,JavaScript还支持用构造函数来创建对象,用法:

```javaScript
function Student(name){
this.name = name;
this.hello = function(){
	alert('Hello,'+this.name+'!');
}
}

var xiaoming = new Student('小明');
xiaoming.name; // '小明'
xiaoming.hello(); // Hello, 小明!
```
**注意:**如果不写new,这就是一个普通函数,它返回undefind,但是如果写了new,它就变成了一个构造函数,它绑定的this指向新创建的对象,并默认返回this,也就是说不用在写 return this;
新创建的原型链接是:
	xiaoming---> Student.prototype-->object.prototype-->null

## 原型继承
因为JavaScript根本不存在class这中类型,我们无法直接扩展一个class,需要修改原型链.
例:我们基于Student扩展出PrimaryStudent.

```javascript
//PrimaryStudent函数
function PrimaryStudent(props){
	Student.call(this,props);
	this.grade = props.grade || 1;
}
//空函数
function F(){}
//把F的原型指向Student.prototype
F.prototype = Student.prototype
//把PrimaryStudent的原型指向一个新的F对象,F对象的原型指向Student.prototype
PrimaryStudent.prototype = new F();
//把PrimaryStudent原型的构造函数修复为PrimaryStudent
PrimaryStudent.prototype.constructor = PrimaryStudent;
//继续在PrimaryStudent原型(就是new F()对象)上定义方法
PrimaryStudent.prototype.getGrade = function(){
	return this.grade;
}
//创建小明对象
var Xiaoming = new PrimaryStudent({
	name:'小明',
	grade:2
});

//验证原型:
Xiaoming.__proto__===PrimaryStudent.prototype;  //true
Xiaoming.__proto__ .__proto__ == PrimaryStudent.prototype; //true

//验证继承关系
Xiaoming instanceof PrimaryStudent; //true
Xiaoming instanceof Student; //true

```

优化后

```javaScript

//封装
function inherits(Child,Parent){
var F = function(){};
F.prototype=Parent.prototype;
Child.prototype = new F();
Child.prototype.constroctor = Child;
}

function Student(props){
this.name = props.name || 'Unnamed'
}
Student.prototype.hello = function(){
	alert('Hello,'+this.name);
}

function PrimaryStudent(props){
	Student.call(this,props);
	this.grade = props.grade || 1;
}

//实现原型继承
inherits(PrimaryStudent,Student);
//绑其他方法到PrimaryStudent
PrimaryStudent.prototype.getGrade = function(){
	return this.grade;
}
```

## ES6 class继承
如果用新的class关键字来编写Student,可以这样写:

```javascript
	class Student{
	constructor(name){
		this.name = name;
	}
	hello(){
	alert('Hello,'+ this.name);
	}
}
```
class继承

```javascript
	class PrimaryStudent extends Student{
		constructor(name,grade){
			super(name);
			this.grade = grade;
		}
		myGrade(){
			alert('I am at grade: ' + this.grade);
		}
	}
```


> 本笔记根据 [廖雪峰老师的JavaScript](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000) 教程记录

