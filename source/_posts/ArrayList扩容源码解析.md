---
title: ArrayList扩容源码解析
date: 2022-11-05
author: YangQun
toc: false
description: ArrayList#grow扩容源码解析、ArrayList#add方法解析、ArrayList overflow-conscious code
categories: JDK
tags:
  - ArrayList
  - 源码分析
---

> 环境信息
> jdk： 1.8.0_341
> macOs 64位

## 介绍
`ArrayList`是`List`接口的一个实现，内部基于**数组**数据结构存储数据；随着元素被添加到`ArrayList`，它的容量会自动增长，也就是说`ArrayList`的容量是可以动态调整的，当空间不够用时，默认增加容量的50%。

在性能方面，`size, isEmpty, get, set, iterator, and listIterator`等操作总是run in constant time（即O(1)）；`add`操作则是amortized constant time，也就是说，添加1个元素需要O(1)的时间，添加n个元素就需要O(n)的时间。对于剩下的其它操作，粗略来讲都是linear time(O(n))。

`ArrayList`不能保证线程安全，如果要在多线程环境下使用`ArrayList`，则应该使用`Vector`或`Collections.synchronizedList(new ArrayList(...));`。

## 关键属性

以下是和扩容相关的一些属性：

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{

    /**
     * 真正用来存储元素的数组，ArrayList的capacity就是这个数组的长度。
     * 注：ArrayList没有capacity这个属性，代码中以elementData.length代表capacity。
     */
    transient Object[] elementData;
    /**
     * 仅使用无参构造器后，添加第一个元素时，容量会默认扩展为10。
     */
    private static final int DEFAULT_CAPACITY = 10;
    /**
     * ArrayList内已有元素的个数。
     * capacity是ArrayList能存放的最大元素个数；size是已有的实际个数。
     */
    private int size;

    /**
     * 使用无参构造器时，elementData会被初始化为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，之后添加元素时代码会
     * 判断element == DEFAULTCAPACITY_EMPTY_ELEMENTDATA，决定是否扩展至DEFAULT_CAPACITY。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 使用有参构造器时，如果传入的初始容量为0或传入空Collection，element会被赋值为EMPTY_ELEMENTDATA。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};
}
````
## 源码分析
### add方法
```java
public boolean add(E e) {
    // 确保容量，如果容量不够先扩容
    ensureCapacityInternal(size + 1);
    // size初始值为0，当调用add时将该元素放到index为0的位置。之后size++为1，代表当前ArryList内元素个数为1，且下次添加元素的index为1。
    elementData[size++] = e;
    return true;
}
```
往下看`ensureCapacityInternal`的内容：
```java
/**
 * minCapacity：代表当前数组至少也要有这个容量才可以，否则就扩容。
 *  如果调用方时`add(E e)`方法，则`minCapacity`传入值为当前`size + 1`;
 *  如果调用方为`addAll(Collection c)`，`minCapacity`则为`size + c.size()`。
 */
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    
    // `minCapacity`大于`ArrayList`的容量时需要扩容。
    if (minCapacity - elementData.length > 0)
      	// 扩容
        grow(minCapacity);
}
```

### grow扩容方法
```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
      	// 扩容
        grow(minCapacity);
}

private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
  
    // 容量增加50%，（oldCapacity >> 1）等于（oldCapacity / 2）
    int newCapacity = oldCapacity + (oldCapacity >> 1);
  
    // 如果容量增加50%仍不够用，则使用minCapacity作为新容量。
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
  
    // 如果newCapacity大于MAX_ARRAY_SIZE，则尝试为它分配MAX_ARRAY_SIZE或Integer.MAX_VALUE的大小。
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
  
    // 创建一个大小为newCapacity的新数组，并将元素统统拷贝过去。
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```
扩容代码里有几个细节需要我们注意：

1. 为什么不直接把`MAX_ARRAY_SIZE`设置为`Integer.MAX_VALUE`，非要等到`newCapacity`大于`MAX_ARRAY_SIZE`时才扩容至`Integer.MAX_VALUE`？
2. 源码中注释`overflow-conscious code`想表达什么意思？以及为什么用`a - b > 0`，而不用`a > b`？

#### MAX_ARRAY_SIZE

先来看第一个问题：通常情况下，`ArrayList`不会返回大于`MAX_ARRAY_SIZE`的容量，除非给定的`minCapacity`大于`MAX_ARRAY_SIZE`。

原因是因为现在很多JVM的实现(例如常用的hotspot)对申请大小为Integer.MAX_VALUE的数组有限制，会抛出`OutOfMemoryError("Requested array size exceeds VM limit") `异常。这个限制值的大小取决于JVM实现的特征，例如对象头大小。所有的JVM的实现最大就限制到了`Integer.MAX_VALUE - 8`，所以JDK保守的选择了最大值8来应对遇到的所有限制。

```java
Object[] arr = new Object[Integer.MAX_VALUE];
// 异常：java.lang.OutOfMemoryError: Requested array size exceeds VM limit。-- jvm会直接限制住。
        
Object[] arr = new Object[Integer.MAX_VALUE - 8];
// 异常：java.lang.OutOfMemoryError: Java heap space。 -- jvm不会限制，尝试分配内存结果堆溢出。
```
#### overflow-conscious code 与 a - b > 0

```java
// 此示例代码引自参考目录编号2的文章。
int a = Integer.MAX_VALUE;
int b = Integer.MIN_VALUE;
if (a < b) {
    System.out.println("a < b");
}
if (a - b < 0) {
    System.out.println("a - b < 0");
}
```
在上面这个例子中，控制台会打印`a - b < 0`。显然，`a < b`是错误的。而`a - b`发生了溢出，结果为-1，所以会打印出`a - b < 0`。

在jdk7就存在由这个问题导致的bug，我们不妨看下jdk7的扩容源码。

```java
// jdk 7 源码
public boolean add(E e) {
    ensureCapacity(size + 1);
    elementData[size++] = e;
    return true;
}

public void ensureCapacity(int minCapacity) {
    modCount++;
    int oldCapacity = elementData.length;
    if (minCapacity > oldCapacity) {
        int newCapacity = (oldCapacity * 3)/2 + 1;
        if (newCapacity < minCapacity)
            newCapacity = minCapacity;
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
}
```
这个BUG很容易重现：假设当前的`capacity`至少为  `Integer.MAX_VALUE`的三分之二，则`(oldCapacity * 3)/2 + 1`的结果会溢出为负数，此时`newCapacity < minCapacity`为`true`，`newCapacity`被设置为`minCapacity`。那这样会造成什么问题呢？

在这种情况调用`add`方法，`capacity`只会+1，而不是增加50%，而且每次`add`操作都会触发扩容导致性能下降(Arrays.copyOf)。

---

理解了上面后，我们再看JDK8的`grow`方法 ，为了方便阅读，我把代码再贴一次。

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```
在`ensureExplicitCapacity`方法里，需要注意的是入参`minCapacity`可能已经溢出为负数。无论`minCapacity`是真的大于`elementData.length`，或是溢出后与`elementData.length`相减再溢出为正数，都会进入到`grow`方法。

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
  
    int newCapacity = oldCapacity + (oldCapacity >> 1);    -----代码①

    if (newCapacity - minCapacity < 0)    ----- 代码②
        newCapacity = minCapacity;		----- 代码③
          
    if (newCapacity - MAX_ARRAY_SIZE > 0)   ----- 代码④
        newCapacity = hugeCapacity(minCapacity);
  
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

如果代码①处`newCapacity`溢出为负数，而且`minCapacity`是正数的话，代码②处会再次溢出为正数（因为minCapacity - elementData.length > 0）。到代码④处，同样溢出为正数，最后会在`hugeCapacity`里返回`Integer.MAX_VALUE`或`MAX_ARRAY_SIZE`。

当然，`minCapacity`也有可能是负数，在这个前提下，无论代码②处结果如何，代码④都会溢出为正数，然后在`hugeCapacity`抛出异常。

> 以上两种情况都是在newCapacity溢出的前提下。

但是！其实这里是有BUG的（JDK后续版本已修复），我们把代码精简如下：

```java
private static int calculate(final int minCapacity, final int length) {
	if (minCapacity - length > 0) {
		int oldCapacity = length;
		int newCapacity = oldCapacity + (oldCapacity >> 1);
		if (newCapacity - minCapacity < 0) {
			newCapacity = minCapacity;
		}
		if (newCapacity - (Integer.MAX_VALUE - 8) > 0) {
			if (minCapacity < 0) {
				throw new OutOfMemoryError();
			}
			newCapacity = minCapacity > (Integer.MAX_VALUE - 8) ?
				Integer.MAX_VALUE :
				(Integer.MAX_VALUE - 8);
		}
		return newCapacity;
	}
	return length;
}
```

当入参是`calculate((Integer.MAX_VALUE - 2) + (Integer.MAX_VALUE - 2), (Integer.MAX_VALUE - 2))`时，结果会返回-6。

可以运行如下代码验证，代码抛出NegativeArraySizeException异常：

```java
// -Xmx17g
public static void main(String[] args) {
  	// Integer.MAX_VALUE - 4: hugeCapacity 抛出 OutOfMemoryError.
		// Integer.MAX_VALUE - 2 or - 3:，申请负数长度的数组，报错：NegativeArraySizeException
		// Integer.MAX_VALUE - 1: OutOfMemoryError: Requested array size exceeds VM limit
    int size = Integer.MAX_VALUE - 2;
    ArrayList<Object> huge = new ArrayList<>(size);
    for (int i = 0; i < size; i++)
        huge.add(null);
    try {
        huge.addAll(huge);
    } catch (OutOfMemoryError e) {
      	e.printStackTrace();
    }
}
```

上面这个测试需要内存 > 32g，可以改掉ArrayList的源码，这样内存只需要16g就可以了，改动如下：

```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

改为
  
public Object[] toArray() {
    return elementData;
}
```

到此扩容方法就讲完了，内容如果有问题，也希望大佬们不吝赐教，共同进步。


## 参考目录
1: [Java SE API Documentation](https://docs.oracle.com/javase/8/docs/api/index.html)

2: [Difference between if (a - b < 0) and if (a < b)](https://stackoverflow.com/questions/33147339/difference-between-if-a-b-0-and-if-a-b/33147610#33147610)