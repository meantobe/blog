---
title: CopyOnWriteArrayList的subList问题
date: 2019-08-20 17:49:09
tags: Java
---

摘要：
1. CopyOnWriteArrayList的subList仅是原始列表的视图，原始列表修改后会出现ConcurrentModificationException问题。
2. 对CopyOnWriteArrayList的subList任何操作都需要获取读锁，效率低于ImmutableList。

<!-- more -->

Review代码时发现同事写了一段加载缓存并从缓存中获取部分数据的逻辑，使用CopyOnWriteArrayList实现，代码如下：

```java
static volatile CopyOnWriteArrayList<Info>> FULL_LIST = new CopyOnWriteArrayList<>();

List<Info> getList(int start, int end) {
    return FULL_LIST.subList(start, end);
}
```

更新缓存时对CopyOnWriteArrayList进行操作：
```java
void updateCache() {
    // some logic
    FULL_LIST.add(...);
    FULL_LIST.remove(...);
}
```

这几个方法（`add`, `remove`, `subList`）都会获取`ReentrantLock`，看起来似乎没什么问题。但是写一个例子就会发现对subList操作时可能会抛出异常：

```java
public class CWALTest {
    public static void main(String[] args) {
        CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>();
        list.add(1);
        list.add(2);
        list.add(3);

        List<Integer> sub = list.subList(1, 2);
        list.add(4);
        sub.get(0); // will throws ConcurrentModificationException
    }
}
```
异常信息如下：

```java
Caused by: java.util.ConcurrentModificationException
	at java.util.concurrent.CopyOnWriteArrayList$COWSubList.checkForComodification(CopyOnWriteArrayList.java:1277)
	at java.util.concurrent.CopyOnWriteArrayList$COWSubList.size(CopyOnWriteArrayList.java:1317)
```

查看CopyOnWriteArrayList中subList方法的代码，可以看到subList返回的是一个静态内部类COWSubList对象。

```java
public List<E> subList(int fromIndex, int toIndex) {
    final ReentrantLock lock = l.lock;
    lock.lock();
    try {
        checkForComodification();
        if (fromIndex < 0 || toIndex > size || fromIndex > toIndex)
            throw new IndexOutOfBoundsException();
        return new COWSubList<E>(l, fromIndex + offset, toIndex + offset);
    } finally {
        lock.unlock();
    }
}
```

而COWSubList中有一个expectedArray属性，指向是对象创建时CopyOnWriteArrayList的array对象。

```java
private static class COWSubList<E> extends AbstractList<E> implements RandomAccess {
    private final CopyOnWriteArrayList<E> l;
    private final int offset;
    private int size;
    private Object[] expectedArray;
    
    // only call this holding l's lock
    COWSubList(CopyOnWriteArrayList<E> list, int fromIndex, int toIndex) {
        l = list;
        expectedArray = l.getArray();
        offset = fromIndex;
        size = toIndex - fromIndex;
    }
```

在之后对子列表操作时，会先调用`checkForComodification`方法，若原CopyOnWriteArrayList被修改，则抛出异常。

```java
// only call this holding l's lock
private void checkForComodification() {
    if (l.getArray() != expectedArray)
        throw new ConcurrentModificationException();
}
```

因此取subList时需要考虑线程安全问题，更好的方式是使用不可变列表（例如Guava的ImmutableList），避免对原始列表的更新导致子列表抛出异常，更新缓存时指向新的ImmutableList对象即可，相关代码如下

```java
static volatile ImmutableList<Info>> FULL_LIST = ImmutableList.of();

List<Info> getList(int start, int end) {
    return FULL_LIST.subList(start, end);
}

void updateCache() {
    // some logic
    FULL_LIST = ImmutableList.copyOf(newList);
}
```

ImmutableList的subList方法取的的是一个子类SubList，和COWSubList类似，仅是一个视图。但由于列表不可变，仅需要检查索引未越界即可。

```java
class SubList extends ImmutableList<E> {
    final transient int offset;
    final transient int length;

    SubList(int offset, int length) {
        this.offset = offset;
        this.length = length;
    }
```

此外，不可变列表无需加锁，避免了COWSubList中操作都需要获得锁的不便。测试对于有大量读线程的情况下，ImmutableList读取效率远高于COWSubList。
