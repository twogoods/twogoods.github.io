title: git的基本操作
date: 2015-07-02 20:17:42
categories: git
tags: git
---
1. 设置Git的user name和email：
`$ git config --global user.name "XXX"`
`$ git config --global user.email "XXX@gmail.com"`
2. 键入命令：`ssh-keygen -t rsa -C "email@email.com"`
`email@email.com`是git账号；
3. 提醒你输入key的名称，输入如`id_rsa`；
4. 在用户user目录下产生两个文件：`id_rsa`和`id_rsa.pub`；
5. 用打开id_rsa.pub文件，复制内容，在github.com的网站上到ssh密钥管理页面，添加新公钥，随便取个名字，内容粘贴刚才复制的内容；

基本操作：
创建 	`git init`
添加到版本库 	 `git add readme.txt`
提交到本地	`git commit -m "fiorst commit"`
版本回退  	`git reset --hard commit_id（head、head^、head~100）`
查看状态	 `git status`
克隆    	`git clone XXXXX`
撤销文件的修改 	`git checkout -- readme.txt`
<!--more-->
创建分支 	`git branch "name"`
切换分支   	 `git checkout "name"`
创建分支+切换 	 `git checkout -b "branchname"`
合并某分支到当前分支	`git merge "name"`
删除分支    	`git branch -d "name"`
查看分支  	 `git branch`
查看分支图    `git log --graph`
添加远程库	`git remote add origin https://git……`  
这里也有阮大神的[总结](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)
最后，不忘出处，学git当然要看[廖雪峰](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/)啦。