---
title: 揭秘instancetype
date: 2018-04-19 14:37:06
tags:
---

苹果官方会建议我们用 `instancetype` 类型代替 `id` 类型作为某个类的初始化方法的返回值。以下内容摘自[Adopting Modern Objective-C](https://developer.apple.com/library/content/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html#//apple_ref/doc/uid/TP40014150-CH1-SW11):

> Use the instancetype keyword as the return type of methods that return an instance of the class they are called on (or a subclass of that class). These methods include alloc, init, and class factory methods.

### 一、初始化方法为什么用id类型而不是 [类名] 类型作为返回值类型

在 `instancetype` 关键字出现之前，我们会用 `id` 作为类初始化方法的返回类型，在 `instancetype` 关键字出现之后，编译器会主动将 `alloc` `init` `new` 开头的方法的返回值类型替换为 `instancetype`， 那么在 `instancetype` 出现之前，为什么不用该类自身的类型而是用 `id` 类型作为初始化方法的返回值。答案是 OC 的类继承体系。

假如，有一个类 `SuperClass`的初始化方法返回类型为它自己的类型，并且不会被编译器替换为 `instancetype` ：

```objc
- (SuperClass *)init;
```
那么当它的子类重写这个初始化方法的时候，只能返回它自身的实例，而无法返回子类的实例。因为重写父类方法， 必须和被重写的方法有相同的返回类型。所以子类永远无法通过重写这个父类初始化方法初始化自身。所以为了实现重写初始化方法（用父类初始化自身）的继承体系，必须要用一种通用类型，既能表述子类实例也能表述父类实例的类型，刚好 `NSObject` 是所有类的根类，由于 OC 的多态性，`id` 类型的变量可以指向任意类型的对象，因此，用 `id` 类型作为初始化方法的返回类型可以很好的解决类继承的问题。

### 二、instancetype 取代 id

有的人会认为 `instancetype`类型和 `id` 类型是一种类型的不同表达方式，其实并不是。 `instancetype` 顾名思义是当前类的实例类型，听起来好像和类名类型并没有上面区别，实则，它更严谨的遵循 OC 的继承体系。`instancetype` 类型只表述当前类的继承线，例如 `NSMutableString -> NSString ->...-> NSObject`，而 `id` 类型相对来说更博爱一点。

在适当的地方使用 `instancetype` 关键字可以提高代码的类型安全。例如：

```objc
interface MyObject : NSObject
+ (instancetype)factoryMethodA;
+ (id)factoryMethodB;
@end
 
@implementation MyObject
+ (instancetype)factoryMethodA { return [[[self class] alloc] init]; }
+ (id)factoryMethodB { return [[[self class] alloc] init]; }
@end
 
void doSomething() {
    NSUInteger x, y;
 
    x = [[MyObject factoryMethodA] count]; // Return type of +factoryMethodA is taken to be "MyObject *"
    y = [[MyObject factoryMethodB] count]; // Return type of +factoryMethodB is "id"
}
```
类 `MyObject` 声明并实现了两个相同类工厂方法，用来返回初始化后的 `MyObject` 对象，只是返回的类型一个是 `instancetype`，一个是 `id`。编译器在代码 `y` 处不会提示任何警告和错误，并且在编译期也没有任何错误，但是当到了运行期就会崩溃。因为此时的 `MyObject` 实例可能是任意一个类的实例，只要某个类中有 `-count` 这个方法存在，那么编译器就会认为返回的实例可能是这个有 `-count` 方法的类，所以它不会报错。但是，当运行期去 `MyObject` 类中查找这个方法的时候，才会出现找不到这个方法并且发送和转发失败的crash。关于[运行时](https://juejin.im/post/5a4c40a35188257d1718e447)。

而代码 `y` 处，该类工厂方法返回的是 `instancetype` 类型，该类型即为 `MyObject` 类型，编译器会去它和它的父类中去寻找调用的方法，如果找不到那么就会报错，并且编译失败。

因此，`instancetype` 类型比 `id` 类型有更好的类型安全性，让隐患和错误的暴露提前到代码的编写期，避免了应用的运行时crash。

### 三、类工厂方法使用[self class]实例化而不是类名

例如：

```objc
@interface SuperClass : NSObject
+ (instancetype)factor;
@end

@implementation SuperClass

+ (instancetype)factor {
    return [[SuperClass alloc] init];
}

@end

```

由于类的继承体系，子类也可以调用父类方法，当子类调用父类的这个类工厂方法初始化自身的时候，实际上返回的实例还是父类的实例，而不是子类自身的实例，但是编译器没有办法判断这些，因为它根据类继承体系找到了正确的方法。当向此实例发送子类的消息的时候，会在运行时crash，因为它会从父类实例方法列表中查找这个子类的方法，然而父类并没有这个方法。

```objc
NSLog(@"%@", NSStringFromClass([[SubClass factor] class]));
// log: SuperClass
```

因此，当我们用类工厂方法初始化自身的时候，一定要用 `[self class]` 实例化自身，而不是类名：

```objc
+ (instancetype)factor {
    return [[[self class] alloc] init];
}
```

这样子类就可以通过调用这个父类的类工厂方法初始化自己：

```objc
NSLog(@"%@", NSStringFromClass([[SubClass factor] class]));
// log: SubClass
```

### 四、单例返回类型用类名

某种程度上来说，单例的初始化方法也是一个类工厂方法，单例使用【类名】类型，而不是 `instancetype` 的原因是：一般情况下，不会有其他类继承自单例类，因此，单例类在初始化的时候不用考虑对其子类的影响，因此单例类可以肆无忌惮的使用类名类型作为初始化方法的返回值类型。
