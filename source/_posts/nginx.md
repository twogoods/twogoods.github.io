title: nginx
date: 2016-03-02 13:59:19
categories: nginx
tags: nginx
---
nginx是著名的反向代理服务器，负载均衡利器，利用一些脚本甚至可以实现动态的负载均衡。这里只是记一下入门的helloworld。  

启动nginx ，sudo nginx ;访问localhost:8080 发现已出现nginx的欢迎页面了。
配置文件：/usr/local/etc/nginx/nginx.conf  
<!--more-->
重新加载配置|重启|停止|退出 nginx 
nginx -s reload|reopen|stop|quit
```
server {
        listen       9001; 监听的端口
        server_name  banka;
        location / {
        	proxy_pass http://local_tomcat; 指向upstream
            root   html;
            index  index.html index.htm;
        }
    }

    upstream local_tomcat {  //配了两个服务，weight为比重
	    server localhost:8080 weight=1;  
	    server localhost:10000 weight=1;  
	} 
```
多加一个服务名如mi,代理访问localhost:9000/mi/demo，实际访问的是localhost:8080/demo/。

```
    server {
        listen       9001;
        server_name  banka;
        location ^~ /mi/ {
        	proxy_pass http://localhost:8080/;
        }

    }
```