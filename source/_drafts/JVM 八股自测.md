---
title: JVM 八股自测
date: 2023-02-28
tags: [jvm, 八股]
categories: JavaSE
toc: true
hide: false
sticky: 0
---

{% note blue 'fas fa-bullhorn' %}
文章开篇内容简介
{% endnote %}

---

## 类加载机制

1. 类加载的生命周期
2. 类加载的层次
3. Class.forName() 和 ClassLoader.loadClass() 区别
4. JVM 有哪些类加载机制

## JVM 内存结构

1. JVM 整体的内存结构包含哪些
	- 哪些线程私有，哪些线程共享
2. 什么是程序计数器
	- 它为什么线程私有
3. 什么是虚拟机栈
	- 有什么特点
	- 该区域有哪些异常
	- 栈帧的内部结构
4. 什么是本地方法栈
5. 什么是方法区
6. 永久代和元空间的差异