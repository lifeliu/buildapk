---
layout: post
title: Flutter Widget 基础与生命周期完全指南
categories: flutter
tags: [Flutter, Widget]
date: 2024/6/10 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_002.jpg)

## 引言

Widget 是 Flutter 应用的基本构建块，一切皆 Widget。与许多框架不同，Flutter 的 Widget 是不可变的（Immutable），每次状态变化都会重建 Widget 树，框架通过 diff 算法高效更新实际渲染的 Element 和 RenderObject。理解 Widget 的分类、生命周期和重建机制是 Flutter 开发的基础。StatelessWidget 用于无状态 UI，StatefulWidget 用于有状态 UI，其 State 对象在 Widget 重建时得以保留，从而持有可变数据。本文将深入探讨 Widget 的创建、更新、销毁流程，以及 build 方法的优化策略和常见陷阱。

## Widget 与 Element 的关系

Widget 是配置描述，Element 是 Widget 在树中的实例，持有对 RenderObject 的引用。当 Widget 重建时，若新 Widget 与旧 Widget 的 runtimeType 和 key 相同，Element 会复用并更新，否则会销毁旧 Element 并创建新的。合理使用 key（如 ValueKey、ObjectKey）可以控制复用行为，在列表重排等场景下避免状态错乱。

## Widget 分类

- **StatelessWidget**：无内部状态，build 仅依赖传入参数。父 Widget 重建时子 StatelessWidget 会随之重建。适合纯展示型组件。
- **StatefulWidget**：有状态，State 对象持有可变数据。当调用 setState 时，State 会触发 rebuild，但 State 对象本身不会销毁，因此可保留数据。适合需要响应用户交互或异步数据的组件。
- **InheritedWidget**：向下传递数据，Theme、MediaQuery、Locale 等都基于此实现。子 Widget 可通过 context.dependOnInheritedWidgetOfExactType 或 context.watch 获取并监听变化。

## StatefulWidget 生命周期

生命周期方法按顺序执行：createState 创建 State，initState 在 State 插入树后调用一次用于初始化，didChangeDependencies 在 initState 之后以及依赖的 InheritedWidget 变化时调用，build 在需要构建 UI 时调用（可能多次），deactivate 在 State 从树中移除时调用（可能随后又插入，如路由切换），dispose 在 State 永久移除时调用，用于释放 Controller、Stream 等资源。

```dart
class MyWidget extends StatefulWidget {
  @override
  _MyWidgetState createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    // 初始化，只调用一次。可在此创建 Controller、订阅 Stream
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // 依赖的 InheritedWidget（如 Theme、Provider）变化时调用
    // 可在此根据 InheritedWidget 做初始化或更新
  }

  @override
  Widget build(BuildContext context) {
    return Container();
  }

  @override
  void deactivate() {
    // 从树中移除时调用，路由切换时可能触发
  }

  @override
  void dispose() {
    super.dispose();
    // 释放资源：取消订阅、关闭 Controller 等，避免内存泄漏
  }
}
```

## build 方法优化

build 可能被频繁调用，应避免在 build 中创建大量对象或执行耗时操作。将不变的子 Widget 提取为 const 或顶层变量，使用 const 构造函数可减少重建开销。对于复杂列表，使用 ListView.builder 懒加载，避免一次性构建所有子项。

## 总结

掌握 Widget 生命周期有助于正确管理资源和避免内存泄漏。理解 StatelessWidget 与 StatefulWidget 的适用场景，在 dispose 中释放资源，并优化 build 方法，是编写高效 Flutter 应用的基础。
