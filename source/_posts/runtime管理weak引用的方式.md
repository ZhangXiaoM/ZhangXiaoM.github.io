---
title: runtime管理weak引用的方式
date: 2018-07-13 16:47:32
tags: 内存管理
---

### 前言

提及 weak 引用，大多数人都知道在什么时候要用它，如果不知道的话：[ARC内存管理以及循环引用](https://zhangxiaom.github.io/2018/01/02/ARC%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%BB%A5%E5%8F%8A%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8/)，其实对于手动管理堆内存来说，比如 C 语言，并不存在所谓的强引用和弱引用，ARC 这种自动引用计数管理内存的方式，导致了两个对象循环引用，从而产生内存泄漏。循环引用就像是双向链表的两个结点的 `next` 指针互相指向，当我们用 C 语言实现循环链表的时候，即使没有 weak，也能很好的管理每个结点的内存。因此，weak 是 ARC 的产物，它需要提供这种机制来避免循环引用造成的内存泄漏。

### runtime 管理 weak 引用

runtime 会管理所有元类、类对象、对象的生命周期以及他们的引用计数，这是 objc 的运行时特性，对象的创建和内存分配和销毁都是 runtime 动态库处理的，因此关于 weak 引用，也是 runtime 来管理的。

我之前看过大多数博客，都会说，runtime 会维护一张 weak 哈希表，以对象的地址为键，以 `__weak` 修饰的变量为值存入哈希表中，当对象被释放了之后，runtime 就会查表将变量设置为 `nil`。比如：

```objc
// 源代码
id obj = [NSObject new];
__weak weakRefer = obj;

// runtime 处理为
weak_table_add_item(&obj, weakRefer);
```

一开始我也对此深信不疑，但是有个问题一直无法理解：哈希表的特性是同一个键只有一个哈希码，也就是这种方式就会造成对象的弱引用能且只能被存储一个，那么假如该对象有两个弱引用呢？例如：

```objc
// 源代码
id obj = [NSObject new];
__weak weakRefer1 = obj;
__weak weakRefer2 = obj;

// runtime 处理为
weak_table_add_item(&obj, weakRefer1);
weak_table_add_item(&obj, weakRefer2);
```

哈希表会将 `weakRefer1` 覆盖，显然这不是一种正确的方式，大多数博客都是互相抄，那不如去看看源码吧。

### 真相到底如何？



