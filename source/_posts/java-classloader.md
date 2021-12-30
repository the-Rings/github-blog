---
title: Java中的类加载器
date: 2021-06-15 10:50:33
categories:
- Java
tags:
- ClassLoader
---

# ClassLoader
A class loader is an object that is responsible for loading classes. The class ClassLoader is an abstract class. Given the binary name of a class, a class loader should attempt to locate or generate data that constitutes a definition for the class. A typical strategy is to transform the name into a file name and then read a "class file" of that name from a file system.
类加载器负责加载类. ClassLoader是抽象类. 对于给定类的二进制名字, 类加载器应该试图去定位或者生成构成一个类定义的数据. 一个典型的策略是将这个名字转换为文件名, 然后从文件体统中读取该名字的class file.

Every Class object contains a reference to the ClassLoader that defined it.
每个Class对象都包含一个引用, 它指向定义它的那个ClassLoader.
> 这一点在源码中表现的就是, 在Class类中, 有一个Filed是ClassLoader类型
```java
public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {

    // ...

    /*
     * Private constructor. Only the Java Virtual Machine creates Class objects.
     * This constructor is not used and prevents the default constructor being
     * generated.
     */
    // NOTE: 这里出现了私有构造器, 说明这个构造函数是能在Class这类里使用
    private Class(ClassLoader loader) {
        // Initialize final field for classLoader.  The initialization value of non-null
        // prevents future JIT optimizations from assuming this final field is null.
        classLoader = loader;
    }

    // ...

    // Initialized in JVM not by private constructor
    // This field is filtered from reflection access, i.e. getDeclaredField
    // will throw NoSuchFieldException
    private final ClassLoader classLoader;

    // ...
}
```

Applications implement subclasses of ClassLoader in order to extend the manner in which the Java virtual machine dynamically loads classes.
应用实现了ClassLoader的子类, 是为了扩展JVM动态加载类的方式.

The ClassLoader class uses a delegation model to search for classes and resources. Each instance of ClassLoader has an associated parent class loader. When requested to find a class or resource, a ClassLoader instance will delegate the search for the class or resource to its parent class loader before attempting to find the class or resource itself. The virtual machine's built-in class loader, called the "bootstrap class loader", does not itself have a parent but may serve as the parent of a ClassLoader instance.
ClassLoader 类使用委托模型来搜索类和资源。 ClassLoader 的每个实例都有一个关联的父类加载器。当请求查找类或资源时，ClassLoader 实例会将类或资源的搜索委托给其父类加载器，然后再尝试查找类或资源本身。虚拟机的内置类加载器，称为“引导类加载器”，它本身没有父级，但可以作为 ClassLoader 实例的父级。

Normally, the Java virtual machine loads classes from the local file system in a platform-dependent manner. For example, on UNIX systems, the virtual machine loads classes from the directory defined by the CLASSPATH environment variable.
一般情况下, JVM加载类通过本地的文件系统. 比如, UNIX系统, JVN从CLASSPATH环境变量定义的目录中来加载类.

However, some classes may not originate from a file; they may originate from other sources, such as the network, or they could be constructed by an application. The method defineClass converts an array of bytes into an instance of class Class. Instances of this newly defined class can be created using Class.newInstance.
然而, 有些类可能不来源于文件; 它们可能来源于其他资源, 比如网络或者被应用构造. `defineClass`方法将字节数组转换为类Class的实例. 这个新定义的类的实例可以被`Class.newInstace`创建.

The methods and constructors of objects created by a class loader may reference other classes. To determine the class(es) referred to, the Java virtual machine invokes the loadClass method of the class loader that originally created the class.
被类加载器创建的对象, 它的方法和构造器可能引用自其他类. 

For example, an application could create a network class loader to download class files from a server. Sample code might look like:

   ClassLoader loader = new NetworkClassLoader(host, port);
   Object main = loader.loadClass("Main", true).newInstance();
        . . .
 

The network class loader subclass must define the methods findClass and loadClassData to load a class from the network. Once it has downloaded the bytes that make up the class, it should use the method defineClass to create a class instance. A sample implementation is:

     class NetworkClassLoader extends ClassLoader {
         String host;
         int port;

         public Class findClass(String name) {
             byte[] b = loadClassData(name);
             return defineClass(name, b, 0, b.length);
         }

         private byte[] loadClassData(String name) {
             // load the class data from the connection
              . . .
         }
     }
