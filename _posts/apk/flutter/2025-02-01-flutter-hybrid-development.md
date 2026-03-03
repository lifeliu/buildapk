---
layout: post
title: Flutter 混合开发与原生集成
categories: flutter
tags: [Flutter, 混合开发]
date: 2025/2/1 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_016.jpg)

## 引言

Flutter 可嵌入现有原生应用（Add-to-App），也可在 Flutter 应用中嵌入原生视图（Platform View）。MethodChannel 实现 Flutter 与原生代码的双向通信：Flutter 调用 invokeMethod 发起请求，原生端在 MethodCallHandler 中处理并返回结果。EventChannel 用于原生向 Flutter 持续推送事件。混合开发支持渐进式迁移，适合已有大量原生代码的项目。本文将介绍 MethodChannel 的配置、通信机制、Platform View 的嵌入以及常见问题处理。

## 混合开发的场景

Add-to-App：在现有 Android Activity 或 iOS ViewController 中嵌入 FlutterEngine 和 FlutterFragment/FlutterViewController，实现部分页面用 Flutter 开发。Platform View：在 Flutter 的 Widget 树中嵌入原生视图，如 AndroidView、UiKitView，适用于地图、WebView、相机预览等 Flutter 尚未很好支持的场景。MethodChannel：调用原生 API（如获取电池电量、打开系统设置）、复用原生 SDK 等。

## MethodChannel

通道名称需在 Flutter 和原生端一致，通常使用反向域名格式。invokeMethod 的第二个参数为可选参数，会传递给原生端。原生端通过 result.success 或 result.error 返回结果。异步操作时原生端需持有 result 并在完成后回调，不可同步返回。注意：MethodChannel 的调用会跨平台边界，避免传递大量数据或频繁调用，可能影响性能。

```dart
static const platform = MethodChannel('com.example.app/channel');

Future<int> getBatteryLevel() async {
  try {
    final result = await platform.invokeMethod<int>('getBatteryLevel');
    return result ?? 0;
  } on PlatformException catch (e) {
    throw Exception('Failed: ${e.message}');
  }
}
```

## 总结

混合开发支持渐进式迁移，MethodChannel 是跨平台通信的核心。合理设计通道和参数格式，注意异步回调和异常处理。Platform View 有性能开销，仅在必要时使用。Add-to-App 需正确管理 FlutterEngine 的生命周期，避免内存泄漏。
