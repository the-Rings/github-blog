---
title: ArrayList
date: 2021-11-15 16:32:10
tags:
- List
---

ArrayList基于数组的List集合, 那么从数组中删除一个元素的底层逻辑是什么样?

### ArrayList.remove(index)
```java
List<String> al = new ArrayList<>(Arrays.asList("a", "b", "c", "d", "e"));
String old = al.remove(0);
System.out.println(old);

// output:
// a
```
> 注意: `Arrays.asList(...).remove(index)`会抛出`UnsupportedOperationException`.
> Arrays.asList方法返回一个ArrayList(内部类), 是`java.util.Arrays`下的一个内部类, 对remove并没有实现逻辑
> 通过`Arrays.asList(...)`得到的List是一个只读的集合, 也没有实现add方法. 所以这里可以通过ArrayList的构造方法新建可以增删改查的集合

ArrayList源码(部分):
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    // ...
    
    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    E elementData(int index) {
        // elementData
        return (E) elementData[index];
    }

    /**
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).
     *
     * @param index the index of the element to be removed
     * @return the element that was removed from the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        // numMoved表示要移动的元素数量
        // 如果等于0说明, 要删除的是最后一位, 不需要移动, 直接将最后一位设为null即可
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    // ...
}
```

ArrayList内部维护了一个数组`elementData`. remove并没有删除数组中的元素, 而是通过`System.arraycopy`方法将index后的元素向前移动一位, 把最后一位的的值设为null.
System.arraycopy是一个native方法
```java
/**
 * 将从src数组的指定位置srcPos开始复制length长度的元素, 对应到dest数组的descPos位置依次往后替换
 * 比如, 当length为2的两个数组, srcPos=3, destPos=0
 * src: [a, b, c, d, e]
 *                ↓  ↓
 * dest:         [1, 2, 3, 4, 5]
 * output: [a, b, 3, 4, 5](dest)
 * 
 * @param src     the source array.
 * @param srcPos  starting position in the source array.
 * @param dest    the destination array.
 * @param destPos starting position in the destination data.
 * @param length  the number of array elements to be copied.
 */
public static native void arraycopy(Object src,  int  srcPos, Object dest, int destPos, int length);
```
寻找native方法的实现C++源码, 下载[OpenJDK](https://github.com/openjdk/jdk/tags?after=jdk8-b114)源码, 在`jdk\src\share\native`你可以找到C/C++源码
 - jdk\src\linux source for linux.
 - jdk\src\windows source for windows.
 - jdk\src\solaris souce for solaris.
 - jd\src\share common source.

最后在`hotspot\src\share\vm\oops\objArrayKlass.cpp`168行, 其中有copy_array方法, 大概思路是这样的:
```java
while(length > 0){
    dest[destPos] = src[srcPos]
    destPos++;
    srcPos++;
    length--;
}
```
如果List集合对应的数组是`[a, b, c, d, e]`, 现在执行`.remove(0)`, 调用`System.arraycopy(elementData, 1, elementData, 0, 4)`后得到的`elementData`是`[b, c, d, e, e]`, 数组的长度不会发生变化. 紧接着执行`elementData[--size] = null`, `[b, c, d, e, null]`, 完成操作. 源码注释中解释到: **clear to let GC do its work**, 让GC完成接下来的工作, 此时ArrayList.size已经减1, 但是由于数组在被创建出来以后长度无法更改, 所以其对应的数组`elementData`的长度依然不变.

```java
elementData[--size] = null; // clear to let GC do its work
```
这样一行代码让人感觉很奇怪. 首先, 数组中的最后一个元素是由于所有位置的元素向前移动而多出来的, 这个元素的值不应该能被外界访问到; 再者, 这个元素可能会在程序中保持很长时间(因为List容器还在使用暂时无法回收), 也不应该再被使用, 设为null, 做为一个垃圾回收的标志.


