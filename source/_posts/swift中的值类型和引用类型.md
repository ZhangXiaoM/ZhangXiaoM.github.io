---
title: swift中的值类型和引用类型
date: 2018-01-02 10:06:30
tags:
---

## 前言 值类型 vs 引用类型
### 1、什么是值类型
值类型就是值直接保存在变量中。例如：

```c
int a = 10;
a = 20;
```
值类型赋新值的时候会直接覆盖旧值。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b5665a4d091ad?w=320&h=94&f=png&s=614)

```c
// 1
int b = a;
// 2
b = 30;
```
按注释：

1.这段代码首先声明了一个`int`类型的变量`b`，然后将`a`中保存的值赋值给`b`。

2.给`b`赋新值，不会影响`a`中保存的值。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b5665a518e28c?w=527&h=94&f=png&s=858)

### 2、什么是引用类型

引用类型，变量中保存的是地址，地址指向实际的对象。例如：

```objc
NSString *foo = @"hello";
// NSString *foo = [[NSString alloc] initWithString:@"hello"];
foo = @"world";
// foo = [[NSString alloc] initWithString:@"world"];
```

引用类型变量重新赋值的时候会改变变量中保存的地址，新的地址会覆盖旧的地址，并且新的地址指向（引用）新创建的对象。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b5665a613c620?w=423&h=294&f=png&s=15000)

```objc
NSMutableString *foo = [NSMutableString stringWithString:@"helllo, "];
// 1
NSMutableString *bar = foo;
// 2
[bar appendString:@"OC"];
NSLog(@"%@", foo);
// print "helllo, OC"
```

1、声明了一个新的变量`bar`，`bar`中保存的地址等于`foo`中保存的地址，因此它们指向同一个对象。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b5665a4667f6a?w=345&h=203&f=png&s=9695)

2、当修改`bar`指向的对象时，`foo`指向的对象也被修改，因为它们俩引用的是同一个对象。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b5665a4eee036?w=375&h=203&f=png&s=10760)

## 一、Swift中的值类型和引用类型

相对于OC，Swift 的优点主要有安全、快速、简洁。Swift的安全性主要体现在，它会在编译时就确定要调用的方法和属性（静态优化），而不是像OC一样到运行时才确定调用哪个方法。Swift会在编译时就告诉你，你的代码是否有bug（比如访问了野指针，数组越界等），而OC会在运行时才发现这些错误并造成APP的crash。

类似于OC中的类，Swift中的类都是引用类型，结构体、枚举、元组都是值类型。

值类型的特点就是拷贝，假如我们创建了一个值类型的实例，当它作为参数传递给函数或者赋值给常量或变量时，这个实例就会生成一份它自身的拷贝（unique copy），当对这个实例进行赋值、参数传递等操作时，实际上被赋值或者传递的是拷贝的那一份，无论怎么修改都不会改变该实例本身。而引用类型的实例进行赋值、参数传递等操作时，实际上传递的就是它本身，我们在函数中修改的就是它自身。类似于 C 函数中的值传递和引用传递。

### 1、引用类型

```swift
// Reference type example
class C { 
	var data: Int = -1 
}
let x = C()
let y = x			    // x is copied to y
x.data = 42				// changes the instance referred to by x (and y)
println("\(x.data), \(y.data)")	// prints "42, 42"
```
上述代码的内存如下图所示，变量`x`和`y`指向同一个类`C`的实例。
![](https://user-gold-cdn.xitu.io/2018/1/2/160b564a2543e978?w=384&h=140&f=png&s=6615)

### 2、值类型

```swift
// Value type example
struct S { 
	var data: Int = -1
}
var a = S()
var b = a						// a is copied to b
a.data = 42						// Changes a, not b
println("\(a.data), \(b.data)")	// prints "42, -1"

```
内存如下图所示，对实例进行赋值操作时，`b = a`，会将`a`的拷贝赋值给`b`，而不是`a`本身，因此，`a`和`b`保存了两个不同的实例，对其中一个的修改不会影响另一个（下图箭头并不是指针的意思，为了表示清楚一点用了箭头）。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b564a1f79f5b2?w=391&h=155&f=png&s=8231)

值类型拷贝操作的时间复杂度为基于被拷贝数据量大小的O(n)。当值类型的实例传递到函数中时，实际上是声明了一个局部变量，它的作用域仅仅是函数的内部，所以当出了函数时，这份拷贝的实例就会被释放，因此不用担心它的空间复杂度或者拷贝浪费内存的问题。

### 3、可变性

对于值类型和引用类型，关键字`var`和`let`是有区别的。对于引用类型来说，`let`的意思是，*引用*必须是常量，换句话你说，你不能修改常量指向其他实例，即不能修改常量中保存的地址，但是你可以修改实例本身。

```swift
let x = C()
let m = C()
x = m //ERROR: note: change 'let' to 'var' to make it mutable
x.data = 55 // It's OK
```
对于值类型来说，`let`的意思是*实例*必须是常量，而常量也是不变的，因此不能修改常量中保存的值，也不能修改值的本身。

```swift
let a = S()
a.data = 2 //ERROR: note: change 'let' to 'var' to make it mutable
let b = S()
a = b //ERROR: note: change 'let' to 'var' to make it mutable
```
因此，相对于引用类型来说，值类型能更好控制实例可变和不可变，例如`NSString `和`NSMutableString `，只需要将字符串声明为`let a = " "`，那么它就是不可变类型的字符串，声明为`var a: String?`，它就是可变字符串。（Swift中的String为值类型）

### 4、如何选择

> If you always get a unique, copied instance, you can trust that no other part of your app is changing the data under the covers. This is especially helpful in multi-threaded environments where a different thread could alter your data out from under you. This can create nasty bugs that are extremely hard to debug.

如果你一直希望持有一个实例的一份独立的备份，即使APP在其他地方修改了这部分数据，你仍然能访问一份原始数据。这在多线程的环境中特别有用，如果其他子线程在你不知道的情况下修改了这个实例，那么你仍然持有一份最原始的数据的备份。

使用值类型的情况：

+ 使用`==`比较两个实例时
+ 当你需要一份独立的备份时
+ 多线程中可能会修改数据时

使用引用类型的情况：

+ 使用`===`比较两个实例时
+ 你需要共享并且修改这个类型的实例时

>`==(Equal To)`表示两个实例的值相等(注意，值类型变量中保存的是值不是地址)；

>`===(Identical To)`表示两个实例完全相等，包括它们在内存中的地址也相同。

在Swift中，`Array ``String ``Dictionary ``Int``Bool`等数据类型都是值类型，换句话说都是`struct`而不是`class`。因此在对它们赋值、传参等操作时，切记它是值传递，而不是引用传递。

```swift
func foo(a: [String])
{
    var b = a
    b.append("haha")
}

var arr: [String] = ["hello", "world"]
foo(a: arr)
print("\(arr)")
// ["hello", "world"]
```

## 二、enum、struct、class

Swift的类型系统：

![](https://user-gold-cdn.xitu.io/2018/1/2/160b564aa578d621?w=480&h=134&f=png&s=12300)

Named type（命名类型）是指在定义时，可以使用特定的名字去命名的类型。例如：

```
let foo: Double = 0.0
// Double: 特定的命名类型
```
Named model type（命名模型类型）是指可以在声明的时候可以重写`setter`或者`getter`的类型。例如：

假如你有一个存储*半径*的变量，然后要声明一个存储*直径*的变量：

```swift
var radius = 5.0;
var diameter: Double {
	get {
		return radius * 2
	}
	set {
		radius = newValue / 2
	}
}
```

Compound type（复合类型）是指没有特定命名的类型。例如：

```swift
// declare a tuple 
let foo: (String, [String]) = ("foo", ["bar"])
// not
let foo: Tuple = ("foo", ["bar"]) 
// error
```

### 1、enum
#### (1)、初始值
枚举类型的每一个选项都可以带有一个初始值，和OC中的枚举类型一样，不同的是，Swift允许你指定这个初始值的类型，并且给每一个选项赋值。

```swift
enum Color: String {
    case black = "black"
    case gray  = "gray"
    case blue  = "blue"
    case white = "white"
}

// 省略写法
//enum Color: String {
//    case black ,gray, blue, white
//}

let black = Color.black.rawValue
print(black)
// black
```

#### (2)关联值
Swift中的枚举类型的每一个选项都允许你传入一个变量作为它的关联值，你可以在`switch`它的时候使用这个变量。例如：

```swift
enum CSColor {
// 声明关联值的类型
    case named(Color)
    case rgb(UInt8, UInt8, UInt8) // 0-255
}
// CustomStringConvertible：这个协议的作用就是用字面量来描述一个类、结构体或者枚举
extension CSColor: CustomStringConvertible {
    var description: String {
        get {
            switch self {
            case .named(let name):
                return name.rawValue
            case .rgb(let r, let g, let b):
                return String.init(format: "0x%02X%02X%02X", r, g, b)
            }
        }
    }
}
let color = CSColor.rgb(255, 255, 255)
// 0xFFFFFF
```
#### *（3）Optional
`Optional`在Swift中首先是个神奇的东西，其次是个非常重要的东西，理解它对于学习Swift重中之重。
它的声明：

```swift
enum Optional<T> {
    case none      // 没有值
    case some(T)   // 有类型为T的某个值
}
```
`Optional`类型其实是一个枚举类型，它只有两个选项，要么是`none`，要么是某个`T`类型的值`some`。

##### （I）关于`?`

`?`的意思是声明的变量是个`Optional`类型。例如：

```swift
var foo: Double？
```
声明一个Double类型的变量，可以不赋值或者赋初值为`nil`，否则必须初始化它，如果一个变量不是`Optional`类型，那它永远不可能是`nil`。

当我们在使用`?`解包的时候，它的执行过程应该为（伪代码）：

```swift
// 1
var bar: String? = "BAR"
// 2
let x = bar?.lowercased
```

```swift
// 1 伪代码
var bar: Optional<String> = "bar"
// 2 ?的作用
switch bar {
case .none:
    bar = nil
case .some(let xx):
    bar = xx
}

if bar == nil {
    return nil
} else {
    return bar.lowercased
}

```
如果`bar = nil`，则返回`nil`，如果`bar != nil`，则返回`bar`的值，并且调用`lowercased`方法。当然返回值赋值给`x`，`x`也是`Optional`类型，因为它也可能为`nil`。

##### （II）关于`!`

`!`的作用是强制解包的意思（unwrap），即，将`Optional`类型的变量强制解包为某个类型的值。如果这个值为`nil`，仍然强制解包的话，应用将会在运行时crash。所以如果不能确定变量的值是一定不是`nil`的话，千万不要用`!`解包。这种方式可以保证应用不会crash：

```swift
var bar: String?
var x: String?
if bar != nil {
    x = bar!.lowercased
}
print(x)
```

##### （III）关于`nil`
OC中的`nil`代表空指针的意思，就是C中的`NULL`，它的定义为：

```swift
// MacType.h
#ifndef NULL
#define NULL    __DARWIN_NULL
#endif /* ! NULL */
#ifndef nil
  #if defined(__has_feature) 
    #if __has_feature(cxx_nullptr)
      #define nil nullptr // nullptr 空指针常量
    #else
      #define nil __DARWIN_NULL
    #endif
  #else
    #define nil __DARWIN_NULL
  #endif
#endif
```
```c++
// C
#define NULL (void *)0
```

在OC中我们可以向`nil`发送消息（调用方法），因为OC是一门动态的语言，在编译期它并不关心一个对象是不是`nil`，它只关心这个类型的对象能不能接收这条消息。它会在运行时判断消息的接收者是不是`nil`，如果是`nil`，它会根据消息的返回类型返回对应的值，例如要求返回`int`类型，返回值就是`0`，`BOOL`类型，返回值就是`NO`，`id`类型，返回值就是`nil`，函数将直接返回。

Swift中的`nil`，并不是空指针的意思，因为Swift并不是一门纯面向对象的语言，大大减少了对引用类型的依赖，使用较多的是值类型，因此空指针对值类型来说，并没有什么意义。因此，`nil`表示的是*没有值*（the absence of a value）即，这块内存的这个变量中没有保存任何值。所以它在Swift中的定义为：

```swift
typealias nil = Optional.none 
```
### 2、struct & class

Swift中的类都是引用类型，结构体都是值类型。

> Comparing Classes and Structures (类和结构体的异同点)

> Classes and structures in Swift have many things in common. Both can (共同点):

> Define properties to store values. (声明属性)

> Define methods to provide functionality. (声明方法)

> Define subscripts to provide access to their values using subscript syntax. (使用点语法)

> Define initializers to set up their initial state. (使用构造方法初始化)

> Be extended to expand their functionality beyond a default
> implementation. (扩展默认实现以外的功能)

> Conform to protocols to provide standard functionality of a certain kind. (使用协议)

> For more information, see [Properties](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Properties.html#//apple_ref/doc/uid/TP40014097-CH14-ID254), [Methods](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Methods.html#//apple_ref/doc/uid/TP40014097-CH15-ID234), [Subscripts](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Subscripts.html#//apple_ref/doc/uid/TP40014097-CH16-ID305), [Initialization](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Initialization.html#//apple_ref/doc/uid/TP40014097-CH18-ID203), [Extensions](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Extensions.html#//apple_ref/doc/uid/TP40014097-CH24-ID151), and [Protocols](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID267). (更多信息)

> Classes have additional capabilities that structures do not (类比结构体多的能力):

> Inheritance enables one class to inherit the characteristics of another. (类可以继承)

> Type casting enables you to check and interpret the type of a class instance at runtime. (类型转换使您能够在运行时检查和解释实例的类型)

> Deinitializers enable an instance of a class to free up any resources it has assigned. (提供析构方法来释放一个类占用的资源)

> Reference counting allows more than one reference to a class instance. (引用计数允许存在多个对实例的引用)