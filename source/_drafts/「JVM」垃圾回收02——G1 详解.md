
---
title: 「JVM」垃圾回收02——G1 详解
date: 2023-02-28
tags: jvm
categories: javase
toc: true
hide: true
sticky: 0
---

{% note blue 'fas fa-bullhorn' %}
内容包含：
1. G1 的内存模型
2. G1 的活动周期
{% endnote %}

---

## 概述

G1 GC 与 CMS GC 的区别：
- 回收后空间状态不同：G1 GC 是可压缩的，回收后得到的空间是连续的，避免了 CMS GC 会产生的空间碎片问题，这也意味着可以不必采用空闲链表的内存分配方式，可以直接采用bump-the-pointer 的方式
- 内存模型不同：G1 将内存划分为一个个固定大小的 region，每个 region 都可以是年轻代或老年代，回收也是以 region 为基本单位的
- G1 可以软实时：软实时是指用户可以规定垃圾回收时间的限时，G1 会**尽可能**地在这个时限下完成垃圾回收

## G1 的内存模型

### 分区概念

*待续...*


