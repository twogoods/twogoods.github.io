title: springboot
date: 2015-11-04 22:54:14
categories: java
tags: springboot
---
[关于微服务](http://www.searchsoa.com.cn/2015soamicroservice/index.html)及[中文文档](https://www.gitbook.com/book/qbgbook/spring-boot-reference-guide-zh/details)。
跟着文档可以快速的上手。springboot确实可以快速的构建一个工程，过程中遇到的一个问题就是配置文件，springboot推荐使用`.properities`或`yml`两种方式。配置文件中会分别配置测试与生产环境，如下是一个完整的配置：  
```
---
#mvn spring-boot:run 时会使用此配置
datasource:
  51banka:
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3310/test
    username : root
    password :
    initialSize : 10
    maxActive : 10
    maxIdle : 10
    minIdle : 10

spring.view.prefix: /WEB-INF/jsp/
spring.view.suffix: .jsp

server:
  port: 10000
  contextPath: /bankaservice

---
#测试环境的配置，打包成jar 使用 java -jar -Dspring.profiles.active=development target/test.jar 启动
spring:
    profiles: development
datasource:
  51banka:
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3310/test
    username : root
    password :
    initialSize : 10
    maxActive : 10
    maxIdle : 10
    minIdle : 10

spring.view.prefix: /WEB-INF/jsp/
spring.view.suffix: .jsp

server:
  port: 10000
  contextPath: /test

---
# 生产环境
spring:
    profiles: production
datasource:
  51banka:
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3310/test
    username : root
    password : 
    initialSize : 30
    maxActive : 40
    maxIdle : 40
    minIdle : 30

spring.view.prefix: /WEB-INF/jsp/
spring.view.suffix: .jsp

server:
  port: 8080
  contextPath: /test
```

另文档的这一处:  
 JSP的限制
在内嵌的servlet容器中运行一个Spring Boot应用时（并打包成一个可执行的存档archive），容器对JSP的支持有一些限制。
- tomcat只支持war的打包方式，不支持可执行的jar
- 内嵌的Jetty目前不支持JSP
- Undertow不支持JSP  
所以为了防坑还是跳过jsp用别的模板吧