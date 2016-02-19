title: Node初步
date: 2015-04-20 20:24:55
categories: Node
tags: Node
---
其实node在去年就接触过了，最近想进一步接触了解一下，发现菜鸟时遇到的问题还是有必要记一下。

全局安装express：
 `npm install -g express`
 express 安装好后命令行还不能使用express命令，需安装express-generato包：
  `npm install -g express-generator`
  启动express：
  `npm start`
  当修改代码时每次都要重启服务器会很不方便，node有一个小工具supervisor，安装后使用此工具会根据代码的改动自动重新部署。
  安装supervisor：
  `npm -g install supervisor`
在 package.json里有这样的代码
```
    "scripts": {
    "start": "node ./bin/www"
    }
```
  所以我们可以使用如下命令来启动express：
`supervisor node ./bin/www`