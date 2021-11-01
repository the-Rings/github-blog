---
title: java-aop
date: 2021-10-31 19:07:03
tags:
- AOP
- Spring
---

软件开发一直追求更加高效, 更加已维护甚至更易扩展的方式.

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
 明天总结动态代理, 需要结合第173页


