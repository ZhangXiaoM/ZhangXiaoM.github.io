---
title: Block
date: 2018-05-31 15:19:11
tags:
---

作为 iOS 开发者，不管是初级还是高级，都应该知道并且熟练应用 OC 中的 block 语法，并且知道 block 会造成循环引用，但是大多数开发者并不知道循环引用产生的原理和如何正确的避免，而不是遇到 block 就用 weak 引用这种简单粗暴的方式解决问题。下面我将从 block 是什么？ block 如何捕获变量？ block 循环引用的原理等几个方面，分析 block 的实质。

### 一、Block 是什么？

runtime 是大家耳熟能详的东西，从 runtime 中可以知道，OC 中所有的类都是用 C 或 C++ 结构体实现的，所有的方法都是通过动态绑定的方式在运行时调用的，具体内容可参见：[从runtime源码解析对象发送消息的动态性](https://zhangxiaom.github.io/2018/01/20/%E4%BB%8Eruntime%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E5%AF%B9%E8%B1%A1%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%E7%9A%84%E5%8A%A8%E6%80%81%E6%80%A7/)。其实 block 的实现方式和 OC 类的实现方式是相同的，只不过 block 的实现不会像 OC 对象一样依赖于运行时动态库，它会在编译时被分配到栈内存或者分配到全局和静态数据区，但是运行时仍然会在某些情况下将分配到栈上的 block 拷贝到堆内存，以便于解决 block 被栈内存销毁的问题。以下内容将用 *Block* 关键字描述 block ”类“（即实现 block 的结构体），用 *block* 关键字描述 blcok “对象”。

当我们在 main 方法中创建一个 block，并且赋值给某个变量时，通过 `clang` 将 OC 代码编译成 C 代码，可以找到 Block ”类“中的内容为：

```c
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```

一眼就看到了熟悉的 `isa` 指针，这是 OC 对象中独有的东西，而且从这个结构体也可以看出，block 并不是一个简简单单的函数指针，而是被包装为 OC 类的 Block 的实例。只不过此时的 block 被分配到栈内存。Block 中包含了一个 `isa` 指针，指向创建它的类，一个标志位 `Flags`，一个保留字段 `Reserved`，当然标志位和保留字段是 OC 源码中的一贯作风，它们没有特别明确的使用场景。`FuncPtr`  即是 Block 作为匿名函数的真相，它指向的函数即为调用 `block()` 时实际调用的函数。这是所有 Block 的通用实现，即所有 block 对象都会有的东西，但是当 block 捕获外部变量，或者被拷贝到堆内存的时候，它还需要一些其他实例和方法完成拷贝和保留被捕获的外部变量。

所以 Block 被实现为：

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

可以看到，此时 Block 是由 `struct __block_impl impl` 和 `struct __main_block_desc_0* Desc` 两个成员变量和一个初始化方法组成的，当 block 捕获不同类型的外部变量时，`struct __main_block_impl_0`  中会多一个成员用来存储捕获的变量。那么为什么要这样写？而不是将 `__block_impl` 在 `__main_block_impl_0`  中展开？我们知道 block 会捕获不同类型的外部变量，有的是引用类型，有的是基础类型，有的是只读的，有的是可变的，所以，假如我们使用了 n 个捕获不同类型的 block，那么就会生成 n 个不同的 `__main_block_impl_n`，`__block_impl `的使用可以减少代码的重复性和耦合性。

再来看 `Desc` ，这是一个指向 `__main_block_desc_0` 结构体的指针。它的内容为：

```c
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

emm，一个保留字段，一个保存 block `size `  的变量`Block_size` ，之前说过， block 会在某些情况下被拷贝到堆内存，假如 block 满足一些被拷贝到堆内存的条件时，`__main_block_desc_0` 就会增加 `copy` 和 `dipose` 方法，用来将 block  拷贝到堆内存和释放 block 占用的堆内存。并且这段代码实例化了一个  `struct __main_block_desc_0` 类型的静态全局变量 `__main_block_desc_0_DATA` 。当然每一个 ` __main_block_impl_n`  都会有一个`__main_block_desc_n_DATA ` 用来保存 block 的 size 和实现 `copy` 方法。

当我们创建一个 block 时，

```objective-c
typedef void (^Block)(void);
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        Block blk = ^{};
        blk();
    }
    return 0;
}
```

它即被 `clang` 翻译为：

```c
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        Block blk = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
        ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    }
    return 0;
}
```

先用指向 `__main_block_func_0` 函数的函数指针和  `__main_block_desc_0_DATA`  全局变量初始化 `__main_block_impl_0`，此即为 block ”对象“。当调用 `blk()` 时，block 会通过他的成员变量 `FuncPtr` 实现，参数即为它自身和我们调用 block 时传入的实参。

### 二、block 捕获外部变量

理解 block 捕获变量的前提是，我们应该掌握虚拟内存和内存分区的概念，即堆、栈、程序代码和数据区、共享库的代码和数据区、内核虚拟存储区，以及每个分区的工作方式。并且了解全局变量，静态全局变量，静态变量，自动变量等的概念以及它们的存储域和销毁时机。并且了解 OC 的内存管理方式，ARC 和 MRC。还有值类型变量和引用类型变量的存储方式的区别和联系。这些内

#### 1、捕获自动变量

如下代码，block 捕获外部变量 `x`：

```c
int x = 0;
Block blk0 = ^(void){
	NSLog(@"%d", x);
};
blk0();
```

此时，生成的 Block 实现结构体为：

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int x;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _x, int flags=0) : x(_x) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

此时，Block 中多了一个成员变量 `x`，并且初始化方法中多了一个参数 `_x`，默认赋值为成员变量 `x`。因此，当 block 捕获自动变量的时候，会生成一个相同类型的成员变量，用来存储该自动变量的 copy。为什么是 copy，而不是该变量？因为当 block 被赋值为实例变量时，该 block 的调用时机可能出了该自动变量的作用域，因此，需要 copy 一份，而不是直接访问。并且此时我们仅仅能访问 `x`，如果尝试改写它，编译器会给你发个 error，因为 `x`  仅仅是 block 的成员变量和自动变量的 copy，而不是自动变量本身，所以我们没有权利修改它，即使修改了，也是修改的 block 成员变量，而不会同步到外部变量，因为它是值类型。

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int x = __cself->x; // bound by copy

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_qd_7zbm76j916n2_dhjbm6nsm480000gn_T_main_19f6cf_mi_0, x);
        }

```

此时，block 中的 `FuncPtr` 指针指向的函数会通过指向它自身的指针 `cself` 访问它的成员变量 `x`。

当自动变量为类类型时，例如：

```objective-c
Block blk0 = ^(void){
	[foo addObject:@""];
};
blk0();
```

此时的 Block 结构体为：

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  NSMutableArray *foo;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, NSMutableArray *_foo, int flags=0) : foo(_foo) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

看样子是和基础类型一样的。