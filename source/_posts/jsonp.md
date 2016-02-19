title: 跨域请求jsonp
date: 2015-07-22 22:13:59
categories: javascript
tags:
---
一次面试的时候被问到了跨域请求，提到了jsonp但一直没好好去看，今天再一次碰到这个，就好好的了解了一下。
这篇文章讲了[原理][1]，而且比较容易懂,推荐一看。
jsonp主要是利用`<script type="text/javascript" src="scripts/jquery.min.js"></script>`
中的src可以跨域获取数据，既然是通过script标签的src属性来请求，那么当然jsonp只有get方式的请求喽。

  [1]: http://www.cnblogs.com/yuzhongwusan/archive/2012/12/11/2812849.html