---
layout: post
title: UIKit 与 SwiftUI 混合开发与桥接
categories: ios
tags: [UIKit, SwiftUI, UIHostingController, UIViewRepresentable, 混合开发]
date: 2025/2/19 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_067.jpg)

## 引言

在 SwiftUI 成熟之前或需要复用现有 UIKit 代码时，混合开发是常见需求。SwiftUI 与 UIKit 可以双向嵌入：`UIHostingController` 将 SwiftUI 视图嵌入 UIKit 层级；`UIViewRepresentable` 将 UIKit 视图嵌入 SwiftUI。这样既能渐进式迁移到 SwiftUI，又能使用 MapKit、WKWebView、AVFoundation 等尚未完全 SwiftUI 化的组件。本文将介绍两种桥接方式、Coordinator 的作用、数据传递与生命周期，以及最佳实践。

## 为什么需要混合开发

SwiftUI 尚无法覆盖所有 UIKit 能力，如复杂的 `MKMapView` 配置、`WKWebView` 的 delegate 回调、自定义相机 UI 等。老项目可能已有大量 UIKit 代码，重写成本高，混合开发允许新功能用 SwiftUI、旧模块保留 UIKit，逐步迁移。此外，某些第三方库仅提供 UIKit 接口，通过桥接可在 SwiftUI 中使用。

## SwiftUI 嵌入 UIKit

`UIHostingController` 是 `UIViewController` 的子类，将 SwiftUI 的 `View` 作为其根视图。创建时传入 SwiftUI 视图，将其作为子 ViewController 添加，或直接作为 window 的 rootViewController。适合在现有 UIKit 应用中嵌入 SwiftUI 页面，或作为导航栈中的一页。

```swift
import SwiftUI

let swiftUIView = ContentView()
let hostingController = UIHostingController(rootView: swiftUIView)
addChild(hostingController)
view.addSubview(hostingController.view)
hostingController.view.frame = view.bounds
hostingController.didMove(toParent: self)
```

若需向 SwiftUI 传递数据，可通过 `rootView` 的初始化参数，如 `ContentView(user: user)`。`UIHostingController` 会管理 SwiftUI 视图的生命周期和更新。

## UIKit 嵌入 SwiftUI

`UIViewRepresentable` 协议将 `UIView` 包装为 SwiftUI 可用的视图。需实现 `makeUIView(context:)` 创建 UIView，以及 `updateUIView(_:context:)` 在 SwiftUI 状态变化时更新 UIView。SwiftUI 会管理 Representable 的创建和更新，`makeUIView` 可能只调用一次，`updateUIView` 会在状态变化时多次调用。

```swift
struct MapViewRepresentable: UIViewRepresentable {
    func makeUIView(context: Context) -> MKMapView {
        MKMapView()
    }
    
    func updateUIView(_ uiView: MKMapView, context: Context) {}
}
```

若需处理 delegate 回调（如地图点击、网页加载完成），需使用 `makeCoordinator` 返回一个 Coordinator，将其设为 UIView 的 delegate，在 Coordinator 中处理回调并更新 SwiftUI 状态（通过 `@Binding` 或父视图的 state）。

## 协调器

Coordinator 是 `UIViewRepresentable` 的可选组件，用于实现 delegate、处理事件、持有跨更新周期的状态。`makeCoordinator` 在 `makeUIView` 之前调用，返回的实例会存入 `context.coordinator`。将 Coordinator 设为 UIView 的 delegate，在 delegate 方法中可通过 `parent` 引用 Representable，从而更新 `@Binding` 或触发回调。Coordinator 通常继承 `NSObject` 以符合 delegate 协议。

```swift
struct WebViewRepresentable: UIViewRepresentable {
    let url: URL
    
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
    
    func makeUIView(context: Context) -> WKWebView {
        let webView = WKWebView()
        webView.navigationDelegate = context.coordinator
        return webView
    }
    
    func updateUIView(_ uiView: WKWebView, context: Context) {
        uiView.load(URLRequest(url: url))
    }
    
    class Coordinator: NSObject, WKNavigationDelegate {
        var parent: WebViewRepresentable
        init(_ parent: WebViewRepresentable) { self.parent = parent }
    }
}
```

## 总结

混合开发允许渐进式迁移，充分利用 SwiftUI 和 UIKit 的优势。掌握 `UIHostingController` 和 `UIViewRepresentable`，以及 Coordinator 模式，可以灵活地在两个框架间桥接，构建兼容现有代码并逐步现代化的应用。
