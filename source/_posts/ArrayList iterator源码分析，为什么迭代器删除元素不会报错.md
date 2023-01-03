---
title: ArrayList iterator源码分析，为什么迭代器删除元素不会报错
date: 2023-01-03 21:45:30
author: YangQun
toc: false
description: ArrayList#grow扩容源码解析、ArrayList#iterator、迭代器删除元素
categories: JDK
tags:
  - ArrayList
  - 源码分析
---

写这篇文章的原因要从前不久的一件事说起。

有一天，朋友问我，“ArrayList遍历中删除元素会怎么样”？

我当时脑子里第一想到的就是forEach这种循环方式，没多想就回答他：“当然会报错了。”

朋友再问：“如果使用iterator迭代器遍历删除呢？”

我：“那不会报错，可以正常删除。”

“为什么？”

为什么？我思索半刻，脑子里没有答案。

心想这种问题实在不该答不出来，只是自己在学习过程中确实没有关注过这些细节，只知道“会报错”或“不会报错”而已。

朋友见我不答，便又带几分引导的问：“forEach和iterator使用的是同一个remove方法吗？”

我又沉默，不久讪笑着说道：“不清楚。”

</br>

事后，我反思了自己的问题：在学习上不够仔细，只知其然而不知其所以然，虽也足以应对工作，但若是想在开发这条路上继续走下去，则是万万要改正的。

基于此教训，我也是开始关注这些“细枝末节”，并成文做记录，此就是其中一篇。

</br>

废话讲完，先看看下面的例子。

```java
// forEach方式
ArrayList<String> l = new ArrayList<>();
l.add("1");
for (String s : l) {    // ----- 结果：java.util.ConcurrentModificationException
    l.remove(s);
}

// 迭代器方式
ArrayList<String> l = new ArrayList<>();
l.add("1");
l.add("2");
Iterator<String> iterator = l.iterator();
while (iterator.hasNext()){
    String next = iterator.next();
    iterator.remove();   // ----- 结果：正常删除
}
```

正如我对朋友所说，`forEach`会报错，`Iterator`能够如我们期望的删除掉元素。那它们两者有什么区别呢？朋友问我它们用的是不是同一个`remove`方法又是何意？

> forEach也不总是报错，会有巧合不报错的情况。

# forEach语法

我们先来说一下`forEach`这个语法，java文档中叫它“The enhanced for statement”，翻译过来就是“增强for循环”。它是Java提供的一个语法糖，经过编译器的翻译后(javac命令)，它会被翻译成普通的for循环。

```java
ArrayList<String> l = new ArrayList<>();
l.add("1");

for (String s : l) {
    l.remove(s);
}
```

以上代码在经过`javac`编译后的结果如下：

```JAVA
ArrayList<String> l = new ArrayList();
l.add("1");
Iterator var2 = l.iterator();

while(var2.hasNext()) {
    String s = (String)var2.next();
    l.remove(s); // ----- 用的是ArrayList的remove方法
}
```

可以看到，原本`forEach`的部分被自动替换成了`Iterator`，方法里通过`Iterator#hasNext`来控制循环次数、通过`Iterator#next`方法来获取下一个元素。乍一看，好像和文章开头我们写的`Iterator`示例代码一样，但仔细看就会发现它的删除操作是使用了`ArrayList`本身的`remove`方法。

正是这个删除操作的不同，会导致我们在使用`forEach`删除元素时抛出`ConcurrentModificationException`异常，要搞清楚抛出异常的原因，我们需要先来看一下`ArrayList`的`remove`方法。

> 使用增强for循环的表达式必须是Iterable接口的子类或是一个数组类型，否则会发生编译时错误。
>
> 至于数组类型的forEach会被翻译成什么样子，读者不妨自己亲自编译看下。

# ArrayList#remove

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null;
}
```

代码很简单，就是遍历找到**第一个相等的元素**，然后把它从当前数组里剔除掉。元素的删除是由`System.arraycopy`方式完成的，删除之后维护一下`size`字段。

需要特别注意的是`modCount++`这个操作，该字段记录`list`在结构上被修改的次数。 结构修改是指那些改变`list`大小的修改，此字段由类`Iterator`和`ListIterator`使用，如果此字段的值意外更改，`Iterator`（或ListIterator）将抛出 `ConcurrentModificationException` 以响应 `next、remove、previous、set `或 `add` 操作。

如下贴出`Iterator`的`remove`方法，报错的原因就一目了然：

```java
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        // ...
    }

    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();           // ----- 看这里
        /*
         * final void checkForComodification() {
         *     if (modCount != expectedModCount)
         *         throw new ConcurrentModificationException();
         * }
         */

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;     // ----- 看这里
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        // ...
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

至此，文章结束，我们再回答文章一开始的问题，“forEach和iterator使用的是同一个remove方法吗”？答：是，只不过iterator里面包装了一些处理modCount字段的逻辑。



# 参考目录

1 ：

[Java Language Specification](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.14.2)
