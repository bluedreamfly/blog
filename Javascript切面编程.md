---
title: Javascript 切面编程
date: 2017-04-15 10:59:55
categories: ['技术', '模式']
---

#### 什么是切面编程
对于切面编程，可能对于Java开发的同学可能很熟悉，但是在前端看来相对陌生，又或者可能无意中用到，但不知道自己用了。那么什么是切面编程呢？维基百科是这样解释的。
  >面向侧面的程序设计（aspect-oriented programming，AOP，又译作面向方面的程序设计、观点导向编程、剖面导向程序设计）是计算机科学中的一个术语，指一种程序设计范型。该范型以一种称为侧面（aspect，又译作方面）的语言构造为基础，侧面是一种新的模块化机制，用来描述分散在对象、类或函数中的横切关注点（crosscutting concern）。

#### AOP应用
看着是有那么点抽象，下面我们来举个现实中的例子。

我们大家都知道我们经常需要在页面中的操作当中来判断用户是否登陆。比如页面中的一些信息根据用户师是否登陆展示不一样，可能很大一部分人会这样写代码。而且可能一个应用有很多地方需要这样做。

```javacript
function showAuthInfo() {
	if (user.isLogin) {
		//登录
	} else {
		//未登录
	}
}
```
突然某一天需求说这些内容登录跟没登录都统一显示，那到时你就坑。要改动的地方很多，而且一改动就可能改出其他问题。而且从上面的业务当中，我们可以知道，其实判断登录不是业务相关，所以我们可以单独抽离放在登录相关模块。而且以后可能在业务当中不止要判断是否登录，可能需要增加上报用户行为的功能，这时候可能又会在业务中杂糅一些业务无关的代码，这样的系统其实是很脆弱，动不动一改需求整个系统很大部分都会相应需要改动。这时候可能我们会这样升级自己的代码。

```javascript
const login = {
	handleAuth() {
	  if (user.isLogin) {
	     
		} else {
	   
	  }
	}
}
const logger = {
	log() {
	  report()
	}
}

function showAuthInfo() {
	login.handleAuth();
	logger.log();
}
```
比起最开始的代码似乎好了不少，但是还是没又解决问题，还是需要改动原有的代码，而且业务无关的代码依然还是存在于业务代码当中，我们只是在原有基础上进行了一些公用的代码的抽离。那么就是AOP出场的时候了，我们先来个简单的代码，虽然实现不是那么优雅，但是已经大概描述了

```javascript
    
	Function.prototype.before = function(beforeFns) {
		const _this = this;
		return function() {
		 	
			if (Array.isArray(beforeFns)) {
				for(let i = 0; i < beforeFns.length; i++) {
					beforeFn[i].apply(this, arguments);
					
				}
			}
			if (typeof beforeFns === 'function') {
				beforeFns.apply(this, arguments);
			}
			
			_this.apply(this, arguments);
		}
	}
	
	showAuthInfo.before([login.handleAuth, logger.log]);
```
那么现在看起来是不是好点了，基本上业务跟验证以及日志上报模块已经算是分离了。 业务跟其他一些系统级别的一些东西已经不是强相关了，随时可以卸载掉。直到这里，我们已经展示了AOP在现实当中的应用，用一张图来表示：

![](http://oofjmuxr0.bkt.clouddn.com/aop.jpeg)

现在只是能够实现AOP中功能，那么如何在应用更好的使用它，因为可能有些相同的业务，但是有的需要权限和日志模块，但是有的却是不需要。所以，为了更好拓展代码，我们需要进行一些改进。把配置的东西统一放在一个地方来处理。

```javascript
	//业务代码
	const bussinessLogic = {
		showAuthInfo() {
		  //具体的业务
		}
	}
	
	const aopBussiness = {
	   //函数名我是随意取的，后面是根据业务不同命名，以后不管删除或者添加模块都很方便，跟业务无关
		aopAuthLogShowAuthInfo() {
		   return bussinessLogic.showAuthInfo.before([login.handleAuth, logger.log])
		},
		aopAuthShowAuthInfo() {
			return bussinessLogic.showAuthInfo.before(login.handleAuth)
		},
		aopLogShowAuthInfo() {
			return bussinessLogic.showAuthInfo.before(logger.log)
		}
		
	}
```
我们可以看到上面这样的代码，这样就将一些模块于具体的业务代码进行分离了。

除了before，还有after, around 具体实现也是相对简单的
```javascript
   //after
	Function.prototype.after = function(afterFn) {
		const _this = this;
		return function() {
		 	beforeFn.apply(this, arguments);
			_this.apply(this, arguments);
		}
	}
	//around
	Function.prototype.around = function(aroundFn) {
		const _this = this;
		return function() {
		 	aroundFn.apply(this, arguments);
			_this.apply(this, arguments);
			aroundFn.apply(this, arguments);
		}
	}
```
#### 总结
AOP的概念及实际应用基本是这样，除了AOP其实还有挺多有意思的代码组织的模式，比如Ioc(控制反转)，中间件的一些方式，后面有机会也会在后续相应的博文进行展开。