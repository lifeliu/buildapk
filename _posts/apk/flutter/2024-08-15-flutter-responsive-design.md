---
layout: post
title: Flutter 响应式设计与多端适配
categories: flutter
tags: [Flutter, 响应式]
date: 2024/8/15 10:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_006.jpg)

## 引言

Flutter 支持手机、平板、桌面和 Web，不同屏幕尺寸需要不同的布局策略。MediaQuery 提供屏幕尺寸、方向、padding、文字缩放等信息；LayoutBuilder 根据父级传递的约束动态构建不同布局。通过断点（如 600、900）区分手机、平板、桌面，可呈现不同的导航方式（底部导航 vs 侧边栏）和内容密度。本文将介绍响应式设计的基本思路、MediaQuery 与 LayoutBuilder 的用法以及常见适配模式。

## 为什么需要响应式设计

同一应用在手机和平板上应有不同的布局：手机可能使用单列和底部导航，平板可能使用多列和侧边栏。固定宽度会导致小屏溢出或大屏留白过多。响应式设计根据可用空间选择布局，提升各设备上的体验。Flutter 的约束驱动模型天然支持根据父级约束调整子布局，LayoutBuilder 是实现响应式的核心组件。

## MediaQuery

MediaQuery.of(context) 提供 size（屏幕逻辑像素）、orientation、padding（刘海、状态栏等）、devicePixelRatio、textScaleFactor 等。size 会随旋转变化。注意：MediaQuery 的数据来自祖先的 MediaQuery  widget，MaterialApp 会插入默认的 MediaQuery，若在路由或 overlay 中可能需手动包装。使用 MediaQuery.sizeOf(context) 可获取当前渲染区域的尺寸，在部分场景下比 MediaQuery.of(context).size 更准确。

```dart
final size = MediaQuery.of(context).size;
final isLandscape = MediaQuery.of(context).orientation == Orientation.landscape;
final padding = MediaQuery.of(context).padding;  // 安全区域
```

## LayoutBuilder

LayoutBuilder 的 builder 接收 constraints，可根据 constraints.maxWidth、constraints.maxHeight 判断可用空间并返回不同布局。例如 maxWidth > 600 时显示平板布局，否则显示手机布局。LayoutBuilder 的约束来自父级，因此会随父级尺寸变化而重建，适合实现自适应的子布局。

```dart
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth > 600) {
      return TabletLayout();  // 侧边栏 + 主内容
    }
    return MobileLayout();    // 单列
  },
)
```

## 总结

结合 MediaQuery、LayoutBuilder 和断点设计，可实现跨设备适配。将断点定义为常量，封装为 ResponsiveBuilder 等工具类，可简化各页面的响应式逻辑。
