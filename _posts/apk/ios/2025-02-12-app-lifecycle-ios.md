---
layout: post
title: iOS 应用生命周期完全解析
categories: ios
tags: [iOS, 应用生命周期]
date: 2025-02-12
---

![title](https://image.sideproject.cn/titlex/titlex_066.jpg)

## 引言

理解 iOS 应用生命周期对于正确处理资源、保存状态和响应用户交互至关重要。从启动到终止，应用会经历多个状态转换；在前后台切换、来电、系统资源紧张等场景下，系统会发送相应回调。开发者需在合适的时机保存数据、释放资源、暂停或恢复任务，以构建稳定、省电的应用。iOS 13 引入的 Scene 支持多窗口，生命周期模型有所扩展。本文将介绍应用状态、AppDelegate 与 SceneDelegate 的职责、各生命周期回调的含义以及最佳实践。

## 应用状态

- **Not Running**：应用未启动或已被系统终止。
- **Inactive**：应用在前台但不接收事件，如拉下控制中心、接听电话时的过渡状态。
- **Active**：应用在前台且接收事件，正常交互状态。
- **Background**：应用在后台，可执行有限的后台任务，系统可能随时挂起。
- **Suspended**：应用被挂起，不执行代码，内存可能被回收以腾出空间。

状态转换由系统驱动，应用通过 delegate 回调感知。从 Active 进入 Background 时，应保存状态、释放非必要资源；从 Background 回到 Active 时，可恢复 UI 和任务。Suspended 时应用无法执行代码，故重要数据应在进入 Background 前持久化。

## AppDelegate 关键方法

`AppDelegate` 负责应用级生命周期，与是否使用 Scene 无关。`didFinishLaunchingWithOptions` 在应用启动时调用，用于初始化全局配置、第三方 SDK 等。`applicationDidEnterBackground` 在进入后台时调用，应在此保存用户数据、暂停计时器、释放大对象，系统给予约 5 秒执行时间。`applicationWillTerminate` 在应用即将被用户或系统终止时调用，但若应用从 Suspended 被终止，此方法可能不会调用，故关键保存逻辑应放在 `applicationDidEnterBackground`。

```swift
func application(_ application: UIApplication, 
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    return true
}

func applicationDidEnterBackground(_ application: UIApplication) {
    // 保存状态，释放资源
}

func applicationWillTerminate(_ application: UIApplication) {
    // 应用即将终止
}
```

## SceneDelegate（多窗口）

iOS 13+ 使用 Scene 管理 UI 生命周期，支持多窗口（iPad）和分屏。每个 Scene 有独立的 `SceneDelegate`，负责创建和管理 `UIWindow`。`scene(_:willConnectTo:options:)` 在 Scene 即将连接时调用，在此创建 `UIWindow` 并设置 `rootViewController`。`sceneDidDisconnect` 在 Scene 断开时调用，可做清理。`sceneWillResignActive`、`sceneDidEnterBackground`、`sceneWillEnterForeground`、`sceneDidBecomeActive` 对应前后台转换，与 AppDelegate 的类似回调对应，但作用域为单个 Scene。若未使用 Scene，则仍由 AppDelegate 的 `applicationDidEnterBackground` 等处理。

```swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
    guard let windowScene = scene as? UIWindowScene else { return }
    window = UIWindow(windowScene: windowScene)
    window?.rootViewController = ContentView()
    window?.makeKeyAndVisible()
}
```

## 最佳实践

1. **状态保存**：在 `applicationDidEnterBackground` 或 `sceneDidEnterBackground` 中保存用户数据、表单状态等，不依赖 `applicationWillTerminate`。
2. **资源释放**：进入后台时释放大图、缓存等，减少被系统终止的风险。
3. **后台任务**：若需在后台执行任务，使用 `beginBackgroundTask` 申请额外时间，或使用 Background Modes 的特定能力（如音频、定位）。
4. **恢复 UI**：从后台返回时，检查数据是否过期，必要时刷新界面。

## 总结

正确理解生命周期有助于构建稳定、资源使用合理的 iOS 应用。将保存和释放逻辑放在合适的回调中，避免依赖不可靠的 `applicationWillTerminate`，并善用 Scene 的多窗口能力，可以提升应用在复杂场景下的表现。
