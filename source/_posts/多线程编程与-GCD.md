---
title: 多线程编程与 GCD
date: 2018-07-07 12:16:27
tags: 并发编程
---

如果你还不了解进程的话，请参考：[并发编程之进程](https://zhangxiaom.github.io/2018/06/12/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E8%BF%9B%E7%A8%8B/)，进程是由完整的逻辑控制流和独立的地址空间构成的，一个线程就是进程中一个单一顺序的逻辑控制流，由进程调度的线程被称为用户级线程，由内核调度的被称为内核线程（轻量级进程），这里仅讨论用户级线程。多个进程可以被操作系统调度而组成多进程程序，同样的，多个线程也可以被进程调度而组成多线程编程，同一进程的多个线程共享该进程的地址空间，也就是整个进程的虚拟内存都是该进程内所有线程的共享内存。

### 一、几个概念

#### 1.1 同步 vs 异步

同步和异步的概念是针对指令，不针对线程，也就是仅用主线程也能进行异步操作，所以异步的并不一定是多线程的，同步的指令也不一定只是主线程执行的。假如将一个同步或者异步任务视为一个指令集的话，执行同步任务的线程会等待该指令集执行完再去执行该指令集的下一条指令，而异步任务会直接跳过当前指令集去执行下一条指令，当执行该指令集的线程空闲的时候才会执行该指令集。所以异步任务不会阻塞当前线程。

#### 1.2 并发 vs 并行

这两个概念是很多人比较容易混淆的概念，有人会说并发是并行的子集，只要是并行的，一定是并发的，但是并发的不一定是并行的，因为牵扯到线程调度（单核 CPU 的情况下，多个线程交替使用 CPU）的问题。那么有个问题请思考一下，假设我们的程序运行在一个四核 CPU 的设备上，也就是此时操作系统是支持并行的，但是我们设计的程序仅仅使用的主线程，比如说打印了一个 Hello world，那么此时能说并行的一定是并发的吗？

其实这两个概念相关联但是又不是那么关联，并行表述的是能力，并发表述的是程序结构，也就是具有双核以上 CPU 的系统具有并行的能力，我们写的代码是支持并发的程序结构。在具有并行能力的系统上执行的并发结构的程序一定是并发的，即使在不具有并行能力的系统（单核）上执行的并发结构的程序仍然是并发的。

#### 1.3 线程 vs 队列

对 iOS 来说，特别是习惯使用 GCD 的开发者，线程和队列也是需要区分的概念，队列和线程本质上并不是一一对应的关系（其实主线程不一定只执行主队列的指令）。GCD 会为我们的程序提供几种类型的队列（主队列，全局并发队列，串行队列，并发队列），我们只需要将任务以同步或者异步的形式添加进队列，GCD 会调度需要的线程帮我们依次执行队列中的任务。

#### 1.4 context（上下文）

**context** 一般被翻译为上下文，是一个抽象的概念，在 iOS 中也经常出现（CGContext），其实我们可以将它理解为一个作为数据模型的结构体，它保存了当前对象此时所有的状态信息，比如要绘制一个 `UILabel`，此时绘制对象的 `context` 里就会保存我们为 `UILabel` 设置的信息，比如背景颜色、字体、字号等等，然后负责绘制的对象会从 `context` 中取出这些属性完成绘制。对进程和线程来说是一样的，当进程被抢占时，它的 `context` 中就会保存进程此时的状态信息，等进程重新进入运行状态时，调度程序就会将进程信息恢复。

### 二、线程调度

和进程一样，线程也有一个上下文保存它当前执行的状态信息，比如堆栈信息、PC、寄存器等等，当该线程被抢占挂起时，上下文就会保存此时线程执行的栈帧、寄存器状态等等，线程的调度也被称为**上下文切换**。就像进程一样，线程也是交替使用 CPU 的，因为对于交互式程序来说，runloop 的存在就造成主线程一直占有 CPU 资源，线程的调度可以避免其他子线程**饿死**。

当多进程和多线程共存的情况下，对于线程的调度就分为两种情况。

- 由进程调度

  调度程序将时间片分配给进程，进程通过调度算法将时间片分配给线程，此时线程的上下文切换由进程决定，比如一个进程得到 10 ms 的时间片，它会根据自身的调度算法将时间片分配给线程，等时间片用完，调度程序会将该进程挂起，进程内正在执行的线程也会挂起，进程会保存所有线程的上下文，这是用户级线程常用的调度方式。

- 由调度程序调度

  调度程序负责调度线程，比如进程 A 和进程 B 分别有三个线程 A1, A2, A3, B1, B2, B3，调度程序分配 10ms 的时间片给线程 A1，10ms 过后分配 10ms 的时间片给 B1，此时既要切换进程的上下文，也要切换线程的上下文，因此，这种调度方式会带来更大的开销。线程的上下文由内核保存，一般来说，内核级线程会使用这种调度方式。

图示：

![](https://upload-images.jianshu.io/upload_images/5314152-ebe04302a61f6c8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 三、线程同步—锁

同一进程的多个线程会共享该进程的地址空间，比如数据段、文本段、堆等。当多个线程并发的访问同一块内存段时，就会产生竞态条件导致的线程安全问题。锁就是为了解决这些问题，也填一下[并发编程之进程](https://zhangxiaom.github.io/2018/06/12/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E8%BF%9B%E7%A8%8B/)里进程同步问题留下的坑。

**临界区：**

理解临界区的概念是解决线程安全和并发编程模型的重要依据。

![](https://upload-images.jianshu.io/upload_images/5314152-efac21c2f2657b32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图即为并发的情况下，临界区不安全的示例，T2 ~ T3 时间段 线程 A 和线程 B 同时访问临界区。我们可能会疑惑的地方是，单核 CPU 的系统中，存在这种并行的情况吗？其实即使在不支持并行的系统中，同样会有这种并发问题：在时间 T2，线程 A 的时间片用完，线程 A 挂起，此时 A 的上下文中记录它的执行状态，切换到线程 B 执行，线程 B 用完时间片记录执行状态，切换回线程 A，此时调度程序读取的线程 A 的执行状态为时间点 T2，因此此时相当于线程 A 回到时间点 T2 继续执行，直到下一次切换。所以我们可以把这种上下文切换的情况描述为上图中的**并行**，实则为**并发**。

```c
// 全局变量 _sum
int _sum = 0;
// 并发执行函数指针
void *thr_fn(void *arg) {
    printf("start %d\n", _sum);
    for (int i = 0; i < 10000; ++i) {
        _sum += i;
    }
    printf("sum of 0 ~ 9999 is %d\n", _sum);
    return NULL;
}

int main(int argc, const char * argv[]) {
    // insert code here...
    // 创建两个线程，执行 thr_fn
    for (int i = 0; i < 2; ++i) {
        pthread_t ntid;
        pthread_create(&ntid, NULL, thr_fn, NULL);
    }
    return 0;
}
```

log 的结果：

```c
// 这个结果不是唯一的，因为结果取决于线程切换的时机(竞态条件)
start 0
start 1021735
sum of 0 ~ 9999 is 46697501
sum of 0 ~ 9999 is 84364406
Program ended with exit code: 0
```

我们可以计算得到 0~9999 的和为 49995000，因此两个子线程的执行的结果都不是我们想要的结果，其实上图很好的解释这段代码的执行过程，线程 B 迭代之前获取的初始值并不是我们想要的线程 A 的执行结果，是因为在时间点 T2，线程 B 读取的临界区的值为线程 A 的 T1 ~ T2 时间段的执行结果，此时我们得到的结果就是竞态条件造成的。

锁能帮助我们解决这个问题，当这段代码加锁后的执行过程为：

![](https://upload-images.jianshu.io/upload_images/5314152-a6f1fe1e040563fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在时间 T2 线程 B 试图进入临界区时，由于临界区被加锁，所以线程 B 被阻塞，当线程 A 将临界区解锁后，线程 B 才能进入临界区。加锁：

```c
void *thr_fn(void *arg) {
 	pthread_mutex_lock(&_mutex);
    printf("start %d\n", _sum);
    for (int i = 0; i < 10000; ++i) {
        _sum += i;
    }
	printf("sum of 0 ~ 10000 is %d\n", _sum);
    pthread_mutex_unlock(&_mutex);
    return NULL;
}

```

log 结果：

```c
start 0
sum of 1 ~ 10000 is 49995000
start 49995000
sum of 1 ~ 10000 is 99990000
Program ended with exit code: 0
```

- 互斥锁

  上述例子中所使用的 `pthread_mutex_t` 即为互斥锁，互斥锁其实也是一个共享的全局变量，当该变量满足一定的条件时，允许线程访问临界区，否则该线程即被挂起。我们可以将该互斥锁想象成现实中的锁，锁的初始值的打开的，线程 A 进入房间（临界区）后，将锁锁住（时刻 T1），当线程 B 想进入房间时（时刻 T2），此时房间上锁，它只能挂起等待，直到线程 A 离开房间并且将锁打开（时刻 T3），线程 B 才能进入房间，并将锁锁住。

- 自旋锁

  自旋锁是特殊的互斥锁，特殊的地方是，当线程访问锁变量时，会一直不停的询问锁变量的状态，也就是在 T2 时刻，线程 B 不会被阻塞而是一直不停（死循环）的访问锁变量，直到它的时间片被消耗完或者锁被打开。因此，互斥锁会在临界区加锁时马上进行上下文切换，而自旋锁会不停的死循环，因此自旋锁会消耗更多的 CPU 资源，在不确定临界区执行时间的前提下，慎用自旋锁。

- 信号量

  互斥锁的概念标定了它只能有两种状态，就是加锁和未加锁，而信号量可以控制进入临界区的线程数量， 信号量被定义为一个正整数，当线程要进入临界区时，会首先访问信号量，当信号量大于 0 时，就代表可以进入临界区， 并将信号量减一，当信号量等于 0 时，线程就会被阻塞，当线程出了临界区时，会将信号量加一。

- 同步锁

  同步锁是 objc 语言特有的一种锁，`@synchronized{}`，代码块中的内容即为临界区，它会被编译器替换为：

  ```c
  // 源代码
  @synchronized{
      work();
  }
  // 编译器替换后
  objc_sync_enter(obj);
  work();
  objc_sync_exit(obj);
  ```

  我们可以从 runtime 动态库中的 `<objc/objc-sync.h>`（[在这](https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-sync.h)）中找到这两个方法的定义：

  ```c
  /** 
   * Begin synchronizing on 'obj'.  
   * Allocates recursive pthread_mutex associated with 'obj' if needed.
   * 
   * @param obj The object to begin synchronizing on.
   * 
   * @return OBJC_SYNC_SUCCESS once lock is acquired.  
   */
  OBJC_EXPORT  int objc_sync_enter(id obj)
      OBJC_AVAILABLE(10.3, 2.0, 9.0, 1.0);
  
  /** 
   * End synchronizing on 'obj'. 
   * 
   * @param obj The objet to end synchronizing on.
   * 
   * @return OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
   */
  OBJC_EXPORT  int objc_sync_exit(id obj)
      OBJC_AVAILABLE(10.3, 2.0, 9.0, 1.0);
  ```

  进一步从实现文件中，我们可以得到的结论是：`objc_sync_enter()` 会为 `obj` 生成并关联一个递归锁，然后将临界区的内容加锁，临界区代码执行完后，调用 `objc_sync_exit()` 解锁。

- NSLock

  NSLock 是对 `pthread_mutex_t` 的对象封装。

**NOTE**：除了同步锁，其他几个锁都是不可重入锁，如果重复加同一个锁，就会造成死锁，例如：

```c
pthread_mutex_lock(&_mutex);
pthread_mutex_lock(&_mutex);
```

这样锁变量将永远不会处于解锁状态导致死锁。

### 四、GCD 中的多线程编程

GCD 是对 POSIX 线程的高级封装，它会帮我们管理线程的生命周期，我们只需要将要执行的任务（block）以同步或者异步的形式添加进队列中，GCD 会选择合适的线程去执行任务。队列就是一种先入先出的数据结构，因此，添加进队列中的任务的执行顺序即为入队的顺序，而执行完成的顺序，取决于线程的调度、任务的长短、队列是串行还是并发、同步任务还是异步任务等等。

**NOTE**：我们所有没有添加进队列的任务其实就是主线程在执行，我们可以将这些任务理解为同步串行任务，也就是所有的函数顺序执行，当前函数执行完才会执行下一个，这些任务包含 `dispatch_sync()` 和 `dispatch_async()`。大概的模型就是这样（忽略任务之间空隙）：

![](https://upload-images.jianshu.io/upload_images/5314152-b3a0b67c91fe605f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面的讨论的队列均不包含主队列。我们将 `dispatch_sync()` 和 `dispatch_async()` 定义为一个 Task。并且将上下文切换以并行（伪并行）的形式描述。

#### 4.1 同步串行

```objc
dispatch_queue_t serialQueue = dispatch_queue_create("com.xm.test.serialQueue", DISPATCH_QUEUE_SERIAL);
// Task1
dispatch_sync(serialQueue, ^{
	// Task2
    NSLog(@"Task2");
    NSLog(@"%d", [NSThread isMainThread]); 
    // log result is 1.
});
// Task3
dispatch_sync(queue, ^{
	// task4
	NSLog(@"Task4");
});
// task5
NSLog(@"Task5");
```

此时这段代码的执行情况理论上应该为：

![](https://upload-images.jianshu.io/upload_images/5314152-7de31071475c4fa6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在时间 T1 处切换到子线程执行加入 `serialQueue` 中的同步任务 `Task2`，直到任务执行完，切回主线程继续执行 `Task3`。同步任务阻塞主线程的原因是：主线程在等待 `dispatch_sync()` 函数返回，而 `dispatch_sync()` 函数在等待 `Task2` 返回，因此，即使同步任务由子线程完成，它依然会阻塞主线程。实际上，GCD 会帮我们做一些优化：

![](https://upload-images.jianshu.io/upload_images/5314152-e50804b386b9354e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

GCD 会直接返回 `dispatch_sync()` 函数，然后在主线程执行同步任务，这样就避免了多余的上下文切换的开销。

#### 4.2 同步并发

同样是上面的示例代码，我们将串行队列，替换为并发队列：

```objc
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.xm.test.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
// Task1
dispatch_sync(concurrentQueue, ^{
	// Task2
    NSLog(@"Task2");
    NSLog(@"%d", [NSThread isMainThread]); 
    // log result is 1.
});
// Task3
dispatch_sync(concurrentQueue, ^{
	// task4
	NSLog(@"Task4");
});
// task5
NSLog(@"Task5");
```

此时，理论上的任务的执行方式为：

![](https://upload-images.jianshu.io/upload_images/5314152-0daa148ae166e0cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实际上 GCD 仍然会做上面的优化：

![](https://upload-images.jianshu.io/upload_images/5314152-f23c4a04836b7903.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

即使是添加进并发队列的同步任务也会阻塞主线程，理由同上，因此，GCD 同样会将任务放到主线程去执行，避免了上下文切换的开销。

#### 4.3 异步串行

我们现在将 4.1 中的同步方法改成异步方法：

```objc
dispatch_queue_t serialQueue = dispatch_queue_create("com.xm.test.serialQueue", DISPATCH_QUEUE_SERIAL);
// Task1
dispatch_async(serialQueue, ^{
	// Task2 (可使用 for 循环模拟线程阻塞，看代码的输出顺序)
	// for (int i = 0; i < 1000; ++i) {}
    NSLog(@"Task2");
    NSLog(@"%d", [NSThread isMainThread]); 
    // log result is 0.
});
// Task3
dispatch_async(queue, ^{
	// task4
	NSLog(@"Task4");
});
// task5
NSLog(@"Task5");
```

主线程会在执行 `dispatch_async()` 时将任务加入 `serialQueue` 然后立刻返回执行下一条指令，同时调度子线程去执行加入队列中的任务，此时任务执行完成的时机依赖于调度程序的调度、任务的长短和加入队列中的顺序。

![](https://upload-images.jianshu.io/upload_images/5314152-e70e3cc5db0846cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**NOTE**：当该示例中的队列为主队列时，异步任务会添加进主队列的队尾，当主线程执行完主队列中其他任务时，才会去执行该任务。

#### 4.4 异步并发

将 4.2 中的同步改为异步：

```objc
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.xm.test.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
// Task1
dispatch_async(concurrentQueue, ^{
	// Task2
    NSLog(@"Task2");
    NSLog(@"%d", [NSThread isMainThread]); 
    // log result is 0.
});
// Task3
dispatch_async(concurrentQueue, ^{
	// task4
	NSLog(@"Task4");
});
// task5
NSLog(@"Task5");
```

此时：

![](https://upload-images.jianshu.io/upload_images/5314152-1710825fa0fa14cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

任务执行完成的顺序仍然依赖于调度程序分配的时间片、任务的长度等。`Task4` 会在 `dispatch_async2()` 执行后加入队列，GCD 会分配合适的线程去执行它。

#### 4.5 死锁

同步任务是造成死锁的主要原因，假如将 4.1 中的串行队列换成主队列的话，此时主队列在等待 `dispatch_sync()` 返回，`dispatch_sync()` 在等待 `block` 返回，block 被添加进主队列，并且在 `dispatch_sync()` 之后，因此，`dispatch_sync()` 返回后才能执行 block，这样就造成了 `dispatch_sync()` 和 block 之间的循环等待而造成死锁（只有添加进非主队列的同步任务，GCD 才会优化）。

**NOTE：**GCD 造成的死锁不是[并发编程之进程](https://zhangxiaom.github.io/2018/06/12/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E8%BF%9B%E7%A8%8B/)中所描述的进程同步造成的死锁，也就是，这里的死锁不是多个线程争夺共享资源造成的死锁，而是由于同步任务和串行队列的性质造成的死锁，两者都是造成死锁的充分非必要条件。

### 总结

在实际工作中，并发带来的问题是比较让人头痛的问题。本文以最简单的模型分析并发时多个线程协同工作的原理，当然必须理解原理才能在工作中更好的分析并发带来的问题。推荐一本书 《现代操作系统》，虽然它不讲 GCD，但是看了它再去理解 GCD 有种豁然开朗的感觉。