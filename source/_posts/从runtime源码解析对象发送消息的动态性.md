---
title: 从runtime源码解析对象发送消息的动态性
date: 2018-01-20 11:30:55
tags:
---

## 写在前面的话

本文不是对runtime的使用的简单的阐述，而是我对runtime中消息发送的一些更深层的理解。

不要相信任何博客或者文章，apple 的 opensource 源代码会告诉我们想知道的一切，所以善用源码可能会事半功倍。
## 一、结构体 vs 类
我们知道，OC 是 C 语言的超集，是对 C 和 C++ 的进一步封装，一开始学习 OC 这门语言的时候，我们就被灌输过一句话：对象存储在堆内存，变量存储在栈内存，而 runtime 告诉我们类是对 C 和 C++ 中结构体的封装，而结构体是值类型（[值类型 vs 引用类型](https://zhangxiaom.github.io/2018/01/02/%E5%80%BC%E7%B1%BB%E5%9E%8B%E5%92%8C%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%E5%9C%A8Swift%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/)），肯定是存储在栈上的，这不是自相矛盾吗？另外，OC1.0 是完全对 C 语言的封装，C 语言的结构体是不能声明和实现函数的，到底是怎么回事呢？现在我们用结构体实现一个简单的类：
```C
// 父类
struct SuperFoo {
    int val;
};

struct Foo {
    int val;
    // 指向自己的指针
    struct Foo *isa;
    // 指向父类的指针
    struct SuperFoo *superFoo;
    // 声明一个指针变量 sum，它的类型为具有一个 int 类型返回值，两个 int 类型参数的函数。
    int(*sum)(int,int);
};

typedef struct Foo* PFoo; // PFoo 为一个指向 Foo 结构体的指针类型

int sum(int a, int b)
{
    return a + b;
}

int main(int argc, const char * argv[]) {
    // 声明一个指针指向 Foo 结构体，PFoo就是引用类型，pFoo 就是分配在栈内存的变量
    PFoo pFoo; 
    // 相当于 OC 中的alloc，将实例存入堆内存，现在 pFoo 就指向（引用）一个堆内
    // 存的实例
    pFoo = (PFoo)malloc(sizeof(PFoo));
    // init 初始化操作
    pFoo -> val = 4;
    pFoo -> isa = pFoo;
    // 将函数 sum() 赋值给 pFoo 的成员变量 sum
    pFoo -> sum = sum;
    // 可以创建一个父类对象，让 superFoo 指向父类对象
    // pFoo -> superFoo = superPFoo;
    // use
    // 通过函数指针调用函数，pFoo -> sum 是一个指向函数sum的指针
    int result = (pFoo -> sum)(4, 5);
    printf("result = %d\n", result);
    // print "result = 9"
    
    // 释放内存
    free(pFoo);
    // 将 pFoo 设置为空指针
    pFoo = NULL;
    
    return 0;
}

```
上述的代码就是用结构体实现一个简单的类，其实真正的runtime对类的实现比这个要复杂的多的多，函数的调用也不是简单的通过函数指针的成员变量调用，说这个只是想引入一下函数指针对类的意义以及值类型和引用类型的关系。

## 二、OC 消息发送的动态性

### 1. 动态性

提及 OC 及 runtime，我们听到最多的一句话就是 OC 是一门动态类型的语言，所谓的动态和静态的区分主要是指程序的执行是依赖于编译期还是运行期。

如果一段程序的执行在编译结束后就决定了它的内存分配，那么我们就可以说它是个静态类型的语言，而 OC 的动态性在于，它在编译期只是进行简单的语义语法检查，而不会分配内存。它在编译期只关心某个类型的某个对象能不能调用某个方法，而不会关心这个对象是不是 `nil`，也不会关心这个方法的实现细节，甚至不关心到底有没有这个方法，这些事都是运行期才会去做的事。

这就决定了我们可以在运行期对我们的程序做更多的更改，当然也存在很多弊端，有句话说得好：“动态类型一时爽，代码重构火葬场”，运行期分配内存确实会让我们的程序出现很多运行时的错误，比如，访问了野指针、内存泄漏等等，确实会给程序带来很多灾难性的bug，甚至于必须重构代码才能解决。

因此，对运行时的充分了解能使我们尽最大可能的规避这些错误，从而减少我们踩坑的几率和填坑的时间。

### 2. 消息发送的动态性

举个例子：
```C
void hello() {
    printf("Hello, world!");
}

void bye() {
    printf("Goodbye, world!");
}

void doSomeThing(int anyState) {
    // 函数的调用由编译时决定，函数的汇编指令是硬编码
    if (anyState) {
        hello();
    } else {
        bye();
    }
}

```
上述代码是一段简单的C语言代码，不管会不会 C 语言，应该都能看得懂，当调用 `doSomeThing()` 的时候，不管 `if` 条件是不是成立，程序都会将 `hello()` 和 `bye()` 这两个函数的汇编指令硬编码进汇编指令集。

![](https://user-gold-cdn.xitu.io/2018/1/3/160ba89f6a138776?w=357&h=130&f=png&s=812)

假设 `hello()` 和 `bye()` 这两个函数在代码区中的存储为上图，则在 `doSomeThing()` 中，编译器会在编译期，在 `if` 和 `else` 中都会将这两块内存生成的汇编指令硬编码进汇编指令集。类似于：
![](https://user-gold-cdn.xitu.io/2018/1/3/160ba950601b8af2?w=142&h=386&f=png&s=5963)
这就是一种静态的调用函数的方式，而动态的调用方法为：

```C
void hello() {
    printf("Hello, world!");
}

void bye() {
    printf("Goodbye, world!");
}

void doSomeThing(int anyState) {
    // 编译时只获取函数的地址，运行时才发出指令，执行函数
    void (*func)();
    if (anyState) {
        func = hello;
    } else {
        func = bye;
    }
}
```
这段代码和上述代码的差异为，在 `if` 条件语句中调用函数的方式变成了函数指针而不是简单的函数调用。它的动态性体现在，编译器在编译期仅仅获取函数的首地址，将指向函数的首地址硬编码进汇编指令集，而不是将整个函数的指令全部硬编码，到运行时再去决定调用那个函数（访问哪个函数的内存）。如果你在运行时强制将这个本来指向某个函数的指针指向另一个函数，那么这就是所谓的方法交换。

![](https://user-gold-cdn.xitu.io/2018/1/3/160baa595f67d978?w=397&h=302&f=png&s=11045)

这就是所谓的调用函数的动态性。OC 这门语言就是采用这种函数指针的方式实现消息发送的动态性。当然也不可能实现的这么简单。

真正的汇编指令集肯定不可能这么简单，只是简单画了一下，更容易理解一点。

## 三、将方法存储到类

OC 的动态性并不仅仅体现在消息发送方面，还有其他的，比如，运行时添加属性、添加成员变量、消息转发等等，其实对属性、变量和方法的封装大同小异，这里仅分析了 runtime 对消息的存储和获取。

大家都知道的一件事就是，OC 中类的实质是结构体，结构体中存储了所有的成员方法列表、属性列表、协议列表等等。存储结构如下图：

![](https://user-gold-cdn.xitu.io/2018/1/3/160baf94d0a49b21?w=828&h=563&f=png&s=51340)

可以看到一个 `method_array_t` 类型的变量 ` methods`，这就是类中的方法列表，`method_array_t` 是一个类，所以 `methods` 指向一个类实例，它在runtime中的组成为：

![](https://user-gold-cdn.xitu.io/2018/1/3/160bb0fb7806de8b?w=1034&h=344&f=png&s=31137)

可以看到，方法列表最终存储的东西为 `method_t` 结构体，它有三个成员变量，一个 `name`，可以理解为方法的签名，OC 会通过方法签名去列表中查找某个方法的实现，runtime 对它的定义为：

```
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```
可以看出这是一个指针类型，指向 `objc_selector` 结构体。另一个成员为：`const char *types` 常量为 OC 运行时方法的 typeEncoding 集合，它指定了方法的参数类型以及在函数调用时参数入栈所要的内存空间，没有这个标识就无法动态的压入参数 [Type Encoding](https://user-gold-cdn.xitu.io/2018/1/3/160bb2b1fa08761a)。

而 `IMP imp` 就是一个指向函数的函数指针，就是一个指向方法的首地址的指针。`IMP` 类型被定义为：

```C
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif

```
可以看出这也是一个指针类型，指向一个函数，即函数指针。当我们向对象的方法列表添加方法的时候，会调用：

```C
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return NO;

    rwlock_writer_t lock(runtimeLock);
    return ! addMethod(cls, name, imp, types ?: "", NO);
}
```

`addMethod()` 会返回一个 `IMP` 类型的函数指针，这个函数会将传入的 `imp` 添加进类的函数列表，并且更新缓存，最后返回这个 `imp`。如果 `addMethod()` 方法返回为空指针，则添加失败，返回 `false`。`addMethod()` 方法的具体实现细节为：

```C
static IMP 
addMethod(Class cls, SEL name, IMP imp, const char *types, bool replace)
{
    IMP result = nil;
    // 1
    runtimeLock.assertWriting();
    
    // 2
    assert(types);
    assert(cls->isRealized());
    
    // 3
    method_t *m;
    if ((m = getMethodNoSuper_nolock(cls, name))) {
        // already exists  
        // 4
        if (!replace) {
            result = m->imp;
        } else {
            result = _method_setImplementation(cls, m, imp);
        }
    } else {
        // fixme optimize
        // 5
        method_list_t *newlist;
        newlist = (method_list_t *)calloc(sizeof(*newlist), 1);
        newlist->entsizeAndFlags = 
            (uint32_t)sizeof(method_t) | fixed_up_method_list;
        newlist->count = 1;
        newlist->first.name = name;
        newlist->first.types = strdupIfMutable(types);
        newlist->first.imp = imp;

        prepareMethodLists(cls, &newlist, 1, NO, NO);
        cls->data()->methods.attachLists(&newlist, 1);
        flushCaches(cls);

        result = nil;
    }

    return result;
}
```
根据注释顺序：

1、加写入锁。

2、检查类型，检查类是否实现。

3、声明一个指针变量，指向 `method_t` 结构体，判断方法是否已经存在。

4、如果方法已经存在，判断是替换方法还是添加方法，如果不是替换，直接返回已经存在的方法的实现，如果是替换，则直接覆盖原方法。

5、如果方法不存在，则将其添加进入方法列表。

更具体的实现：[runtime](https://opensource.apple.com/tarballs/objc4/)，可以下载最新的 runtime 源码查看。

## 四、从类中查找方法

当我们向对象发送消息的时候：

```objc
id returnValue = [obj doSomeThingWithParams:params];
```

编译器会将它编译成原型为：

```objc
void objc_msgSend(id self, SEL cmd, ...);
```
的 C 函数。所以上面的函数会被翻译成：

```objc
id returnValue = objc_msgSend(obj, @selector(doSomeThingWithParams:), params);
```

这是一个标准的 C 函数，而且知道运行时的 iOS 开发者大部分都对它有所了解。我们来看一下，runtime 如何通过这个函数实现 `doSomeThingWithParams` 这个方法的调用。

当我们使用 `objc_msgSend()` 调用函数时，函数的调用栈为：

```
0 lookUpImpOrForward
1 _class_lookupMethodAndLoadCache3
2 objc_msgSend
3 main
4 start
```

可以看到在调用了 `objc_msgSend` 之后，调用了 `class_lookupMethodAndLoadCache3` 这个函数，这个函数名的字面意思为：从类中查找方法并且加载缓存。这个函数的实现为：

```C
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}

```

就调用了一个函数 `lookUpImpOrForward()`，这个函数名的字面意思是：查找 `imp` 或者转发，可以看出来，这个方法应该就是从方法列表中查找函数指针的那个方法了。它的实现为：

```C
/***********************************************************************
* lookUpImpOrForward.
* The standard IMP lookup. 
* initialize==NO tries to avoid +initialize (but sometimes fails)
* cache==NO skips optimistic unlocked lookup (but uses cache elsewhere)
* Most callers should use initialize==YES and cache==YES.
* inst is an instance of cls or a subclass thereof, or nil if none is known. 
*   If cls is an un-initialized metaclass then a non-nil inst is faster.
* May return _objc_msgForward_impcache. IMPs destined for external use 
*   must be converted to _objc_msgForward or _objc_msgForward_stret.
*   If you don't want forwarding at all, use lookUpImpOrNil() instead.
**********************************************************************/
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP imp = nil;
    Method meth;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    if (!cls->isRealized()) {
        rwlock_writer_t lock(runtimeLock);
        realizeClass(cls);
    }

    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }

    // The lock is held to make method-lookup + cache-fill atomic 
    // with respect to method addition. Otherwise, a category could 
    // be added but ignored indefinitely because the cache was re-filled 
    // with the old value after the cache flush on behalf of the category.
 retry:
    runtimeLock.read();

    // Try this class's cache.

    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // Try this class's method lists.

    meth = getMethodNoSuper_nolock(cls, sel);
    if (meth) {
        log_and_fill_cache(cls, meth->imp, sel, inst, cls);
        imp = meth->imp;
        goto done;
    }

    // Try superclass caches and method lists.

    curClass = cls;
    while ((curClass = curClass->superclass)) {
        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (imp) {
            if (imp != (IMP)_objc_msgForward_impcache) {
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, imp, sel, inst, curClass);
                goto done;
            }
            else {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                break;
            }
        }

        // Superclass method list.
        meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
            imp = meth->imp;
            goto done;
        }
    }

    // No implementation found. Try method resolver once.

    if (resolver  &&  !triedResolver) {
        runtimeLock.unlockRead();
        _class_resolveMethod(cls, sel, inst);
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }

    // No implementation found, and method resolver didn't help. 
    // Use forwarding.

    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);

 done:
    runtimeLock.unlockRead();

    return imp;
}
```

源码中给的注释很清楚，先从优化缓存中查找 `imp`，如果有直接返回，如果没有，先判断类是否实现，如果没有就去实现类，然后判断类是否初始化，如果没有就去初始化，再然后去类中的缓存列表中查找，找到就返回，如果没找到，再去父类的缓存和父类的方法列表中查找，找到就返回，如果还是没有，则允许一次 `resolve`，如果还是没有，则进入消息转发。

然后就可以使用返回的 `imp` 和汇编指令完成方法的调用了。对汇编精通的可以参考源码中的 `objc-msg` 模块查看汇编指令对 `imp` 的使用。
