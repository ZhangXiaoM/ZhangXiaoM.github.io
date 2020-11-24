---
title: 深入理解copy关键字
date: 2017-08-30 10:26:27
tags: iOS 内存管理
---

我们在声明 `NSString`、`NSArray `等具有可变子类的属性时，一般都会用 `copy` 关键字来指定它的特质。

## 为什么要用 copy
> `copy` 关键字的解释是：此特质所表达的所属关系与 `strong`（为属性设置新值时，先保留新值，并释放旧值，然后再将新值设置上去）类似。不同的是，设置新值时，不保留新值，而是将其“拷贝”。

看上去并没有什么大不了的，拷贝与否好像和设置成功不成功的关系不大！但是，传递给设置方法的新值可能是一个可变类的实例（父类指针指向子类对象），例如，给类型为 `NSString` 的属性设置新值时，可能会传递一个`NSMutableString`的实例。此时如果不拷贝字符串，设置完属性后，字符串的值可能会在对象不知道的情况下遭人更改。所以要拷贝一份"不可变"的字符串，确保对象中的字符串不会无意间被改变。
## 理解copy的作用
举个例子来说：
```objc
@property (nonatomic, strong) NSString *aString;

self.aString = @"test string";
NSMutableString *mutableString = [[NSMutableString alloc] initWithString:@"hello world"];
self.aString = mutableString;
[mutableString setString:@"hahaha"];
NSLog(@"%@",_aString);
//print: hahaha
```
我们首先声明了一个不可变字符串类型的属性 `aString`，并且给它赋了一个值，然后初始化了一个可变的字符串 `mutableString`，由于OC这门语言的多态性，即**父类指针可以指向子类对象**，因为 `NSMutableString` 是 `NSString` 的子类，所以，我们可以将不可变字符串类型的指针 `self.aString` 指向可变字符串类型的对象 `mutableString`，然后当我们改变 `mutableString` 的值的时候，此时不可变字符串 `self.aString` 的值也随之改变，假如我们再其他较多的地方也使用了 `self.aString` 这个对象，那么该对象的值就会在我们不知情的情况下遭到修改，然后就会造成多处使用错误。
下面我们对上述代码稍作修改：
```objc
self.aString = @"test string";
NSMutableString *mutableString = [[NSMutableString alloc] initWithString:@"hello world"];
self.aString = [mutableString copy];
[mutableString setString:@"hahaha"];
NSLog(@"%@",_aString);
//print: hello world
```
在将可变字符串赋值给  `self.aString`  的时候，进行一步copy的操作，此时可以看到打印的信息为 `hello world` ，这就说明，当我们修改可变对象的时候没对不可变对象造成任何影响。
为了避免每次赋值都进行一次 copy 操作，有的时候也会遗漏，我们可以在声明属性的时候，使用 `copy` 关键字，`copy` 关键字的作用就是每当该属性被赋值的时候，都进行一次拷贝，这样就可以保证这个对象永远不会因为其可变子类对象的修改而被修改。例如：
```objc
@property (nonatomic, copy) NSString *aString;

self.aString = @"test string";
NSMutableString *mutableString = [[NSMutableString alloc] initWithString:@"hello world"];
self.aString = mutableString;
[mutableString setString:@"hahaha"];
NSLog(@"%@",_aString);
//print: hello world
```
使用 `copy` 关键字就可以保证该属性指向的值永远为不可变对象。

## copy 的原理

那么 `copy` 关键字怎么保证我们每次辅助的时候都进行一次拷贝的n呢？

假如我们声明一个 `strong` 修饰的属性：

```objc
@property (nonatomic, strong) NSString *aString;
```

那么它被合成的 `-setter` 方法为：

```objc
// ARC 下
- (void)setAString:(NSString *)aString {
    _aString = aString;
}
```

 ```objc
// MRC 下
- (void)setAString:(NSString *)aString {
    // retain 新值
    id tmp = [aString retain];
    // 释放旧值
    [_aString release];
    // 设置新值
    _aString = tmp;
}
 ```

MRC 下的 `-setter` 方法可以让我们清晰的看到 `strong` 关键字的作用。那么 `copy` 和 `strong` 的不同点在哪呢?

如下代码：

```objc
@property (nonatomic, copy) NSString *aString;
```

它被合成的 `-setter` 方法为：

```objc
// ARC
- (void)setAString:(NSString *)aString {
    _aString = [aString copy];
}
```

```objc
// MRC
- (void)setAString:(NSString *)aString {
    // 拷贝新值
    id tmp = [aString copy];
    // 释放旧值
    [_aString release];
    // 设置
    _aString = tmp;
}
```

不同点在于它会将新值的拷贝设置给属性，而不是直接 `retain`。这就可以确保该属性持有的那一份一直都是被设置的对象的拷贝，所以无论怎么修改被设置的对象，都不会影响它。
