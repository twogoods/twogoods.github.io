title: tiny4j:一个轻量级的类似Spring的实现
date: 2016-12-04 22:06:02
categories: java
tags: tiny4j
---
会点java，做点web，基本也就是spring全家桶，所以打算自己折腾一个，实现最基本最常用的一些功能。断断续续地终于完成了大部分自己想要的功能。实际项目中使用或许还不太现实，不过也提供了一个去了解框架实现的一个简单的版本，也让大家有动力有思路自己去实现一个，源码请戳[github](https://github.com/twogoods/tiny4j)。<!--more-->
### IOC
IOC很大程度借鉴了Spring,简单的使用

```
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("test.xml");
ServiceBean serviceBean=(ServiceBean)applicationContext.getBean("testService");
System.out.println(serviceBean);
serviceBean.service();

ServiceBean serviceBean2=(ServiceBean)applicationContext.getBean("serviceBean");
System.out.println(serviceBean2);
serviceBean2.service();

//全局的容器上下文
ApplicationContextHolder holder=applicationContext.getBean("applicationContextHolder", ApplicationContextHolder.class);
System.out.println("holder get bean : "+holder.getBean("serviceBean"));
```
[IOC详细说明](https://github.com/twogoods/tiny4j/tree/master/core)
### rest
实现了许多SpringMvc里高频使用的功能和一些针对restful改进的功能

```
@Api("/base")
public class TestController extends BaseController {

    @Value("${user.name:test}")
    private String name;

    @Inject
    private UserService userService;

    @RequestMapping
    public String index() {
        userService.query();
        return name;
    }

    @RequestMapping(mapUrl = "/test/{id}", method = HttpMethod.GET)
    @CROS(origins = "www.baidu.com", methods = {HttpMethod.GET}, maxAge = "3600")
    public String patgTest(@PathVariable("id") String id) {
        return id;
    }

    @RequestMapping(mapUrl = "/test", method = HttpMethod.GET)
    @InterceptorSelect(include = {"aInterceptor"}, exclude = {"bInterceptor"})
    public String interceptorTest() {
        return "haha";
    }


    @RequestMapping(mapUrl = "/index")
    @CROS
    public String paramTest(@RequestParam("id") long id, @RequestParam("name") String name) {
        return name + "---" + id;
    }

    @RequestMapping(mapUrl = "/user/{id}", method = HttpMethod.PUT)
    @CROS
    public User insert(@PathVariable("id") long id, @RequestBody User user) {
        return user;
    }
}
```
看着是不是很熟悉-_- [rest详细说明](https://github.com/twogoods/tiny4j/tree/master/rest)

