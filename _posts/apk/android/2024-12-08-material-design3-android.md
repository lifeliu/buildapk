---
layout: post
title: Android Material Design 3 设计规范指南
categories: android
tags: [Material Design 3, Material You, 设计规范, Android]
date: 2024/12/8 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_044.jpg)

## 引言

Material Design 3（Material You）是 Google 在 2021 年推出的最新设计语言，随 Android 12 首次亮相。它强调个性化、动态色彩、更柔和的形状和更自然的交互。Material 3 引入了动态色彩（Dynamic Color），可从壁纸提取主色并生成协调的调色板，使应用与系统主题一致。本文介绍如何在 Android 应用中应用 Material Design 3 规范，包括主题配置、动态色彩、组件使用以及 Compose 与 View 系统的集成。

## Material 3 与 Material 2 的差异

Material 3 采用新的色彩系统（基于角色：primary、secondary、tertiary、surface 等），形状系统更灵活（支持从方形到圆角的连续变化），组件样式有所更新（如 Filled、Tonal、Outlined、Text 按钮变体）。动态色彩是 Material 3 的亮点，在支持的设备上可从壁纸生成主题色。迁移时需注意部分组件 API 和样式名称的变化，参考官方迁移指南逐步调整。

## 添加依赖

```kotlin
dependencies {
    implementation("com.google.android.material:material:1.11.0")
}
```

Material 库包含 Material 3 组件和主题。若使用 Compose，需依赖 `androidx.compose.material3`。

## 主题配置

在 `themes.xml` 中继承 `Theme.Material3.DayNight.NoActionBar` 等父主题，覆盖 `colorPrimary`、`colorSurface` 等属性。DayNight 主题会根据系统深色模式自动切换。可定义多套主题（如浅色、深色、动态色），在 Application 或 Activity 中按需设置。

```xml
<style name="Theme.MyApp" parent="Theme.Material3.DayNight.NoActionBar">
    <item name="colorPrimary">@color/md_theme_primary</item>
    <item name="colorSecondary">@color/md_theme_secondary</item>
    <item name="colorTertiary">@color/md_theme_tertiary</item>
    <item name="colorSurface">@color/md_theme_surface</item>
</style>
```

## 动态色彩 (Android 12+)

Android 12 引入了 `DynamicColors`，可从系统获取动态主题并应用到 Activity。使用 `wrapContextForDynamicThemes` 包装 Context，然后 `setTheme` 使用支持动态色的主题。用户可在系统设置中开启或关闭动态色彩，应用应尊重用户选择。低版本设备可回退到静态主题。

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    val dynamicColor = true
    val superContext = wrapContextForDynamicThemes(this)
    val dynamicColorTheme = if (dynamicColor) {
        R.style.Theme_MyApp_Dynamic
    } else {
        R.style.Theme_MyApp
    }
    superContext.setTheme(dynamicColorTheme)
}
```

## Material 3 组件

Material 3 提供 Filled、Tonal、Outlined、Text 等按钮变体，以及 Filled、Outlined、Elevated 等卡片变体。在 Compose 中，`Button`、`OutlinedButton`、`TextButton` 对应不同样式；`Card`、`OutlinedCard`、`ElevatedCard` 提供不同层级感。使用 `MaterialTheme.colorScheme`、`MaterialTheme.typography`、`MaterialTheme.shapes` 保持一致性。组件应遵循 Material 3 的交互规范（如点击涟漪、状态反馈）。

```kotlin
// Filled Button
Button(
    onClick = { },
    modifier = Modifier.fillMaxWidth()
) {
    Text("Primary Action")
}

// Outlined Card
OutlinedCard(
    onClick = { },
    modifier = Modifier.padding(8.dp)
) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text("Card Title", style = MaterialTheme.typography.titleMedium)
        Text("Card content", style = MaterialTheme.typography.bodyMedium)
    }
}
```

## 总结

Material Design 3 提供了现代化的设计组件和主题系统，能够提升应用的整体视觉体验。合理使用动态色彩、色彩角色和组件变体，可以构建符合 Material 3 规范、与系统协调一致的应用界面。
