---
title: Java动态代理
date: 2021-11-01 09:23:05
tags:
- Proxy
- Spring
---

SpringAOP采用动态代理和字节码生成技术实现.

# Static Proxy
在软件系统中, 代理机制的实现有现成的设计模式支持, 就是代理模式. 在代理模式中, 通常有这几种抽象概念.
 - ISubject 该接口是对被访问资源的抽象
 - SubjectImpl 被访问资源的具体实现
 - SubjectProxy 被访问资源的代理实现
 - Client 访问者
> 没有代理的情况
```java
public class Client {
    private ISubject subject;
    Client(ISubject subject){
        this.subject = subject;
    }

    public void doSomething() {
        String url = ...;
		subject.request(url);
	}
    public static void main(String[] args) {
        Client client = new Client(new SubjectImpl());
        client.doSomething();
    }
}
public class SubjectImpl implements ISubject {
    @Override
    public String request(url) {
        return "OK" + url;
    }
}
```

> 有了代理之后的情况
```java
public class Client {
    private ISubject subject;
    Client(ISubject subject){
        this.subject = subject;
    }

    public void doSomething() {
        String url = ...;
		subject.request(url);
	}
    public static void main(String[] args) {
        // 这里增加了一个多余的强制转换
        ISubject proxy = (ISubject)new SubjectProxy(new SubjectImpl());
        Client client = new Client(proxy);
        client.doSomething();
    }
	
}
public class SubjectImpl implements ISubject {
    @Override
    public String request(url) {
        return "OK" + url;
    }
}
public class SubjectProxy implements ISubject {
    private ISubject subject;
    SubjectProxy(ISubject subject){
        this.subject = subject;
    }
    @Override
    public String request(String url){
        // add pre-process logic if neccessary
        String res = subject.request(url);
        // add post process logic if neccessary
        return "Proxy:" + res;
    }
}
```

可以看出, 没有代理之前是两个类, 有了代理是三个类, 解耦的代价就是必须在其中多加一层, 才能将其分开. 新手来, 乍一看, 这TM不是多此一举吗? 不过SubjectProxy的作用不仅仅是请求的转发, 而是可以对请求添加更多的访问控制. 如注释中的, pre-process和post-process. 在请求转发给Subject之前或之后都可以根据情况插入其他处理逻辑. 这一点就和Python中的装饰器模式一样了.

问题又来了, 我直接修改SubjectImpl.request()方法体不香吗? 为什么要增加一个类? 原因是, 分工不同, 考虑到协同开发和团队合作, 项目中有很多成员, 你怎么可以时不时就修改别人的工作呢? 所以看似软件开发是为了代码的解耦, 其实代码的解耦归根结底是来源于实际工作中的解耦, 脱离现实这一切都毫无意义.

# Dynamic Proxy
加入现在有很多接口都有request方法, 而且都需要代理的时候, 那么就需要单独为每个接口都写一个代理类, 这是不现实的. 所以, 有了动态代理, 为指定接口在系统运行期间生成代理对象. 
动态代理机制的实现主要由一个类和一个接口组成, 即Proxy类和InvocationHandler接口.

使用动态代理实现一个"request服务时间控制"功能:
```java
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

public class Client{
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
public class $Proxy0 implements ISubject {
    InvocationHandler handler;
    public $Proxy0(InvocationHandler handler) {
        this.handler = handler;
    }
    public void request(String url){
        handler.invoke(this, ISubject.class.getMethod("request", String.class), new Object[]{url});
    }
}
```

## 动态代理源码
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
 - 第二步，生产代理对象, `getProxyClass0(loader, intfs)`根据传入`newProxyInstance(...)`的参数得到一个代理对象，其中传入的参数有三个, 根据前两个参数到proxyClassCache中查找是否有缓存，如果没有，根据`ProxyClassFactory`生产出一个代理对象，`ProxyClassFactory`是一个静态内部类, 其中代理对象的生成过程，主要涉及`apply(..)`中的`ProxyGenerator.generateProxyClass(proxyName...)`行(以下附上了`ProxyGenerator`的部分源码)，当我打开`generateProxyClass`方法的源码才恍然大悟，这里直接写入了一个.class文件，来定义代理类，原来Java的动态不过如此。
 ```Java
public class ProxyGenerator {
    // ...
    public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        final byte[] var4 = var3.generateClassFile();
        // ...
        return var4;
    }
}
```
 - 第三步，根据第二步生成的代理类，调用`cl.getConstructor({ InvocationHandler.class })`, 相当于调用了代理类的构造方法, 在通过构造器调用`cons.newInstance(new RequestCtrlInvocationHandler(new ISubjectImpl()))`，最终强制转换ISubject对象。
 - 第四步，当代理对象调用`ISubject`的`request`时，首先进入`RequestCtrlInvocationHandler`的invoke方法，在该方法的最后，通过method.invoke(...)来真正调用`ISubjectImpl.request`



