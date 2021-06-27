---
layout: post
title: Swfit中description和debugDescription的一个特性
date: 2020-10-12 18:49:00.000000000 +09:00

---

# 背景
在Swift开发过程中遇到了这样一个现象：
一个类型同时实现了`CustomStringConvertible和CustomDebugStringConvertible`，输出时作为一个整体和作为`struct、enum、tuple`的参数是不同的，具体如下

```swift
struct MyParameters {
    let a: String
}

enum MyEnum {
    case x(MyParameters)
}

extension MyParameters: CustomStringConvertible, CustomDebugStringConvertible {
    public var description: String {
        return "Parameters"
    }

    public var debugDescription: String {
        return "DebugParameters"
    }
}

func display() {
    let param = MyParameters(a: "a")
    let myEnum = MyEnum.x(param)
    print(param)
    print(myEnum)
}
```



调用`display()`的结果为

```swift
x(DebugParameters)
Parameters
```



同理转化为String也是这样

```swift
let paramString = "\(myEnum)" // Parameters
let myEnumString = "\(myEnum)" // x(DebugParameters)
```

为什么会出现这个现象呢？

# 解析
通过断点观察两个不同的堆栈
**作为整体：**

![整体](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2020-10-12-Swfit中description和debugDescription的一个特性/整体.png "整体")

通过`print_unlocked<A, B>(:_:) ()`直接输出

**作为struct、enum、tuple的参数：**

![参数](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2020-10-12-Swfit中description和debugDescription的一个特性/参数.png "参数")

- _print_unlocked<A, B>(:_:) ()
- _adHocPrint_unlocked<A, B>(:::isDebugPrint:) ()
- _debugPrint_unlocked<A, B>(:_:) ()

通过上述关键字我们可以找到[源码](https://github.com/apple/swift/blob/main/stdlib/public/core/OutputStream.swift)，看一下实现

```swift
internal func _print_unlocked<T, TargetStream: TextOutputStream>(
  _ value: T, _ target: inout TargetStream
) {
  if _openExistential(type(of: value as Any), do: _isOptional) {
    let debugPrintable = value as! CustomDebugStringConvertible
    debugPrintable.debugDescription.write(to: &target)
    return
  }

  if let string = value as? String {
    target.write(string)
    return
  }

  if case let streamableObject as TextOutputStreamable = value {
    streamableObject.write(to: &target)
    return
  }

  if case let printableObject as CustomStringConvertible = value {
    printableObject.description.write(to: &target)
    return
  }

  if case let debugPrintableObject as CustomDebugStringConvertible = value {
    debugPrintableObject.debugDescription.write(to: &target)
    return
  }

  let mirror = Mirror(reflecting: value)
  _adHocPrint_unlocked(value, mirror, &target, isDebugPrint: false)
}
```



显然，当我们直接输出MyParameters时，它是满足CustomStringConvertible协议的，优先取description，符合预期

但当MyParameters作为MyEnum的参数时
同时MyEnum
- 不是optional
- 不是String
- 没有满足TextOutputStreamable
- 没有满足CustomStringConvertible
- 没有满足CustomDebugStringConvertible
就会进入到_adHocPrint_unlocked<A, B>(:::isDebugPrint:) ()中

而`_adHocPrint_unlocked`中通过`Mirror`判断类型
- 如果是optional，则判断是否为空，如果空则输出nil，否则进入到_debugPrint_unlocked
- 如果是tuple、struct，则处理一下输出格式，遍历取值进入到_debugPrint_unlocked
- 如果是enum，则取值进入到_debugPrint_unlocked

```swift
public func _debugPrint_unlocked<T, TargetStream: TextOutputStream>(
    _ value: T, _ target: inout TargetStream
) {
  if let debugPrintableObject = value as? CustomDebugStringConvertible {
    debugPrintableObject.debugDescription.write(to: &target)
    return
  }

  if let printableObject = value as? CustomStringConvertible {
    printableObject.description.write(to: &target)
    return
  }

  if let streamableObject = value as? TextOutputStreamable {
    streamableObject.write(to: &target)
    return
  }

  let mirror = Mirror(reflecting: value)
  _adHocPrint_unlocked(value, mirror, &target, isDebugPrint: true)
}
```



最终都会进入到`_debugPrint_unlocked`
而`_debugPrint_unlocked`中判断`CustomDebugStringConvertible`、`CustomStringConvertible`、`TextOutputStreamable`，如果都不满足则生成新的`Mirror`再次进到`_adHocPrint_unlocked`中

而`_debugPrint_unlocked`中判断`CustomDebugStringConvertible`、`CustomStringConvertible`、`TextOutputStreamable`的优先级和`_print_unlocked`是相反的，即优先取`debugDescription`

这也是为什么当一个类型同时实现了`CustomDebugStringConvertible`和`CustomStringConvertible`时，作为一个整体和作为`struct、enum、tuple`的一部分表现是不同的