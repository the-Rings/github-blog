---
title: 责任链模式
date: 2021-11-02 20:31:25
tags:
- java
- pattern
---

这篇博客需和另一篇将Spring AOP的相结合.

# 责任链模式
用这样一段OnJava8的原文来描述责任链模式

{% blockquote Bruce Eckel, On Java 8 %}
In recursion, one method calls itself over and over until it reaches a termination condition; with **Chain of Responsibility**, a method calls the same base-class method (with a different implementation) which calls another implementation of the base-class method, etc., until it reaches a termination condition.
{% endblockquote %}

从中体会到几点: Java中使用递归调用来执行整个链条, 链条是一个List, 这个List中保存着每个子类重写父类的同一个方法, 但是它们实现逻辑不同. 


在Spring AOP中, 对于某个Joinpoint. 要加入一个横切逻辑, 需要定义一个类实现MethodInterceptor, 重写其invoke方法, 在invoke方法体中添加横切逻辑. 其方法体中必须调用`invocation.proceed()`方法, 不然会造成"短路". 

有多个MethodInterceptor实现的时候, 要按照逐个执行它们的invoke方法(横切逻辑), 这些匹配到的interceptor, 最终集中到了`ReflectiveMethodInvocation.interceptorsAndDynamicMethodMatchers`这个List中, 将在`proceed()`方法中执行. 这里用到了递归, 整体是"责任链模式"

附上ReflectiveMethodInvocation源码来解释

```java

/**
 * Spring's implementation of the AOP Alliance
 * {@link org.aopalliance.intercept.MethodInvocation} interface,
 * implementing the extended
 * {@link org.springframework.aop.ProxyMethodInvocation} interface.
 *
 * <p>Invokes the target object using reflection. Subclasses can override the
 * {@link #invokeJoinpoint()} method to change this behavior, so this is also
 * a useful base class for more specialized MethodInvocation implementations.
 *
 * <p>It is possible to clone an invocation, to invoke {@link #proceed()}
 * repeatedly (once per clone), using the {@link #invocableClone()} method.
 * It is also possible to attach custom attributes to the invocation,
 * using the {@link #setUserAttribute} / {@link #getUserAttribute} methods.
 *
 * <p><b>NOTE:</b> This class is considered internal and should not be
 * directly accessed. The sole reason for it being public is compatibility
 * with existing framework integrations (e.g. Pitchfork). For any other
 * purposes, use the {@link ProxyMethodInvocation} interface instead.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Adrian Colyer
 * @see #invokeJoinpoint
 * @see #proceed
 * @see #invocableClone
 * @see #setUserAttribute
 * @see #getUserAttribute
 */
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {

	protected final Object proxy;

	@Nullable
	protected final Object target;

	protected final Method method;

	protected Object[] arguments = new Object[0];

	@Nullable
	private final Class<?> targetClass;

	/**
	 * Lazily initialized map of user-specific attributes for this invocation.
	 */
	@Nullable
	private Map<String, Object> userAttributes;

	/**
	 * List of MethodInterceptor and InterceptorAndDynamicMethodMatcher
	 * that need dynamic checks.
	 */
	protected final List<?> interceptorsAndDynamicMethodMatchers;

	/**
	 * Index from 0 of the current interceptor we're invoking.
	 * -1 until we invoke: then the current interceptor.
	 */
	private int currentInterceptorIndex = -1;


	/**
	 * Construct a new ReflectiveMethodInvocation with the given arguments.
	 * @param proxy the proxy object that the invocation was made on
	 * @param target the target object to invoke
	 * @param method the method to invoke
	 * @param arguments the arguments to invoke the method with
	 * @param targetClass the target class, for MethodMatcher invocations
	 * @param interceptorsAndDynamicMethodMatchers interceptors that should be applied,
	 * along with any InterceptorAndDynamicMethodMatchers that need evaluation at runtime.
	 * MethodMatchers included in this struct must already have been found to have matched
	 * as far as was possibly statically. Passing an array might be about 10% faster,
	 * but would complicate the code. And it would work only for static pointcuts.
	 */
	protected ReflectiveMethodInvocation(
	protected ReflectiveMethodInvocation(
			Object proxy, @Nullable Object target, Method method, @Nullable Object[] arguments,
			@Nullable Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {

		this.proxy = proxy;
		this.target = target;
		this.targetClass = targetClass;
		// 找到桥接方法，作为最后执行的方法。至于什么是桥接方法，自行百度关键字：bridge method
		// 桥接方法是 JDK 1.5 引入泛型后，为了使Java的泛型方法生成的字节码和 1.5 版本前的字节码相兼容，由编译器自动生成的方法（子类实现父类的泛型方法时会生成桥接方法）
		this.method = BridgeMethodResolver.findBridgedMethod(method);
		// 对参数进行适配
		this.arguments = AopProxyUtils.adaptArgumentsIfNecessary(method, arguments);
		this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
	}

	@Override
	public final Object getProxy() {
		return this.proxy;
	}
	@Override
	@Nullable
	public final Object getThis() {
		return this.target;
	}
	// 此处：getStaticPart返回的就是当前得method
	@Override
	public final AccessibleObject getStaticPart() {
		return this.method;
	}
	// 注意：这里返回的可能是桥接方法哦
	@Override
	public final Method getMethod() {
		return this.method;
	}
	@Override
	public final Object[] getArguments() {
		return this.arguments;
	}
	@Override
	public void setArguments(Object... arguments) {
		this.arguments = arguments;
	}

    // 核心在这里
    @Override
	@Nullable
	public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            // 这个方法相当于调用了目标方法
			return invokeJoinpoint();
		}

        // 获取集合中的 MethodInterceptor（并且currentInterceptorIndex + 1）
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());

			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
                // 执行封装了横切逻辑的invoke方法
                // 所以在interceptor.invoke方法内部, 必须调用invocation.proceed(), 才能再次恢复递归; 否则, 会发生"短路". 就无法执行上面的invokeJoinpoint();
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}

    
	/**
	 * Invoke the joinpoint using reflection.
	 * Subclasses can override this to use custom invocation.
	 * @return the return value of the joinpoint
	 * @throws Throwable if invoking the joinpoint resulted in an exception
	 */
	@Nullable
	protected Object invokeJoinpoint() throws Throwable {
        // this.target指的就是被代理的目标对象
        // 最终invokeJoinpointUsingReflection内部执行的就是method.invoke(arguments);也就是说执行了目标对象的Joinpoint
		return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
	}

    // ...    

}
```

重要的解释信息都在代码的注释中, 这里理清楚它们继承的关系, 可以看出它们是AOP Joinpoint概念实现
{% plantuml %}
Joinpoint <|-- Invocation
interface Joinpoint {}
interface Invocation {}

Invocation <|-- MethodInvocation
interface MethodInvocation {}

MethodInvocation <|-- ProxyMethodInvocation
interface ProxyMethodInvocation {}

ProxyMethodInvocation <|.. ReflectiveMethodInvocation
{% endplantuml %}

```java
package org.aopalliance.intercept;

import java.lang.reflect.AccessibleObject;

/**
 * This interface represents a generic runtime joinpoint (in the AOP
 * terminology).
 *
 * <p>A runtime joinpoint is an <i>event</i> that occurs on a static
 * joinpoint (i.e. a location in a the program). For instance, an
 * invocation is the runtime joinpoint on a method (static joinpoint).
 * The static part of a given joinpoint can be generically retrieved
 * using the {@link #getStaticPart()} method.
 *
 * <p>In the context of an interception framework, a runtime joinpoint
 * is then the reification of an access to an accessible object (a
 * method, a constructor, a field), i.e. the static part of the
 * joinpoint. It is passed to the interceptors that are installed on
 * the static joinpoint.
 *
 * @author Rod Johnson
 * @see Interceptor
 */
public interface Joinpoint {

	/**
	 * Proceed to the next interceptor in the chain.
	 * <p>The implementation and the semantics of this method depends
	 * on the actual joinpoint type (see the children interfaces).
	 * @return see the children interfaces' proceed definition
	 * @throws Throwable if the joinpoint throws an exception
	 */
	Object proceed() throws Throwable;

	/**
	 * Return the object that holds the current joinpoint's static part.
	 * <p>For instance, the target object for an invocation.
	 * @return the object (can be null if the accessible object is static)
	 */
	Object getThis();

	/**
	 * Return the static part of this joinpoint.
	 * <p>The static part is an accessible object on which a chain of
	 * interceptors are installed.
	 */
	AccessibleObject getStaticPart();

}
```

