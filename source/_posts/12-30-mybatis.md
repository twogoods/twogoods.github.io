title: 使用mybatis的注意点
date: 2014-12-30 20:42:49
categories: mybatis
tags: 问题小计
---
 1. 像mybatis,hibernate等等框架大量用到Java的反射，千万要在你的类中定义默认的构造方法
 2. 想这样含多个参数的方法 `public String[] getUserbh(String deptbh,String banzubh);
    `配置文件中可用0,1等序号或者将参数变成 `Map<String,Object> map`

```
  <select id="getUserbh" parameterType="String" resultType="String">
	    select usergh from people where deptbh= #{0} and banzubh= #{1}
  </select> 
```
<!--more-->
而这样是会报错的
(当然你可以用注解的方式让其准确`public String[] getUserbh(@Param("deptbh")String deptbh,@Param("banzubh")String banzubh);`)
```
  <select id="getUserbh" parameterType="String" resultType="String">
	   select usergh from people where deptbh= #{deptbh} and banzubh= #{banzubh}
  </select>
```

错误如下：
org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.binding.BindingException: Parameter 'deptbh' not found. Available parameters are [0, 1, param1, param2]　　
　3. mybatis中的动态SQL使用OGNL 的表达式
```
 <if test="title != null and title!=''">
        AND title like #{title}
 </if>
 ```