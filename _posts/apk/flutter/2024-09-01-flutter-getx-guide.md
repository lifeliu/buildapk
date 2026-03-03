---
layout: post
title: Flutter GetX 状态管理与路由完全指南
categories: flutter
tags: [Flutter, GetX]
date: 2024/9/1 14:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_007.jpg)

## 引言

GetX 是 Flutter 的轻量级全栈解决方案，包含状态管理、路由、依赖注入、国际化等。通过 GetxController 和 .obs 响应式变量，实现简洁的状态管理，无需 Stream 或 ChangeNotifier。Get.to、Get.back 等提供便捷的路由 API。GetX 适合快速开发和小型项目，但需注意与 Flutter 官方推荐的 Provider、Riverpod 等方案的差异，以及部分 API 对 context 的隐式依赖可能带来的测试和可维护性影响。本文将介绍 GetX 的核心用法、状态管理、路由和依赖注入，以及最佳实践。

## GetX 的定位与取舍

GetX 的卖点是"简单"：少量代码即可实现状态管理、路由和依赖注入。.obs 配合 Obx 实现响应式更新，无需手动调用 setState 或 notifyListeners。但 GetX 大量使用全局状态和隐式依赖，在大型项目中可能导致耦合和测试困难。选择 GetX 时需权衡开发速度与长期维护成本。

## 添加依赖

```yaml
dependencies:
  get: ^4.6.6
```

## 状态管理

通过 .obs 将变量变为响应式，Obx 监听其变化并重建子 Widget。GetxController 用于封装业务逻辑，Get.put 注册实例，Get.find 获取。GetView 和 GetWidget 可简化 Controller 的获取。注意：Obx 仅监听其内部访问的 .obs 变量，确保在 Obx 的 builder 中正确引用，否则变化时不会重建。

```dart
class CounterController extends GetxController {
  final count = 0.obs;
  void increment() => count++;
}

// 使用
Get.put(CounterController());
final controller = Get.find<CounterController>();
Obx(() => Text('${controller.count}'));  // count 变化时重建
```

## 路由

Get.to 等同于 Navigator.push，Get.off 替换当前页，Get.offAll 清空栈并跳转。Get.arguments 可获取路由参数。GetMaterialApp 包装 MaterialApp 并注入 Get 的路由能力。对于简单应用，Get 的路由足够；复杂路由和深层链接建议使用 GoRouter。

```dart
Get.to(DetailPage());
Get.toNamed('/detail', arguments: {'id': '123'});
Get.back();
Get.offAll(HomePage());
```

## 总结

GetX 适合快速开发和小型项目，提供一站式解决方案。在大型项目中需注意架构清晰度，可考虑与 Provider、Riverpod 等配合使用，或仅使用 GetX 的路由和依赖注入部分。
