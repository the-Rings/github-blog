---
title: java-reflection\
date: 2021-11-13 11:02:41
tags:
- Reflection
---
反射 (Reflection) 是 Java 的特征之一，它允许运行中的 Java 程序获取自身的信息，并且可以操作类或对象的内部属性。

# 反射原理
{% blockquote Oracle %}
Reflection enables Java code to discover information about the fields, methods and constructors of loaded classes, and to use reflected fields, methods, and constructors to operate on their underlying counterparts, within security restrictions.
The API accommodates applications that need access to either the public members of a target object (based on its runtime class) or the members declared by a given class. It also allows programs to suppress default reflective access control.
{% endblockquote %}

Java反射能够使得Java代码获得已加载类的属性/方法/构造器信息, 并且可以安全地使用这些反射得到的属性/方法/构造器去操作这些属性方法构造器底层对应的事物.
