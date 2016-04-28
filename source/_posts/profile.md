title: profile及配置服务
date: 2016-04-05 21:11:32
categories: java
tags: config-server
---
一套完整的代码肯定有不同的profile，开发环境，测试环境，线上环境；程序需要读取不同的配置文件来完成不同环境的部署。传统的spring项目需要使用`PropertyPlaceholderConfigurer`来完成不同环境下的读取；springboot则特别方便，对于application.yml,application-test.yml,application-dev.yml,在启动参数里加`-Dspring.profiles.active=test`就可以按需读取，这也还是从本地的不同文件中读取，spring-cloud也提供了一个cloud-config服务，可以非常简单的实现从云端读取不同的配置文件。
### 基于XML配置的spring项目
传统的spring项目application.xml里关于配置文件的配置，这个`CloudConfigLoader`继承`PropertyPlaceholderConfigurer`来实现按需读取<!--more-->
```
	<bean id="commonConfigurer" class="com.tg.common.configuration.CloudConfigLoader">
		<property name="locations">
			<list>
				<value>classpath*:testsettings.properties</value>
				<value>classpath*:basesetting.properties</value>
			</list>
		</property>
		<property name="settingkeys">
			<value>managersetting,basesetting</value>
		</property>
	</bean>
```
看一下CloudConfigLoader类
```
public class CloudConfigLoader extends RemoteConfigLoader {
    protected InputStream getConfigByKey(String key) throws Exception {
        String baseurl = System.getProperty("configurl");
        //profile:production/developer
        //启动加jvm参数：-Dconfigurl=xxx
        //-Dprofile=test(或product)
        String profile = System.getProperty("profile", "product");
        String requrl = baseurl;
        if(!baseurl.endsWith("/"))requrl+="/";
        requrl = requrl + key + "-" + profile+".properties";
        logger.info("远程读取配置文件url：" + requrl);
        String result = callUrl(requrl,null,"GET","");
        InputStream in = new ByteArrayInputStream(result.getBytes());
        return in;
    }
}
```
以及RemoteConfigLoader
```
public class RemoteConfigLoader extends PropertyPlaceholderConfigurer{
	
	private String settingkeys;

	protected InputStream getConfigByKey(String key) throws Exception {
		String baseurl = System.getProperty("configurl");
		String requrl = baseurl + "?key=" + key;
		logger.info("远程读取配置文件url：" + requrl);
		String json = callUrl(requrl,null,"GET","");
		InputStream in = new ByteArrayInputStream(json.getBytes());
		return in;
	}

	protected String callUrl(String url, Map<String, String> map,
			String method, String cookie) {
		//http请求读取云端文件
	}
	

	protected void loadProperties(Properties props) throws IOException {
		super.loadProperties(props);
		String configurl = System.getProperty("configurl");
		if (settingkeys != null && configurl != null){
			String [] keyarr = settingkeys.split(",");
			for (String key : keyarr){
				InputStream in = null;
				try {
					in = getConfigByKey(key);
					props.load(in);
				} catch (Exception e) {
					logger.error("",e);
				}finally{
					IOUtils.closeQuietly(in);
				}
			}
			
		}
	}
}

```
相应的从本地读不同的配置文件,修改getConfigByKey方法
```
	 protected InputStream getConfigByKey(String key) throws Exception {
	        //profile:test/product
	        String profile = System.getProperty("profile", "product");
	        String configFile = key + "-" + profile+".properties";
	        logger.info("读取配置文件：" + configFile ",
	                当前环境为："+("test".equals(profile)?"测试":"生产"));
	        InputStream in = Thread.currentThread().getContextClassLoader().
	                getResourceAsStream(configFile);
	        return in;
	 }
```
### springboot
springboot中application.yml里配置不同的profiles,如下分development，production
```
spring:
    profiles: development
server:
  port: 10000
  contextPath: /demo
---
spring:
    profiles: production

server:
  port: 8080
  contextPath: /demo
```
或者分别配置在不同的文件中，application.yml,application-dev.yml,application-test.yml,默认使用application.yml
打成jar包后使用命令`java -jar -Dspring.profiles.active=dev 51banka.jar`

### Spring Cloud Config
Spring Cloud Config 被设计的非常简单，几乎不需要写代码，看官方文档就可以，添加一些spring cloud的pom依赖，这个就不列出来了。
```
@SpringBootApplication
@EnableConfigServer
//EnableConfigServer注解一定要加
public class ConfigServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServiceApplication.class,args);
    }
}
```
resource下有application.yml,bootstrap.yml;这两个的区别是bootstrap.yml里是相对不变的配置，application.yml里是可以根据cloud服务动态变的。bootstrap.yml:
```
spring:
  application:
    name: configService
server:
  port: 9999
```
而allpication.yml里的配置,使用了git服务，uri下的git仓库存放了不同的配置文件,命名规则：{application}-{profile}.yml
```
spring:
  cloud:
    config:
      server:
        git:
          uri: https://git.coding.net/twogoods/ConfigService.git
          username: xxx
          password: yyy
```
文档里除了application和profile外还有一个lable，lable主要是一个版本信息，若使用git做服务，lable就是不同的分支  
启动这个服务，配置服务是否成功可以通过url查看：http://localhost:9999/{application}/{profile}[/{label}]
当使用这个配置服务时，可以增加配置`spring.cloud.config.uri: http://myconfigserver.com`或者启动增加jvm参数`-Dspring.cloud.config.profile=test -Dspring.cloud.config.uri=http://127.0.0.1:9999/`










