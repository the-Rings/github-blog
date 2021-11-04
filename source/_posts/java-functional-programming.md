---
title: Java函数式编程
date: 2021-11-04 11:10:25
tags:
- Java
---

## Lambda表达式
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

## Mehtod References and Unbound Method References
函数式编程的语法:
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
      X x = new X();
      // method references
      MarkString0 n = x::f;
      System.out.println(n.make());

      //unbound method references
      MarkString1 m = X::f;
      System.out.println(m.make(new X()));

   }
}
```