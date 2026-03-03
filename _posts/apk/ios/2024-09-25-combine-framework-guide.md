---
layout: post
title: iOS Combine 响应式编程框架完全指南
categories: ios
tags: [iOS, Combine]
date: 2024-09-25
---

![title](https://image.sideproject.cn/titlex/titlex_051.jpg)

## 引言

Combine 是 Apple 在 WWDC 2019 推出的声明式响应式编程框架，用于处理异步事件流。它借鉴了 ReactiveX 的设计思想，将数据流、用户输入、网络请求等抽象为可组合的事件序列。Combine 与 SwiftUI 深度集成，`@Published` 和 `ObservableObject` 底层都依赖 Combine，是处理数据流、网络请求和用户交互的理想选择。本文将介绍 Combine 的核心概念、常用操作符、错误处理以及在实际项目中的应用场景和最佳实践。

## 为什么需要 Combine

传统的异步编程依赖回调和闭包，容易出现回调地狱、难以取消、错误处理分散等问题。Combine 通过统一的 Publisher-Subscriber 模型，将异步操作表示为可组合、可取消的数据流。开发者可以像操作数组一样对事件流进行 map、filter、merge 等变换，代码更清晰、逻辑更集中。此外，Combine 与 SwiftUI 的 `@Published` 天然配合，数据变化自动驱动 UI 更新，是构建 MVVM 架构的利器。

## 核心概念

### Publisher 与 Subscriber

Publisher 是事件的发出者，定义了一个可以随时间传递若干值的类型。Subscriber 是订阅者，接收 Publisher 发出的值。`sink` 是最简单的订阅方式，接收值和完成事件。`Just` 是一个只发出一个值然后完成的 Publisher，常用于将同步值包装为流。

```swift
import Combine

let publisher = Just(42)
let cancellable = publisher
    .sink { value in
        print("Received: \(value)")
    }
```

订阅会返回 `AnyCancellable`，持有它可保持订阅活跃；调用 `cancel()` 或释放它可取消订阅。在 ViewModel 中通常将 `AnyCancellable` 存入 `Set<AnyCancellable>`，这样在 ViewModel 释放时所有订阅会自动取消，避免内存泄漏。

### 常用操作符

Combine 提供了丰富的操作符，用于转换、过滤、组合事件流。`map` 转换每个元素，`filter` 过滤元素，`flatMap` 将每个元素映射为新的 Publisher 并合并，`merge` 合并多个 Publisher，`zip` 将多个 Publisher 的值按序配对。操作符链式调用形成声明式的数据处理管道。

```swift
[1, 2, 3, 4, 5].publisher
    .filter { $0 % 2 == 0 }
    .map { $0 * 2 }
    .sink { print($0) }
    .store(in: &cancellables)
```

上述管道会输出 4 和 8。注意 `store(in: &cancellables)` 需要 `cancellables` 为 `Set<AnyCancellable>` 类型的可变引用，用于集中管理订阅生命周期。

### 网络请求

`URLSession.dataTaskPublisher` 将网络请求封装为 Publisher，发出 `(Data, URLResponse)` 或 `Error`。通过 `map`、`decode`、`receive(on:)` 等操作符，可以构建类型安全、线程正确的网络层。`receive(on: DispatchQueue.main)` 确保后续操作和订阅回调在主线程执行，便于更新 UI。

```swift
func fetchUser(id: Int) -> AnyPublisher<User, Error> {
    URLSession.shared.dataTaskPublisher(for: url)
        .map(\.data)
        .decode(type: User.self, decoder: JSONDecoder())
        .receive(on: DispatchQueue.main)
        .eraseToAnyPublisher()
}
```

`eraseToAnyPublisher()` 用于类型擦除，将复杂的泛型 Publisher 类型简化为 `AnyPublisher<User, Error>`，便于作为函数返回类型或存储。

### 与 SwiftUI 集成

`ObservableObject` 配合 `@Published` 是 SwiftUI 中常用的 ViewModel 模式。`@Published` 属性变化时会触发 `objectWillChange`，SwiftUI 据此更新视图。在 ViewModel 中订阅网络请求等 Publisher，将结果赋给 `@Published` 属性，即可实现数据驱动的 UI 更新。务必使用 `[weak self]` 避免循环引用，并将订阅 `store(in: &cancellables)` 以便自动取消。

```swift
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
    private var cancellables = Set<AnyCancellable>()
    
    func loadItems() {
        apiService.fetchItems()
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { _ in },
                receiveValue: { [weak self] items in
                    self?.items = items
                }
            )
            .store(in: &cancellables)
    }
}
```

## Combine 与 async/await 的选择

Swift 5.5 引入 async/await 后，许多异步场景可以更简洁地实现。Combine 适合多值流、复杂的事件组合、与 SwiftUI 的深度集成；async/await 适合单次异步操作、顺序执行的逻辑。两者可以互操作：`Publisher` 可通过 `values` 属性转换为 AsyncStream，async 函数可通过 `Future` 或 `Task` 包装为 Publisher。根据项目风格和团队习惯选择即可。

## 总结

Combine 提供了强大的响应式编程能力，与 SwiftUI 配合使用可以构建优雅的异步数据流处理逻辑。掌握 Publisher、Subscriber、操作符和订阅管理是使用 Combine 的关键。在合适的场景下选用 Combine 或 async/await，能够显著提升代码质量和开发效率。
