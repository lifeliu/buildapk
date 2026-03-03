---
layout: post
title: Flutter Riverpod 状态管理完全指南
categories: flutter
tags: [Flutter, Riverpod]
date: 2024/9/18 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_008.jpg)

## 引言

Riverpod 是 Provider 的继任者，由同一作者 Remi Rousselet 开发，解决了 Provider 对 BuildContext 的依赖、测试时需包装 ProviderScope、以及部分边缘场景下的问题。Riverpod 支持编译时安全（通过代码生成）、更好的测试性、无需 BuildContext 的依赖获取，Provider 可声明依赖关系并由 Riverpod 自动解析。flutter_riverpod 提供与 Flutter 的集成，ConsumerWidget、ConsumerStatefulWidget 通过 WidgetRef 访问 Provider。本文将介绍 Riverpod 的 Provider 类型、ref.watch 与 ref.read、与 Flutter 的集成以及最佳实践。

## Riverpod 与 Provider 的对比

Provider 依赖 InheritedWidget，必须在 BuildContext 的子树中访问。Riverpod 的 Provider 是全局的，通过 ProviderScope 存储状态，ref 可在任何有 ProviderScope 祖先的地方使用。Riverpod 的 Provider 可依赖其他 Provider（通过 ref.watch 或 ref.read），形成依赖图，Riverpod 会按依赖顺序创建和更新。测试时只需 ProviderScope 并可 override Provider，无需包装 MaterialApp。

## 添加依赖

```yaml
dependencies:
  flutter_riverpod: ^2.4.9
  riverpod_annotation: ^2.3.3

dev_dependencies:
  build_runner: ^2.4.7
  riverpod_generator: ^2.3.9
```

使用 riverpod_annotation 和 riverpod_generator 可通过注解生成 Provider，减少样板代码。

## Provider 类型

- **Provider**：不可变数据，创建后不变
- **StateProvider**：简单可变状态，如计数器
- **FutureProvider**：异步数据，如网络请求
- **StreamProvider**：流数据，如 WebSocket
- **StateNotifierProvider**：复杂可变状态，配合 StateNotifier
- **NotifierProvider**（Riverpod 2.x）：替代 StateNotifierProvider 的推荐方式

```dart
final counterProvider = StateProvider<int>((ref) => 0);
final userProvider = FutureProvider<User>((ref) => fetchUser());
final configProvider = Provider<Config>((ref) => Config());
```

## 使用

ConsumerWidget 的 build 方法接收 context 和 ref。ref.watch 监听 Provider，当 Provider 更新时重建 Widget；ref.read 仅读取当前值，不监听，适合在事件回调中使用。ref.listen 可监听并执行副作用（如显示 SnackBar），不重建。ProviderScope 需包裹根 Widget，通常放在 MaterialApp 外。

```dart
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Text('$count');
  }
}

// MaterialApp 需使用 ProviderScope
ProviderScope(child: MaterialApp(home: HomePage()))
```

## 总结

Riverpod 是现代化、类型安全的状态管理方案，适合中大型项目。掌握 Provider 类型选择、ref.watch 与 ref.read 的区别、以及依赖声明，可以构建清晰、可测试的状态管理架构。
