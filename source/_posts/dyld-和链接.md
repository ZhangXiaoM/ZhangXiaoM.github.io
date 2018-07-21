---
title: dyld 和链接
date: 2018-07-01 17:46:31
tags: 编译和链接
---

代码从写完到编译到运行之间发生了什么？我们的程序是如何在设备上执行的？为什么我们代码中没有的库函数也能执行？在工作中我们可能常常会疑惑这样的问题，下面我们来探究一下这些问题。在阅读本文之前，需要充分了解进程的概念：[并发编程之进程](https://zhangxiaom.github.io/2018/06/12/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E8%BF%9B%E7%A8%8B/)。

### 一、编译

编译是一个很复杂的过程，往往也是检验我们代码正确与否的第一步，编译器会帮助我们做很多很多很多事，比如，语法分析、词法分析、类型检查、预处理、生成中间代码等等，本文着重介绍链接，所以关于编译的过程也不会展开太多。那么，点击编译器的编译按钮后，编译器主要做了什么工作呢？

- 源程序分析。语法分析、词法分析、语义分析、类型检查等等，这一阶段的目标是主要是检查代码有没有错误，就像我们常见的 `error` 和 `warning` 就是这个阶段确定的。

- 预处理。预处理器会展开目标模块导入的头文件和替换宏定义，预处理后生成 `*.i` 文件。

- 编译。编译器将 `*.i` 文件编译成 ASCII 汇编语言文件 `*.s`。

- 汇编。汇编器将 `*.s` 文件汇编成一个可重定位的二进制目标文件`*.o`，Mac OS 和 iOS 中称为 Mach-O文件。

- 链接。链接分为动态链接和静态链接，链接器将所有的目标文件和系统目标文件组合起来，生成能在机器上运行的可执行文件。iOS 中为 `*.ipa`，Windows 中为 `*.exe`，Android 中为 `*.apk` 等等。

例如两段 c 代码：
```c
// mian.c
#include <stdio.h>
#include "sum.h"

int main(int argc, const char * argv[]) {
    
    int arr[2] = {1, 2};
    sum(arr, 2);
    return 0;
}
```

```c
// sum.h
int sum(int *a, int n);
// sum.c
int sum(int *a, int n) {
    
    int sum = 0;
    
    for (int i = 0; i < n; ++i) {
        sum += a[i];
    }
    
    return sum;
}
```

此时，它们的编译过程为：

![](https://upload-images.jianshu.io/upload_images/5314152-076779743e52d486.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`sum.c` 和 `main.c` 源文件分别被单独编译成目标文件，然后经过链接器链接成可执行文件。

### 二、链接

链接就是上面的例子那样，将若干个目标文件组合成单一可执行文件的过程，这个文件被加载到内存中并执行。链接发生的时机可能为：编译时（源代码被翻译成机器代码时），加载时（程序被加载到内存中时），运行时。

在 iOS 开发中，我们会经常用到 cocoapod，或者引入一些第三方的 sdk，例如，`*.a`，`*.framework` 等等。这些就是可以独立编译成目标文件的动态库和静态库。

#### 2.1 静态链接

如上述例子即为静态链接，在程序加载进内存之前就完成两个目标文件的链接工作。我们知道目标文件是机器可识别的二进制文件，其中指令被存放在 `__TEXT` 内存段中，数据被存储在 `__DATA` 内存段中。

目标文件和进程的地址空间一样也是由一系列不间断的字节序列组成，这些字节序列包括：`.text`（指令集）、`.rodata`（只读数据）、`.data`（已初始化的全局和静态变量）、`.bss`（未初始化的全局和静态变量）、`.symtab`（符号表）、debug 信息等等。

目标文件是可以独立编译并运行的，一旦它被加载进内存中并运行，我们就可以将它视为一个进程，而进程虚拟内存的概念就决定了它所有的数据和指令都是从 `0x0` 开始布局的，因此，目标文件也是从 0x0 开始布局的。

符号表是一个陌生的概念，上面例子中的 `sum` 即为函数 `sum()` 的符号名称被存储在符号表中，只有在该模块中被外部可见的符号或者 `static` 修饰的符号才会被存储在符号表中，该模块私有的符号对其他模块来说没有意义，链接器也不关心它们。符号表中每个符号都对应一个结构体，结构体中存储了该符号对应的函数地址。

拿上面的例子来说，编译器会展开 `sum.h` 得到 `sum()` 的声明，当编译器在 `sum.s` 中找不到该函数的实现时，就会生成一个符号条目，扔给链接器处理，如果链接器在其他模块也没找到该函数的定义，就会造成链接中断，并报错：

```c
Undefined symbols for architecture x86_64:
  "_sum", referenced from:
      _main in main.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

#### 2.2 重定向符号引用

静态链接器解析目标文件中的符号，并且重定位它们，使所有目标文件中的符号在可执行文件中有正确的位置。比如代码：

```c
int main(int argc, const char * argv[]) {
    // insert code here...   
    printf("Hello, World!\n");
    return 0;
}
```

编译器将它编译成机器代码（`clang -c main.c -o main.o -Os`）：

```c
main.o:
(__TEXT,__text) section // __TEXT 段，__text 节
_main:
0000000000000000	pushq	%rbp
0000000000000001	movq	%rsp, %rbp
0000000000000004	leaq	0x9(%rip), %rdi
000000000000000b	callq	0x10
0000000000000010	xorl	%eax, %eax
0000000000000012	popq	%rbp
0000000000000013	retq
```

`callq` 指令是一个有符号偏移的跳转指令，也就是跳转到 `printf()` 函数，地址为 `0x10`（称为符号的引用），我们可以看到 `0x10` 处存放的是一个异或指令，因此汇编器会生成一个符号，存储到符号表中，交给链接器来处理（`printf()` 是 `libc.a` 静态库中的某个目标文件的函数）。

链接器会将定义 `printf()` 的目标文件和 `main.o` 链接成一个新的可执行文件：（`clang main.c -o main -Os`）：

```c
(__TEXT,__text) section // __TEXT 段，__text 节
_main:
0000000100000f76	pushq	%rbp
0000000100000f77	movq	%rsp, %rbp
0000000100000f7a	leaq	0x29(%rip), %rdi
0000000100000f81	callq	0x100000f8a
0000000100000f86	xorl	%eax, %eax
0000000100000f88	popq	%rbp
0000000100000f89	retq
```

链接之后的可执行文件中 `callq` 指令的目标地址被重定向成 `0x100000f8a`，也就是 `main()` 函数调用 `retq` 指令返回后的下一个地址。当可执行文件被加载进内存中时，`prinf()` 的指令集就会被加载器分配到 `0x100000f8a` 处。

链接器使用符号表来完成符号的重定向，如果没有符号表的话，仅靠 `0x10` 这种 magic value，链接器很难知道它是属于哪个目标模块的地址。因此，符号表和符号对链接来说很重要。

#### 2.3 静态库

假如链接器将所有相关的文件打包成一个单独的文件，就被称为静态库，就像我们熟悉的 `.a` 文件。第一个例子中的代码中的 `main()` 修改为：

```c
int main(int argc, const char * argv[]) {
    
    int arr[2] = {1, 2};
    sum(arr, 2);
    printf("%d", sum);
    return 0;
}
```

则整个代码的编译过程为：

![](https://upload-images.jianshu.io/upload_images/5314152-32688dce211add4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`libc.a` 静态库中的 `printf.o` 文件被链接进可执行文件，因为链接器只会复制静态库里被应用程序引用的目标模块。

静态库的存在降低了程序的耦合性和维护成本，但是当一个系统中很多程序都在引用 `libc.a` 时，每个程序都要拷贝一份它的副本，就会造成磁盘内存的极度浪费。

#### 2.4 动态链接和动态共享库

我们在写 C 程序时，第一步要做的就是输入 `#include<stdio.h>`，在写 iOS 代码时，每个类都要导入 `CoreFoundation.h`，`stdio.h` 是 c 语言的标准输入输出库（standard input/output） `libc` 的头文件，几乎每个 c 程序都会用到。假如使用静态链接的话，每个程序都要拷贝一份。当每个程序都引入很多的标准库时，就会造成内存的极大浪费。

动态共享库（linux 中为 `*.so`）的出现就是为了解决静态库的缺陷，动态库同样也是一个目标模块，他不会被链接进任何其他目标模块，而是被加载进任意的内存地址，然后和一个加载到内存中的程序链接，这个过程称为动态链接。iOS 中的 dyld 就是一个动态链接器，负责程序之间的动态链接。

当我们将上述例子中的静态链接换成动态链接时，编译过程为：

![](https://upload-images.jianshu.io/upload_images/5314152-3bfdbe3cc310c982.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

动态链接器也会重定位符号的引用，比如上面例子中的 `0x10`，动态链接器会将动态库中的符号表拷贝到进程的内存中，然后像静态链接那样重定向它们在该进程空间内的引用，就是将目标文件中的地址修改为符号在进程内存中的地址。

静态链接将目标文件拷贝到磁盘中，被链接完成后加载进内存，动态链接在加载的时候拷贝到内存中，节省了磁盘空间。

### 三、dyld

dyld (dynamic loader and linker) 是 iOS 和 Mac OS 系统中的动态加载器和链接器，它负责共享库的动态链接，比如 libobjc.A.dylib，Foundation.framework 等等。加载和链接是在程序启动时，`main()` 函数之前做的，dyld 主要做了什么？

打印一下堆栈信息如下：

```c
* thread #1, stop reason = breakpoint 1.1
  * frame #0: 0x00007fff5e29794a libSystem.B.dylib`libSystem_initializer
    frame #1: 0x000000010001ca7a dyld`ImageLoaderMachO::doModInitFunctions(ImageLoader::LinkContext const&) + 420
    frame #2: 0x000000010001ccaa dyld`ImageLoaderMachO::doInitialization(ImageLoader::LinkContext const&) + 40
    frame #3: 0x00000001000181cc dyld`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 330
    frame #4: 0x000000010001815f dyld`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 221
    frame #5: 0x0000000100017302 dyld`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 134
    frame #6: 0x0000000100017396 dyld`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 74
    frame #7: 0x0000000100008521 dyld`dyld::initializeMainExecutable() + 126
    frame #8: 0x000000010000d239 dyld`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 7242
    frame #9: 0x00000001000073d4 dyld`dyldbootstrap::start(macho_header const*, int, char const**, long, macho_header const*, unsigned long*) + 453
    frame #10: 0x00000001000071d2 dyld`_dyld_start + 54
```

dyld 的工作大致分为以下几步：

- 动态链接器自身就是一个共享目标文件，dyld 会首先将他自己加载进内存中并运行。

- 使用 `ImageLoader` 递归的向进程的内存空间中加载动态库。

- 重定向对动态库中符号的引用，完成链接工作。

- 初始化可执行文件。

- 为 `main()` 函数准备参数和环境变量（argc，argv[]，envp[]）。

- 为运行中的程序动态链接一些懒加载的符号（链接也可能发生在运行时）。

- 在一些情况下，当 `main()` 函数返回时，调用 `exit` 指令退出当前进程。

总结一下就是，当进程开始时，以 iOS 为例，就是点击了某个应用程序，dyld 会将可执行文件和它的共享库加载到内存中，将跨库 C 函数和变量引用链接到一起，然后在 `main` 函数开始执行。因此，从启动应用程序到我们写的代码开始执行，这段时间内都是 dyld 的工作。

注意：dyld 工作时，进程都是被内核管理的，直到可执行文件和动态库被链接并且加载完成，进程才开始运行在用户态。

#### 3.1 共享缓存

我们知道 objc 这门语言是动态特性的语言，对象的内存分配都是运行时来完成的，也就是它是一门重度依赖运行时的语言， 运行时以动态库的形式被 dyld 链接进进程内存空间，因此，所有 iOS 中的应用程序都要在启动时链接运行时库。iOS 中程序所依赖的系统库并不只有运行时库一个，dyld 会在程序启动时链接它们，这样就会造成很大的时间和空间的开销。dyld 会通过共享缓存来优化这一点。

从上面对 dyld 的工作模式的分析，我们可以知道，每个进程的动态库都应该是不同的，因为它们分别被拷贝进不同的进程内存。事实上，dyld 的实现对这里进行了优化，它使用了共享缓存，共享缓存中保存了系统动态库的拷贝，大部分的系统库的加载和链接都是在程序启动前就完成的，所有的进程都会共享这部分内存，这样就可以节省了很多的启动时间和内存。共享内存是进程间通信的一种方式，可以参考我的这篇文章：[并发编程之进程](https://zhangxiaom.github.io/2018/06/12/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E8%BF%9B%E7%A8%8B/)。

共享缓存解决了每个进程都要加载共享库的缺陷，dyld 的另一个工作是重定向符号表，就是上文中的第三步，在 iOS 和 Mac OS 中被称为 `selector` ，它就是 C 语言中符号的名称，例如上文中提到的 `sum`。我们可以从开源库中找到系统的动态库的符号列表：[built-in selector table](http://www.opensource.apple.com/source/objc4/objc4-371/runtime/objc-sel-table.h)，数据量不算小，这些符号保存了函数（指令）的实际地址，dyld 需要将它们拷贝的进程内存中，然后重定向对它们的引用（类似于静态链接的重定向，加载时完成）。

共享缓存会自己创建一个符号表，并且更新它们的引用，这样就可以使所有的进程都共享这些动态库的符号表，不用将符号表拷贝到自己的进程空间中，节约了拷贝的时间和空间，但是 dyld 仍要将目标文件中的符号地址重定向为共享缓存中的地址。

共享缓存的优化会节省一半的程序启动时间，也能为 iOS  设备节省 1MB 的内存。该结论由 runtime 源码的维护者：Hamster Emporium 的[这篇文章](http://www.sealiesoftware.com/blog/archive/2009/09/01/objc_explain_Selector_uniquing_in_the_dyld_shared_cache.html)给出。

#### 3.2  +load 

当 dyld 干完自己要干的事之后，进程就由内核态切换为用户态，这时候，运行时库 libobjc.dylib 会初始化自己，然后向所有静态链接进可执行文件的类对象发送 `+load` 消息，表示该类对象已经被加载进内存。对于动态链接的共享库中的类对象，runtime 会在共享库加载进进程的地址空间之后尽早的发送 `+load` 消息。

因此，无论动态链接还是静态链接的目标模块，我们都可以认为 runtime 发送给类对象的第一条消息就是 `+load`。

从 runtime 源码[`objc-loadmethod.mm`](http://opensource.apple.com/source/objc4/objc4-532.2/runtime/objc-loadmethod.mm) 中，我们可以找到 `+load` 方法的调用时机，以及它在继承关系的类中的调用情况：

```c
/*
*call_load_methods （函数名）
* Call all pending class and category +load methods. （调用所有类和分类的 +load 方法）
* Class +load methods are called superclass-first. （先调用父类的 +load，在调用子类的）
* Category +load methods are not called until after the parent class's +load.（先调用类的 +load，在调用分类的）
*/
// 迭代的向所有类对象发送 `+load` 消息
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.（重复调用该方法无效，最外层的调用将完成所有工作）
    // 注意： loading 是 static 变量
    if (loading) return;
    loading = YES;
	// 自动释放池，管理迭代中生成的对象
    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            // Call all pending class +load methods.
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        // category 的 `+load` 在所有类对象之后调
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

因此，我们可以得到类对象的 `+load` 方法的调用时机和调用顺序，以及每个类对象只会被发送一次 `+load` 消息。

做个试验验证一下，创建几个类为：`Father`、`Son`、`GrandSon`、`Father+Extension`，然后重写每个类的 `+load`，函数的实现为打印 `XXX(类) load`，并且在 `Father` 的 `+load` 和 `main()` 函数分别打个断点，运行程序，程序会先在 `+load` 方法处中断，然后才在 `main()` 函数中断，并且 log 结果为：

```c
father load
son load
grandson load
extension load
```

符合理论结果。

iOS 开发中的 **Method Swizzling**，为什么要在 `+load` 方法中做？

熟悉 runtime 的同学一定知道 runtime 会将函数实现成 C++ 中虚函数表的形式，在运行时调用，参考我的另一篇文章：[从runtime源码解析消息发送的动态性](https://zhangxiaom.github.io/2018/01/20/%E4%BB%8Eruntime%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81%E7%9A%84%E5%8A%A8%E6%80%81%E6%80%A7/)。从上文我们知道了 `+load` 方法的调用时机是类对象加载到进程地址空间的之后的第一时间，也就是可能是该类对象接收到的第一条消息，因此，此时做方法交换是修改类对象的方法列表的最好时机。其实 **Method Swizzling** 的实质有点类似于对符号引用的重定向，当然要在类对象加载的时候完成。

### 四、程序启动时间

从上面我们的分析，可以看到应用程序从启动到我们写的代码的执行之间经历了多少事，链接和加载也是影响程序启动的主要因素，其中 apple 对 dyld 的优化-共享缓存可以帮我们很大程度的提高应用的启动时间，但是我们自己也应该承担一部分责任：

- 删除项目中不再使用的动态库（非系统库，dyld 不会创建共享缓存），因为链接和加载它们需要很多的时间和空间。

- 删除项目中不再使用的类和分类，因为加载它们到进程的地址空间是在启动时完成的。

- 尽量少的使用 **Method Swizzling**，因为方法交换需要遍历方法列表，会拖慢 `+load` 方法的执行，这也是在启动的时候完成的。

- 使用静态库代替非系统库的动态库，让重定向在编译的时候完成，能节省一部分链接的时间。

等等。

 另外：删除不再使用的静态库，即使它不会拖慢启动时间，但是它仍然会被编译成目标文件而占用磁盘空间。

### 五、总结

emm，老规矩，知道这些并不会提高我们的业务能力，但是，你现在知道平时开发中遇到的 Apple Mach-O Linker Error 是什么原因了吗？知道影响程序启动时间的是谁了吗？知道 `Redefinition of 'XX' as different kind of symbol` 的报错原因是什么了吧（符号表）？了解这些的目的是，当我们在某一时间遇到了一些关于 linker 的错误消息时，大概能知道从哪里查找问题，然后解决它。