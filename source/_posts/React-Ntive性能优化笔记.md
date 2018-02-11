---
title: React-Ntive性能优化笔记
date: 2017-07-30 11:38:56
tags: [ReactNative,笔记]
categories: ReactNative
---

## 概述
	ReactNative作为时下流行的跨平台开发语言,其性能可以与原生媲美.但在开发过程种,因为程序编写不当,也会有很多"卡顿"发生,在学习了性能优化方面的知识后,现将具体的优化方法总结如下.
##主要优化方式
> 减少render

## PureComponent & Component
### 优化场景

>在实际开发中,一般我们都会继承Component来实现各种各样的组件,并且也经常会出现组件之间的嵌套,即父子组件.在这样的情况下,如果父组件的State发生改变,那么父组件render将会被重新执行.如果子组件此时也是继承的Component,那么子组件的render方法也将会被重新执行,但是子组件的State并没有发生改变,这样无疑浪费了很多性能.

<!-- more -->


代码示例1:

```JavaScript
class SonComponent extends Component{
		render(){
			const {title,callback} = this.props;
			return(
				<Button title={title} onPress={callback}/>
			)
		}
	}
```

```JavaScript	
class FatherComponent extends Component{
	this.state={
		text: '呵呵'
	}
	render(){
		return(
			<View>
				<Text>{this.state.text}</Text>
				<SonComponent title="点击改变文字"  
				onPress={()=>{this.setState({text:"哈哈"})}}/>
			</View>
		)
	}
}
```
### 优化方式
将子组件继承PureComponent

### 原理

>PureComponent能具备上述功能,是因为其自身在shouldComponentUpdate(object nextProps, object nextState)生命周期方法中做了属性和状态的比较.shouldComponentUpdate方法在返回true时,才会进行componentWillUpdate(),render(),componentDidUpdate()方法更新组件

## 函数方法优化

### 优化场景
在代码示例1中,子组件还是会重新render的.因为在父组件的<SonComponent />中onPress写了一个匿名的箭头函数.这样父组件在每次render的时候,都会重新创建一个onPress方法.这样在子组件的shouldComponentUpdate仍然会返回true,导致子组件重新render
### 优化方式
将onPress中的方法提取出来

代码示例2:

```javaScript
class FatherComponent extends Component{
	this.state={
		text: '呵呵'
	}
	render(){
		return(
			<View>
				<Text>{this.state.text}</Text>
				<SonComponent title="点击改变文字"  
				onPress={this.fatherCallback}/>
			</View>
		)
	}
	fatherCallback = ()=>{
		this.setState({text:"哈哈"})
	}
}
```

### 原理
{()=>{}} === {()=>{}} 为false,所以上述代码示例1中的写法,子组件将会被重新render. 而在成员上使用箭头函数,只要FatherComponent不会被重新创建则,函数的引用也是唯一的

## 合理的拆分组件
### 优化场景
ReactNative是推崇组件化思想的,将具有单独功能的需求封装成一个组件,结合PureComponent优化,就可以再次达到优化的目的

## 组件属性改变的优化

### 优化场景
假设此时我们需要写一个类似ViewPager的组件,在page变化的时候滚动到相应位置,但是ViewPgaer中的子组件并没有任何修改.此时只是因为Page变化而需要重新render则就得不偿失了.
### 优化方式
结合shouldComponentUpdate生命周期方法来进行判断,如果只是page发生变化则return false
### 代码示例

```JavaScript
	class ViewPager extends PureComponent{
		componentWillReceiveProps(newProps){
			if(newProps.page !== this.props.page){
				this.refs.scroll.scrollTo(
					x:newProps.page * PAGE_SIZE
				);
			}	
		}
		shouldComponentUpdate(newProps,newState){
			if(swallowArrayCompare(this.props.children,newProps.children)){
				return ture;
			}
			return false
		}
		render(){
			renturn(
				<ScrollView ref='scroll'>
					{this.props.childern}
				</ScrollView>
			)
		}
	}

```

## 导航动画方面的优化
### 优化场景
在实际开发中,在执行动画的时候,也有较大数据的请求与处理,例如导航动画.此时,页面会有明显的卡顿.

### 优化方式
#### InteractionManager.runAfterInteractions(()=>{耗时操作})
优点:
	此方法在动画执行和用户触摸时不会执行,这样讲动画的执行和业务逻辑分开
缺点:
	整体业务处理时间变长
#### 数据分开渲染

* 不要同时渲染过多数据,在ListView中,常存在的数据放在renderHeader中,并且通过pageSize来分开渲染数据,dataSouce中的数据通过InteractionManager.runAfterInteractions来加载
* 如果不是List这种结构数据,而是许多不同的组件渲染,则需要使用Incremental.在代码示例3中,IncrementalPresenter中的Incremental会依次渲染,不会同时渲染过多组件

代码示例3

```javascript
	render(){
	return(
		<ScrollView>
			{Array(10).fill().map((rowIdx)=>{
				<IncrementalPresenter key={rowIdx}>
					<Row>
						{Array(20).fill().map((weightIdx)=>{
							<Incremental key={weightIdx}>
								<SlowWeight/>
							</Incremental>
						 })}		
					 </Row>
				   </IncrementalPresenter>
				})}
		</ScrollView>
		)
	}
```

