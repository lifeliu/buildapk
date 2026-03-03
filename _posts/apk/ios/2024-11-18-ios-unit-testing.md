---
layout: post
title: iOS 单元测试完全指南
categories: ios
tags: [单元测试, XCTest, Mock, iOS, 测试]
date: 2024/11/18 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_055.jpg)

## 引言

单元测试是保证 iOS 应用质量的重要手段。通过针对单个类或函数的隔离测试，可以快速发现回归、支撑重构、并作为活文档描述预期行为。XCTest 是 Apple 提供的测试框架，与 Xcode 深度集成，支持同步与异步测试、性能测试和 UI 测试。配合 Mock 对象和依赖注入，可以全面覆盖业务逻辑，避免依赖网络、数据库等外部资源。本文将介绍 XCTest 的基础用法、异步测试、Mock 策略以及测试组织的最佳实践。

## 为什么需要单元测试

单元测试在修改代码后能快速反馈是否正确，减少手动回归的工作量。当业务逻辑复杂或多人协作时，测试用例可以明确"正确行为"的定义，避免误改。良好的测试覆盖率也为重构提供信心：有测试保护，可以大胆优化代码结构。此外，编写可测试的代码往往促使更好的设计：依赖注入、职责单一等原则会自然浮现。

## 基础测试

测试类继承 `XCTestCase`，测试方法以 `test` 开头且无参数。`setUp` 和 `tearDown` 在每个测试前后执行，用于初始化被测对象和清理资源。使用 `@testable import MyApp` 可以访问模块内的 `internal` 成员，便于测试。`XCTAssertEqual`、`XCTAssertTrue`、`XCTAssertThrowsError` 等断言用于验证结果，失败时 Xcode 会高亮并输出期望与实际值。

```swift
import XCTest
@testable import MyApp

class CalculatorTests: XCTestCase {
    var calculator: Calculator!
    
    override func setUp() {
        super.setUp()
        calculator = Calculator()
    }
    
    func testAddition() {
        XCTAssertEqual(calculator.add(2, 3), 5)
    }
    
    func testDivisionByZero() {
        XCTAssertThrowsError(try calculator.divide(10, by: 0))
    }
}
```

`XCTAssertThrowsError` 可接受闭包来进一步检查错误类型或信息，例如 `XCTAssertThrowsError(try calculator.divide(10, by: 0)) { error in XCTAssertTrue(error is CalculatorError) }`。

## 异步测试

对于 async 函数或基于回调的异步逻辑，XCTest 支持 async 测试方法：将方法声明为 `async throws`，直接 `await` 异步调用，然后用断言验证结果。这样测试代码呈线性结构，易于编写和维护。对于旧式 completion handler，可使用 `XCTestExpectation` 和 `wait(for:timeout:)` 等待回调完成。

```swift
func testAsyncFetch() async throws {
    let user = try await apiService.fetchUser(id: 1)
    XCTAssertNotNil(user)
    XCTAssertEqual(user?.id, 1)
}
```

若 `apiService` 是真实实现，测试会依赖网络，不稳定且慢。应注入 Mock 实现，返回预设数据，使测试快速且可重复。

## Mock 对象

Mock 是遵循被测对象依赖的协议或基类的测试替身，在测试中返回可控的假数据，并可记录调用情况（如是否被调用、调用参数）以便断言。通过依赖注入将 Mock 传入被测对象，即可实现隔离测试。Mock 应尽量简单，只实现测试所需的行为。

```swift
class MockNetworkService: NetworkServiceProtocol {
    var mockResponse: [User] = []
    var fetchCalled = false
    
    func fetchUsers() async throws -> [User] {
        fetchCalled = true
        return mockResponse
    }
}
```

在测试中创建 `MockNetworkService`，设置 `mockResponse`，注入 ViewModel，调用 `loadUsers()`，然后断言 `viewModel.items == mockResponse` 且 `mockService.fetchCalled == true`。这样无需真实网络即可验证 ViewModel 的逻辑。

## 测试组织与命名

按功能或模块组织测试类，一个被测类对应一个测试类。测试方法名应描述场景和预期，如 `testLoginWithInvalidPasswordShowsError`。遵循 Given-When-Then 结构：准备数据（Given）、执行操作（When）、断言结果（Then）。避免测试间的共享可变状态，每个测试应独立可重复运行。

## 总结

良好的单元测试能够提高代码质量和重构信心，是 iOS 开发中不可或缺的实践。从核心业务逻辑开始编写测试，逐步提高覆盖率；同时保持测试简洁、快速，避免过度测试实现细节。结合 CI 在每次提交时自动运行测试，可以持续保障代码质量。
