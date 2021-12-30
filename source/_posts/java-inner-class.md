---
title: Java内部类
date: 2021-11-04 10:24:37
categories:
- Java
---

## 匿名内部类
{% blockquote %}
Anonymous classes enable you to make your code more concise. They enable you to declare and instantiate a class at the same time. They are like local classes except that they do not have a name. Use them if you need to use a local class only once.
{% endblockquote %}
匿名内部类可以使你的代码更加简洁，你可以在定义一个类的同时对其进行实例化。它与局部类很相似，不同的是它没有类名，如果某个局部类你只需要用一次，那么你就可以使用匿名内部类

语法如下:
```java
interface Contents {
	int value();
}
public class Parcel {
	public Contents getContents () {
		return new Contents() {
			private int i = 11;
			@Override
			public int value() {
				return i;
			}
		};
	}
    public static void main(String[] args) {
		Parcel p = new Parcel();
		p.getContents().value();
	}

// output:
// 11
}
```
声明一个表达式, new后加接口(类)名, 表示这个匿名内部类实现(继承)了对应的接口(类)