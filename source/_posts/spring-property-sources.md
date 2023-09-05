---
title: spring 配置优先级
date: 2023-07-16 20:26:43
categories: 
- Spring
---

Spring Boot获取配置参数有一个顺序, 这个顺序是特殊的:
1. Default properties (specified by setting SpringApplication.setDefaultProperties).
2. `@PropertySource` annotations on your @Configuration classes. Please note that such property sources are not added to the Environment until the application context is being refreshed. This is too late to configure certain properties such as logging.* and spring.main.* which are read before refresh begins.
3. Config data (such as application.properties files)
4. A RandomValuePropertySource that has properties only in random.*.
5. OS environment variables.
6. Java System properties (System.getProperties()).
7. JNDI attributes from java:comp/env.
8. ServletContext init parameters.
9. ServletConfig init parameters.
10. Properties from SPRING_APPLICATION_JSON (inline JSON embedded in an environment variable or system property).
11. Command line arguments.
12. properties attribute on your tests. Available on @SpringBootTest and the test annotations for testing a particular slice of your application.
13. @TestPropertySource annotations on your tests.
14. Devtools global settings properties in the $HOME/.config/spring-boot directory when devtools is active.
以上这些就是Spring Boot启动时, 默认读取Property的位置和顺序.
其中, 注解`@PropertySource`需要注意, 通过注解`@PropertySource`来定位配置文件`.properties`
```java
@PropertySource(value = "classpath:application.properties", ignoreResourceNotFound = true)
```
当你运行SpringBoot启动类的main方法, SpringBoot将会自动按顺序读取这14个PropertySource(低版本的可能是17个, 当前版本是2.5.6), 为的是将这些配置注入到项目中.

> 提示: Spring Boot将配置的值注入bean的属性有两种办法
> 1. @Value
> 2. @ConfigurationProperties
