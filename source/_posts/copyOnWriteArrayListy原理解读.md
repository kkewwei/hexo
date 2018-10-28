---
title: CopyOnWriteArrayList原理解读
date: 2017-09-17 10:14:15
tags:
---
CopyOnWriteArrayList是为了解决并发场景下对ArrayList读写的, 可以根据名称大致可知如何使用: 当写入的时候的对ArrayList进行copy。使用场景: 读多写少。
## 成员变量
+ lock
final transient ReentrantLock lock = new ReentrantLock();
使用ReentrantLock互斥可重入锁, 在向CopyOnWriteArrayList写入数据前, 首先获取锁, 然后才能继续。若获取不到, 则被阻塞。
+ array
private transient volatile Object[] array;
真正存放数据的地方, 由Object可知, CopyOnWriteArrayList可以存放任何类型的对象。 这里使用了volatile属性, 确保该操作是原子可见的, 该线程添加元素对别的线程都是立刻可见的。
## 添加数据add
```
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
主要做了如下事情:
+ 首先获取互斥可重入锁。 若获取不到, 则被阻塞。
+ 将原来数组元素copy到新的数组
+ 将新添加元素都放到数组newElements最后。
+ 替换原来数组elements
过程如下:
<img src="http://pgagp8vnu.bkt.clouddn.com/CopyOnWriteArrayList.png" height="300" width="350"/>
## 总结
CopyOnWriteArrayList在添加元素时, 需要获取锁, 可知互斥写入, 并发为1。 若多线程同时写入, 效率非常低下, 而且会出现线程复制数组而直接丢弃原来数组, 导致大量gc。CopyOnWriteArrayList主要在查多写少的情况下。