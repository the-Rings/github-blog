---
title: 抽象工厂
date: 2021-11-03 20:01:57
tags:
- factory
---

# 动态工厂
{% blockquote Bruce Eckel, On Java 8 %}
Factories: Encapsulating Object Creation

A Factory forces object creation to occur through a common point, preventing creational code from appearing throught your system.
{% endblockquote %}
工厂模式其实是封装对象的创建过程. 强制将让对象的创建发生在一个统一的地方, 阻止代码散落在系统中.

静态工厂就不说了, 就是通过传入对象的标识, 在创建方法内部判断条件new出目标对象.
那么动态工厂, 就更加解耦, 通过传入的标识, 反射得到类, 然后`xxx.getNewInstance()`
```java
class Shape {
  @Override
  public String toString() {
    return getClass().getSimpleName();
  }
}

interface ShapeFactory {
  Shape create(String type);
}

class Circle extends Shape {
  public Circle() {}
}

class Square extends Shape {}

class Triangle extends Shape {}

public class DynamicFactory implements ShapeFactory {

  @Override
  public Shape create(String type) {
    try {
      return (Shape)
          Class.forName("package.name." + type).getConstructor().newInstance();
    } catch (Exception e) {
      System.out.println(e.getLocalizedMessage());
      return null;
    }
  }

  public static void main(String[] args) {
    Shape s = new DynamicFactory().create("Circle");
    System.out.println(s.toString());
  }
}
```
