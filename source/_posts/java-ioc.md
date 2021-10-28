---
title: IoC基础
date: 2019-01-28 15:08:10
tags: 
- java
- ioc
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
注入的方式有三种, 接口注入(废弃), 构造器注入, setter注入(推荐)
## IoC Service Provider
通常被大家称为IoC容器. IoC Service Provider职责只有两个, 业务对象的构建和业务对象之间的依赖绑定. 也就是记录依赖关系, 据此生成业务对象.

Spring的IoC容器是一个IoC Service Provider, 提供了两种类型的支持: BeanFactory和ApplicationContext. 其中ApplicationContext基于BeanFactory, 提供了事件发布等功能.

Spring提倡使用POJO, 每个业务对象看做是一个JavaBean. 只有纳入Spring管理的这些类才能看做是业务对象, 如何纳入Spring管理, 就是这些类上有@Configuration, @Component, @Service等注解. 要是定义了一个普通的类, 那么这并不能归IoC容器管辖.

很久以前, 我们基本上都使用XML进行依赖关系的记录, 通过XML很好的给我们展现了, 依赖的树形关系, 先完成类的声明, 然后对应编写XML, 比如: 
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
## BeanFactory总体逻辑
以下列举了BeanFactory接口源码(重载方法没有列出)
```java
package org.springframework.beans.factory;
/**
 * The root interface for accessing a Spring bean container.
 * This is the basic client view of a bean container;
 * 
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

/** 
  * 其中"prototype", "singleton"是bean的scope属性两种类型值
  * 拥有prototype scope的bean定义, 容器在接到该类型对象的请求时, 会每次都重新生成一个新的对象实例给请求方
  * 这有助于理解BeanFactory中两个方法的意义
  **/
```
在拥有了BeanFactory之后, 我们将"生产图纸"交给BeanFactory, 让其为我们生产一个业务对象即可:
```java
BeanFactory container = new XmlBeanFacotry(new ClassPathResource("XML_PATH"));
/**
 * 或者使用ApplicationContext
 * ApplicationContext container = new ClassPathXmlApplication("XML_PATH");
**/
Order order = (Order)container.getBean("order");
order.persistOrderData();
```

综上, IoC容器, 或者具体点BeanFactory, 完成了, 注册/绑定->生产对象, 三个步骤. 这就是IoC的所有目的了.每个业务对象作为个体, 在Spring的XML配置文件中是</bean>元素一一对应的, 只要我们了解了单个业务对象是如何配置的, 那么剩下的就是"依葫芦画瓢".
