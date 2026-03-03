---
layout: post
title: Flutter Canvas 与自定义绘制完全指南
categories: flutter
tags: [Flutter, Canvas]
date: 2025/7/5 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_025.jpg)

## 引言

CustomPainter 和 Canvas 用于自定义绘制，可绘制路径、形状、文字、图片等。Canvas 提供 drawLine、drawRect、drawCircle、drawPath、drawImage 等 API，Paint 定义颜色、描边、阴影等样式。CustomPainter 的 paint 方法在需要绘制时调用，shouldRepaint 决定何时重绘（返回 true 时触发 repaint）。适用于图表、签名板、自定义进度条、游戏等场景。本文将介绍 Canvas API、CustomPainter 的用法、性能考虑以及常见绘制模式。

## CustomPainter 与 RenderCustomPaint

CustomPaint 接收 painter 和 foregroundPainter，size 指定绘制区域。painter 在子 Widget 之下绘制，foregroundPainter 在之上。CustomPainter 的 paint 接收 Canvas 和 Size，Size 为 CustomPaint 的约束尺寸。Canvas 的坐标系原点在左上角，x 向右、y 向下。可通过 canvas.save、canvas.restore 保存和恢复变换状态，实现平移、缩放、旋转等。

## CustomPainter

paint 方法中使用 Paint 对象设置样式，调用 canvas 的 draw 方法。Path 可定义复杂形状，通过 moveTo、lineTo、quadraticBezierTo、cubicTo 等构建。drawText 需配合 TextPainter 或 ParagraphBuilder 绘制文字。shouldRepaint 在父 Widget 重建时被调用，若新旧的绘制参数相同可返回 false 避免不必要的重绘。对于动画，应在 shouldRepaint 中比较动画进度等变化量，返回 true 以触发重绘。

```dart
class MyPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue
      ..style = PaintingStyle.fill;
    canvas.drawCircle(Offset(size.width/2, size.height/2), 50, paint);

    final path = Path()
      ..moveTo(0, size.height)
      ..lineTo(size.width/2, 0)
      ..lineTo(size.width, size.height)
      ..close();
    canvas.drawPath(path, Paint()..color = Colors.green);
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) => false;
}

CustomPaint(painter: MyPainter(), size: Size.infinite)
```

## 总结

Canvas 提供底层绘制能力，适合复杂自定义图形。注意 shouldRepaint 的优化，避免不必要的重绘。对于简单形状，优先考虑 Container、DecoratedBox 等内置 Widget；复杂图表可考虑 fl_chart、syncfusion_flutter_charts 等库。
