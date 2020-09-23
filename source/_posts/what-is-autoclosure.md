---
title: 什么是@autoclosure
date: 2020-06-07 14:24:39
tags:
    - swift
categories:
---

# Difference between '&&' and ','

本篇源自之前在Swift Forum上看到的一篇讨论: [Is “&&” equal to “,”?](https://forums.swift.org/t/is-equal-to/31121)：Swift里的`&&`和`,`有什么区别?

```swift
var a: Bool
var b: Bool

if a && b {
    //todo
}

if a, b {
    //todo
} 
```
首先这两种写法在语义上应该是完全一样的，那有什么区别的？
<!-- more -->
看下面这个例子
```swift
class TestBoolOperator {
    let a: Bool
    let b: Bool
    init(_ c: Bool) {
        a = true
        //if c, a { // work fine 
        //if a && c { // // work fine 
        if c && a { // error: 'self' captured by a closure before all members were initialized
            print(true)
        }
        b = false
    }
}
TestBoolOperator.init(true)
```

TestBoolOperator有两个未初始化的成员变量a, b，在初始化方法中,传入了布尔值c。用`if a && c` 和 `if a, c`这两种写法的时候都是可以的，但是第三种写法`if c && a`编译器会报错：`'self' captured by a closure before all members were initialized`。这里只是交换了下c和a的位置，并没有写任何closure的代码，为什么会报self被closure捕获？`if c && a` 和 `if c && a`的区别在哪？

# @autoclosure

我们去源码中找答案
[Bool.swift](https://github.com/apple/swift/blob/master/stdlib/public/core/Bool.swift)
```swift
/// Performs a logical AND operation on two Boolean values.
  ///
  /// The logical AND operator (`&&`) combines two Boolean values and returns
  /// `true` if both of the values are `true`. If either of the values is
  /// `false`, the operator returns `false`.
  ///
  /// This operator uses short-circuit evaluation: The left-hand side (`lhs`) is
  /// evaluated first, and the right-hand side (`rhs`) is evaluated only if
  /// `lhs` evaluates to `true`. For example:
  ///
  ///     let measurements = [7.44, 6.51, 4.74, 5.88, 6.27, 6.12, 7.76]
  ///     let sum = measurements.reduce(0, combine: +)
  ///
  ///     if measurements.count > 0 && sum / Double(measurements.count) < 6.5 {
  ///         print("Average measurement is less than 6.5")
  ///     }
  ///     // Prints "Average measurement is less than 6.5"
  ///
  /// In this example, `lhs` tests whether `measurements.count` is greater than
  /// zero. Evaluation of the `&&` operator is one of the following:
  ///
  /// - When `measurements.count` is equal to zero, `lhs` evaluates to `false`
  ///   and `rhs` is not evaluated, preventing a divide-by-zero error in the
  ///   expression `sum / Double(measurements.count)`. The result of the
  ///   operation is `false`.
  /// - When `measurements.count` is greater than zero, `lhs` evaluates to
  ///   `true` and `rhs` is evaluated. The result of evaluating `rhs` is the
  ///   result of the `&&` operation.
  ///
  /// - Parameters:
  ///   - lhs: The left-hand side of the operation.
  ///   - rhs: The right-hand side of the operation.
  @_transparent
  @inline(__always)
  public static func && (lhs: Bool, rhs: @autoclosure () throws -> Bool) rethrows
      -> Bool {
    return lhs ? try rhs() : false
  }
```
通过注释可以看到，Swift针对布尔值的&&操作，使用的是[`短路求值(short-circuit evaluation)`](https://en.wikipedia.org/wiki/Short-circuit_evaluation)，即首先求左操作数的值，只有当左操作数的值未真时才去求右操作数的值。一般来说，短路求值的意义在于减少不必要的计算和调用，或者防止调用可能产生错误和异常的代码。

Swift实现这种短路求值的方式是使用了@autoclosure，把右操作数用@autoclosure标识一下，会自动把右操作数包装成一个闭包，左操作数值为false时，才真正的调用这个闭包求值：`lhs ? try rhs() : false`

@autoclosure还在Swift源码中有很多使用的例子：比如assert, ??

```swift
@_transparent
public func assert(
  _ condition: @autoclosure () -> Bool,
  _ message: @autoclosure () -> String = String(),
  file: StaticString = #file, line: UInt = #line
) {
  // Only assert in debug mode.
  if _isDebugAssertConfiguration() {
    if !_fastPath(condition()) {
      _assertionFailure("Assertion failed", message(), file: file, line: line,
        flags: _fatalErrorFlags())
    }
  }
}
```

??
```swift
@_transparent
public func ?? <T>(optional: T?, defaultValue: @autoclosure () throws -> T)
    rethrows -> T {
  switch optional {
  case .some(let value):
    return value
  case .none:
    return try defaultValue()
  }
}
```
需要注意的是@autoclosure不支持带参数的闭包。

# [short-circuit evaluation](https://en.wikipedia.org/wiki/Short-circuit_evaluation)
相比`&&`, `,`操作符自身就带有短路求值的含义，比如下面这段代码，虽然对一个值为空的optional变量进行了强解包，也不会有任何问题。
```swift
var a: Int? = nil
if a != nil, a! == 1 {
    print(1)
} else {
    print(a) // "nil"
}
```


# more

`&&`和`,`还有什么其它区别？一个就是上面说的，&&会生成右操作数的closure，在使用上需要注意。还有一个是使用`,`不用考虑优先级问题，`,`总是有最高的优先级