title: javascript基础点
date: 2015-02-10 10:46:18
categories: javascript
tags: 问题小计
---
以下是自己学习JavaScript中遇到的问题，网上的这些博文给出了很好的解答，在此一并小计一下。
PS:《javascript高级程序设计》值得一读。
 1. 立即执行函数(function(){...})() 与 (function(){...}()) 的区别？  [解答][1]
 2. [作用域的问题][2]  ，js里只有函数作用域：
```
	var color="red";
	function change(color){
	/*
	在函数作用域内有一个color的局部变量。
	若不是这样，alert(color)使浏览器报color is not defined的错误
	*/
		alert(color);
		color="blue";
	}
	//这里是引用外部的全局变量
	function change2(){
		color="blue";
	}
	change();// undefined
	change(color);// red
	change2();
	alert(color);
```
在执行change()时声明了局部变量color但未初始化所以是undefined；执行change(color)时声明并初始化了局部变量所以是red,紧接着`color="blue"`修改了局部变量的值.
 3. [关于闭包][3]，说的通俗些闭包就是能够读取其他函数内部变量的函数。这里有进一步的[说明][4]


  [1]: http://segmentfault.com/q/1010000000442042
  [2]: http://segmentfault.com/blog/nightire/1190000000348228
  [3]: http://www.ruanyifeng.com/blog/2009/08/learning_javascript_closures.html
  [4]: https://cnodejs.org/topic/5482d05e73dcca8d21292928