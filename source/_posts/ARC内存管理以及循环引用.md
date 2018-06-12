---
title: ARC内存管理以及循环引用
date: 2018-01-02 16:25:52
tags: iOS 内存管理
---

ARC:"Automatic Reference Counting"，自动引用计数。Swift语言延续了OC的做法，也是利用ARC机制进行内存管理，和OC的ARC一样，当一些类的实例不在需要的时候，ARC会释放它们的内存。但是，在少数情况下，ARC需要知道你的代码之间的关系才能更好的为你管理内存，和OC一样，Swift中的ARC也存在循环引用导致内存泄露的情况。    

## 一、ARC的工作机制

每当我们创建一个类的新的实例的时候，ARC会从堆中分配一块内存用来存储有关该实例的信息。这块内存将持有这个实例的类型信息以及和它关联的属性的值。另外，当这个实例不再被需要的时候，ARC将回收这个实例所占有的内存并且将这部分内存给其他需要的实例用。这样就能保证不再被需要的实例不占用多余的内存。
但是，如果ARC释放了正在使用的实例，那么该实例的属性将不能被访问，方法将不能被调用，如果你访问它的属性或者调用它的方法时，应用会崩溃，因为你访问了一个野指针。
为了解决上述问题，ARC会跟踪每个类的实例正在被多少个属性、常量或者变量引用，每当你将类实例赋值给属性，常量或者变量的时候它就会被"强"引用一次，当它的引用计数为0时，表明它不再被需要，ARC就会销毁它。
下面举个例子介绍ARC是如何工作的
```swift
class Person {
    let name: String
    init(name: String) {
        self.name = name
        print("\(name) is being initialized")
    }
    deinit {
        print("\(name) is being deinitialized")
    }
}
```
上述代码创建了一个名为`Person`的类，该类声明了一个非可选的类型的`name`常量，一个给`name`赋值的初始化方法，并且打印了一句话，用来标注初始化成功，同时声明了一个析构函数，打印了一句标志此实例被销毁的信息。
```swift
var reference1: Person?
var reference2: Person?
var reference3: Person?
```
上述代码声明了三个`Person?`类型的变量，这三个变量为可选类型，所以被自动初始化为`nil`，此时三个实例都没有指向任何一个`Person`类的实例。
```swift
reference1 = Person(name: "John Appleseed")
// Prints "John Appleseed is being initialized"
```
现在创建一个`Person`类的实例，并且赋值给`reference1 `，此时控制台会打印`"John Appleseed is being initialized"`。
```swift
reference2 = reference1
reference3 = reference1
```
然后将该实例赋值给`reference2`和`reference3`。现在该实例被三个"强"类型的指针引用。
```swift
reference1 = nil
reference2 = nil
```
如上所示，当我们将其中两个引用赋值给`nil`的时候，这两个"强"引用被打破，但是这个`Person`的实例并没有被释放（释放信息未打印），因为还存在一个对这个实例的强引用。
```swift
reference3 = nil
// Prints "John Appleseed is being deinitialized"
```
当我们将第三个"强"引用打破的时候（赋值为`nil`），可以看到控制台打印的`"John Appleseed is being deinitialized"`析构信息。
## 二、两个类实例之间的循环引用
上述的例子中，ARC可以很好的获取一个实例的引用计数，并且当它的引用计数为0的时候释放它。但是在实际的开发过程中，会存在一些特殊情况，使ARC没办法得到引用计数为0这个关键点，就会造成这个实例的内存一直不被释放，两个类的实例相互"强"引用就会造成这种情况，就是"循环引用"。
苹果官方提供了两种方法来解决两个实例之间的循环引用，`unowned`引用和`weak`引用。
```swift
class Person {
    let name: String
    init(name: String) { self.name = name }
    var apartment: Apartment?
    deinit { print("\(name) is being deinitialized") }
}
 
class Apartment {
    let unit: String
    init(unit: String) { self.unit = unit }
    var tenant: Person?
    deinit { print("Apartment \(unit) is being deinitialized") }
}
```
这个例子，定义了一个`Person`类和一个`Apartment`类。每一个`Person`的实例都有一个`name`的属性和一个`apartment`的可选属性，初始化为`nil`，因为并不是每一个人都拥有一个公寓，所以是可选属性。同样的，每一个`Apartment`实例都有一个`unit`属性和一个`tenant`的可选属性，初始化为`nil`，同理，不是每一个公寓都有人租。同时，两个类都定义了`deinit`方法，并且打印一段信息，用来让我们清楚这个实例何时被销毁。
```swift
var john: Person?
var unit4A: Apartment?
```
分别定义一个`Person`类型和`Apartment`的变量，定义为`optional`（可选类型），初始化为`nil`。
```swift
john = Person(name: "John Appleseed")
unit4A = Apartment(unit: "4A")
```
然后分别创建一个`Person`类的实例和`Apartment`类的实例，并且分别赋值给上面的定义的变量。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fb4d8b351?w=1240&h=373&f=png&s=13249)
上图为此时变量和实例之间的强引用关系。
然后`john`将拥有一座公寓`unit4A`，公寓`unit4A`将被`john`承租。
```swift
john!.apartment = unit4A
unit4A!.tenant = john
```
因为可以确定两个变量都被赋值为相应类型的实例，所以此处用`!`对可选属性强解包。
此时，两个变量和实例以及两个实例之间的"强"引用关系如下图。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fb5f4f8e6?w=1240&h=396&f=png&s=17746)
从图中可以看到两个实例互相"强"引用，也就是说这两个实例的引用计数永远不会为0，ARC也不会释放这两个实例的内存。
```swift
john = nil
unit4A = nil
```
当我们将两个变量设置为`nil`，切断他们与实例之间的"强"引用关系，此时两个实例之间的"强"引用关系为：

![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fb5d5a579?w=1240&h=396&f=png&s=16319)
从图中可以看出，这两个实例的引用计数仍然不为0，它们占用的内存还是得不到释放，因此就会造成内存泄露。
## 三、解决两个类实例之间的循环引用
Swift提供了两种办法解决类实例之间的循环引用。`weak`引用和`unowned`引用。这两种方法都可以使一个实例引用另一个实例的时候，不用保持"强"引用。`weak`一般应用于其中一个实例具有更短的生命周期，或者可以随时设置为`nil`的情况下；`unowned`用于两个实例具有差不多长的生命周期，或者说两个实例都不能被设置为`nil`。
### (1) weak引用
`weak`引用对所引用的实例不会保持"强"引用的关系。假如一个实例同时被若干个"强引用"和一个`weak`引用引用时，当所有其他的"强"引用都被打破时该实例就会被ARC释放，并且ARC会自动将这个`weak`引用置为`nil`。因此，`weak`引用一般被声明为`var`，因为它会被ARC设置为`nil`。
```swift
class Person {
    let name: String
    init(name: String) { self.name = name }
    var apartment: Apartment?
    deinit { print("\(name) is being deinitialized") }
}
 
class Apartment {
    let unit: String
    init(unit: String) { self.unit = unit }
    weak var tenant: Person?
    deinit { print("Apartment \(unit) is being deinitialized") }
}
```
现在，我们将`Apartment`类中的`tenant`变量声明为`weak`引用（在`var`关键字前加`weak`关键字），表明某公寓的承租人并不一定一直都是同一个人。
```swift
var john: Person?
var unit4A: Apartment?
 
john = Person(name: "John Appleseed")
unit4A = Apartment(unit: "4A")
 
john!.apartment = unit4A
unit4A!.tenant = john
```
然后和上文一样，将两个变量和实例关联。此时，它们之间的引用关系如下图。

![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fb5e115f7?w=1240&h=391&f=png&s=18061)
`Person`实例仍然"强"引用`Apartment`实例，但是`Apartment`实例'weak'引用`Person`实例。`john`和`unit4A`两个变量仍然"强"引用两个实例。当我们把`john`变量对`Person`实例的"强"引用打破的时候，即将`john`设置为`nil`，就没有其他的"强"引用引用`Person`实例，此时，`Person`实例被ARC释放，同时`Apartment`实例的`tenant`变量被设置为`nil`。
```swift
john = nil
// Prints "John Appleseed is being deinitialized"
```
![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fb66e2c4a?w=1240&h=373&f=png&s=12990)
然后将变量`unit4A`设为`nil`，可以看到`Apartment`实例也被销毁。
```swift
unit4A = nil
// Prints "Apartment 4A is being deinitialized"
```

![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fb6247120?w=1240&h=373&f=png&s=10562)
### (2) unowned引用
和`weak`引用一样，`unowned`引用也不会保持它和它所引用实例之间的"强"引用关系，而是保持一种非拥有（或未知）的关系，使用的时候也是用`unowned`关键字修饰声明的变量。不同的是，两个互相引用的对象具有差不多长的生命周期，而不是其中一个可以提前被释放（`weak`），有点患难与共的意思。
Swift要求`unowned`修饰的变量必须一直指向一个实例，而不是有些时候为`nil`，因此，ARC也不会将这个变量设置为`nil`，所以我们一般将这个引用声明为非可选类型。PS：请确保你声明的变量一直指向一个实例，如果这个实例被释放了，而`unowned`变量还在引用它的话，你会得到一个运行时错误，因为，这个变量是非可选类型的。
```swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) {
        self.name = name
    }
    deinit { print("\(name) is being deinitialized") }
}
 
class CreditCard {
    let number: UInt64
    unowned let customer: Customer
    init(number: UInt64, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    deinit { print("Card #\(number) is being deinitialized") }
}
```
上面这个例子定义了两个类：`Customer`和`CreditCard`，每个顾客都可能会有一张信用卡（可选类型），每个信用卡都一定会有一个持有他们的顾客（非可选类型，卡片为顾客定制）。因此，`Customer`类有一个`CreditCard?`类型的属性，`CreditCard`类也有一个`Customer`类型的属性，并且被声明为`unowned`，以此来打破循环引用。每张信用卡初始化的时候都需要一名持有它的顾客，因为信用卡本身就是为顾客定制的。
```swift
var john: Customer?
```
然后声明一个`Customer?`类型的变量`john`，初始化为`nil`。接着创建一个`Customer`的实例，并且将它赋值给`john`（让`john`引用它、指向它都是一个意思）。
```swift
john = Customer(name: "John Appleseed")
john!.card = CreditCard(number: 1234_5678_9012_3456, customer: john!)
```
（第一句代码赋值之后，我们知道`john`肯定不是`nil`，所以用`!`解包不会有问题）
然后，两个实例之间的引用关系为：

![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fdb0430e7?w=1240&h=391&f=png&s=17512)
`Customer`实例"强"引用`CreditCard`实例，`CreditCard`实例'unowned'引用`Customer`实例，接着，我们将`john`对`Customer`实例的"强"引用打破，即将`john`设置为`nil`。
```swift
john = nil
// Prints "John Appleseed is being deinitialized"
// Prints "Card #1234567890123456 is being deinitialized"
```
![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fdc54bb51?w=1240&h=391&f=png&s=17585)
可以看到`Customer`实例和`CreditCard`实例都被销毁了。`john`被设置为`nil`之后，就没有"强"引用引用`Customer`实例，所以，`Customer`实例被释放，也就没有"强"引用引用`CreditCard`实例，因此`CreditCard`实例也被释放。
**以上例子证明，两种方式都可以解决循环引用的问题，但是要注意它们使用的范围。`weak`修饰的变量可以被设置为`nil`（引用的实例的生命周期短于另一个实例），`unowned`修饰的变量必须要指向一个实例（造成循环引用的两实例的生命周期差不多长，不会出现一方被提前释放的情况），一旦它被释放了，就千万别再使用了。**
## 四、闭包引起的循环引用
Swift中的闭包是一种独立的函数代码块，它可以像一个类的实例一样在代码中赋值、调用和传递，也可以被认为某个匿名函数的实例，其实就是OC中的*block*。它和类一样也是引用类型的，所以它的函数体中使用的引用都是"强"引用。
```swift
class HTMLElement {
    
    let name: String
    let text: String?
    
    lazy var asHTML: () -> String = {
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }
    
    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }
    
    deinit {
        print("\(name) is being deinitialized")
    }
    
}
```
上述例子中，闭包被赋值给`asHTML `变量，所以闭包被`HTMLElement `实例"强"引用，而闭包又捕获（关于闭包捕获变量，参考官方文档[Capturing Values](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html#//apple_ref/doc/uid/TP40014097-CH11-ID103)）了`HTMLElement `的实例中的`text`和`name`属性，因此它又"强"引用`HTMLElement `实例，这样就造成了循环引用，因为`text`属性可能为空，所以定义为可选属性。
```swift
var paragraph: HTMLElement? = HTMLElement(name: "p", text: "hello, world")
print(paragraph!.asHTML())
// Prints "<p>hello, world</p>"
```
我们创建一个`HTMLElement `实例，并将它赋值给`paragraph `变量，然后访问它的`asHTML `属性。此时的内存示例为下图，可以看到`HTMLElement `实例和闭包之间的循环引用。
![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fdd973a51?w=1240&h=432&f=png&s=14517)
当我们将`paragraph` 设置为` nil`时，控制台并没有打印任何销毁信息，因为循环引用。
![](https://user-gold-cdn.xitu.io/2018/1/2/160b4e8fdf266510?w=1240&h=844&f=png&s=115887)
上图为使用**Instruments**分析得到的循环引用以及造成的内存泄漏。
## 五、使用unowned和weak解决循环引用
通过上文（三）的分析，我们知道`unowned`引用对实例的非拥有关系，因此，我们可以通过如下方式解决循环引用：
```swift
lazy var asHTML: () -> String = {
        [unowned self] in
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }
```
`[unowned self] in`，这段代码，代表闭包中的`self`指针都被`unowned`修饰。这样就可以使闭包对实例的"强"引用变成'unowned'引用，从而打破循环引用。
当HTML的element为标题的时候，此时如果`text`属性为空，我们想返回一个默认的text作为标题，而不是只有`<h/>`这种标签。
```swift
let heading = HTMLElement(name: "h1")
let defaultText = "some default text"
heading.asHTML = {
    return "<\(heading.name)>\(heading.text ?? defaultText)</\(heading.name)>"
}
print(heading.asHTML())
// Prints "<h1>some default text</h1>"
```
这段代码也会造成`HTMLElement `对其自身的循环引用。我们仍然可以使用`unowned`关键字打破循环引用：
```swift
heading.asHTML = {
    [unowned heading] in
    return "<\(heading.name)>\(heading.text ?? defaultText)</\(heading.name)>"
}
// Prints "<h1>some default text</h1>"
// Prints "h1 is being deinitialized"
```
`unowned`会使闭包中对`heading`的"强"都改为'unowned'引用。
或者，可以使用`weak`属性打破循环引用：
```swift
weak var weakHeading = heading
heading.asHTML = {
    return "<\(weakHeading!.name)>\(weakHeading!.text ?? defaultText)</\(weakHeading!.name)>"
}
// Prints "<h1>some default text</h1>"
//Prints "h1 is being deinitialized"
```
上文（三）中可知，`weak`修饰的变量为可选类型，而且，我们对变量进行了一次赋值，就可以确保`weakHeading`指向`heading`引用的实例，所以可以放心的使用`!`对它解包。
上面这段代码同样可以使闭包对`HTMLElement`实例的"强"引用变为`weak`引用，从而打破循环引用。
（ARC会自动回收不被使用的对象，所以不用手动将变量设置为`nil`）

本文参考[Automatic Reference Counting](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/AutomaticReferenceCounting.html#//apple_ref/doc/uid/TP40014097-CH20-ID48)