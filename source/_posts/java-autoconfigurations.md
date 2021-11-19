---
title: Spring Boot自动装配原理
date: 2021-11-17 09:54:27
tags: 
- Spring
- Autoconfigurations
---

Spring Framework’s @Conditional annotation is the core of Autoconfigurations.
事实上, 自动装配是基于标准的@Configuration类来实现的. @Conditional一系列注解用来限制自动装配的实施. 通常情况下, 自动装配的类使用@ConditionalOnClass和@ConditionalOnMissingBean注解. 这确保了自动装配类的实施, 是在当相关类被找到, 并且没有声明你自己的@Configuration的时候.

# 自动装配出现的背景
想象你在一个超级大公司里, 这个公司里有很多使用纯Spring Framework的项目, 大家的配置文件中有80%都是一样的. 这时, 你想到了是不是可以提取出一个共同的ApplicationContextConfiguration, 因为大家的有很多Bean是一样的.
```java
@Configuration
public class SharedContextConfiguration { // (1)

    @Bean
    public Driver neo4jDriver(Neo4jProperties properties, ...) {
        return GraphDatabase.driver(...);
    }

}
```

你可以将此SharedConfiguration项目, 打包发布公司maven仓库的一个依赖. 其他项目可以在自己的@Configuration类上再加上一个注解@Import(...Configuration.class)
```java
@Configuration
@Import(SharedConfiguration.class) // (1)
public class OtherProjectContextConfiguration {

   // you would specify your project-specific beans here, as normal.
}
```

以上方式看起来很好但是却存在一些问题:
如果SharedConfiguration中配置了某些Bean, 比如Neo4jBean, 在其他项目中是不需要的, 那么我是否可以排除这些Bean呢?

这就引出了@Conditional注解.

# 自动装配核心注解@Conditional
@Conditional"系列注解"可以用在@Bean Method, @Components或者其他@Configuration注解上. 它们的作用范围都是`@Target({ElementType.TYPE,ElementType.METHOD})`, 比如@ConditionalOnClass, @ConditionalOnBean, @ConditionalOnMissBean等.
```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {

	/**
	 * The classes that must be present. Since this annotation is parsed by loading class
	 * bytecode, it is safe to specify classes here that may ultimately not be on the
	 * classpath, only if this annotation is directly on the affected component and
	 * <b>not</b> if this annotation is used as a composed, meta-annotation. In order to
	 * use this annotation as a meta-annotation, only use the {@link #name} attribute.
	 * @return the classes that must be present
	 */
	Class<?>[] value() default {};

	/**
	 * The classes names that must be present.
	 * @return the class names that must be present.
	 */
	String[] name() default {};

}
```
这些注解需要@Conditional的加持(注解不支持继承, 只能通过组合的方式), @Conditional有一个"Condition"类的参数, 这个类中必须有一个方法叫matches(继承父类SpringBootCondition得此方法), 并返回一个Boolean值
 - True: (Further Evaluate/Register) Create that @Bean, @Component or @Configuration
 - False: (Stop Evaluating/Registering) Don’t create that @Bean, @Component or @Configuration
关系如下:
{% plantuml %}
    FilteringSpringBootCondition <|-- OnClassCondition
    SpringBootCondition <|-- FilteringSpringBootCondition
    Condition <|.. SpringBootCondition
    interface Condition {
        boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
    }
{% endplantuml %}

附上Condition接口源码:
```java

/**
 * A single {@code condition} that must be {@linkplain #matches matched} in order
 * for a component to be registered.
 *
 * <p>Conditions are checked immediately before the bean-definition is due to be
 * registered and are free to veto registration based on any criteria that can
 * be determined at that point.
 *
 * <p>Conditions must follow the same restrictions as {@link BeanFactoryPostProcessor}
 * and take care to never interact with bean instances. For more fine-grained control
 * of conditions that interact with {@code @Configuration} beans consider implementing
 * the {@link ConfigurationCondition} interface.
 *
 * @author Phillip Webb
 * @since 4.0
 * @see ConfigurationCondition
 * @see Conditional
 * @see ConditionContext
 */
@FunctionalInterface
public interface Condition {

	/**
	 * Determine if the condition matches.
	 * @param context the condition context
	 * @param metadata the metadata of the {@link org.springframework.core.type.AnnotationMetadata class}
	 * or {@link org.springframework.core.type.MethodMetadata method} being checked
	 * @return {@code true} if the condition matches and the component can be registered,
	 * or {@code false} to veto the annotated component's registration
	 */
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

总结一下其中的逻辑:
{% blockquote %}
In short: Even though an ApplicationContextConfiguration comes with certain @Bean definitions, you as the end-user can still somewhat influence if a bean gets created or not.
{% endblockquote %}
即使每个ApplicationContextConfiguration都是来自某个@Bean的定义, 但是你最为最终使用者依然拥有对Bean的创建与否的决定权.


## 使用@Conditional再次改造
制造出新的ApplicationContextConfiguration
```java
@Configuration
public class SharedConfiguration {
    @Bean
    @Conditional(IsRequiredNeo4jDatabaseCondition.class)
    public Driver neo4jDriver(Neo4jProperties properties, ...) {
        return GraphDatabase.driver(...);
    }
}
```
实现Condition接口:
```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class IsRequiredNeo4jDatabaseCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata){
        return neo4jDriverOnClassPath() && databaseUrlSet(context); // [1]
    }
    private boolean databaseUrlSet(ConditioContext context) {
        return context.getEnvironment().containsPreperty("spring.data.neo4j.uri");
    }
    private boolean neo4jDriverOnClassPath() {
        try {
            Class.forName("org.neo4j.driver.Driver");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }
}
```
[1]这里构造了两个条件来实施判断, 一个是检测properties中是否拥有某个配置项, 一个是检测是否引入了neo4jDriver对应的依赖(通过在classpath找寻对应的类是否存在即可).
> 注意: 真是情况下的Neo4j配置与以上做法略有不同.

那么以上两种match方式对应着两个最重要的Condition:
 - create @Beans depending on specific **available properties**.
 - create @Beans depending on specific **libraries on your classpath**.

至此, 我们会有这样的疑问,
 - Conditionals that create a DataSource for you, because you have set specific properties (think: spring.data.neo4j.uri)? 
 - Or @Conditionals that boot up an embedded Tomcat server for you because you have the Tomcat libraries on your classpath?
The Answer is YES, that (and not much more) is exactly what Spring Boot is. 

# Spring Boot Autoconfigurations: Three Internal Core Features
## 定位PropertySources
Spring Boot获取配置参数有一个顺序, 这个顺序是特殊的:
{% blockquote Spring Boot docs%}
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
{% endblockquote %}
以上这些就是Spring Boot启动时, 默认读取Property的位置和顺序.

其中, 注解`@PropertySource`需要注意, 通过注解`@PropertySource`来定位配置文件`.properties`
```java
@PropertySource(value = "classpath:application.properties", ignoreResourceNotFound = true)
```
当你运行SpringBoot启动类的main方法, SpringBoot将会自动按顺序读取这14个PropertySource(低版本的可能是17个, 当前版本是2.5.6), 为的是将这些配置注入到项目中.

> 提示: Spring Boot将配置的值注入bean的属性有两种办法
> 1. @Value
> 2. @ConfigurationProperties


## Read-in META-INF/spring.factories
每个SpringBoot项目都有一个共同的依赖, 就是`org.springframework.boot:spring-boot-autoconfigure`, 这个jar包包含了所有自动配置的magic. 其核心就是`META-INF/spring.factories`文件.
```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
// ...
// 100+ more lines
```
截取其中一段如上所示, EnableAutoConfiguration的值, 有100多个. 这些类都是普通的@Configurations并且有些有@Conditionals, Spring Boot读取这些类并对其Condition进行评估执行.

## Enhanced Conditional Support
@Conditional是一个较底层的注解, 基于此注解, 组合出了很多"增强型"的注解, 比如:
 - @ConditionalOnClass(DataSource.class). The condition is true if the DataSource class is on the classpath.
 - @ConditionalOnMissingClass(DataSource.class). The condition is true if the DataSource class is not on the classpath.
 - @ConditionalOnMissingBean(DataSource.class). The condition is true if the user did not specify a DataSource @Bean in any @Configuration.
 - @ConditionalOnWebApplication. The condition is true if the application is a web application.
 - @ConditionalOnProperty("my.property"). The condition is true if my.property is set.
 - @ConditionalOnResource("classpath:my.properties"). The condition is true if my.properties exists.
 - // e.g.
大多数情况下, 都是用增强型注解更为方便.

@Conditional增强型的注解大概分为以下几种:
 - Class Conditions
 - Bean Conditions
 - Property Conditions
 - Resource Conditions
 - Web Application Conditions
 - SpEL Expression Conditions

### Class Conditions
@ConditionalOnClass或者@ConditionalOnMissingClass, 通过判断指定的class在或者不在classpath中来, 判断Bean的创建与否. 
> **The fact is that annotation metadata is parsed by using ASM.**
Conditional注解一般有两个参数, `value`和`name`, 你可以通过`value`传真实的class参数, 即使这个类并没有出现在application classpath中.

### Bean Conditions
Conditional @Conditional that only matches when no beans meeting the specified
requirements are already contained in the `BeanFactory`.

When placed on a `@Bean` method, the bean class defaults to the return type of
the factory method:
```java
@Configuration
public class MyAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    // @ConditionalOnMissingBean没有传参数, 默认是此method的返回值类型, 同: @ConditionalOnMissingBean(MyService.class)
    public MyService myService() {
        ...
    }
}
```
In the sample above the condition will match if no bean of type {@code MyService} is
already contained in the {@link BeanFactory}.

#### Property Conditions...等不再讨论

