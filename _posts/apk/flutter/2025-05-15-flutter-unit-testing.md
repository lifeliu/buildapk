---
layout: post
title: Flutter 单元测试完全指南
categories: flutter
tags: [Flutter, 单元测试]
date: 2025/5/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_022.jpg)

## 引言

Flutter 使用 Dart 的 test 包编写单元测试，运行在本地 JVM（flutter test），无需模拟器或真机，执行速度快。单元测试针对业务逻辑、工具函数、ViewModel 等，通过 mock 隔离依赖。mockito 用于创建 Mock 对象，when、verify 定义行为和断言。Widget 测试（testWidgets）在本地渲染 Widget 并模拟交互，也属于快速测试范畴。本文将介绍单元测试的编写、mockito 的用法、Widget 测试以及测试组织和最佳实践。

## 为什么需要单元测试

单元测试在修改代码后能快速反馈是否正确，减少手动回归。良好的测试覆盖率支撑重构，确保行为不变。编写可测试的代码往往促使更好的设计：依赖注入、职责单一等。Flutter 的 test 包支持异步测试（async/await）、分组（group）、 setUp/tearDown，满足大多数场景。

## 单元测试

test 函数定义测试用例，expect 断言结果。expect 支持 equals、throwsA、isNull 等 matcher。异步测试使用 async 和 await，或 expectLater 配合 completes、throwsA。group 可组织相关测试，共享 setUp。

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('Counter', () {
    test('increments', () {
      final counter = Counter();
      counter.increment();
      expect(counter.value, 1);
    });

    test('throws when decrement below zero', () {
      final counter = Counter();
      expect(() => counter.decrement(), throwsStateError);
    });
  });
}
```

## Widget 测试

testWidgets 使用 WidgetTester 渲染 Widget，pump 触发一帧，pumpAndSettle 等待动画完成。tap、enterText、drag 等模拟用户操作。find 查找 Widget，findsOneWidget、findsNothing 等断言。Widget 测试会创建 Element 树，需提供 MaterialApp 等祖先以满足依赖 Theme、MediaQuery 的 Widget。

```dart
testWidgets('button tap increments counter', (tester) async {
  await tester.pumpWidget(MaterialApp(home: CounterPage()));
  expect(find.text('0'), findsOneWidget);
  await tester.tap(find.byType(ElevatedButton));
  await tester.pump();
  expect(find.text('1'), findsOneWidget);
});
```

## 总结

单元测试保证业务逻辑正确，Widget 测试验证 UI 行为。结合 mockito 隔离依赖，使用 group 组织测试。在 CI 中运行 flutter test 可自动执行所有测试，保障代码质量。
