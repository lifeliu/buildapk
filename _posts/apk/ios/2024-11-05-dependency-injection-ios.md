---
layout: post
title: iOS 依赖注入模式与 Swinject 实战
categories: ios
tags: [依赖注入, Swinject, DI, iOS, 架构]
date: 2024/11/5 15:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_054.jpg)

## 引言

依赖注入（Dependency Injection，DI）是一种设计模式，通过从外部提供对象所需的依赖，而非在内部直接创建，从而降低耦合、提高可测试性和可维护性。在 iOS 开发中，Swinject 是流行的轻量级依赖注入框架，支持注册、解析、作用域和模块化配置。本文将介绍依赖注入的原则、Swinject 的用法、作用域管理以及在实际项目中的最佳实践。

## 为什么需要依赖注入

当 ViewModel 内部直接 `let api = APIService()` 时，ViewModel 与 APIService 的实现强耦合，难以替换为 Mock 进行单元测试，也难以在不同环境（如测试、预发）使用不同实现。依赖注入将依赖的创建和组装移到外部，ViewModel 只依赖协议或抽象类型，具体实现由注入方提供。这样测试时可以注入 MockAPIService，生产环境注入真实实现，代码更灵活、更易测试。

## 添加 Swinject

Swinject 可通过 Swift Package Manager 或 CocoaPods 集成。在 Xcode 中：File → Add Package Dependencies，输入 `https://github.com/Swinject/Swinject`，选择版本即可。

```swift
// Package.swift 或 CocoaPods
// .package(url: "https://github.com/Swinject/Swinject", from: "2.8.0")
```

## 容器配置

Swinject 使用 `Container` 管理注册和解析。`register` 方法将类型与工厂闭包关联，工厂闭包在解析时执行，用于创建实例。`resolve` 用于获取实例，其参数 `r` 是 `Resolver`，可用来解析其他依赖，实现依赖链的自动组装。建议按层或模块组织注册代码，便于维护和按需加载。

```swift
import Swinject

let container = Container()

container.register(NetworkServiceProtocol.self) { _ in
    NetworkService()
}.inObjectScope(.container)

container.register(UserRepository.self) { r in
    UserRepository(networkService: r.resolve(NetworkServiceProtocol.self)!)
}

container.register(UserViewModel.self) { r in
    UserViewModel(repository: r.resolve(UserRepository.self)!)
}
```

`r.resolve(NetworkServiceProtocol.self)!` 会从容器中解析 NetworkService，注入到 UserRepository。使用 `!` 时需确保已注册，否则运行时崩溃；生产环境可考虑使用 `r.resolve(...) ?? DefaultNetworkService()` 等降级策略。

## 解析依赖

在应用启动或需要创建 ViewController 时，从容器解析依赖。若 ViewController 由容器创建，可在其初始化器中注入 ViewModel；若使用 Storyboard，可通过属性注入或自定义 `instantiateViewController` 逻辑完成注入。

```swift
let viewModel = container.resolve(UserViewModel.self)!
```

## 作用域

作用域决定实例的生命周期。`inObjectScope(.container)` 表示单例，同一容器内多次解析返回同一实例。`inObjectScope(.transient)` 表示每次解析都创建新实例。`inObjectScope(.weak)` 在无强引用时释放。根据依赖的性质选择合适作用域：无状态服务可用单例，有状态的 ViewModel 通常用 transient。

```swift
container.register(APIClient.self) { _ in
    APIClient()
}.inObjectScope(.container)  // 单例

container.register(ViewController.self) { r in
    ViewController(viewModel: r.resolve(ViewModel.self)!)
}.inObjectScope(.transient)  // 每次新建
```

## 最佳实践

1. **面向协议编程**：注册和注入时使用协议类型，便于替换实现和测试。
2. **避免循环依赖**：若 A 依赖 B、B 依赖 A，需重新设计或引入中间层打破循环。
3. **模块化容器**：大型项目可将注册拆分为多个 `Assembly`，按功能模块组织，便于维护和按需加载。
4. **谨慎使用强制解包**：`resolve` 返回可选值，可考虑 `guard let` 或提供默认实现，避免未注册导致的崩溃。

## 总结

合理使用依赖注入能够构建松耦合、易测试的 iOS 应用架构。Swinject 提供了轻量而完整的 DI 能力，适合中小型项目；超大型项目可考虑更模块化的方案或自建轻量容器。将 DI 与 MVVM、Clean Architecture 等结合，可以显著提升代码质量和团队协作效率。
