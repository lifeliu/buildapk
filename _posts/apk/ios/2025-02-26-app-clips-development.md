---
layout: post
title: iOS App Clips 轻应用开发指南
categories: ios
tags: [iOS, App Clips]
date: 2025-02-26
---

![title](https://image.sideproject.cn/titlex/titlex_068.jpg)

## 引言

App Clips 是 iOS 14 引入的轻量级应用体验，用户无需下载完整应用即可使用核心功能。通过 NFC 标签、二维码、App Clip 码或 Safari 链接即可唤起，适合即用即走的场景，如扫码支付、共享单车开锁、餐厅点餐、酒店入住等。App Clips 有独立的 Target、Bundle ID 和大小限制，与主应用共享部分代码和资源，用户可随时升级到完整应用。本文将介绍 App Clips 的创建、配置、唤起方式、与主应用的关系以及开发与审核要点。

## App Clips 的特点与限制

App Clips 体积限制约 15MB（压缩后），需聚焦核心流程，不能包含完整应用的全部功能。支持大部分系统框架，但部分能力受限（如无推送、无后台长时间运行等）。用户使用后，系统会在一段时间内保留 App Clip，再次遇到同一 App Clip 的唤起方式时可快速启动。用户可从 App Clip 内跳转到 App Store 下载完整应用，两者可通过 App Group 共享部分数据（如登录态、临时订单）。

## 创建 App Clips Target

在 Xcode 中：File → New → Target → App Clip，输入名称。创建后会有独立的目录、Info.plist 和入口。Bundle ID 通常为主应用的 ID 加 `.Clip` 后缀。App Clip 与主应用可共享部分 target（如网络层、模型），通过 target membership 将共享代码加入 App Clip target。

## 配置

- **Bundle ID**：主应用 ID + `.Clip`，如 `com.example.app.Clip`
- **大小**：注意依赖和资源，控制在 15MB 内
- **关联域名**：若通过 URL 唤起，需在 App Clip 的 Associated Domains 中配置，并在服务器放置 `apple-app-site-association` 文件
- **App Group**：若与主应用共享数据，配置相同的 App Group

## 唤起方式

App Clips 可通过多种方式唤起：NFC 标签、二维码（需包含 App Clip 的 URL）、App Clip 码（Apple 的轻 App 码）、Safari 链接。用户触发后，系统会加载 App Clip 并传入唤起时的 URL 或 NFC 负载。在 App Clip 的 `SceneDelegate` 或 `AppDelegate` 中，通过 `UIScene.ConnectionOptions` 的 `userActivity` 或 `urlContexts` 获取传入的 URL，解析参数并跳转到对应功能。

```swift
// 通过 URL
// myapp://order/123 或 https://example.com/order/123

// 在 App Clip 中处理
func handle(userActivity: NSUserActivity) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else { return }
    // 解析 URL 参数，如订单 ID，直接进入支付或详情页
}
```

URL 设计应包含完成核心流程所需的全部参数，避免 App Clip 内再要求用户登录或输入大量信息。

## App Clip 与主应用

用户可从 App Clip 内看到"获取完整应用"等入口，跳转 App Store 下载主应用。通过 App Group 共享的 `UserDefaults` 或文件，主应用可读取 App Clip 中保存的临时数据（如未完成的订单），实现无缝衔接。注意：App Clip 的数据可能被系统清理，重要状态应尽快同步到服务器或引导用户下载主应用完成后续操作。

## 总结

App Clips 适合扫码支付、共享单车、点餐等即用即走的场景，能够降低用户使用门槛。掌握创建、配置、唤起方式和数据共享，可以构建轻量、高效的 App Clips。注意体积限制、URL 设计和审核对 App Clips 的专门要求，确保符合 Apple 的即时体验规范。
