---
title: 「JVM」01 类加载机制
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

## 类加载生命周期

按如下顺序开始，而非按如下顺序执行：
加载、连接（验证、准备、解析）、初始化。

