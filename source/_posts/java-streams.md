---
title: 流式编程
date: 2021-06-07 18:59:42
categories:
- Java
tags:
- Stream
---

Java8引入新的概念Streams. 如何将此融入到现有的生态百万行代码的类库中. 一个巨大挑战来自使用接口的库, 如果将一个新的方法添加到接口, 那就破坏了每一个实现接口的类,因为这些类要实现你新添加的方法.   
设计者给出的解决方案是, 在接口中添加被default修饰的方法.
通过这种方案, 将流式平滑地嵌入到现有的类中. 
流的操作有三种: creating streams, intermediate operations, terminal operations.

{% blockquote Bruce Eckel, OnJava8 %}
**Declarative programming** is a style where we state what we want done, rather than
specifying how, and it’s what you see in functional programming. Notice it’s much
more difficult to understand the **imperative programming**.
{% endblockquote %}
对比一下声明式编程和命令式编程
```java
// Declarative programming
new Random(47).ints(5, 20).distinct().limit(7).sorted().forEach(System.out::println);
```

```java
// Imperative programming
Random rand = new Random(47);
SortedSet<Integer> rints = new TreeSet<>();
while(rints.size() < 7) {
    int r = rand.nextInt(20);
    if(r < 5) continue;
    rints.add(r);
}
System.out.println(rints);
```
显然命令式编程更难理解, 而且在Streams过程中, 没有明显地看到任何变量的定义.
语义清晰是Java Stream API的优点之一.

# Creating Streams
1. 通过Streams.of(...)将一组元素转换为Stream
2. 每隔集合可以通过调用其`.stream()`方法来产生一个流.
3. `Stream.generate(...)`, generate接收一个参数`Supplier<T>`, 通过调用实现了`Supplier<T>`接口的类的get方法(也可以传Lambda表达式, 原理也是有个类实现了Supplier接口, 具体在函数式编程里有详细介绍). 不过generate必须有一个限制, 不能无限制的运行. 否则会造成`java.lang.OutOfMemoryError: Java heap space`错误. 比如, 中间加入了limit函数
```java
Stream.generate(() -> "h").limit(30).collect(Collectors.joining());
```
4. `Stream.iterate(...)`, iterate接收的参数第一个是seed, 第二个其实是`Function<T>`, 如下例子, 生成斐波那契数列:
```java
int x = 1;
// seed将会作为i的初始化值, 返回值y将作为参数再次会传递给该函数
Stream.iterate(0, (i) -> {int y = i + x; x = i; return y;}).limit(30).forEach(System.out::println);
```
5. 使用建造者模式. 其实建造者模式类似于新建一个集合, 然后往里put元素.
```java
Stream.Builder<String> builder = Stream.builder();
String res = builder.build().map(...).collect(Collector.joining(" "));
```
6. `Arrays.stream()`

# Intermediate Operations
1. .map(...)
2. .peek(...), 使用peek()帮助调试
3. .skip(...)
4. .limit(...)
5. .sorted(Comparator.reverseOrder())
6. .filter(Predicate)
7. .distinct()
8. parallel(), 这个重点说一下, 这个是将流中计算机制换为并行计算, 所以最终输出元素的顺序是随机的. 要是再此基础上要得到有序的输出, 需要上如`forEachOrdered(...)`等的Terminal Operations


# Terminal Operations
1. .collect(Collector.toList())
2. .toArray()
3. .forEach()
4. .reduce(...), 最终输出的是一个元素. 
```java
// Reduce函数功能展示
import java.util.Random;
import java.util.stream.Stream;

class IntegerSupplier {
    // 生成一个100以内的随机数流(建造者模式)
	static Stream<Integer> supply() {
		Random rand = new Random(15);
		final int BOUND = 100;
		Stream.Builder<Integer> builder = Stream.builder();
		for(int i = 0; i < 30; i++) {
			builder.add(rand.nextInt(BOUND));
		}
		return builder.build();
	}
}

public class Reduce {
	public static void main(String[] args) {
		// reduce(BinaryOperator), BinaryOperator这个函数式接口对应的函数只有两个参数
		// 其中fr0既作为初始化的第一个值, 又作为每次的返回值传入, 在输出中可以看到
		// 最终reduce返回的值, Integer被Optional包裹
		IntegerSupplier.supply().reduce((fr0, fr1) -> {
			System.out.println(fr0 + "--compare to--" + fr1);
			return fr0 < fr1 ? fr0 : fr1;
		}).ifPresent(System.out::println);
	}
}
// output:
/* 41--compare to--72
 * 72--compare to--38
 * 72--compare to--67
 * 72--compare to--85
 * 85--compare to--60
 * 85--compare to--59
 * Optional[85]
*/
```

### Optional类
为了防止流执行的过程中, 出现"空流"而中断抛出异常, 所以引入了Optional类, 为空时将会返回`Optional.empty`
```java
// 空流, 等价于: Stream<String> s = Stream.empty();
Stream.<String>empty().findFirst()
// output: Optional.empty
```
原理是将返回值塞到了Optional对象中, Optional类提供了很多诸如`isPresent()`, `orElse`, `orElseGet`等方法, 在Python3.7以后引入类型系统后也有了Optional类型, 如果一个变量被标识为Optional, 那么这个对象可能为none.

