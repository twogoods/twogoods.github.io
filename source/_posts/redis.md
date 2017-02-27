title: redis的安装
date: 2015-06-28 22:03:13
categories: redis
tags: redis安装
---
一个新东西入门总不是那样简单的，从下载，安装到完成helloworld的过程总是要花些时间的，记下来供以后参考。<!--more--> 
### 下载
下载肯定是去看看[redis](http://redis.io)官网啦,不过官网却并没有官方的Windows版本，提供了 Microsoft Open Tech团队的贡献的windows版本的[链接](https://github.com/MSOpenTech/redis)，可以在这里[下载](https://github.com/MSOpenTech/redis/releases)msi安装包。一些老的博客上的下载地址会指向[这个](https://github.com/dmajkic/redis)，不过已经很久没维护了，文档也说了正式的应该是上一个Open Tech团队的。
### 安装
确定好要下载哪个后，就是安装了，msi安装文件就非常方便，直接安装并且功能添加到了系统服务中。如果是zip版本跟着文档手动的也可以很快的安装，添加服务。安装好后，命令行进入redis目录，使用命令`redis-server.exe redis.windows-service.conf`开启服务（当然添加了服务也可以去系统服务里打开），接下来就可以做一个简单的测试了
```
redis-cli.exe -h 192.168.7.83 -p 6379  //连接服务，注意IP
set name haha   //添加键值对
get name        //得到键为name的值
```
到这来算是把redis装好了。
ps:后来到公司用上了高大上的Mac，mac下安装就容易多了，直接brew安装。
一些基本的命令，集群的配置在这个翻译好的[网站](http://redisdoc.com/)上都有。