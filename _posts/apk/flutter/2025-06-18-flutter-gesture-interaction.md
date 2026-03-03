---
layout: post
title: Flutter 手势与交互完全指南
categories: flutter
tags: [Flutter, 手势]
date: 2025/6/18 09:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_024.jpg)

## 引言

Flutter 通过 GestureDetector、InkWell、Listener 等处理触摸和手势。GestureDetector 支持 tap、longPress、doubleTap、pan、scale、verticalDrag、horizontalDrag 等。InkWell 是 Material 风格的可点击区域，带水波纹效果，适用于卡片、列表项等。Listener 提供更底层的 pointer 事件（down、move、up、cancel）。手势竞技场（Gesture Arena）处理多手势冲突，子 Widget 可通过 winning 或 losing 决定谁响应。本文将介绍常用手势组件、手势冲突处理以及自定义手势识别。

## GestureDetector 与 InkWell 的选择

GestureDetector 无视觉反馈，需自行添加。InkWell 有 Material 水波纹，需在 Material 祖先下使用。对于列表项、卡片等 Material 风格的可点击区域，优先使用 InkWell。对于自定义形状或非 Material 风格，使用 GestureDetector。两者都需指定 child，手势区域为 child 的边界。

## GestureDetector

onTap、onLongPress、onDoubleTap 为点击类手势。onPanUpdate、onVerticalDragUpdate、onHorizontalDragUpdate 为拖拽类，可获取 DragUpdateDetails 的 delta。onScaleUpdate 处理缩放，ScaleUpdateDetails 包含 scale、focalPoint 等。多个手势可能冲突，例如 pan 与 verticalDrag，可通过 GestureDetector 的 behavior（HitTestBehavior.opaque 等）和子 Widget 的 GestureDetector 顺序影响竞技场结果。

```dart
GestureDetector(
  onTap: () => print('tap'),
  onLongPress: () => print('long press'),
  onDoubleTap: () => print('double tap'),
  onVerticalDragUpdate: (details) {
    // details.delta 为垂直位移
  },
  behavior: HitTestBehavior.opaque,  // 确保透明区域也能响应
  child: Container(color: Colors.blue),
)
```

## 总结

合理使用手势可提升交互体验，注意手势冲突的处理。InkWell 适合 Material 风格，GestureDetector 适合自定义。对于复杂手势（如多指、自定义轨迹），可考虑 CustomGestureRecognizer 或第三方库。
