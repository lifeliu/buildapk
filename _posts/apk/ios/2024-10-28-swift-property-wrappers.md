---
layout: post
title: Swift属性包装器完全指南
categories: ios
tags: [属性包装器, PropertyWrapper]
date: 2024-10-28
---

![title](https://image.sideproject.cn/titlex/titlex_053.jpg)

## 引言

属性包装器（Property Wrapper）是 Swift 5.1 引入的强大特性，用于在属性访问时添加额外逻辑。通过 `@propertyWrapper` 和 `wrappedValue`，可以封装取值、赋值、校验、持久化等通用行为，减少重复代码。SwiftUI 中的 `@State`、`@Published`、`@AppStorage` 等都基于属性包装器实现。本文将介绍属性包装器的语法、如何创建自定义包装器、`projectedValue` 的用法，以及常见应用场景和最佳实践。

## 属性包装器的作用

在业务代码中，许多属性需要相同的辅助逻辑：例如数值需要限制在某个范围内、字符串需要去除首尾空格、某些值需要自动持久化到 UserDefaults。若每个属性都手写 getter/setter，代码会冗长且易出错。属性包装器将这类逻辑封装到包装类型中，使用时只需在属性前加 `@WrapperName`，即可自动应用封装逻辑，使代码更简洁、行为更一致。

## 基础定义

属性包装器类型需要提供 `wrappedValue` 属性，用于实际的存储和访问。初始化器可接受 `wrappedValue` 用于默认值。下面的 `Clamped` 包装器确保值始终落在指定范围内，超出时自动截断。

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    let range: ClosedRange<Value>
    
    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
    
    var wrappedValue: Value {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

// 使用
@Clamped(0...100) var score: Int = 50
score = 150  // 实际存储 100
```

`@Clamped(0...100)` 是包装器的初始化调用，`50` 是 `wrappedValue` 的默认值。对 `score` 的读写会经过 `wrappedValue` 的 getter/setter，从而自动应用范围限制。

## UserDefaults 包装器

将 UserDefaults 的存取封装为属性包装器，可以避免到处写 `UserDefaults.standard.string(forKey:)` 和 `set(_:forKey:)`，同时支持类型安全和默认值。注意：UserDefaults 原生支持的类型（如 String、Int、Bool）可直接存储；自定义类型需转为 Data 或使用 PropertyList 编码。

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    
    var wrappedValue: T {
        get {
            UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}

@UserDefault("theme", defaultValue: "light")
var theme: String
```

此包装器适用于简单类型。若 T 为可选类型，需要额外处理 `nil` 的存储与读取，以区分"未设置"和"显式设为 nil"。

## projectedValue 与 $ 语法

属性包装器可提供 `projectedValue`，通过 `$propertyName` 访问。SwiftUI 的 `@State` 的 `projectedValue` 是 `Binding`，因此可以用 `$count` 获取绑定传递给子视图。自定义包装器也可暴露 `projectedValue`，例如暴露校验错误、是否被修改等附加信息。

## 总结

属性包装器能够封装通用逻辑，减少重复代码，是 Swift 和 SwiftUI 开发中的重要工具。合理设计包装器可使业务代码更清晰，同时保持类型安全和可测试性。创建包装器时注意泛型约束、默认值和 `projectedValue` 的语义，以便在不同场景下复用。
