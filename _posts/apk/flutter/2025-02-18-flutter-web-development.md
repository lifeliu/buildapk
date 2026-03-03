---
layout: post
title: Flutter Web 开发完全指南
categories: flutter
tags: [Flutter, Web]
date: 2025/2/18 09:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_017.jpg)

## 引言

Flutter 支持编译为 Web 应用，使用 CanvasKit（Skia 编译为 WebAssembly）或 HTML 渲染器。CanvasKit 渲染质量高、一致性号，但包体积较大；HTML 渲染器体积小，但部分渲染效果可能不同。Web 平台需考虑 URL 路由、浏览器前进后退、SEO、响应式布局等。kIsWeb 可判断当前是否为 Web 平台，用于条件编译或平台特定逻辑。本文将介绍 Flutter Web 的构建、部署、平台差异处理以及性能优化建议。

## Flutter Web 的适用场景

Flutter Web 适合内部工具、管理后台、跨平台 UI 复用等场景。对于面向公众的营销页、内容站，SEO 和首屏加载速度至关重要，需评估 Flutter Web 的包体积和 SEO 支持是否满足需求。Flutter Web 的 SEO 在逐步改善，可通过 --web-renderer html 减小体积，或使用 deferred loading 延迟加载部分库。

## 构建

flutter build web 默认使用 CanvasKit，可指定 --web-renderer html 使用 HTML 渲染器。输出在 build/web 目录，可部署到任何静态托管服务。--base-href 可设置基础路径，用于部署在子路径时。--pwa-strategy 可配置 PWA 策略。

```bash
flutter build web
flutter build web --web-renderer html
```

## 平台判断

在需要区分 Web 与移动端的逻辑中使用 kIsWeb。例如 Web 上使用 url_launcher 打开链接，移动端使用原生分享；Web 上某些插件可能不可用，需提供替代实现或隐藏功能。

```dart
import 'package:flutter/foundation.dart';

if (kIsWeb) {
  // Web 特定逻辑，如使用 dart:html、避免移动端插件
}
```

## 总结

Flutter Web 适合内部工具和跨平台 UI 复用，需注意包体积和 SEO。合理选择渲染器、使用 deferred loading、优化资源加载，可提升 Web 应用体验。关注 Flutter 官方对 Web 的持续改进。
