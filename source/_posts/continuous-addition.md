---
title: 滑动窗口的一次应用
date: 2022-12-12 20:11:43
categories:
- Algorithm
---

## 累加序列
Review同事的代码发现一段关于累加的代码，场景大概是这样的：
日期对应数值序列，现在要累加这些指标，计算出每一天往前推3天的总和，画出一个折线图，并要求可以配置累加天数。举个例子，如果今天是周四，那么今天应该计算周一到周三的指标总和，今天是周五的话，要计算周二到周四的总和。
我看了他的代码，他是这样处理的：整体是两次for循环，外层循环遍历整个序列，内层循环向前倒序遍历3次做累加，然后输出结果值，然后外层循环+1，整体的时间复杂度是O(3n)，虽说可以近似为O(n)，但是直觉告诉我有优化的空间，可以一趟遍历得到答案。
通过分析过程发现，同事的代码，上一次计算和下一次计算累加的过程有重叠的。周四=周三+周二+周一，周五=周四+周三+周二，三个值计算还行，如果要累加30天，这样就重复计算了28天的数据。我给出了以下优化后的代码：
```Java
int[] continuousAddition(int[] arr, int count) {
    int n = arr.length;
    int[] res = new int[n - count - 1];
    if (count > n) return new int[0];
    int p = 0;
    int sum = 0;
    while (p < count) {
      sum += arr[p++];
    }
    res[p - count] = sum;
    p++;
    while (p <= n) {
      // 只计算首尾，减去第一天，加上最后一天
      sum = sum + arr[p - 1] - arr[p - 1 - count];
      res[p - count] = sum;
      p++;
    }
    return res;
 }
```
