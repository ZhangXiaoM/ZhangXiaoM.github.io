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

通过上面的函数判断指针是不是标记指针，如果是标记指针，会用相应的函数取出它存储的值：

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

