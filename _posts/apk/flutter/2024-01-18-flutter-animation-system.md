---
layout: post
title: "Flutter动画系统深度解析：从基础到高级动画实现"
date: 2024-01-18
categories: flutter
tags: [Flutter, Animation, UI, 性能优化, 自定义动画]
author: "Flutter开发团队"
description: "深入探讨Flutter动画系统的核心原理、实现机制和最佳实践，包括基础动画、自定义动画、复杂动画组合、性能优化等内容，帮助开发者构建流畅优雅的用户界面。"
---

动画是现代移动应用用户体验的重要组成部分，能够为用户提供直观的反馈、引导用户注意力并增强应用的视觉吸引力。Flutter提供了强大而灵活的动画系统，支持从简单的属性动画到复杂的自定义动画。本文将深入探讨Flutter动画系统的核心原理、实现机制和最佳实践。

## Flutter动画系统概述

### 动画系统架构

Flutter的动画系统基于几个核心概念构建：

```dart
// lib/src/animation/animation_core.dart

/// 动画系统核心组件
abstract class AnimationCore {
  /// 动画控制器 - 控制动画的播放、暂停、重置等
  AnimationController? controller;
  
  /// 动画对象 - 定义动画的值变化
  Animation<double>? animation;
  
  /// 补间动画 - 定义起始值和结束值之间的插值
  Tween<double>? tween;
  
  /// 曲线动画 - 定义动画的时间曲线
  CurvedAnimation? curvedAnimation;
}

/// 动画状态枚举
enum AnimationStatus {
  /// 动画已停止在开始位置
  dismissed,
  /// 动画正在从开始向结束运行
  forward,
  /// 动画正在从结束向开始运行
  reverse,
  /// 动画已停止在结束位置
  completed,
}

/// 动画监听器接口
abstract class AnimationListener {
  /// 动画值变化回调
  void onAnimationUpdate(double value);
  
  /// 动画状态变化回调
  void onAnimationStatusChanged(AnimationStatus status);
}

/// 基础动画管理器
class AnimationManager with ChangeNotifier {
  final Map<String, AnimationController> _controllers = {};
  final Map<String, Animation> _animations = {};
  
  /// 创建动画控制器
  AnimationController createController({
    required String name,
    required Duration duration,
    Duration? reverseDuration,
    String? debugLabel,
    double lowerBound = 0.0,
    double upperBound = 1.0,
    AnimationBehavior animationBehavior = AnimationBehavior.normal,
    required TickerProvider vsync,
  }) {
    final controller = AnimationController(
      duration: duration,
      reverseDuration: reverseDuration,
      debugLabel: debugLabel,
      lowerBound: lowerBound,
      upperBound: upperBound,
      animationBehavior: animationBehavior,
      vsync: vsync,
    );
    
    _controllers[name] = controller;
    return controller;
  }
  
  /// 创建补间动画
  Animation<T> createTweenAnimation<T>({
    required String name,
    required AnimationController controller,
    required Tween<T> tween,
    Curve curve = Curves.linear,
  }) {
    final curvedAnimation = CurvedAnimation(
      parent: controller,
      curve: curve,
    );
    
    final animation = tween.animate(curvedAnimation);
    _animations[name] = animation;
    
    return animation;
  }
  
  /// 获取动画控制器
  AnimationController? getController(String name) {
    return _controllers[name];
  }
  
  /// 获取动画
  Animation? getAnimation(String name) {
    return _animations[name];
  }
  
  /// 播放动画
  Future<void> playAnimation(String name) async {
    final controller = _controllers[name];
    if (controller != null) {
      await controller.forward();
    }
  }
  
  /// 反向播放动画
  Future<void> reverseAnimation(String name) async {
    final controller = _controllers[name];
    if (controller != null) {
      await controller.reverse();
    }
  }
  
  /// 重置动画
  void resetAnimation(String name) {
    final controller = _controllers[name];
    controller?.reset();
  }
  
  /// 停止动画
  void stopAnimation(String name) {
    final controller = _controllers[name];
    controller?.stop();
  }
  
  /// 释放所有资源
  void dispose() {
    for (final controller in _controllers.values) {
      controller.dispose();
    }
    _controllers.clear();
    _animations.clear();
    super.dispose();
  }
}
```

### 动画类型分类

Flutter支持多种类型的动画：

```dart
// lib/src/animation/animation_types.dart

/// 动画类型枚举
enum AnimationType {
  /// 基础属性动画（透明度、位置、大小等）
  property,
  /// 路径动画
  path,
  /// 物理动画（弹簧、重力等）
  physics,
  /// 序列动画
  sequence,
  /// 并行动画
  parallel,
  /// 交错动画
  staggered,
  /// 自定义动画
  custom,
}

/// 动画配置基类
abstract class AnimationConfig {
  final Duration duration;
  final Curve curve;
  final bool autoReverse;
  final int repeatCount;
  
  const AnimationConfig({
    required this.duration,
    this.curve = Curves.easeInOut,
    this.autoReverse = false,
    this.repeatCount = 1,
  });
}

/// 属性动画配置
class PropertyAnimationConfig extends AnimationConfig {
  final double fromValue;
  final double toValue;
  final AnimationProperty property;
  
  const PropertyAnimationConfig({
    required this.fromValue,
    required this.toValue,
    required this.property,
    required Duration duration,
    Curve curve = Curves.easeInOut,
    bool autoReverse = false,
    int repeatCount = 1,
  }) : super(
    duration: duration,
    curve: curve,
    autoReverse: autoReverse,
    repeatCount: repeatCount,
  );
}

/// 动画属性枚举
enum AnimationProperty {
  opacity,
  scale,
  rotation,
  translation,
  color,
  size,
}

/// 路径动画配置
class PathAnimationConfig extends AnimationConfig {
  final Path path;
  final bool followPath;
  
  const PathAnimationConfig({
    required this.path,
    this.followPath = true,
    required Duration duration,
    Curve curve = Curves.easeInOut,
    bool autoReverse = false,
    int repeatCount = 1,
  }) : super(
    duration: duration,
    curve: curve,
    autoReverse: autoReverse,
    repeatCount: repeatCount,
  );
}

/// 物理动画配置
class PhysicsAnimationConfig extends AnimationConfig {
  final SpringDescription spring;
  final double velocity;
  final double target;
  
  const PhysicsAnimationConfig({
    required this.spring,
    this.velocity = 0.0,
    required this.target,
    required Duration duration,
  }) : super(duration: duration);
}

/// 弹簧描述
class SpringDescription {
  final double mass;
  final double stiffness;
  final double damping;
  
  const SpringDescription({
    this.mass = 1.0,
    this.stiffness = 100.0,
    this.damping = 10.0,
  });
  
  /// 预定义弹簧配置
  static const SpringDescription soft = SpringDescription(
    mass: 1.0,
    stiffness: 50.0,
    damping: 8.0,
  );
  
  static const SpringDescription medium = SpringDescription(
    mass: 1.0,
    stiffness: 100.0,
    damping: 10.0,
  );
  
  static const SpringDescription stiff = SpringDescription(
    mass: 1.0,
    stiffness: 200.0,
    damping: 15.0,
  );
}
```

## 基础动画实现

### 简单属性动画

最常见的动画类型是属性动画，如透明度、大小、位置变化：

```dart
// lib/src/animation/basic_animations.dart

/// 基础动画Widget
class BasicAnimationWidget extends StatefulWidget {
  final Widget child;
  final AnimationConfig config;
  final VoidCallback? onAnimationComplete;
  
  const BasicAnimationWidget({
    Key? key,
    required this.child,
    required this.config,
    this.onAnimationComplete,
  }) : super(key: key);
  
  @override
  State<SpringButton> createState() => _SpringButtonState();
}

class _SpringButtonState extends State<SpringButton>
    with TickerProviderStateMixin {
  late SpringAnimationController _springController;
  late Animation<double> _scaleAnimation;
  bool _isPressed = false;
  
  @override
  void initState() {
    super.initState();
    
    _springController = SpringAnimationController(vsync: this);
    _scaleAnimation = _springController.createSpringAnimation(
      from: 1.0,
      to: widget.scaleDown,
      spring: widget.spring,
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: _onTapDown,
      onTapUp: _onTapUp,
      onTapCancel: _onTapCancel,
      onTap: widget.onTap,
      child: AnimatedBuilder(
        animation: _scaleAnimation,
        builder: (context, child) {
          return Transform.scale(
            scale: _isPressed ? _scaleAnimation.value : 1.0,
            child: widget.child,
          );
        },
      ),
    );
  }
  
  void _onTapDown(TapDownDetails details) {
    setState(() {
      _isPressed = true;
    });
    _springController.forward();
  }
  
  void _onTapUp(TapUpDetails details) {
    _resetScale();
  }
  
  void _onTapCancel() {
    _resetScale();
  }
  
  void _resetScale() {
    setState(() {
      _isPressed = false;
    });
    
    // 创建回弹动画
    final resetController = SpringAnimationController(vsync: this);
    final resetAnimation = resetController.createSpringAnimation(
      from: widget.scaleDown,
      to: 1.0,
      spring: widget.spring,
    );
    
    resetController.forward(onComplete: () {
      resetController.dispose();
    });
  }
  
  @override
  void dispose() {
    _springController.dispose();
    super.dispose();
  }
}

/// 重力动画
class GravityAnimationWidget extends StatefulWidget {
  final Widget child;
  final double gravity;
  final double initialVelocity;
  final double bounciness;
  final VoidCallback? onComplete;
  
  const GravityAnimationWidget({
    Key? key,
    required this.child,
    this.gravity = 9.8,
    this.initialVelocity = 0.0,
    this.bounciness = 0.8,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<GravityAnimationWidget> createState() => _GravityAnimationWidgetState();
}

class _GravityAnimationWidgetState extends State<GravityAnimationWidget>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;
  
  double _currentPosition = 0.0;
  double _currentVelocity = 0.0;
  bool _isAnimating = false;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: const Duration(seconds: 3),
      vsync: this,
    );
    
    _currentVelocity = widget.initialVelocity;
    _startGravityAnimation();
  }
  
  void _startGravityAnimation() {
    _isAnimating = true;
    
    _animation = Tween<double>(
      begin: _currentPosition,
      end: 300.0, // 假设容器高度为300
    ).animate(_controller);
    
    _controller.addListener(_updatePosition);
    _controller.addStatusListener(_onAnimationStatus);
    
    _controller.forward();
  }
  
  void _updatePosition() {
    final progress = _controller.value;
    final deltaTime = 1.0 / 60.0; // 假设60fps
    
    // 应用重力
    _currentVelocity += widget.gravity * deltaTime;
    _currentPosition += _currentVelocity * deltaTime;
    
    // 检查边界碰撞
    if (_currentPosition >= 300.0) {
      _currentPosition = 300.0;
      _currentVelocity = -_currentVelocity * widget.bounciness;
      
      // 如果速度太小，停止动画
      if (_currentVelocity.abs() < 0.1) {
        _isAnimating = false;
        widget.onComplete?.call();
      }
    }
  }
  
  void _onAnimationStatus(AnimationStatus status) {
    if (status == AnimationStatus.completed && _isAnimating) {
      _controller.reset();
      _controller.forward();
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return Transform.translate(
          offset: Offset(0, _currentPosition),
          child: widget.child,
        );
      },
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

## 路径动画与自定义动画

### 路径动画实现

路径动画允许Widget沿着自定义路径移动：

```dart
// lib/src/animation/path_animations.dart

/// 路径动画控制器
class PathAnimationController {
  final Path _path;
  final TickerProvider _vsync;
  final Duration _duration;
  final Curve _curve;
  
  late AnimationController _controller;
  late Animation<double> _animation;
  late PathMetrics _pathMetrics;
  
  PathAnimationController({
    required Path path,
    required TickerProvider vsync,
    required Duration duration,
    Curve curve = Curves.linear,
  }) : _path = path,
       _vsync = vsync,
       _duration = duration,
       _curve = curve {
    _initializeAnimation();
  }
  
  void _initializeAnimation() {
    _controller = AnimationController(
      duration: _duration,
      vsync: _vsync,
    );
    
    _animation = CurvedAnimation(
      parent: _controller,
      curve: _curve,
    );
    
    _pathMetrics = _path.computeMetrics();
  }
  
  /// 获取路径上指定进度的位置
  Offset getPositionAtProgress(double progress) {
    if (_pathMetrics.isEmpty) return Offset.zero;
    
    final pathMetric = _pathMetrics.first;
    final distance = pathMetric.length * progress.clamp(0.0, 1.0);
    final tangent = pathMetric.getTangentForOffset(distance);
    
    return tangent?.position ?? Offset.zero;
  }
  
  /// 获取路径上指定进度的切线角度
  double getAngleAtProgress(double progress) {
    if (_pathMetrics.isEmpty) return 0.0;
    
    final pathMetric = _pathMetrics.first;
    final distance = pathMetric.length * progress.clamp(0.0, 1.0);
    final tangent = pathMetric.getTangentForOffset(distance);
    
    if (tangent?.vector != null) {
      return math.atan2(tangent!.vector.dy, tangent.vector.dx);
    }
    
    return 0.0;
  }
  
  /// 开始路径动画
  Future<void> forward() async {
    await _controller.forward();
  }
  
  /// 反向播放路径动画
  Future<void> reverse() async {
    await _controller.reverse();
  }
  
  /// 重置动画
  void reset() {
    _controller.reset();
  }
  
  /// 获取动画对象
  Animation<double> get animation => _animation;
  
  /// 释放资源
  void dispose() {
    _controller.dispose();
  }
}

/// 路径动画Widget
class PathAnimationWidget extends StatefulWidget {
  final Widget child;
  final Path path;
  final Duration duration;
  final Curve curve;
  final bool followPath;
  final bool autoStart;
  final VoidCallback? onComplete;
  
  const PathAnimationWidget({
    Key? key,
    required this.child,
    required this.path,
    this.duration = const Duration(seconds: 2),
    this.curve = Curves.easeInOut,
    this.followPath = true,
    this.autoStart = true,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<PathAnimationWidget> createState() => _PathAnimationWidgetState();
}

class _PathAnimationWidgetState extends State<PathAnimationWidget>
    with TickerProviderStateMixin {
  late PathAnimationController _pathController;
  
  @override
  void initState() {
    super.initState();
    
    _pathController = PathAnimationController(
      path: widget.path,
      vsync: this,
      duration: widget.duration,
      curve: widget.curve,
    );
    
    _pathController.animation.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onComplete?.call();
      }
    });
    
    if (widget.autoStart) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _pathController.forward();
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _pathController.animation,
      builder: (context, child) {
        final progress = _pathController.animation.value;
        final position = _pathController.getPositionAtProgress(progress);
        
        Widget animatedChild = widget.child;
        
        if (widget.followPath) {
          final angle = _pathController.getAngleAtProgress(progress);
          animatedChild = Transform.rotate(
            angle: angle,
            child: animatedChild,
          );
        }
        
        return Transform.translate(
          offset: position,
          child: animatedChild,
        );
      },
    );
  }
  
  @override
  void dispose() {
    _pathController.dispose();
    super.dispose();
  }
}

/// 预定义路径生成器
class PathGenerator {
  /// 创建圆形路径
  static Path createCirclePath({
    required Offset center,
    required double radius,
    bool clockwise = true,
  }) {
    final path = Path();
    final rect = Rect.fromCircle(center: center, radius: radius);
    
    path.addOval(rect);
    
    if (!clockwise) {
      // 反向路径
      final reversePath = Path();
      for (double angle = 2 * math.pi; angle >= 0; angle -= 0.1) {
        final x = center.dx + radius * math.cos(angle);
        final y = center.dy + radius * math.sin(angle);
        
        if (angle == 2 * math.pi) {
          reversePath.moveTo(x, y);
        } else {
          reversePath.lineTo(x, y);
        }
      }
      return reversePath;
    }
    
    return path;
  }
  
  /// 创建波浪路径
  static Path createWavePath({
    required Offset start,
    required Offset end,
    required double amplitude,
    required int frequency,
  }) {
    final path = Path();
    final distance = (end - start).distance;
    final direction = (end - start) / distance;
    
    path.moveTo(start.dx, start.dy);
    
    for (double t = 0; t <= 1; t += 0.01) {
      final basePosition = start + direction * distance * t;
      final waveOffset = amplitude * math.sin(frequency * 2 * math.pi * t);
      
      // 计算垂直于方向的偏移
      final perpendicular = Offset(-direction.dy, direction.dx);
      final wavePosition = basePosition + perpendicular * waveOffset;
      
      path.lineTo(wavePosition.dx, wavePosition.dy);
    }
    
    return path;
  }
  
  /// 创建贝塞尔曲线路径
  static Path createBezierPath({
    required Offset start,
    required Offset end,
    required List<Offset> controlPoints,
  }) {
    final path = Path();
    path.moveTo(start.dx, start.dy);
    
    if (controlPoints.length == 1) {
      // 二次贝塞尔曲线
      path.quadraticBezierTo(
        controlPoints[0].dx,
        controlPoints[0].dy,
        end.dx,
        end.dy,
      );
    } else if (controlPoints.length == 2) {
      // 三次贝塞尔曲线
      path.cubicTo(
        controlPoints[0].dx,
        controlPoints[0].dy,
        controlPoints[1].dx,
        controlPoints[1].dy,
        end.dx,
        end.dy,
      );
    }
    
    return path;
  }
  
  /// 创建螺旋路径
  static Path createSpiralPath({
    required Offset center,
    required double startRadius,
    required double endRadius,
    required double turns,
  }) {
    final path = Path();
    final totalAngle = turns * 2 * math.pi;
    
    bool isFirst = true;
    
    for (double angle = 0; angle <= totalAngle; angle += 0.1) {
      final progress = angle / totalAngle;
      final radius = startRadius + (endRadius - startRadius) * progress;
      
      final x = center.dx + radius * math.cos(angle);
      final y = center.dy + radius * math.sin(angle);
      
      if (isFirst) {
        path.moveTo(x, y);
        isFirst = false;
      } else {
        path.lineTo(x, y);
      }
    }
    
    return path;
  }
  
  /// 创建心形路径
  static Path createHeartPath({
    required Offset center,
    required double size,
  }) {
    final path = Path();
    
    bool isFirst = true;
    
    for (double t = 0; t <= 2 * math.pi; t += 0.01) {
      // 心形参数方程
      final x = center.dx + size * (16 * math.pow(math.sin(t), 3));
      final y = center.dy - size * (13 * math.cos(t) - 5 * math.cos(2 * t) - 2 * math.cos(3 * t) - math.cos(4 * t));
      
      if (isFirst) {
        path.moveTo(x, y);
        isFirst = false;
      } else {
        path.lineTo(x, y);
      }
    }
    
    path.close();
    return path;
  }
}

/// 路径动画示例Widget
class PathAnimationExample extends StatefulWidget {
  @override
  State<PathAnimationExample> createState() => _PathAnimationExampleState();
}

class _PathAnimationExampleState extends State<PathAnimationExample> {
  PathAnimationType _currentType = PathAnimationType.circle;
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('路径动画示例'),
      ),
      body: Column(
        children: [
          // 路径类型选择器
          Wrap(
            children: PathAnimationType.values.map((type) {
              return Padding(
                padding: const EdgeInsets.all(4.0),
                child: ChoiceChip(
                  label: Text(_getPathTypeName(type)),
                  selected: _currentType == type,
                  onSelected: (selected) {
                    if (selected) {
                      setState(() {
                        _currentType = type;
                      });
                    }
                  },
                ),
              );
            }).toList(),
          ),
          
          // 动画展示区域
          Expanded(
            child: Container(
              width: double.infinity,
              child: _buildPathAnimation(),
            ),
          ),
        ],
      ),
    );
  }
  
  Widget _buildPathAnimation() {
    final center = Offset(200, 200);
    Path path;
    
    switch (_currentType) {
      case PathAnimationType.circle:
        path = PathGenerator.createCirclePath(
          center: center,
          radius: 100,
        );
        break;
      
      case PathAnimationType.wave:
        path = PathGenerator.createWavePath(
          start: Offset(50, 200),
          end: Offset(350, 200),
          amplitude: 50,
          frequency: 3,
        );
        break;
      
      case PathAnimationType.bezier:
        path = PathGenerator.createBezierPath(
          start: Offset(50, 300),
          end: Offset(350, 100),
          controlPoints: [Offset(150, 50), Offset(250, 350)],
        );
        break;
      
      case PathAnimationType.spiral:
        path = PathGenerator.createSpiralPath(
          center: center,
          startRadius: 20,
          endRadius: 100,
          turns: 3,
        );
        break;
      
      case PathAnimationType.heart:
        path = PathGenerator.createHeartPath(
          center: center,
          size: 5,
        );
        break;
    }
    
    return PathAnimationWidget(
      key: ValueKey(_currentType),
      path: path,
      duration: Duration(seconds: 3),
      curve: Curves.easeInOut,
      followPath: true,
      child: Container(
        width: 20,
        height: 20,
        decoration: BoxDecoration(
          color: Colors.blue,
          shape: BoxShape.circle,
        ),
      ),
    );
  }
  
  String _getPathTypeName(PathAnimationType type) {
    switch (type) {
      case PathAnimationType.circle:
        return '圆形';
      case PathAnimationType.wave:
        return '波浪';
      case PathAnimationType.bezier:
        return '贝塞尔';
      case PathAnimationType.spiral:
        return '螺旋';
      case PathAnimationType.heart:
        return '心形';
    }
  }
}

enum PathAnimationType {
  circle,
  wave,
  bezier,
  spiral,
  heart,
}
```

### 自定义动画实现

自定义动画允许开发者创建完全定制的动画效果：

```dart
// lib/src/animation/custom_animations.dart

/// 自定义动画基类
abstract class CustomAnimation {
  final Duration duration;
  final Curve curve;
  
  const CustomAnimation({
    required this.duration,
    this.curve = Curves.linear,
  });
  
  /// 计算指定进度的动画值
  double calculateValue(double progress);
  
  /// 应用动画值到Widget
  Widget applyAnimation(Widget child, double value);
}

/// 波动动画
class WaveAnimation extends CustomAnimation {
  final double amplitude;
  final double frequency;
  final WaveDirection direction;
  
  const WaveAnimation({
    required Duration duration,
    this.amplitude = 10.0,
    this.frequency = 2.0,
    this.direction = WaveDirection.horizontal,
    Curve curve = Curves.linear,
  }) : super(duration: duration, curve: curve);
  
  @override
  double calculateValue(double progress) {
    return amplitude * math.sin(frequency * 2 * math.pi * progress);
  }
  
  @override
  Widget applyAnimation(Widget child, double value) {
    final offset = direction == WaveDirection.horizontal
        ? Offset(value, 0)
        : Offset(0, value);
    
    return Transform.translate(
      offset: offset,
      child: child,
    );
  }
}

/// 脉冲动画
class PulseAnimation extends CustomAnimation {
  final double minScale;
  final double maxScale;
  
  const PulseAnimation({
    required Duration duration,
    this.minScale = 0.8,
    this.maxScale = 1.2,
    Curve curve = Curves.easeInOut,
  }) : super(duration: duration, curve: curve);
  
  @override
  double calculateValue(double progress) {
    // 创建脉冲效果：0 -> 1 -> 0
    final pulseProgress = math.sin(progress * math.pi);
    return minScale + (maxScale - minScale) * pulseProgress;
  }
  
  @override
  Widget applyAnimation(Widget child, double value) {
    return Transform.scale(
      scale: value,
      child: child,
    );
  }
}

/// 抖动动画
class ShakeAnimation extends CustomAnimation {
  final double intensity;
  final int shakeCount;
  
  const ShakeAnimation({
    required Duration duration,
    this.intensity = 5.0,
    this.shakeCount = 3,
    Curve curve = Curves.elasticInOut,
  }) : super(duration: duration, curve: curve);
  
  @override
  double calculateValue(double progress) {
    return intensity * math.sin(shakeCount * 2 * math.pi * progress) * (1 - progress);
  }
  
  @override
  Widget applyAnimation(Widget child, double value) {
    return Transform.translate(
      offset: Offset(value, 0),
      child: child,
    );
  }
}

/// 翻转动画
class FlipAnimation extends CustomAnimation {
  final FlipDirection direction;
  
  const FlipAnimation({
    required Duration duration,
    this.direction = FlipDirection.horizontal,
    Curve curve = Curves.easeInOut,
  }) : super(duration: duration, curve: curve);
  
  @override
  double calculateValue(double progress) {
    return progress * math.pi;
  }
  
  @override
  Widget applyAnimation(Widget child, double value) {
    if (direction == FlipDirection.horizontal) {
      return Transform(
        alignment: Alignment.center,
        transform: Matrix4.identity()
          ..setEntry(3, 2, 0.001)
          ..rotateY(value),
        child: child,
      );
    } else {
      return Transform(
        alignment: Alignment.center,
        transform: Matrix4.identity()
          ..setEntry(3, 2, 0.001)
          ..rotateX(value),
        child: child,
      );
    }
  }
}

/// 自定义动画Widget
class CustomAnimationWidget extends StatefulWidget {
  final Widget child;
  final CustomAnimation animation;
  final bool autoStart;
  final bool repeat;
  final VoidCallback? onComplete;
  
  const CustomAnimationWidget({
    Key? key,
    required this.child,
    required this.animation,
    this.autoStart = true,
    this.repeat = false,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<CustomAnimationWidget> createState() => _CustomAnimationWidgetState();
}

class _CustomAnimationWidgetState extends State<CustomAnimationWidget>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: widget.animation.duration,
      vsync: this,
    );
    
    _animation = CurvedAnimation(
      parent: _controller,
      curve: widget.animation.curve,
    );
    
    _controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onComplete?.call();
        
        if (widget.repeat) {
          _controller.reset();
          _controller.forward();
        }
      }
    });
    
    if (widget.autoStart) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _controller.forward();
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        final progress = _animation.value;
        final animationValue = widget.animation.calculateValue(progress);
        
        return widget.animation.applyAnimation(widget.child, animationValue);
      },
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

/// 动画方向枚举
enum WaveDirection {
  horizontal,
  vertical,
}

enum FlipDirection {
  horizontal,
  vertical,
}

/// 复合自定义动画
class CompositeAnimation extends CustomAnimation {
  final List<CustomAnimation> animations;
  final CompositeMode mode;
  
  const CompositeAnimation({
    required this.animations,
    required Duration duration,
    this.mode = CompositeMode.sequential,
    Curve curve = Curves.linear,
  }) : super(duration: duration, curve: curve);
  
  @override
  double calculateValue(double progress) {
    // 复合动画的值计算取决于模式
    switch (mode) {
      case CompositeMode.sequential:
        return _calculateSequentialValue(progress);
      case CompositeMode.parallel:
        return _calculateParallelValue(progress);
    }
  }
  
  double _calculateSequentialValue(double progress) {
    final stepDuration = 1.0 / animations.length;
    final currentStep = (progress / stepDuration).floor();
    final stepProgress = (progress % stepDuration) / stepDuration;
    
    if (currentStep >= animations.length) {
      return animations.last.calculateValue(1.0);
    }
    
    return animations[currentStep].calculateValue(stepProgress);
  }
  
  double _calculateParallelValue(double progress) {
    // 并行模式下，返回所有动画值的平均值
    double totalValue = 0.0;
    for (final animation in animations) {
      totalValue += animation.calculateValue(progress);
    }
    return totalValue / animations.length;
  }
  
  @override
  Widget applyAnimation(Widget child, double value) {
    // 应用第一个动画的变换方式
    if (animations.isNotEmpty) {
      return animations.first.applyAnimation(child, value);
    }
    return child;
  }
}

enum CompositeMode {
  sequential,
  parallel,
}
```

## 动画性能优化

### 性能监控与分析

动画性能对用户体验至关重要，需要持续监控和优化：

```dart
// lib/src/animation/performance_monitor.dart

/// 动画性能监控器
class AnimationPerformanceMonitor {
  static final AnimationPerformanceMonitor _instance = AnimationPerformanceMonitor._internal();
  factory AnimationPerformanceMonitor() => _instance;
  AnimationPerformanceMonitor._internal();
  
  final Map<String, AnimationMetrics> _metrics = {};
  final List<PerformanceListener> _listeners = [];
  
  /// 开始监控动画
  void startMonitoring(String animationId) {
    _metrics[animationId] = AnimationMetrics(
      animationId: animationId,
      startTime: DateTime.now(),
    );
  }
  
  /// 记录帧渲染时间
  void recordFrameTime(String animationId, Duration frameTime) {
    final metrics = _metrics[animationId];
    if (metrics != null) {
      metrics.frameTimes.add(frameTime);
      metrics.totalFrames++;
      
      // 检查是否有掉帧
      if (frameTime.inMilliseconds > 16) { // 60fps = 16.67ms per frame
        metrics.droppedFrames++;
      }
      
      _notifyListeners(metrics);
    }
  }
  
  /// 结束监控
  void stopMonitoring(String animationId) {
    final metrics = _metrics[animationId];
    if (metrics != null) {
      metrics.endTime = DateTime.now();
      metrics.duration = metrics.endTime!.difference(metrics.startTime);
      
      _calculateStatistics(metrics);
      _notifyListeners(metrics);
    }
  }
  
  void _calculateStatistics(AnimationMetrics metrics) {
    if (metrics.frameTimes.isNotEmpty) {
      // 计算平均帧时间
      final totalTime = metrics.frameTimes.fold<Duration>(
        Duration.zero,
        (sum, time) => sum + time,
      );
      metrics.averageFrameTime = Duration(
        microseconds: totalTime.inMicroseconds ~/ metrics.frameTimes.length,
      );
      
      // 计算最大帧时间
      metrics.maxFrameTime = metrics.frameTimes.reduce(
        (max, time) => time > max ? time : max,
      );
      
      // 计算帧率
      metrics.averageFps = 1000 / metrics.averageFrameTime.inMilliseconds;
      
      // 计算掉帧率
      metrics.dropRate = metrics.droppedFrames / metrics.totalFrames;
    }
  }
  
  /// 添加性能监听器
  void addListener(PerformanceListener listener) {
    _listeners.add(listener);
  }
  
  /// 移除性能监听器
  void removeListener(PerformanceListener listener) {
    _listeners.remove(listener);
  }
  
  void _notifyListeners(AnimationMetrics metrics) {
    for (final listener in _listeners) {
      listener.onMetricsUpdated(metrics);
    }
  }
  
  /// 获取动画指标
  AnimationMetrics? getMetrics(String animationId) {
    return _metrics[animationId];
  }
  
  /// 获取所有指标
  Map<String, AnimationMetrics> getAllMetrics() {
    return Map.unmodifiable(_metrics);
  }
  
  /// 清除指标
  void clearMetrics() {
    _metrics.clear();
  }
  
  /// 生成性能报告
  PerformanceReport generateReport() {
    final allMetrics = _metrics.values.toList();
    
    if (allMetrics.isEmpty) {
      return PerformanceReport.empty();
    }
    
    // 计算总体统计
    final totalAnimations = allMetrics.length;
    final averageFps = allMetrics
        .map((m) => m.averageFps)
        .reduce((a, b) => a + b) / totalAnimations;
    
    final totalDroppedFrames = allMetrics
        .map((m) => m.droppedFrames)
        .reduce((a, b) => a + b);
    
    final totalFrames = allMetrics
        .map((m) => m.totalFrames)
        .reduce((a, b) => a + b);
    
    final overallDropRate = totalDroppedFrames / totalFrames;
    
    return PerformanceReport(
      totalAnimations: totalAnimations,
      averageFps: averageFps,
      overallDropRate: overallDropRate,
      metrics: allMetrics,
    );
  }
}

/// 动画性能指标
class AnimationMetrics {
  final String animationId;
  final DateTime startTime;
  DateTime? endTime;
  Duration? duration;
  
  final List<Duration> frameTimes = [];
  int totalFrames = 0;
  int droppedFrames = 0;
  
  Duration averageFrameTime = Duration.zero;
  Duration maxFrameTime = Duration.zero;
  double averageFps = 0.0;
  double dropRate = 0.0;
  
  AnimationMetrics({
    required this.animationId,
    required this.startTime,
  });
  
  /// 是否为高性能动画（60fps，掉帧率<5%）
  bool get isHighPerformance {
    return averageFps >= 55 && dropRate < 0.05;
  }
  
  /// 性能等级
  PerformanceLevel get performanceLevel {
    if (averageFps >= 55 && dropRate < 0.05) {
      return PerformanceLevel.excellent;
    } else if (averageFps >= 45 && dropRate < 0.1) {
      return PerformanceLevel.good;
    } else if (averageFps >= 30 && dropRate < 0.2) {
      return PerformanceLevel.fair;
    } else {
      return PerformanceLevel.poor;
    }
  }
}

/// 性能监听器接口
abstract class PerformanceListener {
  void onMetricsUpdated(AnimationMetrics metrics);
}

/// 性能等级枚举
enum PerformanceLevel {
  excellent,
  good,
  fair,
  poor,
}

/// 性能报告
class PerformanceReport {
  final int totalAnimations;
  final double averageFps;
  final double overallDropRate;
  final List<AnimationMetrics> metrics;
  
  const PerformanceReport({
    required this.totalAnimations,
    required this.averageFps,
    required this.overallDropRate,
    required this.metrics,
  });
  
  factory PerformanceReport.empty() {
    return const PerformanceReport(
      totalAnimations: 0,
      averageFps: 0.0,
      overallDropRate: 0.0,
      metrics: [],
    );
  }
  
  /// 获取性能建议
  List<String> getPerformanceRecommendations() {
    final recommendations = <String>[];
    
    if (averageFps < 45) {
      recommendations.add('整体帧率偏低，建议优化动画复杂度');
    }
    
    if (overallDropRate > 0.1) {
      recommendations.add('掉帧率过高，建议减少同时运行的动画数量');
    }
    
    final poorPerformanceAnimations = metrics
        .where((m) => m.performanceLevel == PerformanceLevel.poor)
        .toList();
    
    if (poorPerformanceAnimations.isNotEmpty) {
      recommendations.add(
        '发现${poorPerformanceAnimations.length}个低性能动画，建议重点优化',
      );
    }
    
    if (recommendations.isEmpty) {
      recommendations.add('动画性能良好，继续保持');
    }
    
    return recommendations;
  }
}

/// 性能优化的动画Widget
class OptimizedAnimationWidget extends StatefulWidget {
  final Widget child;
  final Animation<double> animation;
  final String? animationId;
  final bool enablePerformanceMonitoring;
  
  const OptimizedAnimationWidget({
    Key? key,
    required this.child,
    required this.animation,
    this.animationId,
    this.enablePerformanceMonitoring = false,
  }) : super(key: key);
  
  @override
  State<OptimizedAnimationWidget> createState() => _OptimizedAnimationWidgetState();
}

class _OptimizedAnimationWidgetState extends State<OptimizedAnimationWidget> {
  final Stopwatch _frameStopwatch = Stopwatch();
  String? _monitoringId;
  
  @override
  void initState() {
    super.initState();
    
    if (widget.enablePerformanceMonitoring) {
      _monitoringId = widget.animationId ?? 'animation_${widget.hashCode}';
      AnimationPerformanceMonitor().startMonitoring(_monitoringId!);
      
      widget.animation.addListener(_onAnimationFrame);
      widget.animation.addStatusListener(_onAnimationStatus);
    }
  }
  
  void _onAnimationFrame() {
    if (_monitoringId != null) {
      _frameStopwatch.stop();
      final frameTime = _frameStopwatch.elapsed;
      
      AnimationPerformanceMonitor().recordFrameTime(_monitoringId!, frameTime);
      
      _frameStopwatch.reset();
      _frameStopwatch.start();
    }
  }
  
  void _onAnimationStatus(AnimationStatus status) {
    if (status == AnimationStatus.completed || status == AnimationStatus.dismissed) {
      if (_monitoringId != null) {
        AnimationPerformanceMonitor().stopMonitoring(_monitoringId!);
      }
    }
  }
  
  @override
  Widget build(BuildContext context) {
    if (widget.enablePerformanceMonitoring && !_frameStopwatch.isRunning) {
      _frameStopwatch.start();
    }
    
    return RepaintBoundary(
      child: widget.child,
    );
  }
  
  @override
  void dispose() {
    if (widget.enablePerformanceMonitoring) {
      widget.animation.removeListener(_onAnimationFrame);
      widget.animation.removeStatusListener(_onAnimationStatus);
    }
    
    _frameStopwatch.stop();
    super.dispose();
  }
}
```

### 动画优化最佳实践

```dart
// lib/src/animation/optimization_utils.dart

/// 动画优化工具类
class AnimationOptimizationUtils {
  /// 创建优化的动画控制器
  static AnimationController createOptimizedController({
    required Duration duration,
    required TickerProvider vsync,
    String? debugLabel,
    double lowerBound = 0.0,
    double upperBound = 1.0,
  }) {
    return AnimationController(
      duration: duration,
      vsync: vsync,
      debugLabel: debugLabel,
      lowerBound: lowerBound,
      upperBound: upperBound,
      // 启用动画行为优化
      animationBehavior: AnimationBehavior.preserve,
    );
  }
  
  /// 创建优化的补间动画
  static Animation<T> createOptimizedTween<T>({
    required AnimationController controller,
    required T begin,
    required T end,
    Curve curve = Curves.linear,
  }) {
    final tween = Tween<T>(begin: begin, end: end);
    
    // 使用CurvedAnimation进行曲线优化
    final curvedAnimation = CurvedAnimation(
      parent: controller,
      curve: curve,
    );
    
    return tween.animate(curvedAnimation);
  }
  
  /// 批量动画管理器
  static AnimationController createBatchController({
    required List<Duration> durations,
    required TickerProvider vsync,
  }) {
    // 计算总持续时间
    final totalDuration = durations.fold<Duration>(
      Duration.zero,
      (sum, duration) => sum + duration,
    );
    
    return AnimationController(
      duration: totalDuration,
      vsync: vsync,
    );
  }
  
  /// 创建间隔动画
  static List<Animation<double>> createStaggeredAnimations({
    required AnimationController controller,
    required int count,
    required Duration staggerDelay,
  }) {
    final animations = <Animation<double>>[];
    final totalDuration = controller.duration!.inMilliseconds;
    final delayMs = staggerDelay.inMilliseconds;
    
    for (int i = 0; i < count; i++) {
      final startTime = (delayMs * i) / totalDuration;
      final endTime = math.min(1.0, startTime + 0.6); // 每个动画占60%的时间
      
      final animation = Tween<double>(
        begin: 0.0,
        end: 1.0,
      ).animate(CurvedAnimation(
        parent: controller,
        curve: Interval(startTime, endTime, curve: Curves.easeOut),
      ));
      
      animations.add(animation);
    }
    
    return animations;
  }
  
  /// 动画缓存管理
  static final Map<String, AnimationController> _controllerCache = {};
  
  static AnimationController getCachedController({
    required String key,
    required Duration duration,
    required TickerProvider vsync,
  }) {
    if (_controllerCache.containsKey(key)) {
      final controller = _controllerCache[key]!;
      if (controller.duration == duration) {
        return controller;
      } else {
        // 持续时间不匹配，移除旧的控制器
        controller.dispose();
        _controllerCache.remove(key);
      }
    }
    
    final controller = createOptimizedController(
      duration: duration,
      vsync: vsync,
      debugLabel: 'cached_$key',
    );
    
    _controllerCache[key] = controller;
    return controller;
  }
  
  /// 清理缓存的控制器
  static void clearControllerCache() {
    for (final controller in _controllerCache.values) {
      controller.dispose();
    }
    _controllerCache.clear();
  }
  
  /// 动画预热
  static Future<void> preWarmAnimations(List<AnimationController> controllers) async {
    for (final controller in controllers) {
      // 快速运行一次动画进行预热
      controller.duration = const Duration(milliseconds: 1);
      await controller.forward();
      controller.reset();
      // 恢复原始持续时间需要重新创建控制器
    }
  }
  
  /// 检查动画是否应该运行
  static bool shouldRunAnimation(BuildContext context) {
    // 检查设备性能设置
    final mediaQuery = MediaQuery.of(context);
    if (mediaQuery.disableAnimations) {
      return false;
    }
    
    // 检查电池状态（如果可用）
    // 这里可以集成battery_plus插件
    
    // 检查应用是否在前台
    final binding = WidgetsBinding.instance;
    if (binding.lifecycleState != AppLifecycleState.resumed) {
      return false;
    }
    
    return true;
  }
  
  /// 自适应动画持续时间
  static Duration adaptiveDuration({
    required Duration baseDuration,
    required BuildContext context,
  }) {
    final mediaQuery = MediaQuery.of(context);
    
    // 根据设备性能调整动画持续时间
    double multiplier = 1.0;
    
    // 检查是否启用了减少动画
    if (mediaQuery.disableAnimations) {
      return Duration.zero;
    }
    
    // 根据设备像素密度调整
    final devicePixelRatio = mediaQuery.devicePixelRatio;
    if (devicePixelRatio > 3.0) {
      // 高密度屏幕，可能需要更长的动画时间
      multiplier *= 1.2;
    } else if (devicePixelRatio < 2.0) {
      // 低密度屏幕，可以缩短动画时间
      multiplier *= 0.8;
    }
    
    // 根据屏幕尺寸调整
    final screenSize = mediaQuery.size;
    final screenArea = screenSize.width * screenSize.height;
    if (screenArea > 1000000) { // 大屏设备
      multiplier *= 1.1;
    }
    
    final adjustedDuration = Duration(
      milliseconds: (baseDuration.inMilliseconds * multiplier).round(),
    );
    
    return adjustedDuration;
  }
}

/// 动画性能分析器
class AnimationProfiler {
  static final Map<String, List<Duration>> _animationTimes = {};
  
  /// 开始分析动画
  static void startProfiling(String animationName) {
    _animationTimes[animationName] = [];
  }
  
  /// 记录动画帧时间
  static void recordFrameTime(String animationName, Duration frameTime) {
    _animationTimes[animationName]?.add(frameTime);
  }
  
  /// 生成性能报告
  static Map<String, dynamic> generateReport(String animationName) {
    final times = _animationTimes[animationName];
    if (times == null || times.isEmpty) {
      return {'error': 'No data available for $animationName'};
    }
    
    final totalFrames = times.length;
    final totalTime = times.fold<Duration>(
      Duration.zero,
      (sum, time) => sum + time,
    );
    
    final averageTime = Duration(
      microseconds: totalTime.inMicroseconds ~/ totalFrames,
    );
    
    final maxTime = times.reduce((max, time) => time > max ? time : max);
    final minTime = times.reduce((min, time) => time < min ? time : min);
    
    final droppedFrames = times.where((time) => time.inMilliseconds > 16).length;
    final dropRate = droppedFrames / totalFrames;
    
    return {
      'animationName': animationName,
      'totalFrames': totalFrames,
      'averageFrameTime': averageTime.inMicroseconds,
      'maxFrameTime': maxTime.inMicroseconds,
      'minFrameTime': minTime.inMicroseconds,
      'droppedFrames': droppedFrames,
      'dropRate': dropRate,
      'averageFps': 1000000 / averageTime.inMicroseconds,
    };
  }
  
  /// 清理分析数据
  static void clearProfiling(String animationName) {
    _animationTimes.remove(animationName);
  }
  
  /// 清理所有分析数据
  static void clearAllProfiling() {
    _animationTimes.clear();
  }
}
```

## 总结

Flutter的动画系统为开发者提供了强大而灵活的工具来创建流畅、优雅的用户界面。通过本文的深入探讨，我们了解了：

### 核心要点

1. **动画系统架构**：理解AnimationController、Animation、Tween和Curve的关系和作用
2. **基础动画实现**：掌握常见的属性动画、颜色动画等基础动画类型
3. **复杂动画组合**：学会使用序列动画、并行动画和交错动画创建复杂效果
4. **物理动画**：利用弹簧动画和重力动画创建更自然的动画效果
5. **路径动画**：实现沿自定义路径的动画移动
6. **自定义动画**：创建完全定制的动画效果
7. **性能优化**：监控动画性能并应用最佳实践

### 最佳实践建议

1. **合理使用RepaintBoundary**：避免不必要的重绘
2. **优化动画层级**：减少动画Widget的嵌套深度
3. **使用const构造函数**：提高Widget创建效率
4. **监控动画性能**：及时发现和解决性能问题
5. **适配不同设备**：根据设备性能调整动画参数
6. **缓存动画控制器**：避免重复创建相同的动画
7. **合理使用动画曲线**：选择合适的Curve提升用户体验

### 发展趋势

Flutter动画系统持续发展，未来可能的改进方向包括：

1. **更好的性能优化**：自动检测和优化动画性能
2. **更丰富的预设动画**：提供更多开箱即用的动画效果
3. **更强的物理模拟**：支持更复杂的物理动画
4. **更好的调试工具**：提供更强大的动画调试和分析工具
5. **跨平台一致性**：确保动画在不同平台上的一致表现

通过掌握这些动画技术和最佳实践，开发者可以创建出既美观又高性能的Flutter应用，为用户提供卓越的交互体验。动画不仅仅是视觉效果，更是提升应用品质和用户满意度的重要手段。

class _BasicAnimationWidgetState extends State<BasicAnimationWidget>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;
  
  @override
  void initState() {
    super.initState();
    _initializeAnimation();
  }
  
  void _initializeAnimation() {
    _controller = AnimationController(
      duration: widget.config.duration,
      vsync: this,
    );
    
    _animation = CurvedAnimation(
      parent: _controller,
      curve: widget.config.curve,
    );
    
    _controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onAnimationComplete?.call();
        
        if (widget.config.autoReverse) {
          _controller.reverse();
        }
      } else if (status == AnimationStatus.dismissed && 
                 widget.config.autoReverse) {
        _controller.forward();
      }
    });
    
    // 开始动画
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return _buildAnimatedWidget();
      },
      child: widget.child,
    );
  }
  
  Widget _buildAnimatedWidget() {
    if (widget.config is PropertyAnimationConfig) {
      final config = widget.config as PropertyAnimationConfig;
      final value = Tween<double>(
        begin: config.fromValue,
        end: config.toValue,
      ).evaluate(_animation);
      
      switch (config.property) {
        case AnimationProperty.opacity:
          return Opacity(
            opacity: value,
            child: widget.child,
          );
        case AnimationProperty.scale:
          return Transform.scale(
            scale: value,
            child: widget.child,
          );
        case AnimationProperty.rotation:
          return Transform.rotate(
            angle: value,
            child: widget.child,
          );
        case AnimationProperty.translation:
          return Transform.translate(
            offset: Offset(value, 0),
            child: widget.child,
          );
        default:
          return widget.child;
      }
    }
    
    return widget.child;
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

/// 淡入动画
class FadeInAnimation extends StatelessWidget {
  final Widget child;
  final Duration duration;
  final Curve curve;
  final VoidCallback? onComplete;
  
  const FadeInAnimation({
    Key? key,
    required this.child,
    this.duration = const Duration(milliseconds: 300),
    this.curve = Curves.easeIn,
    this.onComplete,
  }) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return BasicAnimationWidget(
      config: PropertyAnimationConfig(
        fromValue: 0.0,
        toValue: 1.0,
        property: AnimationProperty.opacity,
        duration: duration,
        curve: curve,
      ),
      onAnimationComplete: onComplete,
      child: child,
    );
  }
}

/// 缩放动画
class ScaleAnimation extends StatelessWidget {
  final Widget child;
  final double fromScale;
  final double toScale;
  final Duration duration;
  final Curve curve;
  final VoidCallback? onComplete;
  
  const ScaleAnimation({
    Key? key,
    required this.child,
    this.fromScale = 0.0,
    this.toScale = 1.0,
    this.duration = const Duration(milliseconds: 300),
    this.curve = Curves.elasticOut,
    this.onComplete,
  }) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return BasicAnimationWidget(
      config: PropertyAnimationConfig(
        fromValue: fromScale,
        toValue: toScale,
        property: AnimationProperty.scale,
        duration: duration,
        curve: curve,
      ),
      onAnimationComplete: onComplete,
      child: child,
    );
  }
}

/// 滑动动画
class SlideAnimation extends StatefulWidget {
  final Widget child;
  final Offset fromOffset;
  final Offset toOffset;
  final Duration duration;
  final Curve curve;
  final VoidCallback? onComplete;
  
  const SlideAnimation({
    Key? key,
    required this.child,
    this.fromOffset = const Offset(-1.0, 0.0),
    this.toOffset = Offset.zero,
    this.duration = const Duration(milliseconds: 300),
    this.curve = Curves.easeOut,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<SlideAnimation> createState() => _SlideAnimationState();
}

class _SlideAnimationState extends State<SlideAnimation>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Offset> _offsetAnimation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: widget.duration,
      vsync: this,
    );
    
    _offsetAnimation = Tween<Offset>(
      begin: widget.fromOffset,
      end: widget.toOffset,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: widget.curve,
    ));
    
    _controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onComplete?.call();
      }
    });
    
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return SlideTransition(
      position: _offsetAnimation,
      child: widget.child,
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

/// 旋转动画
class RotationAnimation extends StatefulWidget {
  final Widget child;
  final double fromAngle;
  final double toAngle;
  final Duration duration;
  final Curve curve;
  final bool continuous;
  final VoidCallback? onComplete;
  
  const RotationAnimation({
    Key? key,
    required this.child,
    this.fromAngle = 0.0,
    this.toAngle = 2 * math.pi,
    this.duration = const Duration(seconds: 1),
    this.curve = Curves.linear,
    this.continuous = false,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<RotationAnimation> createState() => _RotationAnimationState();
}

class _RotationAnimationState extends State<RotationAnimation>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _rotationAnimation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: widget.duration,
      vsync: this,
    );
    
    _rotationAnimation = Tween<double>(
      begin: widget.fromAngle,
      end: widget.toAngle,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: widget.curve,
    ));
    
    _controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onComplete?.call();
        
        if (widget.continuous) {
          _controller.reset();
          _controller.forward();
        }
      }
    });
    
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _rotationAnimation,
      builder: (context, child) {
        return Transform.rotate(
          angle: _rotationAnimation.value,
          child: widget.child,
        );
      },
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

### 颜色动画

颜色动画在UI设计中非常常见，用于主题切换、状态指示等：

```dart
// lib/src/animation/color_animations.dart

/// 颜色动画Widget
class ColorAnimation extends StatefulWidget {
  final Widget child;
  final Color fromColor;
  final Color toColor;
  final Duration duration;
  final Curve curve;
  final VoidCallback? onComplete;
  
  const ColorAnimation({
    Key? key,
    required this.child,
    required this.fromColor,
    required this.toColor,
    this.duration = const Duration(milliseconds: 300),
    this.curve = Curves.easeInOut,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<ColorAnimation> createState() => _ColorAnimationState();
}

class _ColorAnimationState extends State<ColorAnimation>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Color?> _colorAnimation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: widget.duration,
      vsync: this,
    );
    
    _colorAnimation = ColorTween(
      begin: widget.fromColor,
      end: widget.toColor,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: widget.curve,
    ));
    
    _controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onComplete?.call();
      }
    });
    
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _colorAnimation,
      builder: (context, child) {
        return ColorFiltered(
          colorFilter: ColorFilter.mode(
            _colorAnimation.value ?? widget.fromColor,
            BlendMode.modulate,
          ),
          child: widget.child,
        );
      },
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

/// 背景颜色动画
class BackgroundColorAnimation extends StatefulWidget {
  final Widget child;
  final Color fromColor;
  final Color toColor;
  final Duration duration;
  final Curve curve;
  
  const BackgroundColorAnimation({
    Key? key,
    required this.child,
    required this.fromColor,
    required this.toColor,
    this.duration = const Duration(milliseconds: 300),
    this.curve = Curves.easeInOut,
  }) : super(key: key);
  
  @override
  State<BackgroundColorAnimation> createState() => _BackgroundColorAnimationState();
}

class _BackgroundColorAnimationState extends State<BackgroundColorAnimation>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Color?> _colorAnimation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: widget.duration,
      vsync: this,
    );
    
    _colorAnimation = ColorTween(
      begin: widget.fromColor,
      end: widget.toColor,
    ).animate(CurvedAnimation(
      parent: _controller,
      curve: widget.curve,
    ));
    
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _colorAnimation,
      builder: (context, child) {
        return Container(
          color: _colorAnimation.value,
          child: widget.child,
        );
      },
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

/// 渐变颜色动画
class GradientAnimation extends StatefulWidget {
  final Widget child;
  final Gradient fromGradient;
  final Gradient toGradient;
  final Duration duration;
  final Curve curve;
  
  const GradientAnimation({
    Key? key,
    required this.child,
    required this.fromGradient,
    required this.toGradient,
    this.duration = const Duration(milliseconds: 500),
    this.curve = Curves.easeInOut,
  }) : super(key: key);
  
  @override
  State<GradientAnimation> createState() => _GradientAnimationState();
}

class _GradientAnimationState extends State<GradientAnimation>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: widget.duration,
      vsync: this,
    );
    
    _animation = CurvedAnimation(
      parent: _controller,
      curve: widget.curve,
    );
    
    _controller.forward();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return Container(
          decoration: BoxDecoration(
            gradient: _interpolateGradient(
              widget.fromGradient,
              widget.toGradient,
              _animation.value,
            ),
          ),
          child: widget.child,
        );
      },
    );
  }
  
  Gradient _interpolateGradient(Gradient from, Gradient to, double t) {
    if (from is LinearGradient && to is LinearGradient) {
      return LinearGradient(
        begin: Alignment.lerp(from.begin, to.begin, t) ?? from.begin,
        end: Alignment.lerp(from.end, to.end, t) ?? from.end,
        colors: _interpolateColors(from.colors, to.colors, t),
        stops: _interpolateStops(from.stops, to.stops, t),
      );
    } else if (from is RadialGradient && to is RadialGradient) {
      return RadialGradient(
        center: Alignment.lerp(from.center, to.center, t) ?? from.center,
        radius: lerpDouble(from.radius, to.radius, t) ?? from.radius,
        colors: _interpolateColors(from.colors, to.colors, t),
        stops: _interpolateStops(from.stops, to.stops, t),
      );
    }
    
    // 默认返回起始渐变
    return from;
  }
  
  List<Color> _interpolateColors(List<Color> from, List<Color> to, double t) {
    final maxLength = math.max(from.length, to.length);
    final result = <Color>[];
    
    for (int i = 0; i < maxLength; i++) {
      final fromColor = i < from.length ? from[i] : from.last;
      final toColor = i < to.length ? to[i] : to.last;
      result.add(Color.lerp(fromColor, toColor, t) ?? fromColor);
    }
    
    return result;
  }
  
  List<double>? _interpolateStops(List<double>? from, List<double>? to, double t) {
    if (from == null || to == null) return null;
    
    final maxLength = math.max(from.length, to.length);
    final result = <double>[];
    
    for (int i = 0; i < maxLength; i++) {
      final fromStop = i < from.length ? from[i] : from.last;
      final toStop = i < to.length ? to[i] : to.last;
      result.add(lerpDouble(fromStop, toStop, t) ?? fromStop);
    }
    
    return result;
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

## 复杂动画组合

### 序列动画

序列动画允许多个动画按顺序执行：

```dart
// lib/src/animation/sequence_animations.dart

/// 序列动画管理器
class SequenceAnimationManager {
  final List<AnimationStep> _steps = [];
  final List<AnimationController> _controllers = [];
  final TickerProvider _vsync;
  
  int _currentStepIndex = 0;
  bool _isPlaying = false;
  VoidCallback? _onComplete;
  
  SequenceAnimationManager({required TickerProvider vsync}) : _vsync = vsync;
  
  /// 添加动画步骤
  void addStep(AnimationStep step) {
    _steps.add(step);
  }
  
  /// 添加多个步骤
  void addSteps(List<AnimationStep> steps) {
    _steps.addAll(steps);
  }
  
  /// 开始播放序列动画
  Future<void> play({VoidCallback? onComplete}) async {
    if (_isPlaying) return;
    
    _onComplete = onComplete;
    _isPlaying = true;
    _currentStepIndex = 0;
    
    await _playNextStep();
  }
  
  Future<void> _playNextStep() async {
    if (_currentStepIndex >= _steps.length) {
      _isPlaying = false;
      _onComplete?.call();
      return;
    }
    
    final step = _steps[_currentStepIndex];
    final controller = AnimationController(
      duration: step.duration,
      vsync: _vsync,
    );
    
    _controllers.add(controller);
    
    // 设置动画监听器
    controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        step.onComplete?.call();
        _currentStepIndex++;
        
        // 延迟后播放下一步
        Future.delayed(step.delay ?? Duration.zero, () {
          _playNextStep();
        });
      }
    });
    
    // 执行动画步骤
    await step.execute(controller);
  }
  
  /// 停止序列动画
  void stop() {
    _isPlaying = false;
    for (final controller in _controllers) {
      controller.stop();
    }
  }
  
  /// 重置序列动画
  void reset() {
    _isPlaying = false;
    _currentStepIndex = 0;
    for (final controller in _controllers) {
      controller.reset();
    }
  }
  
  /// 释放资源
  void dispose() {
    for (final controller in _controllers) {
      controller.dispose();
    }
    _controllers.clear();
    _steps.clear();
  }
}

/// 动画步骤
abstract class AnimationStep {
  final Duration duration;
  final Duration? delay;
  final VoidCallback? onComplete;
  
  const AnimationStep({
    required this.duration,
    this.delay,
    this.onComplete,
  });
  
  /// 执行动画步骤
  Future<void> execute(AnimationController controller);
}

/// 属性动画步骤
class PropertyAnimationStep extends AnimationStep {
  final Widget target;
  final AnimationProperty property;
  final double fromValue;
  final double toValue;
  final Curve curve;
  
  const PropertyAnimationStep({
    required this.target,
    required this.property,
    required this.fromValue,
    required this.toValue,
    this.curve = Curves.easeInOut,
    required Duration duration,
    Duration? delay,
    VoidCallback? onComplete,
  }) : super(
    duration: duration,
    delay: delay,
    onComplete: onComplete,
  );
  
  @override
  Future<void> execute(AnimationController controller) async {
    final animation = Tween<double>(
      begin: fromValue,
      end: toValue,
    ).animate(CurvedAnimation(
      parent: controller,
      curve: curve,
    ));
    
    await controller.forward();
  }
}

/// 延迟步骤
class DelayStep extends AnimationStep {
  const DelayStep({
    required Duration duration,
    VoidCallback? onComplete,
  }) : super(
    duration: duration,
    onComplete: onComplete,
  );
  
  @override
  Future<void> execute(AnimationController controller) async {
    await Future.delayed(duration);
    onComplete?.call();
  }
}

/// 序列动画Widget
class SequenceAnimationWidget extends StatefulWidget {
  final List<AnimationStep> steps;
  final Widget child;
  final bool autoStart;
  final VoidCallback? onComplete;
  
  const SequenceAnimationWidget({
    Key? key,
    required this.steps,
    required this.child,
    this.autoStart = true,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<SequenceAnimationWidget> createState() => _SequenceAnimationWidgetState();
}

class _SequenceAnimationWidgetState extends State<SequenceAnimationWidget>
    with TickerProviderStateMixin {
  late SequenceAnimationManager _manager;
  
  @override
  void initState() {
    super.initState();
    
    _manager = SequenceAnimationManager(vsync: this);
    _manager.addSteps(widget.steps);
    
    if (widget.autoStart) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _manager.play(onComplete: widget.onComplete);
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
  
  @override
  void dispose() {
    _manager.dispose();
    super.dispose();
  }
}
```

### 并行动画

并行动画允许多个动画同时执行：

```dart
// lib/src/animation/parallel_animations.dart

/// 并行动画管理器
class ParallelAnimationManager {
  final List<ParallelAnimationItem> _items = [];
  final TickerProvider _vsync;
  
  bool _isPlaying = false;
  VoidCallback? _onComplete;
  int _completedCount = 0;
  
  ParallelAnimationManager({required TickerProvider vsync}) : _vsync = vsync;
  
  /// 添加并行动画项
  void addItem(ParallelAnimationItem item) {
    _items.add(item);
  }
  
  /// 添加多个并行动画项
  void addItems(List<ParallelAnimationItem> items) {
    _items.addAll(items);
  }
  
  /// 开始播放并行动画
  Future<void> play({VoidCallback? onComplete}) async {
    if (_isPlaying || _items.isEmpty) return;
    
    _onComplete = onComplete;
    _isPlaying = true;
    _completedCount = 0;
    
    // 同时启动所有动画
    for (final item in _items) {
      _playItem(item);
    }
  }
  
  void _playItem(ParallelAnimationItem item) {
    final controller = AnimationController(
      duration: item.duration,
      vsync: _vsync,
    );
    
    controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        item.onComplete?.call();
        _completedCount++;
        
        // 检查是否所有动画都完成
        if (_completedCount >= _items.length) {
          _isPlaying = false;
          _onComplete?.call();
        }
      }
    });
    
    // 延迟启动
    Future.delayed(item.delay ?? Duration.zero, () {
      item.execute(controller);
    });
  }
  
  /// 停止所有并行动画
  void stop() {
    _isPlaying = false;
    // 这里需要保存controller引用来停止动画
  }
  
  /// 重置所有并行动画
  void reset() {
    _isPlaying = false;
    _completedCount = 0;
  }
  
  /// 释放资源
  void dispose() {
    _items.clear();
  }
}

/// 并行动画项
abstract class ParallelAnimationItem {
  final Duration duration;
  final Duration? delay;
  final VoidCallback? onComplete;
  
  const ParallelAnimationItem({
    required this.duration,
    this.delay,
    this.onComplete,
  });
  
  /// 执行动画
  Future<void> execute(AnimationController controller);
}

/// 并行属性动画项
class ParallelPropertyAnimationItem extends ParallelAnimationItem {
  final AnimationProperty property;
  final double fromValue;
  final double toValue;
  final Curve curve;
  
  const ParallelPropertyAnimationItem({
    required this.property,
    required this.fromValue,
    required this.toValue,
    this.curve = Curves.easeInOut,
    required Duration duration,
    Duration? delay,
    VoidCallback? onComplete,
  }) : super(
    duration: duration,
    delay: delay,
    onComplete: onComplete,
  );
  
  @override
  Future<void> execute(AnimationController controller) async {
    final animation = Tween<double>(
      begin: fromValue,
      end: toValue,
    ).animate(CurvedAnimation(
      parent: controller,
      curve: curve,
    ));
    
    await controller.forward();
  }
}

/// 并行动画Widget
class ParallelAnimationWidget extends StatefulWidget {
  final List<ParallelAnimationItem> items;
  final Widget child;
  final bool autoStart;
  final VoidCallback? onComplete;
  
  const ParallelAnimationWidget({
    Key? key,
    required this.items,
    required this.child,
    this.autoStart = true,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<ParallelAnimationWidget> createState() => _ParallelAnimationWidgetState();
}

class _ParallelAnimationWidgetState extends State<ParallelAnimationWidget>
    with TickerProviderStateMixin {
  late ParallelAnimationManager _manager;
  
  @override
  void initState() {
    super.initState();
    
    _manager = ParallelAnimationManager(vsync: this);
    _manager.addItems(widget.items);
    
    if (widget.autoStart) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _manager.play(onComplete: widget.onComplete);
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
  
  @override
  void dispose() {
    _manager.dispose();
    super.dispose();
  }
}
```

### 交错动画

交错动画在列表或网格中特别有用，可以创建波浪式的动画效果：

```dart
// lib/src/animation/staggered_animations.dart

/// 交错动画管理器
class StaggeredAnimationManager {
  final List<Widget> _children;
  final Duration _duration;
  final Duration _staggerDelay;
  final Curve _curve;
  final TickerProvider _vsync;
  
  late AnimationController _controller;
  late List<Animation<double>> _animations;
  
  StaggeredAnimationManager({
    required List<Widget> children,
    required Duration duration,
    required Duration staggerDelay,
    required TickerProvider vsync,
    Curve curve = Curves.easeOut,
  }) : _children = children,
       _duration = duration,
       _staggerDelay = staggerDelay,
       _curve = curve,
       _vsync = vsync {
    _initializeAnimations();
  }
  
  void _initializeAnimations() {
    _controller = AnimationController(
      duration: _duration,
      vsync: _vsync,
    );
    
    _animations = [];
    
    for (int i = 0; i < _children.length; i++) {
      final startTime = (_staggerDelay.inMilliseconds * i) / _duration.inMilliseconds;
      final endTime = math.min(1.0, startTime + 0.5); // 每个动画占总时长的50%
      
      final animation = Tween<double>(
        begin: 0.0,
        end: 1.0,
      ).animate(CurvedAnimation(
        parent: _controller,
        curve: Interval(startTime, endTime, curve: _curve),
      ));
      
      _animations.add(animation);
    }
  }
  
  /// 开始交错动画
  Future<void> forward() async {
    await _controller.forward();
  }
  
  /// 反向播放交错动画
  Future<void> reverse() async {
    await _controller.reverse();
  }
  
  /// 重置动画
  void reset() {
    _controller.reset();
  }
  
  /// 获取指定索引的动画
  Animation<double> getAnimation(int index) {
    return _animations[index];
  }
  
  /// 获取控制器
  AnimationController get controller => _controller;
  
  /// 释放资源
  void dispose() {
    _controller.dispose();
  }
}

/// 交错动画Widget
class StaggeredAnimationWidget extends StatefulWidget {
  final List<Widget> children;
  final Duration duration;
  final Duration staggerDelay;
  final Curve curve;
  final StaggeredAnimationType animationType;
  final bool autoStart;
  final VoidCallback? onComplete;
  
  const StaggeredAnimationWidget({
    Key? key,
    required this.children,
    this.duration = const Duration(milliseconds: 600),
    this.staggerDelay = const Duration(milliseconds: 100),
    this.curve = Curves.easeOut,
    this.animationType = StaggeredAnimationType.fadeInUp,
    this.autoStart = true,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<StaggeredAnimationWidget> createState() => _StaggeredAnimationWidgetState();
}

class _StaggeredAnimationWidgetState extends State<StaggeredAnimationWidget>
    with TickerProviderStateMixin {
  late StaggeredAnimationManager _manager;
  
  @override
  void initState() {
    super.initState();
    
    _manager = StaggeredAnimationManager(
      children: widget.children,
      duration: widget.duration,
      staggerDelay: widget.staggerDelay,
      curve: widget.curve,
      vsync: this,
    );
    
    _manager.controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onComplete?.call();
      }
    });
    
    if (widget.autoStart) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _manager.forward();
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: widget.children.asMap().entries.map((entry) {
        final index = entry.key;
        final child = entry.value;
        final animation = _manager.getAnimation(index);
        
        return AnimatedBuilder(
          animation: animation,
          builder: (context, _) {
            return _buildAnimatedChild(child, animation.value);
          },
        );
      }).toList(),
    );
  }
  
  Widget _buildAnimatedChild(Widget child, double animationValue) {
    switch (widget.animationType) {
      case StaggeredAnimationType.fadeIn:
        return Opacity(
          opacity: animationValue,
          child: child,
        );
      
      case StaggeredAnimationType.fadeInUp:
        return Transform.translate(
          offset: Offset(0, 50 * (1 - animationValue)),
          child: Opacity(
            opacity: animationValue,
            child: child,
          ),
        );
      
      case StaggeredAnimationType.fadeInDown:
        return Transform.translate(
          offset: Offset(0, -50 * (1 - animationValue)),
          child: Opacity(
            opacity: animationValue,
            child: child,
          ),
        );
      
      case StaggeredAnimationType.fadeInLeft:
        return Transform.translate(
          offset: Offset(-50 * (1 - animationValue), 0),
          child: Opacity(
            opacity: animationValue,
            child: child,
          ),
        );
      
      case StaggeredAnimationType.fadeInRight:
        return Transform.translate(
          offset: Offset(50 * (1 - animationValue), 0),
          child: Opacity(
            opacity: animationValue,
            child: child,
          ),
        );
      
      case StaggeredAnimationType.scaleIn:
        return Transform.scale(
          scale: animationValue,
          child: child,
        );
      
      case StaggeredAnimationType.slideInUp:
        return ClipRect(
          child: Transform.translate(
            offset: Offset(0, 100 * (1 - animationValue)),
            child: child,
          ),
        );
      
      default:
        return child;
    }
  }
  
  @override
  void dispose() {
    _manager.dispose();
    super.dispose();
  }
}

/// 交错动画类型
enum StaggeredAnimationType {
  fadeIn,
  fadeInUp,
  fadeInDown,
  fadeInLeft,
  fadeInRight,
  scaleIn,
  slideInUp,
}

/// 交错列表动画
class StaggeredListAnimation extends StatefulWidget {
  final List<Widget> children;
  final Duration duration;
  final Duration staggerDelay;
  final StaggeredAnimationType animationType;
  final ScrollController? scrollController;
  
  const StaggeredListAnimation({
    Key? key,
    required this.children,
    this.duration = const Duration(milliseconds: 600),
    this.staggerDelay = const Duration(milliseconds: 100),
    this.animationType = StaggeredAnimationType.fadeInUp,
    this.scrollController,
  }) : super(key: key);
  
  @override
  State<StaggeredListAnimation> createState() => _StaggeredListAnimationState();
}

class _StaggeredListAnimationState extends State<StaggeredListAnimation> {
  final Set<int> _visibleItems = {};
  
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      controller: widget.scrollController,
      itemCount: widget.children.length,
      itemBuilder: (context, index) {
        return VisibilityDetector(
          key: Key('staggered_item_$index'),
          onVisibilityChanged: (info) {
            if (info.visibleFraction > 0.1 && !_visibleItems.contains(index)) {
              setState(() {
                _visibleItems.add(index);
              });
            }
          },
          child: _visibleItems.contains(index)
              ? StaggeredItemAnimation(
                  delay: Duration(milliseconds: widget.staggerDelay.inMilliseconds * (index % 3)),
                  duration: widget.duration,
                  animationType: widget.animationType,
                  child: widget.children[index],
                )
              : Opacity(
                  opacity: 0,
                  child: widget.children[index],
                ),
        );
      },
    );
  }
}

/// 单个交错动画项
class StaggeredItemAnimation extends StatefulWidget {
  final Widget child;
  final Duration delay;
  final Duration duration;
  final StaggeredAnimationType animationType;
  
  const StaggeredItemAnimation({
    Key? key,
    required this.child,
    required this.delay,
    required this.duration,
    required this.animationType,
  }) : super(key: key);
  
  @override
  State<StaggeredItemAnimation> createState() => _StaggeredItemAnimationState();
}

class _StaggeredItemAnimationState extends State<StaggeredItemAnimation>
    with TickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: widget.duration,
      vsync: this,
    );
    
    _animation = CurvedAnimation(
      parent: _controller,
      curve: Curves.easeOut,
    );
    
    // 延迟启动动画
    Future.delayed(widget.delay, () {
      if (mounted) {
        _controller.forward();
      }
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return _buildAnimatedWidget();
      },
    );
  }
  
  Widget _buildAnimatedWidget() {
    final value = _animation.value;
    
    switch (widget.animationType) {
      case StaggeredAnimationType.fadeIn:
        return Opacity(
          opacity: value,
          child: widget.child,
        );
      
      case StaggeredAnimationType.fadeInUp:
        return Transform.translate(
          offset: Offset(0, 30 * (1 - value)),
          child: Opacity(
            opacity: value,
            child: widget.child,
          ),
        );
      
      case StaggeredAnimationType.scaleIn:
        return Transform.scale(
          scale: 0.8 + (0.2 * value),
          child: Opacity(
            opacity: value,
            child: widget.child,
          ),
        );
      
      default:
        return widget.child;
    }
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

## 物理动画与弹簧效果

### 弹簧动画系统

物理动画能够创建更自然、更有趣的动画效果：

```dart
// lib/src/animation/physics_animations.dart

/// 弹簧动画控制器
class SpringAnimationController {
  final TickerProvider _vsync;
  late AnimationController _controller;
  late SpringSimulation _simulation;
  
  Animation<double>? _animation;
  VoidCallback? _onComplete;
  
  SpringAnimationController({required TickerProvider vsync}) : _vsync = vsync;
  
  /// 创建弹簧动画
  Animation<double> createSpringAnimation({
    required double from,
    required double to,
    required SpringDescription spring,
    double velocity = 0.0,
  }) {
    // 计算动画持续时间
    final duration = _calculateDuration(spring, from, to, velocity);
    
    _controller = AnimationController(
      duration: duration,
      vsync: _vsync,
    );
    
    // 创建弹簧模拟
    _simulation = SpringSimulation(
      SpringDescription(
        mass: spring.mass,
        stiffness: spring.stiffness,
        damping: spring.damping,
      ),
      from,
      to,
      velocity,
    );
    
    // 创建动画
    _animation = _controller.drive(
      Tween<double>(begin: from, end: to),
    );
    
    return _animation!;
  }
  
  Duration _calculateDuration(
    SpringDescription spring,
    double from,
    double to,
    double velocity,
  ) {
    // 基于弹簧参数计算合适的持续时间
    final distance = (to - from).abs();
    final dampingRatio = spring.damping / (2 * math.sqrt(spring.mass * spring.stiffness));
    
    double duration;
    if (dampingRatio < 1.0) {
      // 欠阻尼
      final naturalFreq = math.sqrt(spring.stiffness / spring.mass);
      final dampedFreq = naturalFreq * math.sqrt(1 - dampingRatio * dampingRatio);
      duration = (4 * math.pi) / dampedFreq;
    } else {
      // 过阻尼或临界阻尼
      duration = 4 / math.sqrt(spring.stiffness / spring.mass);
    }
    
    return Duration(milliseconds: (duration * 1000).round());
  }
  
  /// 开始弹簧动画
  Future<void> forward({VoidCallback? onComplete}) async {
    _onComplete = onComplete;
    
    _controller.addStatusListener(_onStatusChanged);
    await _controller.forward();
  }
  
  void _onStatusChanged(AnimationStatus status) {
    if (status == AnimationStatus.completed) {
      _onComplete?.call();
      _controller.removeStatusListener(_onStatusChanged);
    }
  }
  
  /// 停止动画
  void stop() {
    _controller.stop();
  }
  
  /// 重置动画
  void reset() {
    _controller.reset();
  }
  
  /// 释放资源
  void dispose() {
    _controller.dispose();
  }
}

/// 弹簧动画Widget
class SpringAnimationWidget extends StatefulWidget {
  final Widget child;
  final SpringDescription spring;
  final double fromValue;
  final double toValue;
  final SpringAnimationProperty property;
  final bool autoStart;
  final VoidCallback? onComplete;
  
  const SpringAnimationWidget({
    Key? key,
    required this.child,
    required this.spring,
    required this.fromValue,
    required this.toValue,
    required this.property,
    this.autoStart = true,
    this.onComplete,
  }) : super(key: key);
  
  @override
  State<SpringAnimationWidget> createState() => _SpringAnimationWidgetState();
}

class _SpringAnimationWidgetState extends State<SpringAnimationWidget>
    with TickerProviderStateMixin {
  late SpringAnimationController _springController;
  late Animation<double> _animation;
  
  @override
  void initState() {
    super.initState();
    
    _springController = SpringAnimationController(vsync: this);
    _animation = _springController.createSpringAnimation(
      from: widget.fromValue,
      to: widget.toValue,
      spring: widget.spring,
    );
    
    if (widget.autoStart) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _springController.forward(onComplete: widget.onComplete);
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return _buildAnimatedWidget();
      },
    );
  }
  
  Widget _buildAnimatedWidget() {
    final value = _animation.value;
    
    switch (widget.property) {
      case SpringAnimationProperty.scale:
        return Transform.scale(
          scale: value,
          child: widget.child,
        );
      
      case SpringAnimationProperty.opacity:
        return Opacity(
          opacity: value.clamp(0.0, 1.0),
          child: widget.child,
        );
      
      case SpringAnimationProperty.rotation:
        return Transform.rotate(
          angle: value,
          child: widget.child,
        );
      
      case SpringAnimationProperty.translation:
        return Transform.translate(
          offset: Offset(value, 0),
          child: widget.child,
        );
      
      default:
        return widget.child;
    }
  }
  
  @override
  void dispose() {
    _springController.dispose();
    super.dispose();
  }
}

/// 弹簧动画属性
enum SpringAnimationProperty {
  scale,
  opacity,
  rotation,
  translation,
}

/// 弹性按钮
class SpringButton extends StatefulWidget {
  final Widget child;
  final VoidCallback? onTap;
  final SpringDescription spring;
  final double scaleDown;
  
  const SpringButton({
    Key? key,
    required this.child,
    this.onTap,
    this.spring = SpringDescription.medium,
    this.scaleDown = 0.95,
  }) : super(key: key);
  
  @override
  State