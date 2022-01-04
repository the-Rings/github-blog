---
title: Java函数式编程
date: 2021-08-04 11:10:25
categories:
- Java
---
编程风格可以分为命令式(Imperative)和声明式(Declarative), 它声明了要做什么, 而不是每一步如何做. 
这正是我们在函数式编程中所看到的的.

# Lambda表达式
从概念上来说, Lambda表达式, 生产的是函数, 而不是类
但是在JVM上, everything is a class, 但是经过各种幕后操作之后, **使得Lambda看起来像函数**
```java
interface IntCall {
	int call(int arg);
}

class IntCallImpl implements IntCall {

	@Override
	public int call(int arg) {
		if (arg == 0) {
			return 1;
		} else {
			return arg * call(arg - 1);
		}
	}
}
public class FunctionalPrograming {
	static IntCall fact;

	/**
	 * 传统方法实现
	 */
	public static void oldApproach() {
		IntCallImpl intCall = new IntCallImpl();
		for (int i = 0; i <= 10; i++) {
			System.out.println("oldApproach --> " + intCall.call(i));
		}
	}

	/**
	 * 函数式编程实现
	 */
	public static void functionalApproach() {
        // 这里使用Lambda表达式的简洁语法, 但是其底层实现仍然是类和对象, IntCallImpl的步骤可能一步也没有少, 只是看起来像生成了一个函数.
        fact = n -> n == 0 ? 1 : n * fact.call(n - 1);
        for (int i = 0; i <= 10; i++) {
            System.out.println("functionalApproach --> " + fact.call(i));
        }
	}

    public static void anonymousInnerClassApproach() {
        IntCall fact = IntCallImpl::call
    }

	public static void main(String[] args) {
		FunctionalPrograming.oldApproach();
		FunctionalPrograming.functionalApproach();
	}
}
```

# Mehtod References
方法引用的语法:
{% blockquote %}
A method reference is a class name or an object name, follow by a ::, then the name of the method 
{% endblockquote %}

```java
class X {
   String f() {
      return "X::f";
   }
}

interface MarkString0 {
   String make();
}

interface MarkString1 {
   String make(X x);
}

public class UnboundMethodReferences {
   public static void main(String[] args) {
      // MarkString0 n = X::f                             [1]
      X x = new X();
      MarkString0 n = x::f;   // method references        [2]
      System.out.println(n.make());

      
      MarkString1 m = X::f;   //unbound method references [3]
      System.out.println(m.make(new X()));

   }
}
```
从逻辑上来将, 方法的引用是指向一个Method类的一个实例, 方法对象必须绑定在某个类的对象上. 按照这个逻辑, 所以`[2]`可以赋值成功, `[1]`不能. `[1]`这种方式, `class X`和`interface MarkString0`中都没有可以创建`X`的对象, 没有对象如何调用f()方法呢? 这就引出Unbound Method References

## Unbound Method References
对于这种方式(感觉不常用), 我们需要让对应的接口的第一个参数(隐性规定)必须是X. 举一反三, 如下示例:
```java
class This {
  void two(int i, double d) {}
  void three(int i, double d, String s) {}
  void four(int i, double d, String s, char c) {}
}
interface TwoArgs {
  void call2(This athis, int i, double d);
}
interface ThreeArgs {
  void call3(This athis, int i, double d, String s);
}
public class MultiUnbound {
  public static void main(String[] args) {
    TwoArgs twoargs = This::two;
    ThreeArgs threeargs = This::three;
    This athis = new This();
    twoargs.call2(athis, 11, 3.14);
    threeargs.call3(athis, 11, 3.14, "Three");
  }
}
```

## 构造函数引用
```java
class Dog {
  String name;
  int age = -1; // For "unknown"
  Dog() { name = "stray"; }
  Dog(String nm) { name = nm; }
  Dog(String nm, int yrs) { name = nm; age = yrs; }
}
interface MakeNoArgs {
  Dog make();
}
interface Make1Arg {
  Dog make(String nm);
}
interface Make2Args {
  Dog make(String nm, int age);
}
public class CtorReference {
  public static void main(String[] args) {
    MakeNoArgs mna = Dog::new; // [1]
    Make1Arg m1a = Dog::new;   // [2]
    Make2Args m2a = Dog::new;  // [3]
    Dog dn = mna.make();
    Dog d1 = m1a.make("Comet");
    Dog d2 = m2a.make("Ralph", 4);
  }
}
```
Dog 有三个构造函数，函数式接口内的 make() 方法反映了构造函数参数列表（ make() 方法名称可以不同）。

注意我们如何对 [1]，[2] 和 [3] 中的每一个使用 Dog :: new。 这三个构造函数只有一个相同名称：:: new，但在每种情况下赋值给不同的接口，编译器可以从中知道具体使用哪个构造函数。


# 函数式接口
现在我们讨论本后的本质. Java是强类型语言, 编译器必须得知每个对象的类型信息, 所以method reference和Lambda表达式都必须被赋值. 通过赋值让编译器推测出准确的类型信息. 比如:
```java
x -> x.toString()
```
编译器可以轻松地知道返回值肯定是String, 但是x是什么类型呢? Lambda表达式包含类型推导. 再比如:
```java
(x, y) -> x + y
```
必须当Lambda表达式被指派给某个接口, 才能确定其类型. 比如: 
```java
@FunctionalInterface
interface Functional {
  String goodbye(String arg);
}
interface FunctionalNoAnn {
  String goodbye(String arg);
}
/*
@FunctionalInterface
interface NotFunctional {
  String goodbye(String arg);
  String hello(String arg);
}
产生错误信息:
NotFunctional is not a functional interface
multiple non-overriding abstract methods
found in interface NotFunctional
*/
public class FunctionalAnnotation {
  public String goodbye(String arg) {
    return "Goodbye, " + arg;
  }
  public static void main(String[] args) {
    FunctionalAnnotation fa =
      new FunctionalAnnotation();
    Functional f = fa::goodbye;
    FunctionalNoAnn fna = fa::goodbye;
    // Functional fac = fa; // Incompatible
    Functional fl = a -> "Goodbye, " + a;
    FunctionalNoAnn fnal = a -> "Goodbye, " + a;
  }
}
```

其中, `@FunctionalInterface`是一个可选注解, 没有也行. 
通过以上例子我们可以得知, Lambda表达式被指派的接口只能有一个抽象方法对应. 摘录一段解释

{% blockquote Bruce Eckel, OnJava8 %}
Look closely at what happens in the definitions of f and fna. Functional and
FunctionalNoAnn define interfaces. Yet what is assigned is just the method goodbye.
First, this is only a method and not a class. Second, it’s not even a method of a class
that implements one of those interfaces. This is a bit of magic that was added to Java
8: if you assign a method reference or a lambda expression to a functional interface
(and the types fit), Java will adapt your assignment to the target interface. **Under
the covers, the compiler wraps your method reference or lambda expression in an
instance of a class that implements the target interface.**

A @FunctionalInterface is also called a Single Abstract Method (SAM) type.
{% endblockquote %}


如果将Lambda表达式作为参数传入, 看流式编程中, `Stream.of(...).map((x, y) -> x + y))`, 其中map方法接收了一个表达式. 查看其源码:

```java
<R> Stream<R> map(Function<? super T, ? extends R> mapper);
```

然后, 再查看Function接口:
```java
package java.util.function;

import java.util.Objects;

/**
 * Represents a function that accepts one argument and produces a result.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #apply(Object)}.
 *
 * @param <T> the type of the input to the function  方法输入的类型
 * @param <R> the type of the result of the function 方法输出的类型
 *
 * @since 1.8
 */
@FunctionalInterface
public interface Function<T, R> {

    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);

    // ...
}
```
由Function也可以看出其中只有一个抽象方法`apply`, 那么这个肯定与Lambda表达式的格式对应. 方法的输入和输出分别由泛型`<T, R> `与之对应.

那么@FunctionalInterface的意义是什么, 在源码注释中解释很清楚:

```java
package java.lang;

import java.lang.annotation.*;

/**
 * An informative annotation type used to indicate that an interface
 * type declaration is intended to be a <i>functional interface</i> as
 * defined by the Java Language Specification.
 *
 * Conceptually, a functional interface has exactly one abstract
 * method.  Since {@linkplain java.lang.reflect.Method#isDefault()
 * default methods} have an implementation, they are not abstract.  If
 * an interface declares an abstract method overriding one of the
 * public methods of {@code java.lang.Object}, that also does
 * <em>not</em> count toward the interface's abstract method count
 * since any implementation of the interface will have an
 * implementation from {@code java.lang.Object} or elsewhere.
 *
 * <p>Note that instances of functional interfaces can be created with
 * lambda expressions, method references, or constructor references.
 *
 * <p>If a type is annotated with this annotation type, compilers are
 * required to generate an error message unless:
 *
 * <ul>
 * <li> The type is an interface type and not an annotation type, enum, or class.
 * <li> The annotated type satisfies the requirements of a functional interface.
 * </ul>
 *
 * <p>However, the compiler will treat any interface meeting the
 * definition of a functional interface as a functional interface
 * regardless of whether or not a {@code FunctionalInterface}
 * annotation is present on the interface declaration.
 *
 * @jls 4.3.2. The Class Object
 * @jls 9.8 Functional Interfaces
 * @jls 9.4.3 Interface Method Body
 * @since 1.8
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface FunctionalInterface {}
```

`@FunctionalInterface`是为了检测, 以限制所注释的接口中只能有一个抽象方法.

在`java.util.function`中还有很多函数式接口, 这里就不在列举了.


## 高阶函数
其实就是生成函数的函数, 比如返回一个Lambda表达式.
```java
public Function<String, String> a() {
		return s -> s.toLowerCase();
	}
```

## 闭包(Closure)
一般情况下, 函数执行完成后, 其内部定义的局部变量, 将会被垃圾回收. 如果有一个函数使用函数作用域之外的变量, 并将此函数返回, 这些变量将会在函数中继续存在, 即使母函数已经执行完毕.

```java
import java.util.function.IntSupplier;

public class Closure2 {
  IntSupplier makeFun(int x) {
    int i = 0;
    return () -> x + i;
  }
}
```
makeFun方法返回的IntSupplier将i和x进行了"close over", 即使makeFun执行完毕, x和i仍然有效, 一般情况下, 它们就被回收了.

此时, 考虑如下情况:
```java
import java.util.function.IntSupplier;

public class Closure3 {
  IntSupplier makeFun(int x) {
    int i = 0;
    // 在编译时, x++ 和 i++ 都会报错
    return () -> x++ + i++;
  }
}
```
*Variable used in lambda expression should be final or effectively final*
Lambda表达式中引用的局部变量必须是final, 或者等同final. 使用时, final可以省略. 

那么, 为什么呢?
考虑在一般情况, 不使用Lambda表达式时, 我们可能会这么写:
```java
import java.util.function.IntSupplier;

public class Closure3 {
  void makeFun(int x) {
    int i = 0;
    lambda(x, i);
  }
  void lambda(int x, int i) {
      x++;
      i++;
  }
}
```
x和i传入lambda时, 一定是一个定量, 不可能是一个变量, 这不符合思维逻辑. 只是Lambda表达式给我们一个将函数返回的机会, 早期的Java版本没有这个特性.

#### 闭包的结论
```java
import java.util.function.*;
public class Closure1 {
  int i;
  IntSupplier makeFun(int x) {
    // 一切正常, 因为i是成员, 非局部.
    return () -> x + i++;
  }
}
```
Lambda可以无限制的引用成员变量(members), 但是当其引用局部变量(local variable)时, 局部变量必须声明为final.

# 总结
Lambda表达式与Method Reference原理上是一样的, 编译器会解析它们, 之后将它们包裹在一个类中, 这个类实现了目标接口. 所以, 当我们去书写Lambda表达式的时候, 编译器也会据此生成代码.

解释的更加仔细一点, 写了一个例子
```java
interface TargetInterface {
  String apply(String str);
}
class Tool {
  String connect(String data) {
    return data + "<--->" + data;
  }
}
public class MethodReferencesNature {
  public static void main(String[] args) {
    Tool tool = new Tool();
    TargetInterface t = tool::connect;
    System.out.println(t.apply("you"));
  }
}
```

以下这个示例反应了运行时生成字节码的伪代码, `$Example1`在目标接口和类之间搭建了一个桥梁
```java
/** 示例: 自动生成的类 */
class $Example1 implements TargetInterface {
  private final Tool tool;
  $Example1(Tool tool) {
    this.tool = tool;
  }
  @Override
  public String apply(String data) {
    return tool.connect(data);
  }
}

// 与Method Reference是一样的(省略main方法声明)
$Example1 example1 = new $Example1(new Tool());
example1.apply("you");
```

那么Lambda表达式, 就更进一步. 首先表达式是一个函数, 函数必须属于某个类. 所以, 生成的字节码时`Tool`类和`$Example1`都生成了, 所以Lambda和Method References是本质是一样的.
