---
title: 编程小技巧
date: 2022-01-29 14:11:35
tags:
---

### 双向遍历
1. 单指针
回文字符串验证
```java
for (int i = 0; i < array.length/2; i++) {
    if (array[i] != array[array.length-1-i]) {
        // do something
    }
}
```
2. 双指针
回文字符串验证
```java
int i = 0;
int j = array.length - 1;
while (i < j) {
    if (array[i] != array[j]) {
        // do something
    }
    i++;
    j--;
}
```

### 判断是否是字母或者数字字符
ASCII码中简记如下：
A-Z 是 65-90
a-z 是 97-122
```java
private boolean isAlpha(char c) {
    if (c >= 'a' && c <= 'z') return true;
    if (c >= 'A' && c <= 'Z') return true;
    if (c >= '0' && c <= '9') return true;
    return false;
}
```

### 大写转小写
在ASCII中大写字母A与小写字母a相差了32
```java
public char toLowerCase(char c) {
    if (c >= 'A' && c <= 'Z') return c;
    if (c >= '0' && c <= '9') return c;
    return (char)(c + 32);
}
```

### 数字字符转为数字
ASCII码中数字字符减去字符0可以得到十进制真是的数字，每个数字的大小都是与0之间的距离。
```java
public int parseInt(char[] c) {
    int digit = 0;
    int i = 0;
    while (i < c.length) {
        digit = digit*10 + (c[i]-'0');
        i++;
    }
    return digit;
}
```

### 分离整数为数组
将一个int类型的整数分离为int数组
```java
int[] intToArray(int s) {
    int[] digits = new int[10]; // int类型十进制的数值最大十位
    int i = 0;
    while (s > 0) {
        int digit = s % 10;
        s = s / 10;
        digits[i] = digit;
        i++;
    }
    return digits;
}
```



### continue与while
continue逻辑判断和与while可以转化，比如，对一串字母、标点和空格组成的字符串，验证其是否是回文字符串
```javascript
int i = 0;
int j = array.length - 1;
while (i < j) {
    if (!isAlpha(array[i])) {
        i++;
        continue;
    }
    if (!isAlpha(array[j])) {
        j--;
        continue;
    }
    if (array[i] != array[j]) {
        return false;
    } else {
        i++;
        j--;    
    }
}
```
其中 continue 可以用while来代替，比如：
```javascript
// ...
while (i < j) {
    while (i < j && !isAlpha(array[i])) {
        i++;
    }
    // ...
}
// ...
```
第一段代码中，相当于continue将`i<j`的情况抽出来了，走最外层的while，接着走if，然后continue。与第二段代码中的，内层的while一直循环将i执行加加，到字母字符的位置，是一个效果。
但是加上continue之后，代码的抽象层次更高了。


