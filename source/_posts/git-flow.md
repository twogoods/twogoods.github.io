title: git flow
date: 2016-01-05 17:35:02
categories: git
tags: git
---
用过git，那么肯定要碰到gitflow，gitflow个人认为是多人协作下使用git的一个规范，完全就是git操作，不过安装git flow后特定的命令可以帮我们做很多的事情，理解可以参考阮一峰的[文章](http://www.ruanyifeng.com/blog/2012/07/git.html)。  

功能开发
```
# 初始化工作目录(一直回车即可)
git flow init 
# 开始创建新的需求分支(这个会从develop分出来，创建新的分支)
git flow feature start editimage #这时项目会自动切换 feature/editimage分支
# 更改部分代码后
# git commit -a -m "修改完了"
# 完成开发分支合并develop(自动完成,只合并到develop，没有合并到master)
git flow feature finish editimage
# 发布到远程开发分支
git push origin develop
```
<!--more-->
bug修复
```
    
# 切换到master分支
git checkout master
# 更新master分支
git pull origin master（更新master分支为最新） 
#生成一个hotfix分支(这个从master分支上切出来，不是从develop)
git flow hotfix start hfx     
# 最终修改完成后，结束hotfix以供发布(修改后的代码会自动合并到master和develop两个分支)
git flow hotfix finish hfx
﻿# 发布最终的master分支和develop分支
git push origin master
git push origin develop
```