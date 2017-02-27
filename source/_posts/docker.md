title: docker
date: 2017-02-25 20:43:46
categories: docker
tags: docker
---
折腾下docker，由于是Mac，安装非常得简单，下载dmg直接安装就行。docker一些基本的概念image，container也都好理解，跑一下nginx，熟悉一下命令行，也算是完成了helloworld了。镜像pull很慢，可以使用[daocloud](https://www.daocloud.io/)的镜像。<!--more-->  

接下来看看自己写的Java程序怎么build一个镜像.SpringBoot其实官方就有[文档](http://spring.io/guides/gs/spring-boot-docker/)，也非常的简单，用的maven插件spotify。自己在打包的时候报了个500错误，还找了好一会儿，结果发现自己二了，`dockerfile`写成了`dockfile`，丢人了(捂脸)...找错误的时候也发现maven里`imageName`不要出现大写否则会出错。一条基本的启动命令:
`docker run -e "SPRING_PROFILES_ACTIVE=dev" -p 9002:9001 --name book-dev -t bookplatform`
`docker logs {containername}`可以查看日志，不过感觉这种方式不好，当然也可以把日志挂载到本地的目录方便查看，当然这一块的最佳实践还是去看看人家生产环境的经验分享比较好。既然是虚拟机当然也可以进入终端了`docker exec -i -t {containername} /bin/sh`  

以上都挺简单的，`dockerfile`的使用还是要多看看，docker算是开了个头。