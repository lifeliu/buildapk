---
layout: post
title: Flutter 布局系统完全指南
categories: flutter
tags: [Flutter, 布局]
date: 2024/6/25 14:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_003.jpg)

## 引言

Flutter 采用约束驱动的布局模型：父 Widget 向子 Widget 传递约束（BoxConstraints），子 Widget 在约束内选择自己的尺寸，父 Widget 根据子尺寸进行定位。这种自底向上的布局过程与 CSS 的流式布局不同，需要理解约束的传递和"紧约束"与"松约束"的区别。Row、Column、Stack、Flex 是常用布局组件，Expanded、Flexible 用于在 Row/Column 中分配剩余空间。本文将介绍约束传递、常用布局组件、Expanded 与 Flexible 的区别以及复杂布局的实现方式。

## 约束传递原理

约束包含 minWidth、maxWidth、minHeight、maxHeight。子 Widget 的 size 必须在此范围内。若 maxWidth 与 minWidth 相等，则为"紧约束"，子 Widget 必须填满该宽度；若 max 远大于 min，则为"松约束"，子 Widget 可自由选择尺寸。常见的布局错误如"RenderFlex overflowed"往往源于子 Widget 在有限约束下仍请求无限尺寸（如未包裹在 Expanded 中的 ListView）。

## 约束传递

```dart
// 约束：minWidth, maxWidth, minHeight, maxHeight
// 子 Widget 的 size 必须在约束范围内
// 无界约束（如 maxWidth = infinity）通常来自 ScrollView、Column 等
```

## 常用布局

Row 和 Column 分别沿主轴（水平/垂直）排列子项。mainAxisAlignment 控制主轴对齐，crossAxisAlignment 控制交叉轴对齐。Expanded 和 Flexible 只能作为 Row/Column 的直接子项，用于吸收剩余空间。Expanded 的 flex 默认为 1，表示按比例分配；Flexible 的 fit 为 FlexFit.tight 时类似 Expanded，为 FlexFit.loose 时子项可小于分配空间。Stack 用于层叠布局，Positioned 可指定子项的位置和尺寸，alignment 控制未定位子项的对齐方式。

```dart
// Row 水平排列
Row(
  mainAxisAlignment: MainAxisAlignment.spaceBetween,
  children: [
    Text('Left'),
    Text('Right'),
  ],
)

// Column 垂直排列，Expanded 吸收剩余空间
Column(
  crossAxisAlignment: CrossAxisAlignment.start,
  children: [
    Text('Title'),
    Expanded(child: ListView()),  // 必须包裹在 Expanded 中，否则会报错
  ],
)

// Stack 层叠，Positioned 定位
Stack(
  alignment: Alignment.center,
  children: [
    Image.network(url),
    Positioned(bottom: 0, left: 0, right: 0, child: Text('Label')),
  ],
)
```

## 总结

理解约束驱动模型是掌握 Flutter 布局的关键。遇到 overflow 时，检查是否需要在 Row/Column 中使用 Expanded 或 Flexible，或为 ScrollView 指定有限高度。合理使用 SizedBox、Spacer、Padding 等可简化布局代码。
