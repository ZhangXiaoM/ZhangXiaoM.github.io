---
title: 'Swift集合函数:Reduce、Map、FlatMap、Filter详解'
date: 2018-01-02 11:20:58
tags: iOS API 详解
---

## Reduce
声明
```swift
func reduce<Result>(_ initialResult: Result, _ nextPartialResult: (Result, Element) throws -> Result) rethrows -> Result
```
> Returns the result of combining the elements of the sequence using the given closure.

> 使用给定的block来组合集合中的元素，并且返回组合后的结果

参数
> initialResult

> The value to use as the initial accumulating value. initialResult is passed to nextPartialResult the first time the closure is executed.

> 初始值

> nextPartialResult

> A closure that combines an accumulating value and an element of the sequence into a new accumulating value, to be used in the next call of the nextPartialResult closure or returned to the caller.

> 带有两个参数的block，block的第一个参数为之前的计算结果，如果是第一次计算，则默认为initialResult。第二个参数为集合中的下一个元素。返回值为遍历完整个集合后的组合结果。block中可以定义两个参数的组合规则。

例如
```swift
let da = [1, 2, 3, 4, 5]
let sum = da.reduce(0) { (result: Int, ele: Int) -> Int in
	return result + ele
}
// sum = 15
```
对数组`da`中的元素求和，第一个参数`0`为初始值，当数组第一次执行`result + ele //ele = data[0]`时，此时的`result`即为初始值。

可简写为
```swift
let da = [1, 2, 3, 4, 5]
let sum = da.reduce(0) {
	return $0 + $1
}
// sum = 15
```
或者
```swift
let da = [1, 2, 3, 4, 5]
let sum = da.reduce(0, +)
// sum = 15
```
注：Swift中操作符为函数，函数允许使用和block具有相同参数的函数作为参数代替block

上述代码的作用相当于
```swift
let da = [1, 2, 3, 4, 5]
var sum = 0
for i in 0..<da.count {
	sum += da[i]
}
// sum = 15
```
## Map & FlatMap
### Map
> Returns an array containing the results of mapping the given closure over the sequence’s elements.

> 使用给定的block将数组映射为一个新的数组。新的数组元素为block中设置的映射规则确定。 

参数：
> transform

> A mapping closure. transform accepts an element of this sequence as its parameter and returns a transformed value of the same or of a different type.

> 一个block，参数为数组中的一个元素，block返回一个使用映射规则转换之后的元素。

例如
```swift
let cast = ["Vivien", "Marlon", "Kim", "Karl"]
let lowercaseNames = cast.map { $0.lowercased() }
// 'lowercaseNames' == ["vivien", "marlon", "kim", "karl"]
let letterCounts = cast.map { $0.count }
// 'letterCounts' == [6, 6, 3, 4]
```
### FlatMap

`flatMap(_:)`和`map(_:)`一样，也是可以将一个集合通过某种映射规则映射为另一个集合，不同的地方是，`flatMap(_:)`会将映射之后的元素强解包（unwraped），如果遇到`nil`，则将其过滤到，返回的新集合为过滤掉`nil`之后的集合；`map(_:)`返回的元素为`Optional`类型元素，如果集合中有`nil`，则返回的集合中也包·括`nil`，其他元素均为`Optional`类型。
例如：
```swift
let cast = [nil, "Marlon", "Kim", "Karl"]
let uppercaseNames = cast.map { $0?.uppercased() } //1
// 'uppercaseNames' = [nil, Optional("MARLON"), Optional("KIM"), Optional("KARL")]
let foo = cast.flatMap { $0?.uppercased() }//2
// 'foo' = ["MARLON", "KIM", "KARL"]
```
1、将数组`cast`映射为一个新的数组，给定的映射规则是`$0?.uppercased()`，即将数组中的字符串全部转换成大写的形式，返回的新数组的元素为`optional`类型。

2、将数组`cast`映射为一个新的数组，给定的映射规则是`$0?.uppercased()`，即将数组中的字符串全部转换成大写的形式，并且将返回的数组元素解包， 如果元素为`nil`，则过滤掉。
### Filter
> Returns an array containing, in order, the elements of the sequence that satisfy the given predicate.

> 返回一个数组，该数组按顺序包含满足给定谓词的序列元素。

参数
> isIncluded

> A closure that takes an element of the sequence as its argument and returns a Boolean value indicating whether the element should be included in the returned array.

> 一个block，将集合的元素作为参数，并返回一个Boolean，用来指示这个元素是否包含在返回的数组中，如果包括，则将其添加入数组。

例如
```swift
let cast = ["Vivien", "Marlon", "Kim", "Karl"]
let shortNames = cast.filter { $0.count < 5 }
print(shortNames)
// Prints "["Kim", "Karl"]"
```
上述代码的作用为：筛选出名字的字数小于5的名字，`{ $0.count < 5 }`这个block为筛选的谓词，`$0`为block的参数，即`shortNames`的元素。
