## 线程基础知识复习

start 一个线程

Java 多线程相关概念

- 进程
- 线程

用户线程和守护线程

## CompletableFuture

## 锁

乐观锁、悲观锁

案例演示，锁是什么

公平锁、非公平锁

可重入锁（递归锁）

死锁及排查

写锁（独占锁）和读锁（共享锁）

## LockSupport 与线程中断

线程中断机制

LockSupport 是什么

线程等待唤醒机制

## JMM

计算机硬件存储体系

JMM

JMM 规范下的三大特性

JMM 规范下，多线程对线程变量的读写过程

JMM 规范下，多线程先行发生原则只 happens-before

## volatile 与 Java 内存模型

内存屏障

volatile 特性

正确使用 vloatile

总结

## CAS

未使用 CAS 之前

CAS 是什么

底层原理（Unsafe 的理解）

原子引用

自旋锁

CAS 缺点

## 线程池

原理

线程池几个参数的意义

流程分析

源码解析

业务中的实践

总结

## 原子操作类

是什么

分类

- 基本类型原子类
- 数组类型原子类
- 引用类型原子类
- 对象的属性修改原子类

原子操作类远离深度解析

## ThreadLocal

简介

阿里规范

源码分析

内存泄露问题

总结

## Java 对象内存布局和对象头

## Synchronized 与锁升级

Synchronized 锁

Synchronized 的性能变化

Synchronized 锁种类及升级步骤

JIT 编译器读锁的优化

## AQS（AbstractQueuedSynchronized）

AQS 是什么

AQS 是JUC 内容中最重要的基石

AQS 能做什么

初步学习

解读 AQS

## ReentrantLock及相关锁

路线

ReentrantReadWriteLock

有没有比读写锁更快的锁

StampedLock

## 总结