---
title: CopyOnWriteArrayList的线程安全问题——subList
date: 2019-08-20 17:49:09
tags: Java
---

Review代码时发现同事写了一段加载缓存并从缓存中获取部分数据的逻辑，使用CopyOnWriteArrayList实现，代码如下：

```java
static volatile CopyOnWriteArrayList<Info>> FULL_LIST = new CopyOnWriteArrayList<>();

List<Info> getList(int start, int end) {
    return FULL_LIST.subList(start, end);
}
```

更新缓存时直接增删
```java
void updateCache() {
    // some logic
    FULL_LIST.add(...);
    FULL_LIST.remove(...);
}
```

这几个方法（`add`, `remove`, `subList`）都会获取`ReentrantLock`，看起来似乎没什么问题。但是写一个demo模拟多线程并发读并输出，一个线程修改列表，代码会抛出异常：

```java
Caused by: java.util.ConcurrentModificationException
	at java.util.concurrent.CopyOnWriteArrayList$COWSubList.checkForComodification(CopyOnWriteArrayList.java:1277)
	at java.util.concurrent.CopyOnWriteArrayList$COWSubList.size(CopyOnWriteArrayList.java:1317)
```

看一下CopyOnWriteArrayList中subList方法的代码，可以发现subList返回的是一个静态内部类COWSubList对象。

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

此外，CopyOnWriteArrayList读取时需要加锁，不可变列表无需加锁。测试对于有大量读线程的情况下，ImmutableList效率远高于CopyOnWriteArrayList。
