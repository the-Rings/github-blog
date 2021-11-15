---
title: java-reflection
date: 2021-11-12 17:46:37
tags:
- Reflection
---

# 反射
`Class.forName(...).getContructor()`只能获取到`Member.PUBLIC`的构造方法, 一个类如果不声明任何构造方法就会由一个默认的构造方法, 如果这个类是`Member.PUBLIC`那么默认的构造方法也是, 如果这个类是包访问权限, 那么默认构造器也是包访问权限啊