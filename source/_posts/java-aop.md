---
title: Spring AOP原理
date: 2021-10-31 19:07:03
tags:
- AOP
- Spring
---

# AOP概念
对于系统中普遍的业务关注点, OOP可以很好地对其进行分解并使之模块化, 但是却无法更好地避免类似于系统需求的实现在系统中各处散落这样的问题. 所以, 我们要寻求一种更好的办法在OOP的基础上更上一层楼. 我们可以推翻OOP的概念提出一套全新的思路, 但是也可以在此基础上提供一种补足方案. 后来我们找到了AOP.
 - 静态AOP, 第一代AOP, 以AspectJ为代表. 特点是, 相应的很切关注点以Aspect形式实现之后, 会通过特定的编译器, 将实现后的Aspect编译并织入到系统的静态类中. 比如, AspectJ会使用ajc编译器将各个Aspect以Java字节码的形式编译到Java类中, Java虚拟机可以像通常一样加载Java类运行.
 - 动态AOP, 第二代AOP, 通过Java语言的动态特性来实现Aspect织入到系统的过程, 比如Spring AOP.

 ### Joinpoint
 首先我们需要知道在哪些执行点上进行织入操作, 这些将要在其之上进行织入操作的系统执行点, 称之为Joinpoint. 比如, 方法调用时, 执行时, 字段设置时, 异常处理时, 类初始化时等等.
 ### Pointcut
 接下来需要知道在什么地方(Joinpoint)织入横切逻辑. 比如, 指定Joinpoint所在方法的名称, 或者利用正则表达式表述出所有符合条件的多组Joinpoint.
 ### Advice
 逻辑载体, 即织入到Joinpoint的横切逻辑. 它分为几种形式:
  - Before Advice, 在Joinpoint位置之前执行的Advice类型
  - After Advice, 在Joinpoint位置之后, 包括After returning Advice, Afterthrowing Advice等
  - Around Advice, 这里应该叫做拦截器比较好, Interceptor. 它也可以完成Before Advice和After Advice的功能.
  - Introduction
 ### Aspect
 Aspect是对以上三者进行封装的AOP概念.

# Spring AOP
AOP是一种理论, Spring AOP是针对Spring框架落地的一种AOP实现. Spring的设计哲学是简单而强大, 用20%的AOP的支持来满足80%的场景.

 ## 动态代理与CGLIB
 动态代理, 这里不再赘述, 在博客里有专门剖析.
 结合Spring AOP来说, 动态代理实现InvocationHandler的类是我们实现横切逻辑的地方, 它是横切逻辑的载体, 作用和Advice是一样的. 这就理解了Advice是什么了.
 动态代理虽好, 但是不能满足所有需求. 因为动态代理机制只能对实现了相应Interface的类使用, 如果某个类没有实现任何Interface, 就无法使用动态代理机制为其生成相应的动态代理对象.

 使用动态字节码生成技术扩展对象行为的原理是, 我们可以对目标对象进行继承扩展, 为其生成相应的子类, 而子类可以通过重写来扩展父类的行为, 只要将横切逻辑放的实现放到子类中, 然后让系统使用扩展后的目标对象的子类, 就可以达到相同的目的了. CGLIB可以对实现了某种接口的类, 或者没有实现任何接口的类都可以进行扩展.
 通常我们会直接使用`net.sf.cglib.proxy.MethodInterceptor`接口(扩展了net.sf.cglib.proxy.Callback接口): 
 ```java
 class Requestable {
     public void request(){
         System.out.println("OK");
     }
 }
 public class RequestCtrlInterceptor implements MethodInterceptor {
     private static final Log logger = LogFactory.getLog(RequestCtrlInterceptor.class);
     public Object intercept(Object object, Method method, Object[] args, MethodProxy proxy) throws Throwable {
         if(method.getName().equals("request")) {
            TimeOfDay startTime = new TiemOfDay(0, 0, 0);
            TimeOfDay endTime = new TiemOfDay(5, 59, 59);
            TiemOfDay currentTime = new TimeDay();
            if(currentTime.isAfter(startTime) && currentTime.isBefore(endTime)) {
                logger.warn("Service is not available now.");
                return null;
            }
            return proxy.invokeSuper(object, args);
         }
         return null;
     }
 }
 ```
 这样, RequestCtrlInterceptor就实现了对request()方法请求进行访问控制的逻辑. 现在我们要通过CGLIB的Enhance类为目标动态生成一个类, 并将RequestCtrlInterceptor中的横切逻辑附加到该子类中, 代码如下:
 ```Java
 Enhancer enhancer = new Enhancer();
 enhancer.setSuperclass(Requestable.class);
 enhancer.setCallback(new RequestCtrlInterceptor());
 
 Requestable proxy = (Requestable) enhancer.create();
 proxy.request();
 ```
 这里是使用Enhancer来演示在对象中加入横切逻辑, 在后续的内容中我们将会使用经过封装的更高级的Spring AOP工具, Advisor

# Spring AOP
对于Joinpoint, Spring AOP仅支持方法级别的Joinpoint, 更确切的说, 仅支持方法执行(Method Execution)类型的Joinpoint, 原因有以下几点:
 - Spring AOP设计理念是简单而强大
 - 对于类属性Field级别的Joinpoint, 完全可以使用getter/setter方法的拦截来达到同样的目的
 - 如果要求十分特殊, 借助AspectJ即可(当然即使是AspectJ这样支持很多Joinpoint类型的AOP实现产品, 也无法保证能捕捉到程序流程中的任何一个点)

对于Pointcut, Spring AOP提供了org.springframework.aop.Pointcut接口
```java
public interface Pointcut {
    ClassFilter getClassFilter;
    MethodMatcher getMethodMatcher;
    Pointcut TRUE = TruePointcut.INSTANCE;
}
```
见名知意, 不是重点, 不再赘述

对于Advice, Advice是实现了将织入到Pointcut规定的Joinpoint处的横切逻辑. Advice的几种类型, 重点介绍Around Advice.
Srping中没有直接定义Around Advice的实现接口, 而是采用AOP Alliance的标准接口, 即: `org.aopalliance.intercept.MethodInterceptor`, 该接口定义如下: 
```java
package org.aopalliance.intercept;

/**
 * Intercepts calls on an interface on its way to the target. These
 * are nested "on top" of the target.
 * ...
 */
@FunctionalInterface
public interface MethodInterceptor extends Interceptor {

	/**
	 * Implement this method to perform extra treatments before and
	 * after the invocation. Polite implementations would certainly
	 * like to invoke {@link Joinpoint#proceed()}.
	 * @param invocation the method invocation joinpoint
	 * @return the result of the call to {@link Joinpoint#proceed()};
	 * might be intercepted by the interceptor
	 * @throws Throwable if the interceptors or the target object
	 * throws an exception
	 */
	Object invoke(MethodInvocation invocation) throws Throwable;
}
```
其他的Advice能做到的事情, 它都可以做, 或者说Around Advice可以应用的场景很多, 例如: 系统安全检查/系统各处性能检测/日志记录/系统附件行为的添加等.

接下来是一个场景: 销售系统, 在商场优惠期间, 所有的商品一律8折, 那么我们在系统中所有取得商品价格的地方插入如下横切逻辑.
```java
public class DiscountMethodInterceptor implements MethodInterceptor {
    private static final Integer DEFAULT_DISCOUNT_RATIO = 80;

    public Object invoke (MethodInvocation invocation) throws Throwable {
        Object returnValue = invocation.proceed();
        return ((Integer) returnValue) * DEFAULT_DISCOUNT_RATIO / 100;
    }
}
```
> 通过MethodInvocation的invoke方法的MethodInvocation参数, 我们可以控制相应的Joinpoint的拦截行为. 通过调用MethodInvocation的proceed()方法, 可以让程序执行继续沿着调用链传播, 这是我们所希望的行为. 如果我们在哪一个MethodInterceptor中没有调用proceed(),那么程序的执行将会在MethodInterceptor处"短路".

我们使用了Spring框架, 并且这些Advice实现都是普通的POJO, 更多时候, 会直接将其集成到IoC容器中, 如下所示:
```XML
<bean id="discountInterceptor" class="...DiscountMethodInterceptor"></bean>
```

当我定义了多个MethodInterceptor时, 他们是如何执行呢?这就要用到**{% post_link java-chain-of-responsibility '责任链模式' %}**

# SpringAOP的织入
AspectJ使用ajc编译器作为它的织入器, 在SpringAOP中使用`org.springframework.aop.ProxyFactory`, 在Spring AOP中这是最基本的织入器.

总的思路是: **Spring AIO是基于代理模式的AOP实现, 织入完成后, 会返回织入横切逻辑的代理对象**. 也就是说ProxyFactory返回织入了横切逻辑的代理对象.

使用ProxyFactory需要两个基本的东西, 一个是传入进行织入的目标对象, 一个是将要应用到目标对象的Aspect(在Spring中叫做Advisor). 指定对应的Advisor, 就可以添加各种类型的Advice, 比如伪代码:
```Java
ProxyFactory weaver = new ProxyFactory(new Executable());
NameMatchMethodPointcutAdvisor advisor = new NameMathMethodPointcutAdvisor();
advisor.setMappedName("request");
advisor.setAdvice(new RequestCtrlInterceptor());
weaver.addAdvisor(advisor);

// 
weaver.setProxyTargetClass(true);

Requestable proxyObject = (Requestable) weaver.getProxy();
proxyObject.request();
System.out.println(proxyObject.getClass());

// out
// class ...Requestable$$EnhancerByCGLIB$$9e62fc83
// 从输出中可以看出ProxyFactory底层还是使用了之前的Enhancer, 也可以看出通过CGLIB得到的类的命名方式与动态代理的方式不太一样, 动态代理是"$Proxy0"
```

我们知道Spring AOP在使用代理模式实现AOP的过程中采用了动态代理和CGLIB两种机制, 分别对实现了某些接口的目标类和没有实现任何接口的目标类进行代理. 当目标类没有实现任何接口时, 默认使用动态代理, 但是这里设置了`proxyTargetClass`的属性为`true`, 强制其采用基于类的代理, 即使目标类确实是基于接口的.

## 探索ProxyFactory本质
先来看一下类之间的关系
{% plantuml %}
AdvisedSupport <.. AopProxyFactory: according to
interface AopProxyFactory {
    AopProxy createAopProxy(AdvisedSupport config)
}
AopProxyFactory <|.. DefaultAopProxyFactory

AopFactory <.. AopProxyFactory: create
interface AopFactory {
    Object getProxy()
    Object getProxy(ClassLoader classLoader)
}

AopFactory <|.. JdkDynamicProxy

JdkDynamicProxy ..|> InvocationHandler
interface InvocationHandler {}

AopFactory <|.. Cglib2AopProxy

{% endplantuml%}

其中AopProxy有Cglib2AopProxy和JdkDynamicProxy两种实现. 因为动态代理需要通过InvocationHandler提供调用拦截, 所以JdkDynamicProxy同时也要实现InvocationHandler接口.

AopProxy实例化代理对象的过程采用了抽象工厂模式. 


一个类如果不声明, 默认的构造方法, 那么他到底有没有默认的构造方法?
一个类声明了默认的构造方法, 但是没有写访问控制符, 那么这个访问控制符是什么?
一个类声明了默认的构造方法, 但是没有写访问控制符, 为什么通过Class.forName(...).getConstructor()会抛出NoSuchMethodException?




