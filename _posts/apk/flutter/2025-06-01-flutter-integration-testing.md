---
layout: post
title: Flutter 集成测试与 E2E 测试指南
categories: flutter
tags: [Flutter, 集成测试]
date: 2025/6/1 14:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_023.jpg)

## 引言

integration_test 包用于端到端测试，在真机或模拟器上运行完整应用，模拟真实用户操作。与单元测试和 Widget 测试不同，集成测试启动整个应用，经过完整的初始化流程，可验证导航、网络、存储等集成行为。集成测试较慢、较脆弱（UI 变化可能导致失败），应聚焦关键用户流程。本文将介绍集成测试的配置、编写、运行以及最佳实践。

## 集成测试的定位

单元测试和 Widget 测试快速、隔离，覆盖逻辑和 UI 组件。集成测试验证完整流程，如登录、下单、设置等端到端场景，能发现集成问题、导航错误、平台差异等。集成测试通常作为 CI 的后期阶段，在单元测试通过后运行，控制数量和稳定性以平衡反馈速度。

## 集成测试

集成测试文件放在 integration_test 目录，与 lib 平级。使用 IntegrationTestWidgetsFlutterBinding.ensureInitialized 初始化绑定。调用 app.main() 启动应用，pumpAndSettle 等待应用就绪。使用与 Widget 测试相同的 find、tap、enterText 等 API。可配置 IntegrationTestDriver 和 IntegrationTest 的 reportKey 以在设备上生成报告。

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('login flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(Key('email')), 'user@test.com');
    await tester.enterText(find.byKey(Key('password')), 'password123');
    await tester.tap(find.text('Login'));
    await tester.pumpAndSettle();

    expect(find.text('Welcome'), findsOneWidget);
  });
}
```

## 运行

```bash
flutter test integration_test/app_test.dart
# 或在设备上
flutter drive --driver=test_driver/integration_test.dart --target=integration_test/app_test.dart
```

## 总结

集成测试覆盖关键用户流程，是质量保障的重要环节。合理使用 find 的 Key 提升稳定性，避免依赖易变的文本。控制测试数量，聚焦核心路径，平衡覆盖与维护成本。
