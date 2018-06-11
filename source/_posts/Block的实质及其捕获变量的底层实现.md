---
title: Block的实质及其捕获变量的底层实现
date: 2018-05-31 15:19:11
tags:
---

作为 iOS 开发者，不管是初级还是高级，都应该知道并且熟练应用 OC 中的 block 语法。大多数开发者都知道 block 会造成循环引用，但是很少有人会关心 block 造成循环引用的原理和如何正确的避免，大多数人遇到 block 就用 weak 引用这种简单粗暴的方式来解决循环引用的问题，这种做法并不可取。下面我将从 block 是什么？ block 如何捕获变量？ block 循环引用的原理等几个方面，分析 block 的实质。

### 一、Block 是什么？

runtime 是大家耳熟能详的东西，从 runtime 中可以知道，OC 中所有的类都是用 C 或 C++ 结构体实现的，所有的方法都是通过动态绑定的方式在运行时调用的，具体内容可参见：[从runtime源码解析对象发送消息的动态性](https://zhangxiaom.github.io/2018/01/20/%E4%BB%8Eruntime%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E5%AF%B9%E8%B1%A1%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%E7%9A%84%E5%8A%A8%E6%80%81%E6%80%A7/)。其实 block 的实现方式和 OC 类的实现方式是相同的，只不过 block 的实现不会像 OC 对象一样依赖于运行时动态库，它会在编译时被分配到栈内存或者分配到全局和静态数据区，但是运行时仍然会在某些情况下将分配到栈上的 block 拷贝到堆内存，以便于解决 block 被栈内存销毁的问题。以下内容将用 *Block* 关键字描述 block ”类“（即实现 block 的结构体），用 *block* 关键字描述 block ”对象“。

当我们在 main 方法中创建一个 block，并且赋值给某个变量时，通过 `clang` 将 OC 代码编译成 C 代码，可以找到 Block ”类“中的内容为：

```c
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```

一眼就看到了熟悉的 `isa` 指针，这是 OC 对象中独有的东西，而且从这个结构体也可以看出，block 并不是一个简简单单的函数指针，而是被包装为 OC 类的 Block 的实例。只不过此时实例化的 block 被分配到栈内存。Block 中包含了一个 `isa` 指针，指向保存它所有信息的*类对象*，一个标志位 `Flags`，一个保留字段 `Reserved`，当然标志位和保留字段是 OC 源码中的一贯作风，它们没有特别明确的使用场景。`FuncPtr`  即是 Block 作为匿名函数的真相，它指向的函数即为调用 `block()` 时实际调用的函数。这是 Block 的通用实现，即所有 block 对象都会有的东西，但是当 block 捕获外部变量，或者被拷贝到堆内存的时候，它还需要一些其他实例和方法完成拷贝和保留被捕获的外部变量。

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

emm，一个保留字段，一个保存 block 大小的变量`Block_size` ，之前说过 block 会在某些情况下被拷贝到堆内存，假如 block 满足一些被拷贝到堆内存的条件时，它捕获的内容也将被拷贝到堆内存， `__main_block_desc_0` 就会增加 `copy` 和 `dipose` 方法，前者用来将 block  捕获的内容拷贝到堆内存，后者用来释放捕获的内容。并且这段代码实例化了一个  `struct __main_block_desc_0` 类型的静态全局变量 `__main_block_desc_0_DATA` 。当然每一个 ` __main_block_impl_n`  都会有一个`__main_block_desc_n_DATA ` 用来保存 block 的 size 和实现 `copy` 方法。

当我们创建一个 block 时，

```objc
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

### 二、block 存储位置

block 在内存中有三个存储域，它的 `isa` 指针会描述它的存储域，上文中可看到 block 的存储在栈内存中，当我们声明一个全局 block 并且实现它的时候，block 就会被存储在静态和全局区。

```objc
void (^Block)(void) = ^(void){}; 
```

```c
impl0.isa = &_NSConcreteGlobalBlock;
```

当 block 满足一定的条件时，分配在栈上的 block 会在运行时被拷贝到堆内存，并且 ARC 会帮我们管理内存。下面几种情况 block 会被拷贝到堆内存：

- 捕获对象时；
- 作为函数返回值时；
- 捕获 `__block` 修饰的变量时；
- 被赋值给 `__strong` 修饰的成员变量时；
- 主动调用 `- copy` 方法时。

等等。

### 三、block 捕获外部变量

理解 block 捕获变量的前提是，我们应该掌握虚拟内存和内存分区的概念，即堆、栈、程序代码和数据区、共享库的代码和数据区、内核虚拟存储区，以及每个分区的工作方式。并且了解全局变量，静态全局变量，静态变量，自动变量等的概念以及它们的存储域和销毁时机。并且了解 OC 的内存管理方式，ARC 和 MRC。还有值类型变量和引用类型变量的存储方式的区别和联系。

#### 1、捕获自动变量

如下代码，block 捕获外部变量 `x`：

```objc
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

此时，Block 中多了一个成员变量 `x`，并且初始化方法中多了一个参数 `_x`，并且赋值给成员变量 `x`。因此，当 block 捕获自动变量的时候，会生成一个相同类型的成员变量，用来存储该自动变量的 copy。为什么是 copy，而不是该变量？因为当 block 被赋值为实例变量时，该 block 的调用时机可能出了该自动变量的作用域，那么，该自动变量就会因为被弹出栈而销毁，因此需要 copy 一份，而不是直接访问。并且此时我们仅仅能访问 `x`，如果尝试改写它，编译器会给你发个 error，因为 `x`  仅仅是 block 的成员变量和自动变量的 copy，而不是自动变量本身，所以我们没有权利修改它，即使修改了，也是修改的 block 成员变量，而不会同步到外部变量，因为它是值类型。

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int x = __cself->x; // bound by copy
    NSLog((NSString*)&__NSConstantStringImpl__var_folders_qd_7zbm76j916n2_dhjbm6nsm480000gn_T_main_19f6cf_mi_0, x);
}

```

此时，block 中的 `FuncPtr` 指针指向的函数会通过指向它自身的指针 `cself` 访问它的成员变量 `x`。

#### 2、捕获对象

关于 block 的循环引用是我们在工作中老生常谈的话题，有一句话大家都不会陌生：因为某个对象强引用了 block，block 又强引用了某个对象，所以会造成循环引用，那么 block 是如何强引用对象的呢？这就要从 block 捕获对象说起了。

例如：

```objc
NSMutableArray *foo = [NSMutableArray new];
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

Block 结构体中也多了一个变量 `foo`，就像上文 **1** 一样，Block 中也会多一个变量用来保存它捕获的内容。但是不同的是，对象是引用类型，所以 Block 的中的变量是对指向可变数组的变量的 copy，而不是对对象的 copy，它是 `foo` 的别名，和 `foo` 指向同一块堆内存。用 C 语言的语法可表述为：Block 中的成员变量 `foo` 是对指针 `foo` 的拷贝，而不是对对象 `*foo` 的拷贝。

此时， `__main_block_desc_0` 结构体为：

```c
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
```

可以看到多了两个成员 `copy` 和 `dispose`，说明当 block 捕获对象时，block 会被拷贝到堆内存。

这两个函数指针的定义为：

```c
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    _Block_object_assign((void*)&dst->foo, (void*)src->foo, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
```

```c
static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    _Block_object_dispose((void*)src->foo, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
```

从代码可看出，它们的作用是拷贝 block 捕获的变量和废弃 block 捕获的变量。

下面的代码可以验证 block 此时被拷贝到堆内存。

```objc
NSMutableArray *foo = [NSMutableArray new];
NSLog(@"%ld", CFGetRetainCount((__bridge CFTypeRef)(foo)));
void (^blk0)(void) = ^(void){
    NSLog(@"%ld", CFGetRetainCount((__bridge CFTypeRef)(foo)));
};
blk0();
NSLog(@"%@",blk0);
```

log 结果为：

```objc
2018-06-04 16:44:44.581703+0800 XXX[12276:1661853] 1
2018-06-04 16:44:44.582176+0800 XXX[12276:1661853] 3
2018-06-04 16:44:44.582949+0800 XXX[12276:1661853] <__NSMallocBlock__: 0x100745a50>
```

可以看出，在被 block 捕获之前对象 `foo`  的引用计数为 1，被捕获之后，引用计数增加到 3，并且 log 结果可看到 block 被拷贝到堆内存（*NSMallocBlock*）中，此时我们可以猜测，对象 `foo` 分别被栈上的 block 强引用一次、堆上的 block 强引用一次，变量 `foo` 强引用一次，因此，在被 block 捕获之后，它的引用计数为 3。此时内存布局为：

![](https://upload-images.jianshu.io/upload_images/5314152-0862ec404f661233.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面证明我们的猜测，如下代码：

```objc
NSMutableArray *foo = [NSMutableArray new];
void (^blk0)(void);
NSLog(@"%ld", CFGetRetainCount((__bridge CFTypeRef)(foo)));
{
    blk0 = ^(void){
        NSLog(@"%ld", CFGetRetainCount((__bridge CFTypeRef)(foo)));
    };
    blk0();
}

NSLog(@"%ld", CFGetRetainCount((__bridge CFTypeRef)(foo)));
NSLog(@"%@",blk0);
```

log 的结果为：

```objc
2018-06-04 16:58:09.573400+0800 XXX[13849:1684686] 1
2018-06-04 16:58:09.576588+0800 XXX[13849:1684686] 3
2018-06-04 16:58:09.576820+0800 XXX[13849:1684686] 2
2018-06-04 16:59:57.620217+0800 XXX[14913:1696697] <__NSMallocBlock__: 0x100616990>
```

从结果来看，此时的 block 依然在堆上， `foo` 变量仍然强引用对象，但是当出了中间的大括号作用域，`foo` 对象的引用计数变为 2，这说明，分配在栈上的 block 在出了作用域之后被弹出栈，同时栈内存中拷贝的 `foo` 变量也被废弃（dispose），所以此时栈上的 block 不再强引用对象 `foo`，所以它的引用计数变为 2。这就是 block 捕获对象的真相。我们可以得出的结论是，被捕获的对象会随着 block 的拷贝而被拷贝到堆内存，随着 block 的销毁而销毁，因此即使 block 不像 OC 对象一样遵循 ARC 的内存管理方式，OC 也会帮我们管理它和它捕获的对象的内存。

#### 3、捕获全局和静态变量

注意：全局变量并不是类中的全局变量，类中的全局变量只是属于该类的成员变量，它会在运行时被保存到该对象成员变量列表中，对象是存储在堆内存中的，因此类中的全局变量也是存储在堆内存中的。此处说的全局变量是分配在*全局和静态变量区*的全局变量，又分为已初始化的全局和静态数据区和未初始化的全局和静态数据区，此处不再多余赘述。

如下代码：

```objc
int _foo = 3;
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        void (^blk)(void) = ^(){
            NSLog(@"%d", _foo);
        };
        blk();
    }
    return 0;
}
```

此时，block 不会对全局变量和静态全局变量作任何多余的拷贝，而是在函数中直接访问全局和静态全局变量。因为，静态和全局数据区存储的内容在整个进程的生命周期内都有效，block 不用担心作用域和野指针等等内存问题。

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_qd_7zbm76j916n2_dhjbm6nsm480000gn_T_main_d97d1e_mi_0, _foo);
}
```

block 的实现函数也是直接访问了全局变量 `_foo` 。

当捕获静态自动变量时：

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        static int foo = 3;
        void (^blk)(void) = ^(){
            NSLog(@"%d", foo);
        };
        blk();
    }
    return 0;
}

```

静态自动变量和全局变量不同的是：它有作用域的概念，当出了当前作用域的时候，它就不能再被访问，但是它仍然被存储在静态和全局数据区，随着进程的销毁而销毁。因此，block 会生成一份对该变量地址的拷贝。

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *foo;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_foo, int flags=0) : foo(_foo) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

```c
impl.foo = &foo; 
```

访问的时候也是通过指针访问：

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *foo = __cself->foo; // bound by copy
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_qd_7zbm76j916n2_dhjbm6nsm480000gn_T_main_3b4b26_mi_0, (*foo));
}
```

#### 4、捕获 `__block` 修饰的变量

从上面的分析我们可以得到的结论是，当 block 捕获的内容为对象、全局变量、静态全局变量、静态自动变量时，我们都可以通过指针或者变量本身访问到它们的存储域，并且可以在随意修改它们（上面内容没涉及到在 block 内修改变量，可以自行尝试）。但是对于值类型的自动变量，我们不能在 block 内部修改它，具体原因上面也谈到了。OC 提供了一个 `__block` 关键字给开发者实现对自动变量的捕获和修改，那么当我们用 `__block` 修饰自动变量之后，block 对变量做了什么呢？如下代码：

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        __block int foo = 3;
        void (^blk)(void) = ^(){
            foo = 4;
        };
        blk();
        foo = 5;
    }
    return 0;
}
```

当变量 `foo` 用 `__block` 修饰之后，在 block 中修改它，编译器不会报错。

此时，将上述代码编译成 C 代码后发现 block 的实现中多了一个结构体：

```c
struct __Block_byref_foo_0 {
  void *__isa;
__Block_byref_foo_0 *__forwarding;
 int __flags;
 int __size;
 int foo;
};
```

又看到了老朋友 `isa`，因此我们可以认为 block 使用了 OC 类的结构来存储它捕获到的 `__block` 变量。令人诧异的是，这个结构体中多了一个 `__forwarding` 指针。

此时的 Block：

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_foo_0 *foo; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_foo_0 *_foo, int flags=0) : foo(_foo->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

多了一个 `struct __Block_byref_foo_0` 类型的成员 `foo`。初始化方法多了一个参数，并且默认将传入参数 `_foo ` 的 `__forwarding)` 指针赋值给成员变量 `foo`。

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_foo_0 *foo = __cself->foo; // bound by ref
  (foo->__forwarding->foo) = 4;
}
```

block 同样也会用 `__forwarding` 指针去访问捕获的 `foo` 变量（注意：C语言中， `->` 和 `.` 的区别，值类型的结构体用点语法，引用类型的结构体用 `->`）。

即使出了 block 的作用域，block 仍然会用 `__forwarding` 访问 `foo` 变量。

```c
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        __attribute__((__blocks__(byref))) __Block_byref_foo_0 foo = {(void*)0,(__Block_byref_foo_0 *)&foo, 0, sizeof(__Block_byref_foo_0), 3};
        void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_foo_0 *)&foo, 570425344));
        ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
        (foo.__forwarding->foo) = 5;
    }
    return 0;
}
```

这段代码首先将 `foo` 变量封装为 `__Block_byref_foo_0` 类型的对象 `foo`，然后用对象 `foo` 实例化一个 block，从 Block 的初始化方法可看出，此时 block 的成员变量 `foo` 和对象 `foo` 的 `__forwarding` 指针指向同一块内存。令人费解的是这行代码：

```c
(foo.__forwarding->foo) = 5;
```

因为此时，已经出了 block 的作用域，但是仍然用对象 `foo` 的 `__forwarding` 指针访问和修改 `foo` 变量。而且，和捕获自动变量不同的是，block 的实现中多了 `copy` 和 `dispose` 函数：

```c
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    _Block_object_assign((void*)&dst->foo, (void*)src->foo, 8/*BLOCK_FIELD_IS_BYREF*/);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    _Block_object_dispose((void*)src->foo, 8/*BLOCK_FIELD_IS_BYREF*/);
}
```

之前说过，当 block 满足被拷贝进堆内存的条件时，它才会实现这两个方法，来管理被捕获对象的内存。并且此时函数的注释是 `BLOCK_FIELD_IS_BYREF`，这说明当 block 捕获 `__block` 修饰的变量时，block 也会被拷贝到堆内存，block 会通过枚举值 `BLOCK_FIELD_IS_BYREF` 和 `BLOCK_FIELD_IS_OBJECT` 等等来区分拷贝的内容，并且管理它的内存。从上面的实现代码中，我们大概可以运行时的内存布局如下所示：

![](https://upload-images.jianshu.io/upload_images/5314152-b95d42dfdd257763.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

block 被拷贝到堆内存，并且 `foo` 对象作为 block 的成员也被拷贝到堆内存，此时栈中的 `foo` 对象和堆中的 `foo` 对象的 `__fowarding` 指针都指向了堆中的 `foo` 对象，block 就是通过这种方式完成对 `__block` 类型的自动变量的捕获，所以即使自动变量 `foo` 被弹出栈，block 仍然可以修改它，因为自从 `__block` 变量被捕获的那一刻起，无论在 block 中，还是 block 外部，对它的访问和修改全部都是堆内存中的那一份，并且栈 block 和堆 block 都会对它强引用，当然它也会服从 ARC 的内存管理方式，当栈 block 和堆 block 都不再强引用它的时候，它就会被释放。

### 四、 block 循环引用

理解 block 循环引用需要熟知 OC 内存管理的方式 ARC，参考：[ARC内存管理以及循环引用](https://zhangxiaom.github.io/2018/01/02/ARC%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%BB%A5%E5%8F%8A%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8/)。一个运行中的程序是由它的完整的逻辑控制流和一整块的虚拟内存构成的，因此对内存的理解更优先于对语言语法和 API 的掌握。我们应该从更深层次理解 ARC 的工作方式（包括自动释放池、内存管理关键字、一个对象何时创建、何时释放、何时何地引用计数为何等等），而不是用一句使用自动引用计数的方式管理内存涵盖一切。

#### 1、何时会产生循环引用

从上文中可以知道 block 捕获对象的时候，会强引用一次对象，此时假如 block 被赋值为该对象的某一个 `strong` 修饰的属性或者实例变量时，那么该对象也就会强引用一次 block，因此就造成了它们之间的互相循环引用。ARC 会在一个对象的引用计数为 0 的时候回收一个对象占有的堆内存，而循环引用会导致 block 和对象的引用计数永远都不为 0，从而导致它们占用的内存永远不会被释放。

那么什么时候对象会强引用 block？如果对内存有一定了解的话，应该知道只有堆内存是需要我们手动管理的，也就是只有堆内存中的数据才遵循自动引用计数的规则，因此只有当 block 被拷贝到堆内存时才会有循环引用的概念。上文中提及的几个 block 会被拷贝到堆内存的情况就是我们需要注意循环引用的地方。当 block 存储在栈内存，例如作为函数的参数并且在函数内不会被函数的调用者强引用，或者存储在全局区都不用担心循环引用的问题。当 block 被赋值给一个 `strong` 或者 `copy ` 关键字修饰的成员变量时，我们就应该注意使用它的时候是否会产生循环引用，因为此时 block 会被拷贝到堆内存，并且拥有该成员变量的对象会间接地强引用 block，假如此时 block 也强引用该对象的话，就会产生循环引用。这里（[ARC内存管理以及循环引用](https://zhangxiaom.github.io/2018/01/02/ARC%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%BB%A5%E5%8F%8A%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8/)）会详细介绍啥叫循环引用。

#### 2、block 捕获 weak 变量

使用 `weak` 关键字解决循环引用是大多数人都掌握的一个手段，但是好像并没有多少人关心 `weak` 关键字如何解决循环引用的，当然大家可能都会说一个弱引用，像这种问题我们最好不要这样一言以蔽之。

关于 `weak` 关键字的特性不是我们本文讨论的重点，下面的论述默认大家对 `weak` 关键字的特性有一定的了解。上文说了，block 捕获对象的时候会生成一个引用该对象的变量的[别名](https://zh.wikipedia.org/wiki/%E5%88%AB%E5%90%8D_(%E8%AE%A1%E7%AE%97) )，此别名和外部变量具有相同的内存管理语义。因此，当 block 捕获 `weak` 变量引用的对象时，实际上生成的别名仍然是具有 `weak` 特质的。如下代码：

```objc
id foo = [NSMutableArray new];
__weak id bar = foo;
void (^blk)(void) = ^(){
    [bar addObject:@""];
};
blk();
```

此时的内存布局为：

![](https://upload-images.jianshu.io/upload_images/5314152-f44da8c134a5824f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因此，此时的 block 不会强引用 `NSMutableArray` 对象，当变量不再强引用对象时，对象就会被释放。下面的代码可证明上述结论：

```objc
void (^blk)(void);
{
    id foo = [NSMutableArray new];
    __weak id bar = foo;
    blk = ^(){
        NSLog(@"%@", bar);
    };
}
blk();
```

执行结果为：

```objc
2018-06-06 17:44:45.199394+0800 XXX[49068:7235369] (null)
```

由此可见，当 `foo` 变量出了其作用域之后（不再强引用对象），`NSMutableArray` 对象即被释放，证明了我们的结论（注意：不要用类工厂方法创建 `NSMutableArray` 对象，因为自动释放池会强引用它一次，即使 `foo` 不再强引用它，它的引用计数仍然为 1）。

#### 3、关于 weak-strong dance

像上文中的情况，假如我们使用了 `weak` 解决循环引用，就会造成一些问题，就是被捕获的对象会被释放，那么我们就会得到错误的结果，很多人会告诉我们使用 weak-strong dance 解决这类问题，但是真的有效吗？如下代码：

```objc
void (^blk)(void);
{
    id foo = [NSMutableArray new];
    __weak id weak_foo = foo;
    blk = ^(){
        __strong id strong_foo = weak_foo;
        NSLog(@"%@", strong_foo);
    };
}
blk();
NSLog(@"%@",blk);
```

程序执行结果为：

```objc
2018-06-06 18:06:58.564651+0800 XXX[49189:7362233] (null)
2018-06-06 18:06:58.565225+0800 XXX[49189:7362233] <__NSMallocBlock__: 0x100653040>
```

好像并没有生效，从理论上来说，block 的执行函数会强引用一次捕获的对象以此来使对象不被释放，但是假如 block 的执行函数执行的时候，对象已经被释放，那么强引用不还是对 `nil` 强引用的吗？所以，weak-strong dance 只能帮助我们防止在block 执行的过程中捕获的对象被释放，并不能解决 block 执行之前对象被释放的问题，因此请慎用 `weak` ，weak-strong dance 并不能保证 block 执行时对象不被释放。 所以即使大部分开发者认为 weak-strong dance 能解决这样的问题，我们仍然要抱着质疑的态度去面对这样的观点。

### 写在最后的话

请不要完全相信本文的任何一个观点和结论。尽信书不如无书，请抱着学习的态度研究问题，而不是为了应付面试。

源码和官方文档会告诉你想知道的一切！！