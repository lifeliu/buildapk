---
layout: post
title: Swift async/await 并发编程完全指南
categories: ios
tags: [iOS, async/await]
date: 2024-10-12
---

![title](https://image.sideproject.cn/titlex/titlex_052.jpg)

## 引言

Swift 5.5 引入的 async/await 语法彻底简化了异步编程。开发者可以用同步风格的代码编写异步逻辑，无需嵌套回调和复杂的错误处理。配合 Actor 模型和结构化并发，Swift 并发模型提供了数据竞争安全、可取消、可优先级的并发能力。本文将深入探讨 async/await 的基础用法、Task 与结构化并发、Actor 数据隔离、MainActor 的使用，以及与传统回调的互操作。

## 为什么需要 async/await

传统的异步编程依赖 completion handler 和闭包，多层嵌套会导致"回调地狱"，错误处理分散且容易遗漏。async/await 将异步操作抽象为可挂起、可恢复的函数，代码呈线性结构，错误通过 `throws` 和 `try` 统一处理，可读性和可维护性大幅提升。此外，Swift 并发模型在编译器层面检查数据竞争，配合 Actor 可在编写并发代码时获得更强的安全保障。

## 基础用法

async 函数使用 `async` 关键字标记，内部可调用其他 async 函数或使用 `await` 等待异步操作。`await` 表示挂起点：当等待的异步操作未完成时，当前任务会挂起，线程可去执行其他工作；操作完成后任务恢复执行。从同步上下文调用 async 函数必须使用 `Task { }` 创建异步任务。

```swift
func fetchUser(id: Int) async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// 调用
Task {
    do {
        let user = try await fetchUser(id: 1)
        print(user.name)
    } catch {
        print("Error: \(error)")
    }
}
```

`URLSession.shared.data(from:)` 在 iOS 15+ 提供了 async 版本，直接 `try await` 即可，无需 completion handler。错误通过 `throws` 传播，调用方用 `do-catch` 处理。

## 并发执行

当多个异步操作相互独立时，可使用 `async let` 并发启动，再在需要时 `await` 结果。这样多个操作会并行执行，总耗时接近最慢的那个，而非顺序执行的总和。

```swift
async let user = fetchUser(id: 1)
async let posts = fetchPosts(userId: 1)

let (userResult, postsResult) = try await (user, posts)
```

`async let` 创建子任务，两个 fetch 同时进行；`try await (user, posts)` 会等待两者都完成，若有任一抛出错误则整体抛出。

## Actor 数据隔离

Actor 是 Swift 并发中用于共享可变状态的类型。Actor 隔离其内部状态，所有对状态的访问必须通过 actor 的方法，且方法默认是异步的（从外部调用需 `await`）。这保证了同一时刻只有一个任务能访问 actor 内部数据，从而避免数据竞争。

```swift
actor Counter {
    private var value = 0
    
    func increment() {
        value += 1
    }
    
    func getValue() -> Int {
        return value
    }
}

let counter = Counter()
Task {
    await counter.increment()
    print(await counter.getValue())
}
```

从非 actor 隔离的上下文（如 `Task` 闭包内）访问 actor 的方法或属性必须加 `await`，因为调用可能会挂起以等待 actor 空闲。

## 与 MainActor

UI 更新必须在主线程执行。`@MainActor` 将类型或方法标记为在主线程执行，在 async 函数中调用被 `@MainActor` 标记的代码时，会自动切换到主线程。ViewModel 若需要更新 UI，可标记为 `@MainActor class`，其方法内的赋值会安全地在主线程执行。

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
    
    func loadItems() async {
        let fetched = await apiService.fetchItems()
        items = fetched  // 已在主线程
    }
}
```

`loadItems` 是 async 的，可能在后台执行，但 `items = fetched` 会切换到主线程，因为 `ViewModel` 是 `@MainActor` 隔离的。

## 任务取消与 TaskGroup

`Task` 支持取消：`Task.isCancelled` 可检查，`Task.checkCancellation()` 在取消时抛出 `CancellationError`。`TaskGroup` 用于动态数量的并发子任务，适合批量处理。合理利用取消和 TaskGroup 可以构建可响应、高效的并发逻辑。

## 总结

async/await 是 Swift 并发的现代方案，能够显著提升异步代码的可读性和可维护性。掌握 async/await、Task、Actor 和 MainActor 是编写高质量并发代码的基础。在新项目中优先使用 async/await，老项目可逐步迁移，并与 Combine 等现有方案配合使用。
