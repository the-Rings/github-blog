---
title: Spring Boot自动装配原理
date: 2021-11-11 22:02:18
categories: 
- Spring
---

在Spring Boot项目流行之前，Spring团队有一款代码生成器就是Spring Roo项目，几分钟就可以构建一个应用，但是现实情况是，大家对此不买账，后来这个项目就永远停留在了2017年。虽然Spring Roo能够帮助生成各种代码和配置，但是总体的代码数量并没有减少。Spring团队的Josh Long一语道破：如果一个东西可以生成出来，那为什么还要生成它呢？

Spring Boot在SpringFramework的基础上引入了4个特性，来提高开发效率
- starter dependency
- auto configuration
- actuator
- CLI
本次我只谈谈前两项。

starter dependency(起步依赖)这个名字取得很差，总是让我迷惑，其实应该叫做“依赖分组”。他的目的是要解决依赖管理问题，需要引入什么依赖，版本是什么，互相之间不能有冲突。要解决这个问题根本无需写额外的代码，使用maven就能天然解决，只需要创建一个空项目，然后把同一组依赖(加上版本)放到pom.xml文件中，然后主项目依赖这个空项目即可，等我读懂这些技术以后，我十分失望，因为这是多么的弱智。

auto configuration(自动配置)，它的目的是省去自己动手配置的一模一样的模板配置。框架猜测你会这么配置，如果配置不是你想要的，你就再手动修改就好了。要想我们的配置要在Spring中生效，配置文件的内容要加载到一个对象中，对应一个@Configuration修饰的类。在Spring项目的根路径下，会被自动扫描到，就会加载到容器中。但是现在，它需要在某些条件下才能向Spring容器注入这个Bean，所以这就是构成了自定义配置项的基础。

我们以JdbcTemplateAutoConfiguration为例：
```Java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })
@ConditionOnSingleCandidate(DataSource.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@EnableConfigurationProperties(JdbcProperties.class)
@Import({ JdbcTemplateConfiguration.class, NamedParameterJdbcTemplateConfiguration.class })
public class JdbcTemplateAutoConfiguration {}
```
这个配置类生效的条件是存在DataSource和JdbcTemplate类，且上下文中只能有一个DataSource。此外，这个自动配置需要在DataSourceAutoConfiguration之后再配置（可以用@AutoConfigure、@AutoConfigureAfter和@AutoConfigureOrder来控制配置的顺序）。这个配置类还会同时导入JdbcTemplateConfiguration，NamedParameterJdbcTemplateConfiguration里的配置。

更直白地说，比如，`@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })`的语义其实是，`IF EXISTS DataSource.class&JdbcTemplate.class DO...`

还有一个很重要的问题，普通配置类需要被扫描到才能生效，可是自动配置类并不在我们项目的扫描路径中，那么我们是如何将它们加载到容器中的呢？

秘密在于`@EnableAutoConfiguration`上的`@Import(AutoConfigurationImportSelector.class)`，其中的AutoConfigurationImportSelector类是ImportSelector的实现，这个接口作用就是根据特定条件决定可以导入哪些配置类。接口中的selectImports()方法返回的就是可以导入的配置的类名(String数组)。

AutoConfigurationImportSelector通过SpringFactoriesLoader来加载`/META-INF/spring.factories`里边的配置列表，所用的key是`org.springframework.boot.autoconfigure.EnableAutoConfiguration`，value是以都好分隔的自动配置类全限定的类名（包含了完整的包名与类名清单）。所以，只要在我们类上增加`@SpringBootApplication`或者`@EnableAutoConfiguration`就会自动加载所有的自动配置类。
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

# 自动装配核心注解@Conditional
Spring Framework’s @Conditional annotation is the core of Autoconfigurations.
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


## AutoConfiguration测试
```java
public class MyService {

  private String name;

  public MyService(String name) {
    this.name = name;
  }

  public MyService() {}

  public String getName() {
    return this.name;
  }
}
```

```java
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
public class MyServiceAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean // equivalent to @ConditionalOnMissingBean(MyService.class)
  public MyService myService() {
    // When MyService(bean type) doesn't exist, this bean will create.
    return new MyService("test123");
  }
}
```

测试AutoConfiguration需要借助ApplicationContextRunner, ApplicationContextRunner is usually defined as field of the "test class" to gather the base, common configuration.
```java
import com.example.multimodule.service.auto.MyService;
import com.example.multimodule.service.auto.MyServiceAutoConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
public class TestAutoConfiguration {
  // load MyServiceAutoConfiguration.class
  private final ApplicationContextRunner contextRunner =
      new ApplicationContextRunner()
          .withConfiguration(AutoConfigurations.of(MyServiceAutoConfiguration.class));

  @Configuration(proxyBeanMethods = false)
  static class UserConfiguration {
    @Bean
    MyService myCustomService() {
      // This bean is always created.
      return new MyService("mine");
    }
  }

  @Test
  void conditionEffectedTest() {
    // @ConditionalOnMissingBean effect
    this.contextRunner.run(
        (context) -> {
          assertThat(context).getBean("myService");
          assertThat(context).hasSingleBean(MyService.class);
          assertThat(context.getBean(MyService.class).getName()).isEqualTo("test123");
        });
  }

  @Test
  void conditionNotEffectedTest() {
    // @ConditionalOnMissingBean effect
    this.contextRunner
        .withUserConfiguration(UserConfiguration.class) // add UserConfiguration into context.
        .run(
            (context -> {
              assertThat(context).hasSingleBean(MyService.class);
              assertThat(context)
                  .getBean("myCustomService")
                  .isSameAs(context.getBean(MyService.class));
              assertThat(context.getBean(MyService.class).getName()).isEqualTo("mine");
            }));
  }
}
```
从以上例子中可以看出, 当仅仅ApplicationContext仅仅加载MyServiceAutoConfiguration类时, 注入的Bean是`myService`, 当MyServiceAutoConfiguration和UserConfiguration都加载是注入的Bean是`myCustomService`，因为UserConfiguration加载时，注入了MyService类型的Bean，当从中体现出了@ConditionalOnMissingBean的作用（`IF NOT EXISTS MyService.class Bean DO ...`）

更多关于Autoconfiguration相关Conditional注解的测试可以查看: https://www.baeldung.com/spring-boot-context-runner
