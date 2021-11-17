---
title: java-reflection
date: 2021-11-12 17:46:37
tags:
- Reflection
---
反射 (Reflection) 是 Java 的特征之一，它允许运行中的 Java 程序获取自身的信息，并且可以操作类或对象的内部属性。

{% blockquote Oracle %}
Reflection enables Java code to discover information about the fields, methods and constructors of loaded classes, and to use reflected fields, methods, and constructors to operate on their underlying counterparts, within security restrictions.
The API accommodates applications that need access to either the public members of a target object (based on its runtime class) or the members declared by a given class. It also allows programs to suppress default reflective access control.
{% endblockquote %}


# 反射原理
简言之，反射是为了在运行时获得程序中类的属性和方法等信息。在Java中程序中的类在编译时期就确定下来了，反射机制可以动态创建对象并调用其members，对象的类型在编译时是未知的，在运行时才知道运行的对象是谁。

反射应用的三种形式，总结起来分别是：
1. 运行时得到类的信息
2. 类型转换
3. instanceof

Java中使用反射遵循以下流程。java.lang.reflect包中有Field, Method和Constructor（它们都继承AccessibleObject，实现了Member接口），在运行时JVM为类创建Field，Method和Constructor等的对象。你可以使用Constructor创建新对象，使用某个特定的Method对象调用invoke方法，或者使用Class.forName（当你没有对象的时候，你可以使用Class.forName得到类，当你拥有对象的时候，你可以通过该对象调用getClass，他们的结果是等价的），getSuperClass，getFields，getMethods，getConstructors来得到这些Member对象的数组。

以上一直在强调“动态”，但是归根结底Java仍然是一门静态语言，与Python才是真动态，在Java里所有的类在编译时期就已经完全确定了，不会再增加新的类，但是Python不一样，在运行时才知道某个类的存在，可以在运行时往类中增加额外的属性，不像Java要使用什么必须先声明出来。

很多人都认为反射在实际的 Java 开发应用中并不广泛，其实不然。当我们在使用 IDE(如 Eclipse，IDEA)时，当我们输入一个对象或类并想调用它的属性或方法时，一按点号，编译器就会自动列出它的属性或方法，这里就会用到反射。

反射最重要的用途就是开发各种通用框架。很多框架（比如 Spring）都是配置化的（比如通过 XML 文件配置 Bean），为了保证框架的通用性，它们可能需要根据配置文件加载不同的对象或类，调用不同的方法，这个时候就必须用到反射，运行时动态加载需要加载的对象。

## Class.forName(...)
反射中最常用的方法, 它与XXX.class.getClass()等价, 不做过多介绍
`Class.forName(...).getContructor()`只能获取到`Member.PUBLIC`的构造方法, 一个类如果不声明任何构造方法就会由一个默认的构造方法, 如果这个类是`Member.PUBLIC`那么默认的构造方法也是, 如果这个类是包访问权限, 那么默认构造器也是包访问权限啊

## Method Invoke
在实际应用中, 经常会用到method.invoke, 比如在动态代理中, 附上其源码来分析一下: 
```java

    /**
     * Invokes the underlying method represented by this {@code Method}
     * object, on the specified object with the specified parameters.
     * Individual parameters are automatically unwrapped to match
     * primitive formal parameters, and both primitive and reference
     * parameters are subject to method invocation conversions as
     * necessary.
     *
     * <p>If the underlying method is static, then the specified {@code obj}
     * argument is ignored. It may be null.
     *
     * <p>If the number of formal parameters required by the underlying method is
     * 0, the supplied {@code args} array may be of length 0 or null.
     *
     * <p>If the underlying method is an instance method, it is invoked
     * using dynamic method lookup as documented in The Java Language
     * Specification, Second Edition, section 15.12.4.4; in particular,
     * overriding based on the runtime type of the target object will occur.
     *
     * <p>If the underlying method is static, the class that declared
     * the method is initialized if it has not already been initialized.
     *
     * <p>If the method completes normally, the value it returns is
     * returned to the caller of invoke; if the value has a primitive
     * type, it is first appropriately wrapped in an object. However,
     * if the value has the type of an array of a primitive type, the
     * elements of the array are <i>not</i> wrapped in objects; in
     * other words, an array of primitive type is returned.  If the
     * underlying method return type is void, the invocation returns
     * null.
     *
     * @param obj  the object the underlying method is invoked from
     * @param args the arguments used for the method call
     * @return the result of dispatching the method represented by
     * this object on {@code obj} with parameters
     * {@code args}
     *
     * @exception IllegalAccessException    if this {@code Method} object
     *              is enforcing Java language access control and the underlying
     *              method is inaccessible.
     * @exception IllegalArgumentException  if the method is an
     *              instance method and the specified object argument
     *              is not an instance of the class or interface
     *              declaring the underlying method (or of a subclass
     *              or implementor thereof); if the number of actual
     *              and formal parameters differ; if an unwrapping
     *              conversion for primitive arguments fails; or if,
     *              after possible unwrapping, a parameter value
     *              cannot be converted to the corresponding formal
     *              parameter type by a method invocation conversion.
     * @exception InvocationTargetException if the underlying method
     *              throws an exception.
     * @exception NullPointerException      if the specified object is null
     *              and the method is an instance method.
     * @exception ExceptionInInitializerError if the initialization
     * provoked by this method fails.
     */
    @CallerSensitive
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
    }
```
以上源码中, 首先进行了权限检查，接着调用了methodAccessor的invoke方法，其中methodAccessor是一个共享的变量(volatile)。Reflection.getCallerClass()方法是一个native修饰的方法，native表示该方法的实现由C/C++来编写，放在本地的共享库中。

MethodAccessor是一个接口, 最终通过NativeMethodAccessorImpl来实现invoke, 然后调用native方法invoke0:
```java
class NativeMethodAccessorImpl extends MethodAccessorImpl {
    private final Method method;
    private DelegatingMethodAccessorImpl parent;
    private int numInvocations;

    NativeMethodAccessorImpl(Method var1) {
        this.method = var1;
    }

    public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException {
        if (++this.numInvocations > ReflectionFactory.inflationThreshold() && !ReflectUtil.isVMAnonymousClass(this.method.getDeclaringClass())) {
            MethodAccessorImpl var3 = (MethodAccessorImpl)(new MethodAccessorGenerator()).generateMethod(this.method.getDeclaringClass(), this.method.getName(), this.method.getParameterTypes(), this.method.getReturnType(), this.method.getExceptionTypes(), this.method.getModifiers());
            this.parent.setDelegate(var3);
        }

        return invoke0(this.method, var1, var2);
    }

    void setParent(DelegatingMethodAccessorImpl var1) {
        this.parent = var1;
    }

    private static native Object invoke0(Method var0, Object var1, Object[] var2);
}
```
