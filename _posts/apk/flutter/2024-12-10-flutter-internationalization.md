---
layout: post
title: Flutter 国际化与本地化完全指南
categories: flutter
tags: [Flutter, 国际化]
date: 2024/12/10 09:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_013.jpg)

## 引言

Flutter 通过 flutter_localizations 和 intl 实现国际化。arb（Application Resource Bundle）文件存储各语言的翻译，格式为 JSON。通过 flutter gen-l10n 或 intl_utils 等工具可生成 Dart 代码，提供类型安全的翻译访问。MaterialApp 的 localizationsDelegates 和 supportedLocales 配置支持的语言和本地化委托。系统语言变化时，MaterialApp 会重建并更新 Locale，依赖 Locale 的 Widget 会随之更新。本文将介绍国际化的配置、arb 文件格式、代码生成以及日期、数字的本地化。

## 国际化配置

在 pubspec.yaml 的 flutter 下配置 l10n 的 arb-dir 和 output-dir。flutter gen-l10n 会读取 arb 文件并生成 AppLocalizations 类。arb 文件名格式为 app_zh.arb、app_en.arb 等，其中 app 为默认前缀，zh、en 为语言代码。无后缀的 app.arb 作为默认/回退语言。

## 配置

```yaml
flutter:
  generate: true
```

在 l10n.yaml 中配置：

```yaml
arb-dir: lib/l10n
template-arb-file: app_zh.arb
output-localization-file: app_localizations.dart
```

```dart
MaterialApp(
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
  home: HomePage(),
);
```

## 使用

AppLocalizations.of(context) 获取当前 Locale 对应的本地化实例。若 context 的 Locale 在 supportedLocales 中无匹配，会回退到默认语言。对于带参数的翻译，arb 中使用占位符如 {name}，生成的方法会接收对应参数。

```dart
Text(AppLocalizations.of(context)!.title);
Text(AppLocalizations.of(context)!.greeting('张三'));
```

## 总结

国际化可让应用支持多语言，扩大用户覆盖。合理组织 arb 文件、使用代码生成保证类型安全，并注意 RTL 语言（如阿拉伯语）的布局适配。日期、数字的格式化可使用 intl 的 DateFormat、NumberFormat 并传入 Locale。
