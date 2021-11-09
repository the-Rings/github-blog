---
title: java-streams
date: 2021-11-07 18:59:42
tags:
- streams
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
2. 每隔几何可以通过调用其`.stream()`方法来产生一个流.

# Intermediate Operations
使用peek()帮助调试

# Terminal Operations
1. collect(...)


