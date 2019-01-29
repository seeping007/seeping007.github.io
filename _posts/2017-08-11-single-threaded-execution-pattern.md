---
layout: post
title: 多线程设计模式 | Single Threaded Execution
description: Single Threaded Execution模式：设置限制以确保同一时间内只能让一个线程执行处理
category: blog
---

## 多线程程序设计的难点

- 就算没检查出错误，也不能说程序就一定安全。测试次数不够、时间点不对，都有可能检查不出错误。
- 如果显示调试信息的代码本身就是非线程安全的，那么显示的调试信息就很可能是错误的。



## Single Threaded Execution模式

> 该模式用于设置限制，以确保同一时间内只能让一个线程执行处理。

### synchronized的作用

- synchronized方法能够确保该方法同时只能由一个线程执行。
- 临界区：只允许单个线程执行的程序范围。

### 适用场景

- 当共享资源的实例有可能被多个线程同时访问时
- 共享资源的状态会发生变化时

### 生存性与死锁

> 死锁是指两个线程分别持有着锁，并互相等待对方释放锁的现象。

满足下列条件时，死锁就会发生：

1. 存在多个SharedResource角色
2. 线程在持有者某个SharedResource角色的锁的同时，还想获取其他SharedResource角色的锁
3. 获取SharedResource角色的锁的顺序并不固定

只要破坏三个条件中的一个，就可以防止死锁发生。

### 临界区的大小和性能

一般情况下，该模式会降低程序性能，原因如下：

- 获取锁花费时间
- 线程冲突引起的等待

解决方式：尽可能地缩小临界区的范围，降低线程冲突的概率。



## 关于synchronized

### synchronized语法

```java
// synchronized方法
synchronized void method() {
    ...
}

// synchronized代码块
synchronized (obj) {
    ...
}

// synchronized方法和代码块无论是执行return还是抛出异常，都一定能够释放锁。

// 显式处理锁的方法
void method() {
    lock();
    // do something
    // 若这中间存在return或调用的方法抛出异常时，那么锁就有可能无法被释放
    unlock();
}
```

### synchronized在保护着什么

synchronized就像是门上的锁。当看到门上了锁时，还应该确认其他的门和窗户是不是都锁好了。只要是访问多个线程共享的字段的方法，就需要使用synchronized进行保护。

### 该以什么单位来保护

> 原子操作（不可分割的操作）

- 大部分基本类型（char、int等）、引用类型的赋值和引用是原子操作。
- long和double的赋值和引用是非原子操作。在线程间共享时，需要将其放入synchronized中操作，或者声明为volatile。

### 使用哪个锁保护

- synchronized方法：获取的是this的锁。
- synchronized代码块：需要明确指定获取哪个实例的锁。

- 实例不同，锁也就不一样。



## 参考资料

[《图解Java多线程设计模式》][1]

[1]: https://www.amazon.cn/dp/B074WVZK8B/ref=sr_1_1?s=books&ie=UTF8&qid=1548786260&sr=1-1&keywords=%E5%9B%BE%E8%A7%A3java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F