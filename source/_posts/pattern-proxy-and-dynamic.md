---
title: Java动态代理
date: 2021-08-01 09:23:05
categories:
- Pattern
tags:
- Proxy
---

# Static Proxy
一个代理机制，可以看作是对调用目标的一个包装，这样我们对目标代码的调用不是**直接**发生的，而是通过代理完成。**通过代理可以让调用者与实现者之间解耦。**

在代理模式中, 通常有这几种抽象概念：
 - ISubject 该接口是对被访问资源的抽象
 - SubjectImpl 被访问资源的具体实现
 - SubjectProxy 被访问资源的代理实现
 - Client 访问者
SubjectImpl不会出现在SubjectProxy类中，取而代之的是ISubject被注入到SubjectProxy中，作为一个“桥梁”，就满足了调用者和实现者的解耦

> 没有代理
```Java
public class SubjectImpl implements ISubject {
    @Override
    public String request(url) {
        return "OK" + url;
    }
}

public class Client {
    public static void main(String[] args) {
        ISubject subject = new SubjectImpl();
        String url = ...;
        subject.request(url);
    }
}
```
> 增加了代理之后
```Java
public class SubjectImpl implements ISubject {
    @Override
    public String request(url) {
        return "OK" + url;
    }
}

public class SubjectProxy implements ISubject {
    ISubject subject = new SubjectImpl();

    public static ISubject getProxy() {
        return new SubjectProxy();
    }

    @Override
    public String request(String url){
        // add pre-process logic if neccessary
        String res = subject.request(url);
        // add post process logic if neccessary
        return "Proxy:" + res;
    }
}

public class Client {
    public static void main(String[] args) {
        String url = ...;
        // 不直接使用SubjectImpl去调用，而是通过proxy去调用
        ISubject proxy = SubjectProxy.getProxy();
        proxy.request(url);
    }
}
```
不过SubjectProxy的作用不仅仅是请求的转发, 而是可以对请求添加更多的访问控制. 如注释中的, pre-process和post-process. 在请求转发给Subject之前或之后都可以根据情况插入其他处理逻辑. 这一点就和Python中的装饰器模式一样了.

# Dynamic Proxy
如果接口和方法很多，那么我们要手写很多（静态）代理类，通过观察研究这些代理类，我们抽象出一个**高阶代理类**，这个类要有这样几种行为：
- public static Class<?> **getProxyClass**(ClassLoader loader, Class<?>... interfaces)
- public static Object **newProxyInstance**(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
- public static InvocationHandler **getInvocationHandler**(Object proxy)
而动态代理的巧妙之处在于，可以在程序的运行期间，动态的生成不同的代理类且加载这个代理类再实例化，来完成相同的工作。
所以, 有了动态代理, 为指定接口在系统运行期间生成代理对象. 在执行代码的过程中，动态生成了代理类 Class 的字节码byte[]，然后通过defineClass0 加载到JVM中，简要代码如下：
```Java
Proxy.newProxyInstance
    ProxyClassFactory.apply
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
```

动态代理机制的实现，需要Proxy类和InvocationHandler接口. Proxy类是Java提供的，如果要利用动态代理处理业务，我们要做的就是写一个类实现InvocationHandler，比如，使用动态代理实现一个"request服务时间控制"功能:
```Java
public class RequestCtrlInvocationHandler implements InvocationHandler {
    private static final Log logger = LogFactory.getLog(RequestCtrlInvocationHandler.class);
    private Object target;
    public RequestCtrlInvocationHandler(Objec target) {
        this.target = target;
    }
    public Objec invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if(method.getName.equal("request")) {
            TimeOfDay startTime = new TiemOfDay(0, 0, 0);
            TimeOfDay endTime = new TiemOfDay(5, 59, 59);
            TiemOfDay currentTime = new TimeDay();
            if(currentTime.isAfter(startTime) && currentTime.isBefore(endTime)) {
                logger.warn("Service is not available now.");
                return null;
            }
            return method.invoke(target, args);
        }
        return null;
    }
}

public class Client {
    public static void main(String[] args) {
        ISubject subject = (ISubject)Proxy.newProxyInstance(ISubject.class.getClassLoader(), new Class[]{ISubject.class}, new RequestCtrlInvocationHandler(new ISubjectImpl()));
        // 这里不增加doSomething方法了, 简单演示一下调用
        String url = ...;
        subject.request(url);
    }
}
```
那么在运行期间, JVM会为我们创建class字节码, 翻译过来大概如下:
```Java
public class $Proxy0 extends Proxy implements ISubject {
    InvocationHandler handler;
    public $Proxy0(InvocationHandler handler) {
        this.handler = handler;
    }
    public void request(String url) {
        handler.invoke(this, ISubject.class.getMethod("request", String.class), new Object[]{url});
    }
}
```
最常用的动态代理实现方式有两种：JDK原生实现和Cglib开源实现。日常业务能够使用jdk动态代理编码的场景非常少。Spring在5.X之前默认的动态代理实现一直是JDK动态代理。但是从5.X开始，Spring就开始默认使用Cglib来作为动态代理实现。并且SpringBoot从2.X开始也转向了Cglib动态代理实现。我们可以在源码中找到对应的线索，SpringAOP的顶级接口`org.springframework.aop.framework.AopProxy`，这个接口只有两个实现类
- CglibAopProxy
- JdkDynamicAopProxy

# Questions
1. 为什么要用 InvocationHandler？因为sun.misc.ProxyGenerator 在生成 Proxy 字节码 byte[]时，自然希望具体的方法实现是一个模式化的code，这样才方便自动生成代码。所以**将差异化的逻辑转移到了InvocationHandler中**。
2. 为什么JDK动态代理只支持Interface的代理？如果只有SubjectImpl没有ISubject的话，就失去了代理模式的意义，不能通过ISubject将SubjectProxy和SubjectImpl解耦。

# JDK动态代理分析
有必要动态代理的源码扒出来加固上边的分析过程
```java
public class Proxy implements java.io.Serializable {

    /** parameter types of a proxy class constructor */
    private static final Class<?>[] constructorParams =
        { InvocationHandler.class };
    
    /**
     * a cache of proxy classes
     */
    private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());

    // ...
    
    /**
     * Generate a proxy class.  Must call the checkProxyAccess method
     * to perform permission checks before calling this.
     */
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }

    
    /**
     * A factory function that generates, defines and returns the proxy class given
     * the ClassLoader and array of interfaces.
     */
    private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);

            //... 

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            // ...
        }
    }
    
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        // ...

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            // ...
        } catch (InvocationTargetException e) {
            // ...
        } catch (NoSuchMethodException e) {
            // ...
        }
    }
}
```
> 其中这个newProxyInstance方法要注重说一下，大概分为四步：
 - 第一步，先进行权限检测`checkProxyAccess`，主要是借助于`System.getSecurityManager().checkxxxPermission(...)`，其中权限主要涉及：文件，socket等
 - 第二步，生产代理对象, `getProxyClass0(loader, intfs)`根据传入`newProxyInstance(...)`的参数得到一个代理对象，其中传入的参数有三个, 根据前两个参数到proxyClassCache中查找是否有缓存，如果没有，根据`ProxyClassFactory`生产出一个代理对象，`ProxyClassFactory`是一个静态内部类, 其中代理对象的生成过程，主要涉及`apply(..)`中的`ProxyGenerator.generateProxyClass(proxyName...)`行(以下附上了`ProxyGenerator`的部分源码)，当我打开`generateProxyClass`方法的源码才恍然大悟，`generateClassFile()`在内存中生成一个class类(文件一般不保存, 类名一般叫做`$Proxy0`)，来定义代理类.
 ```Java
public class ProxyGenerator {
    // ...
    public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        final byte[] var4 = var3.generateClassFile();
        if (saveGeneratedFiles) {
            // ...
            // 生成一个简单的代理对象, 一般不需要进行.class的保存, 所以这里的`saveGeneratedFiles`为false
        }
        
        return var4;
    }
}
```
 - 第三步，根据第二步生成的代理类，调用`cl.getConstructor({ InvocationHandler.class })`, 相当于调用了代理类的构造方法, 在通过构造器调用`cons.newInstance(new RequestCtrlInvocationHandler(new ISubjectImpl()))`，最终强制转换ISubject对象。
 - 第四步，当代理对象调用`ISubject`的`request`时，首先进入`RequestCtrlInvocationHandler`的invoke方法，在该方法的最后，通过method.invoke(...)来真正调用`ISubjectImpl.request`

# Cglib动态代理分析
JDK动态代理必须让SubjectImpl实现某个接口，也就是基于接口代理（interface-based proxies），有局限性。

>注：CGLIB(Code Generation Library)是一个开源项目！是一个强大的，高性能，高质量的Code生成类库，它可以在运行期扩展Java类与实现Java接口。
>CGLIB是一个强大的高性能的代码生成包。它广泛的被许多AOP的框架使用，例如Spring AOP为他们提供方法的interception（拦截）。CGLIB包的底层是通过使用一个小而快的字节码处理框架ASM，来转换字节码并生成新的类。除了CGLIB包，脚本语言例如Groovy和BeanShell，也是使用ASM来生成java的字节码。当然不鼓励直接使用ASM，因为它要求你必须对JVM内部结构包括class文件的格式和指令集都很熟悉。

如果可以生成一个子类继承SubjectImpl，子类重写父类的方法，将子类注入代理类中，通过子类调用父类的方法来实现代理，这就是Cglib动态代理（subclass-based proxies）。
使用SpringBoot时，如果Subject实现了某个接口就会使用JDK动态代理，如果实现任何接口就会使用CGLIB动态代理。我们可以在启动类上`@EnableAspectJAutoProxy(proxyTargetClass = true)`进行全局配置。
Spring框架中的Cglib动态代理，源码在org.springframework.cglib.proxy包下，也有对应的Proxy类和InvocationHandler接口。
这里我们只需要Subject类，代理它的request方法，在运行期间生成的代理类也就是子类，部分反编译的代码如下：
```Java
public class Subject$$EnhancerBySpringCGLIB$$5889ab8 extends Subject implements SpringProxy, Advised, Factory {
  // ... code ...
  private MethodInterceptor CGLIB$CALLBACK_0;
  // ... code ...
  private static final Method CGLIB$request$0$Method;
  private static final MethodProxy CGLIB$request$0$Proxy;
  // ... code ...

  static void CGLIB$STATICHOOK229() {
    CGLIB$THREAD_CALLBACKS = new ThreadLocal();
    CGLIB$emptyArgs = new Object[0];
    Class var0 = Class.forName("com.xxx.xxx.Subject$$EnhancerBySpringCGLIB$$5889ab8");
    Class var1;
    Method[] var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
    // ... code ...
    CGLIB$request$0$Method = ReflectUtils.findMethods(new String[]{"request", "()V"}, (var1 = Class.forName("com.ficc.numtalk.scheduled.Subject")).getDeclaredMethods())[0];
    CGLIB$request$0$Proxy = MethodProxy.create(var1, var0, "()V", "request", "CGLIB$request$0");
  }

  public final void request() {
    try {
      MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
      if (var10000 == null) {
        CGLIB$BIND_CALLBACKS(this);
        var10000 = this.CGLIB$CALLBACK_0;
      }

      if (var10000 != null) {
        // 
        var10000.intercept(this, CGLIB$request$0$Method, CGLIB$emptyArgs, CGLIB$request$0$Proxy);
      } else {
        super.request();
      }
    } catch (Error | RuntimeException var1) {
      throw var1;
    } catch (Throwable var2) {
      throw new UndeclaredThrowableException(var2);
    }
  }
```
