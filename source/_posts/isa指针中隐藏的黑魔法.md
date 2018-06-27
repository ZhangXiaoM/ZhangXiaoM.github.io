---
title: isa指针中隐藏的黑魔法
date: 2018-06-26 14:46:11
tags: 内存管理
---

#### 前言

在 objc 的对象系统中，`isa` 指针是一个非常重要的角色，每个对象都有一个 `isa` 指针，它的含义用中文可以解释为**是一个**，例如：

```objc
 id xiaoming = [Person new];
```

可以解释为，xiaoming is a person（小明是一个人）。我们知道在 32 位架构下，指针变量的存储空间是 4 字节，在 64 位架构下，指针变量的存储空间是 8 字节，从 iPhone 5s 往后的 iPhone 设备都是搭载了 64 位的处理器和指令集，在保持原有的运行时和对象系统的设计不变的情况下，指针变量的字节扩充会造成一部分多余的内存开销。runtime 对 `isa` 指针的设计很巧妙的规避了这部分的内存消耗，让 `isa` 不再仅仅是一个指针。当然，深入了解这些的前提是你要足够了解指针变量的存储域以及**位操作**。

#### 一、Tagged Pointer （标记指针）

在深入了解 `isa` 之前，我们需要了解一下运行时系统对64位的指针变量所做的一些优化。例如当我们初始化一个 `NSNumber` 对象时：

```objc
id foo = @(2);
```

此时，`foo` 变量指向堆内存的 `NSNumber` 对象，在64位架构下，`foo` 变量的存储空间为 8 字节，并且 `NSNumber`  对象中还包含该对象的引用计数，`weak` 引用表，数值 `2` 等等，同时，runtime 还要处理它的生命周期，这样就会给程序增加许多额外的内存开销和逻辑处理的时间开销。而我们的目的仅仅是为了以对象的形式存储和访问数值 `2`，一个 4 字节的整型数值而已。

当我们使用一些基础数据类型（例如：`int`，`uint`，`c string` 等等）的封装对象（`NSNumber`， `NSString`）时，Tagged Pointer 会将指针的其中几个位作为标志位，标记该指针是否为 Tagged Pointer，什么类型等等，另外的部分用来存储内容。当使用 Tagged Pointer优化后，上文中  `foo` 变量中存储的不再是一个堆内存地址，而是值 `2` 和几个标志位。这样就可以省去对象的内存开销和对象的持有释放等时间开销。`int` 和 `uint`  类型的变量最高只需要 4 字节的存储空间，`c string` 一个字符占 1 个字节，8 字节长度的指针去掉标志位，可以存储 7 个字符。因此当存储的内容低于指针的地址长度减去标志位长度时，这些内容即被存储到指针变量中，当内容超过指针地址长度减去标志位长度时，内容就会被存储到普通对象中。如下：

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        
        id foo = [NSString stringWithCString:"aaaaa" encoding:NSUTF8StringEncoding];
        id bar = [NSNumber numberWithInteger:2];
        id baz = @(0x111111111111111);
        
        NSLog(@"foo is %p", foo);
        NSLog(@"bar is %p", bar);
        NSLog(@"baz is %p", baz);
        
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

log 结果：

```objc
foo is 0xa000061616161615 // 字符 a 的 ASCII 码是97(0x61)
bar is 0xb000000000000023
baz is 0x600000034880
```

`foo` 和 `bar` 的最高四位和最低四位去掉之后正是实际存储的值。而 `baz` 的存储空间超过了 Tagged Pointer 所能分配的极限，因此被当做常规对象处理。

```c
#if TARGET_OS_OSX && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1
#endif

#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1ULL<<63)
#else
#   define _OBJC_TAG_MASK 1
#endif
```

runtime 中描述了在 64 位 Mac 中区分指针变量是标记指针还是普通指针的标志位为最低位，在其他系统中，标志位为最高位（`1ULL<<63`）。runtime 会根据标志位区分一个指针是标记指针还是普通指针，从而对指针做出不同的处理：

```c
// 内联函数，通过位操作判断指针是不是 tagged pointer
static inline bool 
_objc_isTaggedPointer(const void *ptr) 
{
    return ((intptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```

通过上面的函数可看出，runtime 通过使用[掩码](https://zh.wikipedia.org/wiki/%E6%8E%A9%E7%A0%81)判断指针是不是标记指针，如果是标记指针，会用相应的函数取出它存储的值：

```c
static inline uintptr_t
_objc_getTaggedPointerValue(const void *ptr) 
{
    // assert(_objc_isTaggedPointer(ptr));
    uintptr_t basicTag = ((uintptr_t)ptr >> _OBJC_TAG_INDEX_SHIFT) & _OBJC_TAG_INDEX_MASK;
    // 标志位在不同的情况下有区别
    if (basicTag == _OBJC_TAG_INDEX_MASK) {
        // 先左移 n 位，然后右移 n 位，保留中间位
        // 无符号长整型，逻辑移位后补 0
        return ((uintptr_t)ptr << _OBJC_TAG_EXT_PAYLOAD_LSHIFT) >> _OBJC_TAG_EXT_PAYLOAD_RSHIFT;
    } else {
        return ((uintptr_t)ptr << _OBJC_TAG_PAYLOAD_LSHIFT) >> _OBJC_TAG_PAYLOAD_RSHIFT;
    }
}
```

通过左移和右移去掉标志位，剩下的中间几位即为指针中存储的值。runtime 给出了几种可能会用 Tagged Pointer 优化的对象：

```c
// 从命名中就可以看出是哪种类型的对象
OBJC_TAG_NSAtom            = 0, 
OBJC_TAG_1                 = 1, 
OBJC_TAG_NSString          = 2, 
OBJC_TAG_NSNumber          = 3, 
OBJC_TAG_NSIndexPath       = 4, 
OBJC_TAG_NSManagedObjectID = 5, 
OBJC_TAG_NSDate            = 6, 
OBJC_TAG_RESERVED_7        = 7, 
```

runtime 会根据这些枚举值来判断一个 Tagged Pointer 是哪种类型的指针，以便于将取出来的值当做这种类型的对象来处理。

#### 二、isa

`isa` 指针也是指针，在 64 位架构下也需要 8 个字节的存储空间，而在原有的 32 位架构的设计中，`isa` 只需要 4 个字节的存储空间， 因此在保持原有的设计下，`isa` 指针同样会有 4 个字节的内存被浪费。就像 Tagged Pointer 一样，`isa` 也不再仅仅是一个指针，而是一个 `nonpointer`。

`isa` 指针的一些位仍然被编码为指向类对象的指针，但是被编码为指针的位不是全部的 64 位地址空间，iOS 和 Mac OS 都是对 `isa` 指针做了相同的处理。和标记指针一样，`isa` 也使用掩码的形式最大程度使用指针的全部地址空间。runtime 会用 `isa` 的其他额外的位来存储对象的其他数据，例如引用计数，是否被弱引用等等。

这样就可以避免用额外的数据结构存储每个对象的引用计数和弱引用情况，也可以减少对象 `-retain`，`-release`，`-alloc`，`-dealloc` 时通过查表修改和获取引用计数的时间。

那么 `isa` 的 64 位地址空间都用来存储什么数据了呢？记住，看源码，而不是百度或者google。

```c
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
    
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        // LSB
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
        // MSB
        // bits + RC_ONE is equivalent to extra_rc + 1
#       define RC_ONE   (1ULL<<45)
		// RC_HALF is the high bit of extra_rc (i.e. half of its range)
#       define RC_HALF  (1ULL<<18)
    };
}
```

可以看到 `isa` 被定义为一个联合体，其中 `Class` 类型的变量 `cls` 指向类对象，还有一个无名位域指定了 64 位地址空间的存储内容。从最低有效位到最高有效位依次表示：

- `nonpointer`：1 bit，置位（1）代表是 `nonpointer` ，复位（0）代表是原始的 `isa` 指针。

- `has_assoc`：1bit，对象是否被变量引用过。没有被引用的对象会很快被释放。

- `has_cxx_dtor`：1bit，对象有一个 c++ 或者 ARC 的析构函数，没有析构函数的对象会被很快释放。

- `shiftcls`：33 bit，这 33 位地址空间存储的就是真正的指向堆内存的地址。

- `magic`：6 bit，用来给调试器区分该对象是真正的对象还是未初始化过的垃圾内存。

- `weakly_referenced`：1 bit，对象是否被弱引用。

- `deallocating`：1 bit，对象正在被释放。

- `has_sidetable_rc`：1 bit，对象的引用计数太大，要用额外的数据结构存储。

- `extra_rc`：19 bit，如果对象的引用计数超过 1，就用这 19 位是用来存储额外的引用计数，最大可以表示的数字为 2^19 - 1。也就是当一个对象的引用计数低于 2 ^ 19 时就用这 19 位来存储，否则的话用额外的数据结构存储。例如对象的引用计数为 6，那么 extra_rc = 5。

这就是全部的 64 位地址空间的存储内容。宏定义 `ISA_MASK` 作为指向类对象的指针的掩码。`0x0000000ffffffff8ULL` 正是第 3 到第 35 位的掩码值，`ISA_MAGIC_MASK` 是 6 位 `magic` 的掩码，当 `isa &ISA_MAGIC_MASK == ISA_MAGIC_VALUE ` 时，表明该对象是被初始化过的对象。（我猜 `0x000001a000000001ULL` 的意义和命名一样，magic value...）

`RC_ONE` 表示 `extra_rc` 的最低位，因此 `bits + RC_ONE = extra_rc + 1`。`RC_HALF` 表示 `extra_rc` 范围的一半，此处为 2 ^ 19 / 2。

如何获取一个对象的引用计数呢？

```c
inline uintptr_t objc_object::rootRetainCount()
{
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    ClearExclusive(&isa.bits);
    if (bits.nonpointer) {
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}
```

objc_object 命名空间的内联函数，当 `isa` 指针是 Tagged Pointer 时，直接返回该指针的值，因为它不受 ARC 内存管理。当 `isa` 指针是 `nonpointer` 时，对象的引用计数为 1 + `extra_rc`，如果 `extra_rc` 的地址空间不足以存储对象的引用计数，则用 hash 表存储额外的引用计数，将三部分的结果求和即为对象的引用计数。

```c
uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable& table = SideTables()[this];

    size_t refcnt_result = 1;
    
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        // 右移是因为设置的时候做了左移操作，我也不知道为什么要左移
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
    }
    table.unlock();
    return refcnt_result;
}
```

对象的引用计数存储在一个以 `this` 指针为键，引用计数为值的 Map 中，该函数即为获取非 `nonpointer` 的对象的引用计数。因此，如果 `isa` 指针是 `nonpointer`，那么就节省了这个存储引用计数的 Map，同时也减少了管理对象生命周期时查表的时间。

#### 总结

小小的 `isa` 指针就做了这么多不可思议的事情，可见 apple 对空间和时间的优化多么登峰造极，当然知道这些并不会提高我们的业务能力，但是我们有必要知道 runtime 动态库为整个 objc 的对象系统提供了多么大的支撑，这也是 objc 的核心能力。