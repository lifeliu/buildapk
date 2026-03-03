---
layout: post
title: Swift Codable 与 JSON 编解码完全指南
categories: ios
tags: [Codable, JSON, 编解码, Swift, 网络]
date: 2025/1/15 11:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_062.jpg)

## 引言

Codable 是 Swift 4 引入的编解码协议，由 `Encodable` 和 `Decodable` 组成，用于将类型与外部表示（如 JSON、Plist）相互转换。`JSONEncoder` 和 `JSONDecoder` 是处理 JSON 的标准工具，类型安全、性能良好，是网络层和本地存储的常用选择。通过自定义 `CodingKeys`、`init(from decoder:)` 和 `encode(to encoder:)`，可以灵活处理键名映射、可选字段、默认值、嵌套结构等复杂情况。本文将介绍 Codable 的基础用法、键名映射、可选与默认值处理以及常见陷阱和最佳实践。

## 为什么使用 Codable

相比手写 JSON 解析或第三方库，Codable 是 Swift 标准库的一部分，无需额外依赖。编译器可自动为简单类型合成编解码实现，减少样板代码。类型安全：解码失败会抛出错误，可精确捕获；编码时由类型系统保证结构正确。与 `URLSession`、`Alamofire` 等网络库配合良好，是 iOS 开发中处理 JSON 的事实标准。

## 基础用法

类型声明遵循 `Codable` 后，若所有属性都是 Codable 的且键名与 JSON 一致，无需额外实现即可编解码。`JSONDecoder().decode(User.self, from: data)` 将 Data 解码为 User，失败时抛出 `DecodingError`。`JSONEncoder().encode(user)` 将对象编码为 Data。默认情况下，`JSONDecoder` 使用 `.convertFromSnakeCase` 可将 `user_name` 转为 `userName`，需键名匹配。

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

let json = """
{"id": 1, "name": "张三", "email": "zhang@example.com"}
""".data(using: .utf8)!

let user = try JSONDecoder().decode(User.self, from: json)
```

## 自定义键名

当 JSON 键名与 Swift 属性名不一致时，定义 `CodingKeys` 枚举并实现 `String` 原始值映射。只需列出需要映射的 case，未列出的键默认使用属性名。编码时也会按 `CodingKeys` 输出键名。

```swift
struct User: Codable {
    let id: Int
    let userName: String
    
    enum CodingKeys: String, CodingKey {
        case id
        case userName = "user_name"
    }
}
```

## 可选与默认值

JSON 中可能缺失某些字段，或值为 `null`。若属性为可选类型，缺失或 `null` 会解码为 `nil`。若需要非可选类型且提供默认值，需自定义 `init(from decoder:)`，使用 `decodeIfPresent` 尝试解码，失败时使用默认值。注意：`decodeIfPresent` 对缺失的键返回 `nil`，对类型不匹配会抛出错误。

```swift
struct Config: Codable {
    let timeout: Int
    let retryCount: Int
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        timeout = try container.decodeIfPresent(Int.self, forKey: .timeout) ?? 30
        retryCount = try container.decodeIfPresent(Int.self, forKey: .retryCount) ?? 3
    }
}
```

## 日期与自定义转换

`JSONDecoder.dateDecodingStrategy` 可配置日期解析，如 `.iso8601`、`.formatted(dateFormatter)` 或自定义闭包。对于枚举、URL 等类型，若 JSON 表示与 Swift 类型不一致，可自定义 `init(from:)` 实现转换。复杂嵌套结构可拆分为多个 Codable 类型，逐层解码。

## 总结

Codable 是 Swift 处理 JSON 的标准方式，类型安全且易于扩展。掌握键名映射、可选处理和自定义解码，可以应对大多数 API 结构。注意错误处理、日期格式和可选字段的语义，以构建健壮的网络层解析逻辑。
