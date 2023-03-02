---
title: Java 并发_理论基础篇
date: 2023-03-01
tags: 并发
categories: JavaSE
toc: true
hide: false
sticky: 0
---

{% note blue 'fas fa-bullhorn' %}
内容包括：
1. 多线程要解决的问题
2. 什么是线程不安全
3. 线程不安全的本质
4. Java 如何解决并发问题：JMM
5. 线程安全实现思路：互斥同步、非阻塞同步、无同步
{% endnote %}

---

# 为什么需要多线程

CPU、内存、I/O 设备 的速度差异很大，为合理利用 CPU 性能，平衡三者速度差异：
- `CPU 增加缓存`：CPU 增加缓存平衡与内存之间的速度差异——导致可见性问题
- `操作系统增加进程、线程`：以分时复用 CPU 来平衡 CPU 与 I/O 设备之间的速度差异——导致原子性问题
- `优化程序执行顺序`：编译程序优化指令的执行顺序，更合理地利用缓存——导致有序性问题

![](../pictures/Pasted%20image%2020230301094603.png)

# 并发问题根源：并发三要素

## 可见性：CPU 缓存引起

可见性：一个线程对共享变量的修改，另一个线程能够立即看到。

## 原子性：分时复用引起

原子性：一个或多个操作要么全部执行且不会被中断，要么不执行。

## 有序性：重排序引起

有序性：程序的执行顺序按代码的先后顺序执行。

执行程序时，为提高性能，编译器和处理器常会对指令进行重排序，有三种：
- `编译器优化的重排序`
- `指令级并行的重排序`
- `内存系统的重排序`

![](../pictures/Pasted%20image%2020230301100043.png)


# Java 如何解决并发问题：JMM

[详见 Java 内存模型]

 1. 从 JMM 核心理论的角度：

JMM 规范了 JVM 如何提供按需禁用缓存和编译优化的方法：
- `volatile`、`synchronized`、`final`
- `Happens-before 原则`

2. 从并发三要素的角度：

- 原子性

Java 中，JMM 只保证了对基本的读取和赋值操作时原子操作，如果要实现大范围的原子性，需要通过 `synchronized` 和 `Lock` 来实现。

- 可见性

使用 `volatile` 关键字保证可见性，共享变量被 `volatile` 修饰时，它会将保存修改的值立即写入到主存，这样其他线程读取时，会去从主存中读取最新值。

`synchronized` 和 `Lock` 也可以实现可见性，在锁释放之前，会将共享变量的值更新到主存中。

- 有序性

通过 `volatile` 关键字保证 “一定的有序性”，也可以通过 `synchronized` 和 `Lock` 保证有序性。

JMM 则是通过 `Happens-before 原则` 来保证有序性。

## 关键字：volatile、synchronized、final

- [详见 volatile 详解]
- [详见 synchronized 详解]
- [详见 final 详解]

## Happens-Before 原则

理论：
- （本质）如果一个操作 happens-before 另一个操作，那么第一个操作的执行结果对第二个操作可见，并且第一个操作的执行顺序在第二个操作之前
- 如果重排序后的执行结果和在该原则下的执行结果一样，那么允许这种重排序

# 线程安全并不绝对

线程安全问题不是一个非真即假的问题，将共享数据的线程安全程度由强到弱分为如下五类：
- 不可变
- 绝对线程安全
- 相对线程安全
- 线程兼容
- 线程对立

### 不可变

对象不可变，也就不会发生线程安全问题，不可变类型有如下几种：
- 被 final 修饰的基本数据类型
- String
- 枚举类型
- Number 部分子类，如：Integer 等包装类、BigInteger 等大数据类型

### 绝对线程安全

不管运行环境如何，调用者都不需要任何额外的同步操作。

### 相对线程安全

对单个对象的单独操作时线程安全的，调用时不需要额外的保障措施，但对一些特定顺序的连续使用，还是需要进行额外的同步操作。

Java 语言中，大部分都属于这种类型，如：Vector、HashTable、Collections 的 synchronizedCollection() 方法包装的集合等。

# 线程安全的实现方法

## 互斥同步：synchronized & ReetrantLock

- [详见 synchronized 详解] 
- [详见 ReetrantLock 详解]

## 非阻塞同步：CAS & AtomicInteger 原子类

[详见 JUC 原子类：CAS、Unsafe和原子类详解]

## 无同步方案

### 栈封闭

[详见 JUC 线程池部分]

### 线程本地存储

使用 `java.lang.ThreadLocal` 类实现线程本地存储功能。

每个 `ThreadLocal` 都有一个 `ThreadLocal.ThreadLocalMap` 对象，`Thread` 类中定义了 `ThreadLocal.ThreadLocalMap` 的成员。

调用一个 `ThreadLocal` 的 `set(T value)` 方法时：
```java
public void set(T value) { 
	Thread t = Thread.currentThread(); 
	ThreadLocalMap map = getMap(t); 
	if (map != null) 
		map.set(this, value);
	else 
		createMap(t, value); 
}
```

`get()` 方法同理：
```java
public T get() { 
	Thread t = Thread.currentThread(); 
	ThreadLocalMap map = getMap(t); 
	if (map != null) { 
		ThreadLocalMap.Entry e = map.getEntry(this); 
		if (e != null) { 
			@SuppressWarnings("unchecked") 
			T result = (T)e.value; 
			return result; 
		} 
	} 
	return setInitialValue(); 
}

```

`ThreadLocal` 理论上并不是为了解决并发问题的，因为根本就不存在多线程竞争。

[详见 ThreadLocal 详解]

### 可重入代码（Reentrant Code）

即，纯代码。

