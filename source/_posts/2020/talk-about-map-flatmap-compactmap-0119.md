---
title: "细谈map，flatMap和compactMap"
catalog: true
toc_nav_num: true
date: 2020-01-19 20:01:29
subtitle: "Talk About map, flatMap and compactMap"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

# 前言

`map`是Swift中常用的函数，同时它还有两个兄弟级函数`flatMap`和`compactMap`，下面就根据官方声明和源码来谈谈这个三个函数之间的联系和区别，加以深刻记忆。

<!-- more -->

# map

首先是`map`函数的使用：

``` swift
let nums = [1, 2, 3]
let res = nums.map { $0 + 1 }
print(res) // [2, 3, 4]
```

`map`函数接受一个闭包函数作为参数，他会遍历整个序列元素，并对每个元素执行闭包函数中定义的操作。

然后我们再看下`map`函数的官方声明：

``` swift
func map<T>(_ transform: (Value) throws -> T) rethrows -> [T]
```

我们来稍微解释下这个声明的意思，所需参数`transform`的类型很明显是一个闭包类型：将`(Value)`（序列元素的类型）转换为类型`T`，最终返回一个`T`类型的序列。

这就是`map`的初步了解，我们再看下一个。

# flatMap

`flatMap`的声明和使用有两种，让我们分别来讲解一下。

先看第一种使用方法：

``` swift
let nums = [[1, 2], [3, 4]]
let mapRes = nums.map { $0.map { $0 + 1 } }
let flatRes = nums.flatMap { $0.map { $0 + 1 } }
print(mapRes)  // [[2, 3], [4, 5]]
print(flatRes) // [2, 3, 4, 5]
```

这样我们就看出`map`和`flatMap`的区别了。

我们看下`map`函数的官方声明：

``` swift
func flatMap<SegmentOfResult>(_ transform: (Value) throws -> SegmentOfResult) rethrows -> [SegmentOfResult.Element] where SegmentOfResult : Sequence
```

这个声明比`map`相比就长了很多，但我们可以将它转化一下：

``` swift
func flatMap<T>(_ transform: (Value) throws -> T) rethrows -> [T.Element] where T : Sequence
```

这么看其实就已经和`map`的声明差不多了，同时不同的地方表现在以下几个方面：`T`的类型有了约束，必须是继承`Sequence`协议，因为它的返回值也变成了`[T.Element]`，这个地方的关键点就是造成与`map`函数不同的地方。

然后我们看下`flatMap`[源码部分](https://github.com/apple/swift/blob/master/stdlib/public/core/FlatMap.swift)：

``` swift
extension LazySequenceProtocol {
  /// Returns the concatenated results of mapping the given transformation over
  /// this sequence.
  ///
  /// Use this method to receive a single-level sequence when your
  /// transformation produces a sequence or collection for each element.
  /// Calling `flatMap(_:)` on a sequence `s` is equivalent to calling
  /// `s.map(transform).joined()`.
  ///
  /// - Complexity: O(1)
  @inlinable // lazy-performance
  public func flatMap<SegmentOfResult>(
    _ transform: @escaping (Elements.Element) -> SegmentOfResult
  ) -> LazySequence<
    FlattenSequence<LazyMapSequence<Elements, SegmentOfResult>>> {
    return self.map(transform).joined()
  }
}
```

我们通过源码可以看出，`flatMap`函数本身也是调用了`map`函数，同时进行了`joined`操作，`joined`操作可以将序列包裹序列的元素重新组合成一个序列（只能拆第二层的序列，第三层的依旧是原类型），所以这就是`flatMap`函数的不同点。

`flatMap`函数的另一种使用方式其实已经弃用，功能已经划分给`compactMap`函数，但我们可以看下它的声明：

``` swift
@available(swift, deprecated: 4.1, renamed: "compactMap(_:)", message: "Please use compactMap(_:) for the case where closure returns an optional value")
func flatMap<ElementOfResult>(_ transform: (Value) throws -> ElementOfResult?) rethrows -> [ElementOfResult] // Deprecated
```

下面来看下`compactMap`函数。

# compactMap

使用方法：

``` swift
let nums = [1, 2, nil, 3, 4]
let res = nums.compactMap { $0 }
print(res) // [1, 2, 3, 4]
```

如果使用`map`函数实现上面代码，会直接报错无法运行，但是`compactMap`函数不会报错，还会把`nil`剔除，同时可以将可选类型解包。这在实际场景中是个非常有用的机制，可以用条件闭包将非法或无效的内容过滤。

然后我们先来看下`compactMap`的官方声明：

``` swift
func compactMap<ElementOfResult>(_ transform: (Value) throws -> ElementOfResult?) rethrows -> [ElementOfResult]
```

将它转化一下：

``` swift
func compactMap<T>(_ transform: (Value) throws -> T?) rethrows -> [T]
```

和`map`函数不同地方就在于`transform`闭包参数的返回值是`T`的可选类型，最终的返回结果是解包后的`[T]`。

我们在根据[源码](https://github.com/apple/swift/blob/master/stdlib/public/core/FlatMap.swift)分析下它是如何筛选并解包的：

``` swift
extension LazySequenceProtocol {
  /// Returns the non-`nil` results of mapping the given transformation over
  /// this sequence.
  ///
  /// Use this method to receive a sequence of non-optional values when your
  /// transformation produces an optional value.
  ///
  /// - Parameter transform: A closure that accepts an element of this sequence
  ///   as its argument and returns an optional value.
  ///
  /// - Complexity: O(1)
  @inlinable // lazy-performance
  public func compactMap<ElementOfResult>(
    _ transform: @escaping (Elements.Element) -> ElementOfResult?
  ) -> LazyMapSequence<
    LazyFilterSequence<
      LazyMapSequence<Elements, ElementOfResult?>>,
    ElementOfResult
  > {
    return self.map(transform).filter { $0 != nil }.map { $0! }
  }
}
```

非常容易理解，同样是调用了`map`函数，并调用`filter`函数进行非空筛选，最终进行解包。这也就不难理解`compactMap`函数的实现结果了。

# 结语

将这个三个函数剖析之后更加利于我们对于函数的理解，在实际场景中也能更好地根据对应情况使用不同的函数。