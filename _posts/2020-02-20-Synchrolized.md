---
title: synchronized和volatile
categories: [concurrent]
comments: true
---

##### synchronized关键字
synchronized 是 java 内置的同步锁实现，一个关键字实现对共享资源的锁定。synchronized 有 3 种使用场景，场景不同，加锁对象也不同：
* 普通的同步方法：锁是当前实例对象
* 静态同步方法：锁是当前class对象
* 同步方法块：锁是synchronized括号里的对象