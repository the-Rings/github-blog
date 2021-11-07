---
title: java-streams
date: 2021-11-07 18:59:42
tags:
- streams
---

引入新的概念, 如何将此融入到现有的生态百万行代码的类库中. 一个巨大挑战来自使用接口的库, 如果将一个新的方法添加到接口, 那就破坏了每一个实现接口的类,因为这些类要实现你新添加的方法.   
设计者给出的解决方案是, 在接口中添加被default修饰的方法.
通过这种方案, 将流式平滑地嵌入到现有的类中. 
流的操作有三种: creating streams, intermediate operations, terminal operations.

# Creating Streams
1. Streams.of(...)
2. collectionObject.stream()

# Intermediate Operations
使用peek()帮助调试

# Terminal Operations
1. collect(...)


