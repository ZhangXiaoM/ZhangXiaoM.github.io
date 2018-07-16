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

我们可以找到 runtime 源码中对 weak 的处理：[objc-weak.h](https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-weak.h) 和 [objc-weak.mm](https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-weak.mm)。在看源码之前，先了解一下[哈希表](https://zhangxiaom.github.io/2018/03/23/%E5%93%88%E5%B8%8C%E8%A1%A8/)吧。

```c
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```

确实是一个全局哈希表，保存了所有对象的弱引用，不过保存的方式是，以对象为键，以 `weak_entry_t` 结构体为值，而不是之前提到的 weak 变量。`mask` 和 `max_hash_displacement` 是用来计算哈希码的。

```c
/**
 * The internal structure stored in the weak references table. 
 * It maintains and stores
 * a hash set of weak references pointing to an object.
 * If out_of_line_ness != REFERRERS_OUT_OF_LINE then the set
 * is instead a small inline array.
 */
struct weak_entry_t {
    objc_object* referent;
    union {
        struct {
            // typedef id weak_referrer_t
            weak_referrer_t *referrers; // 保存弱引用地址的 hash set
            uintptr_t        out_of_line_ness : 2; // 标记是否需要用 set
            uintptr_t        num_refs : PTR_MINUS_2; // 弱引用的数量
            uintptr_t        mask; // 计算哈希码
            uintptr_t        max_hash_displacement; // 计算哈希码
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT]; // 轻量级内联数组
        };
    };
}
```

其实 `weak_table_t` 实现了一个简易的对象类型的哈希表，`weak_entry_t` 里面保存了哈希表中某一条目的 key 和 value，至于为什么保存 key，是为了发生哈希冲突时，找到这个值。当该对象的弱引用数量低于4个时，`weak_entry_t` 会用一个轻量级的数组存储它们的**地址**，当数量大于4 时，`weak_entry_t` 会用哈希 set（哈希 set 是一个只有 keys 的哈希表，相当于 `map.allKeys()`，为了避免存储两个相同的地址）存储它们的**地址**。

[objc-weak.h](https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-weak.h)  提供了几个接口方法，分别是向 `weak_table_t` 中添加、删除、清空某一个对象的数据：

```c
/// Adds an (object, weak pointer) pair to the weak table.
/// 向 weak table 中添加一个 <object、weak pointer> 的键值对
/// 如果 object 已经存在，则向 weak_entry 中插入 *referrer,否则创建一个 weak_entry,将 *referrer 插入
/// 当哈希表的装填因子满足一定条件时，扩大哈希表长度，重新哈希。
id weak_register_no_lock(weak_table_t *weak_table, id referent, 
                         id *referrer, bool crashIfDeallocating);

/// Removes an (object, weak pointer) pair from the weak table.
/// 从 weak_table 中删除一个键值对
void weak_unregister_no_lock(weak_table_t *weak_table, id referent, id *referrer);

#if DEBUG
/// Returns true if an object is weakly referenced somewhere.
/// 如果一个对象被弱引用过的话，则返回 true
bool weak_is_registered_no_lock(weak_table_t *weak_table, id referent);
#endif

/// Called on object destruction. Sets all remaining weak pointers to nil.
/// 在对象的析构方法中调用，设置所有的弱引用指针为 nil
void weak_clear_no_lock(weak_table_t *weak_table, id referent);
```

这几个方法都是常规的操作哈希表的方法，计算哈希码，插入或者删除合适的数据，重新哈希等等。

我们知道，weak 引用的特性是，当对象被释放后，它就会被设置为 `nil`，避免造成野指针。

**野指针**：野指针的概念是，当某个指针指向的内存被回收后，该指针变量保存的扔然是那块内存的地址，就会造成访问错误的内存的 crash，通常的崩溃信息是 `BAD_ACCESS`。例如：

```c
int *a = (int *)malloc(sizeof(int *));
free(a);
printf("%p\n", a); // BAD_ACCESS crash
```

崩溃的前提是，指针指向的内存被回收，因为 `free()` 函数释放的内存不会被立刻回收，所以这段代码可能不会直接崩溃，一旦 `a` 指向的内存被回收，就肯定会崩溃。因此，我们需要手动将指针设置为 `NULL`：

```c
int *a = (int *)malloc(sizeof(int *));
free(a);
a = NULL;
printf("%p\n", a); // OK, note: 被设置为 NULL 的指针，不要再用*访问它
```

weak 引用的这种特性配合 ARC 就大大减少了野指针出现的概率。我们来看 runtime  的实现：

```c++
/** 
 * Called by dealloc; nils out all weak pointers that point to the 
 * provided object so that they can no longer be used.
 * 
 * @param weak_table 
 * @param referent The object being deallocated. 
 */
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    // 将 id 类型转换为 objc 类型
    objc_object *referent = (objc_object *)referent_id; 
	// 通过 object 找到 table 中存的值
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    // assert(entry)
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    // 判断是否需要 set
    if (entry->out_of_line()) {
        // 如果是，将 referrers 指向 set
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } else {
        // 如果不是，将 referrers 指向内联数组
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    // 迭代集合，依次将集合内的指针设置为 nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            } else if (*referrer) {
                // 如果 *referrer != nil && *referrer != referent，断点并报错
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    // 最后将 entry 从 weak_table 中删除
    weak_entry_remove(weak_table, entry);
}
```

代码的具体实现请看注释。

**NOTE：**`referent` 指向 `object`，`referrer` 指向 `referent`，也就是 `referrer` 中保存的是 weak 变量的地址。

这个方法的作用是清空某个对象的所有弱引用，并将它们设置为 `nil`，它会在对象的析构方法（`-dealloc`）中调用，runtime 在管理对象生命周期的过程中会帮助我们统一调用。

在对 `weak_table` 进行删除、清空等操作之前，runtime 会根据 `isa` 指针的某一位判断这个对象是不是有弱引用，[isa指针中隐藏的黑魔法](https://zhangxiaom.github.io/2018/06/26/isa%E6%8C%87%E9%92%88%E4%B8%AD%E9%9A%90%E8%97%8F%E7%9A%84%E9%BB%91%E9%AD%94%E6%B3%95/)。



### 未完待续...