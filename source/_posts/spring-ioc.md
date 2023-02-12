---
title: Spring IoC基础
date: 2021-09-28 15:08:10
categories: 
- Spring
- IoC
---

为了方便, 通篇的例子都采用这个模型, 订单, 订单产品, 订单地址的关系, 订单需要依赖订单地址和产品等信息. 一般需要在构造函数中构造Order
```Java
public class Order {
    private InterfaceItem orderItem;
    private InterfaceAddress orderAddress;
    // ...
    Order (InterfaceItem orderItem, InterfaceAddress orderAddress, ...) {
        this.orderItem = orderItem;
        this.orderAddress = orderAddress;
        // ...
    }

    public persistOrderData() {
        this.save();
    }
}
public AmazonOrderItem implements InterfaceItem {
    // ...
}
public AmazonOrderAddress implements InterfaceItem {
    // ...
}
```

## IoC是如何诞生的
IoC中文被翻译为"控制反转", 一直都让我一头雾水, 软件工程师取名总是带着一种"我提出了一个改变世界的概念"的感觉, 不实在. 老外取得名, 在翻译为中文, 更增加了神秘感.
得到一个类的实例, 只有一个办法, 就是new YourClass(). 在只有没有Spring框架的时代, 随着业务逻辑的增加, 类中的属性越来越多,当我们要得到实例时, 发现要提前准备很多很多对象, 比如: new YourClass(obj_1, obj_2, obj_3, ...), 这些obj_n都是通过new得到的.
当人们把很多项目放在一起比较发现, 这些"new操作", 其实是一种高级别的相似, 那么就可以"抽出它们像的部分", 让机器帮助我们干这些活.于是, 有人能够把我们需要的某个依赖对象"主动"送过来, 而不是我们自己去new, 所以就是"控制反转".
达到的目的就是"依赖注入", 将依赖对象注入到被注入对象中.
注入的方式有三种, 构造器注入(推荐), setter注入(推荐), Field注入(不推荐)
整个IoC的过程可以简化为：加载解析XML > 封装BeanDefination > 实例化 > 放到容器中 > 从容器中Get

## IoC Service Provider
**通常被大家称为IoC容器**. IoC Service Provider职责只有两个, 业务对象的构建和业务对象之间的依赖绑定. 也就是记录依赖关系, 据此生成业务对象.
既然是容器，那么容器使用什么数据结构存储呢？必然是Map，Map的key的类型可以是String/Class，储存的value首先得是Object，另外Spring还设计了BeanFactory和BeanDefination等类型
**Spring的IoC容器是一个IoC Service Provider, 提供了两种类型的支持: BeanFactory和ApplicationContext. 其中ApplicationContext基于BeanFactory, 提供了事件发布等功能.**
Spring提倡使用POJO, 每个业务对象看做是一个JavaBean. 只有纳入Spring管理的这些类才能看做是业务对象, 如何纳入Spring管理, 就是这些类上有`@Configuration`, `@Component`, `@Service`等注解. 要是定义了一个普通的类, 那么这并不能归IoC容器管辖.

很久以前, 我们基本上都使用XML进行依赖关系的记录, 通过XML很好的给我们展现了,依赖的树形关系, 先完成类的声明, 然后对应编写XML, 比如: 
```xml
<bean id="order" class="..Order">
    <property name="orderItem">
        <ref bean="amazonOrderItem">
    </property>
    <property name="orderAddress">
        <ref bean="amazonOrderAddress">
    </property>
</bean>
<bean id="amazonOrderItem" class="..impl.AmazonOrderItem"></bean>
<bean id="amazonOrderAddress" class="..impl.AmazonOrderAddress"></bean>
```

`org.springframework.beans.factory.BeanFactory`是Spring IoC中最重要类，注释中的第一句话：*The root interface for accessing a Spring bean container.*
BeanFactory接口，部分源码。其中注释中有句话很重要，*Bean factory implementations should support the standard bean lifecycle interfaces as far as possible*
```Java
package org.springframework.beans.factory;
/**
 * The root interface for accessing a Spring bean container.
 * This is the basic client view of a bean container;
 * 
 * ....
 * 
 * <p>Bean factory implementations should support the standard bean lifecycle interfaces
 * as far as possible. The full set of initialization methods and their standard order is:
 * <ol>
 * <li>BeanNameAware's {@code setBeanName}
 * <li>BeanClassLoaderAware's {@code setBeanClassLoader}
 * <li>BeanFactoryAware's {@code setBeanFactory}
 * <li>EnvironmentAware's {@code setEnvironment}
 * <li>EmbeddedValueResolverAware's {@code setEmbeddedValueResolver}
 * <li>ResourceLoaderAware's {@code setResourceLoader}
 * (only applicable when running in an application context)
 * <li>ApplicationEventPublisherAware's {@code setApplicationEventPublisher}
 * (only applicable when running in an application context)
 * <li>MessageSourceAware's {@code setMessageSource}
 * (only applicable when running in an application context)
 * <li>ApplicationContextAware's {@code setApplicationContext}
 * (only applicable when running in an application context)
 * <li>ServletContextAware's {@code setServletContext}
 * (only applicable when running in a web application context)
 * <li>{@code postProcessBeforeInitialization} methods of BeanPostProcessors
 * <li>InitializingBean's {@code afterPropertiesSet}
 * <li>a custom {@code init-method} definition
 * <li>{@code postProcessAfterInitialization} methods of BeanPostProcessors
 * </ol>
 *
 * <p>On shutdown of a bean factory, the following lifecycle methods apply:
 * <ol>
 * <li>{@code postProcessBeforeDestruction} methods of DestructionAwareBeanPostProcessors
 * <li>DisposableBean's {@code destroy}
 * <li>a custom {@code destroy-method} definition
 * </ol>
 * ....
 */
public interface BeanFactory {
	String FACTORY_BEAN_PREFIX = "&";

	/**
	 * Return an instance, which may be shared or independent, of the specified bean.
	 */
	Object getBean(String name) throws BeansException;
	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);
	boolean containsBean(String name);
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;
	String[] getAliases(String name);
}

```
>注：其中`prototype`, `singleton`是Spring Bean的scope的两个范围，其次还有`request`和`session`，这两个用的少
>- prototype, 容器在接到该类型对象的请求时, 会每次都重新生成一个新的对象实例给请求方
>- singleton，即单例，全局只有一个对象，这是Spring默认的scope

在拥有了BeanFactory之后, 我们将"生产图纸"交给BeanFactory, 让其为我们生产一个业务对象即可:
```Java
BeanFactory container = new XmlBeanFacotry(new ClassPathResource("YOUR_XML_PATH"));
Order order = (Order) container.getBean("order");
order.persistOrderData();
```
或者使用ApplicationContext
```Java
ApplicationContext container = new ClassPathXmlApplication("YOUR_XML_PATH");
Order order = (Order) container.getBean("order");
order.persistOrderData();
```
>注：不仅仅是XML这种格式，.json/.properties/.yml都可以当作配置文件来定义Bean，只要做好规范（接口）即可，对应`org.springframework.beans.factory.support.BeanDefinitionReader`接口，不同类型的配置文件就有了规范
XML定义的Bean最终被“翻译”为`BeanDefinition`的一个个的实例，将这些BeanDefinition进行实例化流程中，Spring加入了很多可扩展的地方，比如：PostProcessor
- BeanFactoryPostProcessor，增强（修改）BeanDefination信息，在xml生成BeanDefinition阶段起作用
- BeanPostProcessor，增强（修改）Bean信息，在Bean实例化阶段起作用
- 等等

接下来，明确两个概念，一个是Instantiate（实例化），一个是Initialize（初始化），对应Python中的`__new__()`和`__init__()`
- Instantiate，是调用构造方法，在堆中开辟一个内存空间，属性赋默认值。
- Initialize，是调用init-method，为对象赋值，比如在XML中定义Bean的时候，指定init-method

### BeanFactory和FactoryBean的区别
实现了FactoryBean接口的类，将作为一个Bean放入SpringIOC容器中，然后在某个地方调用FactoryBean.getObject()方法来进行对象的实例化。FactoryBean是Spring框架的一个扩展，方便用户自己灵活进行Bean的创建

### 工厂方法
这里额外介绍一下工厂模式. 在强调面向接口编程的同时, 有一点需要注意: **虽然对象可以通过声明接口来避免对特定接口实现类的过度耦合**, 但总归需要一种方式将声明依赖接口的对象与接口实现类关联起来,. 只依赖一个不做任何事情的接口是没有任何用处的.
```java
public class Foo {
    private BarInterface barInterface;
    public Foo() {
        // 我们应该避免这样做
        // instance = new BarInterfaceImpl();
    }
}
```
如果以上类Foo是由我们自定义的, 我们可以在其上`@Component`或者`@Service`纳入Spring IoC容器管理, 让容器帮我们解除接口和实现类之间的耦合性. 但是, 如果BarInterface来自于第三方库, 接口与实现类的耦合性需要其他方式来避免. 这是我们可以写一个工厂方法(Factory Method), 提供一个工厂类来实例化具体接口实现类. Foo类只需要依赖于工厂类, 当实现类有变更的时候, 只是变更工厂类, Foo类代码不需要做出任何变动.
```java
public class Foo {
    private BarInterface barInterface;
    public Foo() {
        // 静态工厂
        // barInterface = BarInterfaceFactory.getInterface()
        // 抽象工厂
        // barInterface = new BarInterfaceFactory().getInstance();
    }
}
// 以上操作实际上是在BarInterface与实现类之间加了一层而已. 美其名曰: "解除耦合".
```
在XML中, 我们可以这样声明, 将这个工厂方法交给Spring容器管理
```xml
<bean id="foo" class="...Foo">
    <property name="barInterface">
        <ref bean="bar"/>
    </property>
</bean>
<bean id="bar" class="...StaticBarInterfaceFactory" factory-method="getInstance"/>
```
factory-method指定工厂方法名, 然后容器调用静态方法getInstance. 也就是说, 为对象foo注入的bar对象实际是BarInterfaceImpl的实例.

## 容器启动过程分析
Spring IoC容器实现其功能, 基本上可以按照类似的流程分为两个阶段: 容器启动阶段，Bean实例化和初始化阶段
- 容器启动阶段: 加载环境配置 > 分析配置信息 > 装备到BeanDefinition > PostProcessor
- Bean实例化和初始化阶段: 实例化 > 填充属性 > 初始化
用一个流程图来具体展示：
[Spring IOC执行流程](https://www.processon.com/diagraming/63dcfa02392a4b25fec64888)

### 生产产品
容器启动之后, 并不会马上进行实例化Bean. 容器现在拥有对象的BeanDefinition来存储实例化必要信息. 当通过BeanFactory.getBean()方法来请求某个对象实例时, 才可能触发Bean实例化阶段的活动. 

#### Bean的实例化与BeanWrapper
容器在内部实现的时候, 采用"策略模式(Strategy Pattern)"来决定使用何种方式初始化bean实例, 通常是通过反射或者CGLIB动态字节码来生成bean实例, 或者其子类. 默认情况下, 容器采用的是CglibSubclassingInstantiationStrategy.

按照正常的逻辑, 容器只需要根据BeanDefintion取得实例化信息, 结合CglibInstantiationStrategy返回对象实例. 但是, 这里的做法不是直接返回构造完成的实例, 而是以BeanWrapper对构造完成的对象实例进行包裹, 返回相应的BeanWrapper实例. 
```Java
Object order = Class.forName("package.name.Order").newInstance();
Object orderItem = Class.forName("package.name.AmazonOrderItem").newInstance();
Object orderAddress = Class.forName("package.name.AmazonOrderAddress").newInstace();

BeanWrapper newOrder = new BeanWrapperImpl(order);
newOrder.setPropertyValue("newOrderItem", orderItem);
newOrder.setPropertyValue("newOrderAddress", orderAddress);

assertTrue(newOrder.getWrappedInstance() instanceof Order);
assertSame(order, newOrder.getWrappedInstance());
assertSame(orderItem, newOrder.getPropertyValue("newOrderItem"));
assertSame(orderAddress, newOrder.getPropertyValue("newOrderAddress"));
```
针对以上示例, 截一段源码, BeanWrapperImpl是实现类, 这里调用的都是父类AbstractNestablePropertyAccessor的方法.
```java
package org.springframework.beans;
public class BeanWrapperImpl extends AbstractNestablePropertyAccessor implements BeanWrapper {
    // ...
    public BeanWrapperImpl(Object object) {
		super(object);
	}
    // ...
}
// ...
public abstract class AbstractNestablePropertyAccessor extends AbstractPropertyAccessor {
    @Nullable
	Object wrappedObject;
    // ...
    public final Object getWrappedInstance() {
		Assert.state(this.wrappedObject != null, "No wrapped object");
		return this.wrappedObject;
	}
    protected AbstractNestablePropertyAccessor(Object object) {
		registerDefaultEditors();
		setWrappedInstance(object);
	}
    public void setWrappedInstance(Object object) {
		setWrappedInstance(object, "", null);
	}
    public void setWrappedInstance(Object object, @Nullable String nestedPath, @Nullable Object rootObject) {
		this.wrappedObject = ObjectUtils.unwrapOptional(object);
		// ...
	}
    @Override
	public void setPropertyValue(String propertyName, @Nullable Object value) throws BeansException {
		// ... 这里用了很多反射的方法.
	}
    // ...
}
```
如果粗略的看没有什么复杂的逻辑, 但是里边的细节很多, 要是让我去写Spring的架构, 最终要的是接口的设计, 层次的划分, 以及需求的抽象. 

#### 给实例注入依赖对象
上一步我们已经将, 属性set到了bean实例上, 但是没有赋值. Spring容器会检查当前实例实现了哪个Aware命名结尾的接口, 然后将对应Aware接口中对顶的依赖注入进去.
比如, BeanNameAware, 如果Spring容器检测到当前对象实例实现了该接口, 会将该对象实例的bean定义对应的beanName设置到当前实例. BeanClassLoaderAware, 会将当前bean的ClassLoader注入当前对象实例.

#### 对实例进行前(后)置处理
现在依赖注入已经完成, 那么接下来可以对实例进行后置处理(hook), Spring提供了侵入的办法, 就是BeanPostProcessor. 
BeanPostProcessor与BeanFactoryPostProcessor容器混淆. 只要记住BeanPostProcessor存在于对象实例化阶段, 而BeanFactoryPostProcessor存在于容器启动阶段. BeanPostProcessor会处理容器内所有符合条件的实例化后的对象实例. 该接口很简单, 从方法的命名上就可以看出其意义, 一个可以唤作前置处理, 一个唤作后置处理:
```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;
import org.springframework.lang.Nullable;

/**
 * Factory hook that allows for custom modification of new bean instances,
 * e.g. checking for marker interfaces or wrapping them with proxies.
 *
 * <p>ApplicationContexts can autodetect BeanPostProcessor beans in their
 * bean definitions and apply them to any beans subsequently created.
 * Plain bean factories allow for programmatic registration of post-processors,
 * applying to all beans created through this factory.
 *
 * <p>Typically, post-processors that populate beans via marker interfaces
 * or the like will implement {@link #postProcessBeforeInitialization},
 * while post-processors that wrap beans with proxies will normally
 * implement {@link #postProcessAfterInitialization}.
 *
 * @author Juergen Hoeller
 * @since 10.10.2003
 * @see InstantiationAwareBeanPostProcessor
 * @see DestructionAwareBeanPostProcessor
 * @see ConfigurableBeanFactory#addBeanPostProcessor
 * @see BeanFactoryPostProcessor
 */
public interface BeanPostProcessor {

	/**
	 * Apply this BeanPostProcessor to the given new bean instance <i>before</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 */
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	/**
	 * Apply this BeanPostProcessor to the given new bean instance <i>after</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>In case of a FactoryBean, this callback will be invoked for both the FactoryBean
	 * instance and the objects created by the FactoryBean (as of Spring 2.0). The
	 * post-processor can decide whether to apply to either the FactoryBean or created
	 * objects or both through corresponding {@code bean instanceof FactoryBean} checks.
	 * <p>This callback will also be invoked after a short-circuiting triggered by a
	 * {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation} method,
	 * in contrast to all other BeanPostProcessor callbacks.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.FactoryBean
	 */
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```
我们也可以自定义BeanPostProcessor, 定义一个类implements BeanPostProcessor, 来把自己的逻辑侵入到bean实例化的过程当中去.

#### InitialzingBean和init-method
```java
package org.springframework.beans.factory;

/**
 * Interface to be implemented by beans that need to react once all their properties
 * have been set by a {@link BeanFactory}: e.g. to perform custom initialization,
 * or merely to check that all mandatory properties have been set.
 *
 * <p>An alternative to implementing {@code InitializingBean} is specifying a custom
 * init method, for example in an XML bean definition. For a list of all bean
 * lifecycle methods, see the {@link BeanFactory BeanFactory javadocs}.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see DisposableBean
 * @see org.springframework.beans.factory.config.BeanDefinition#getPropertyValues()
 * @see org.springframework.beans.factory.support.AbstractBeanDefinition#getInitMethodName()
 */
public interface InitializingBean {

	/**
	 * Invoked by the containing {@code BeanFactory} after it has set all bean properties
	 * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
	 * <p>This method allows the bean instance to perform validation of its overall
	 * configuration and final initialization when all bean properties have been set.
	 * @throws Exception in the event of misconfiguration (such as failure to set an
	 * essential property) or if initialization fails for any other reason
	 */
	void afterPropertiesSet() throws Exception;

}
```
InitializingBean的作用在于, 对象实例化调用过"BeanPostProcessor的前置处理"方法之后, 会接着检测对象是否实现了InitializingBean接口, 如果是, 就会调用afterPropertiesSet()方法进一步调整对象实例的状态. 
但是, 以上操作显得Spring容器比较具有侵入性, 那么Spring还提供了另一种方式, 那就是在XML的`<bean>`标签中配置init-method, 可以认为在InitializingBean和init-method中任选其一帮助你完成类似的初始化工作.
> 到这里我不仅感叹, 这篇博客是我对《Spring揭秘》的读书和实践的笔记, 可能大部分书籍的文字都来源于对Spring源码中注释的解读.

#### DisposableBean与destroy-method
同样地, 当所有的一切, 该设置的设置, 该注入的注入, 该调用的调用完成之后, 容器会检查singleton类型的bean实例, 是否实现了DisposableBean接口. 或者对应的bean在`<bean>`里定义了destory-method. 是的话, 就会为该实例注册一个用于对象销毁的回调(Callback), 以便这些singleton类型的对象实例销毁之前, 执行销毁逻辑.
> 容器不会去管理, scope为prototype类型的bean实例.


## 使用注解代替XML
在XML配置成功的基础上, 引入了注解来减少冗余操作.
`@Autowired`四基于注解的依赖注入的核心注解. 它们都是触发容器对相应对象给与依赖注入的标志. `@Autowired`是按照类型匹配进行依赖注入的. 现在, 容器的配置文件就只剩下一个个孤零零的bean定义了.

有了注解必须得有Annotation Processor, 要不然注解和注释没什么区别, Spring提供了AutowiredAnnotationBeanPostProcessor来得到这一目的. 通过反射检查每个bean定义对应的类上的各种可能位置上的`@Autowired`. 存在, 就从当前容器管理的对象中获取符合条件的对象, 设置给`@Autowired`锁标注的属性或方法. 伪代码如下: 
```java
Object[] beans = ...;
for (Objec bean: beans) {
    if(autowiredExistsOnField(bean)){
        Field f = getQulifiedField(bean);
        setAccessiableIfNeccessary(f);
        f.set(getBeanByTypeFromContainer());
    }
    if(autowiredExistsOnMethod(bean)) {
        // ...
    }
    // ...
}
```

如果当前的`@Autowired`标注的依赖在容器中找到了两个以上的实例的话, 就需要@Qualifier的配合, 出入自定义的name(String)条件作出进一步限定. `@Qualifier`实际上是byName自动绑定的注解版.

#### classpath-scanning
到目前为止, 我们已经通过注解将依赖关系xml定义转移到了源码中. 为了"一套代码, 一处定义"的理念, 要将革命进行彻底. classpath-scanning的诞生!
使用相应的注解(`@Component`, `@Service`, `@Configuration`)进行标注之后, classpath-scanning功能从某一顶层包(base package)开始扫描, 当扫描到相应的注解之后, 就会提取该类的信息, 构建对应的BeanDefinition, 然后把构建完成的BeanDefinition注册到容器. 
classpath-scanning由`<context:component-scan>`决定. `<context:component-scan>`默认扫描的注解时`@Component`. 其中, 在@Component语义的基础上细化后又有了`@Repository`, `@Service`/`@Controller`, 他们同样都会被扫描. `@Component`的语义更宽泛, 而`@Service`以及`@Repository`等更具体. 另外, 对于服务层类定义来说, 使用`@Service`标注它, 比`@Component`更加确切.


学习Spring框架, 是不是要抓住Spring中几个大的接口来进行, 比如BeanFactory, BeanPostProcessor等, 毕竟是面向接口的编程. 

