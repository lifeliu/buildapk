---
layout: post
title: Flutter 主题与样式完全指南
categories: flutter
tags: [Flutter, 主题]
date: 2024/7/28 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_005.jpg)

## 引言

Flutter 通过 ThemeData 统一管理颜色、字体、组件样式。MaterialApp 的 theme 和 darkTheme 分别控制浅色和深色主题，themeMode 决定使用哪个（system、light、dark）。Theme.of(context) 在子 Widget 中获取当前主题，当主题变化时依赖 Theme 的 Widget 会自动重建。本文将介绍主题配置、ColorScheme、TextTheme、深色模式以及自定义组件样式的继承与覆盖。

## 主题的作用

统一使用主题可保持应用视觉一致性，便于切换深色模式和品牌定制。ThemeData 包含 colorScheme、textTheme、appBarTheme、elevatedButtonTheme 等，Material 组件会自动使用这些样式。自定义组件应通过 Theme.of(context) 获取颜色和字体，而非硬编码，以支持主题切换。

## 主题配置

ColorScheme 定义 primary、secondary、surface、error 等角色颜色，Material 3 推荐使用 ColorScheme.fromSeed 从种子色生成协调的调色板。useMaterial3 为 true 时启用 Material 3 组件样式。textTheme 定义各文本样式，可基于 Typography 的 black 或 white 做修改。darkTheme 通常使用 ThemeData.dark() 并覆盖部分颜色。

```dart
MaterialApp(
  theme: ThemeData(
    colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
    useMaterial3: true,
    textTheme: TextTheme(
      headlineLarge: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
    ),
  ),
  darkTheme: ThemeData.dark(),
  themeMode: ThemeMode.system,  // 跟随系统
  home: HomePage(),
);
```

## 使用主题

通过 Theme.of(context) 获取 ThemeData，再访问 colorScheme、textTheme 等。若在 build 外使用，需确保 context 已包含 MaterialApp 的 Theme 祖先。部分组件可直接使用主题，如 ElevatedButton 默认使用 colorScheme.primary。

```dart
// 获取主题
final theme = Theme.of(context);
Text('Title', style: theme.textTheme.headlineMedium);
Container(color: theme.colorScheme.primary);
```

## 总结

统一使用主题可保持应用视觉一致性，便于切换深色模式。合理配置 ColorScheme 和 TextTheme，自定义组件通过 Theme.of(context) 获取样式，可构建易于维护的 UI 系统。
