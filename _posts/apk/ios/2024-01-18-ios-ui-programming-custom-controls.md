---
layout: post
title: "iOS UI编程深度解析：自定义控件开发与高级界面技术"
date: 2024-01-18
categories: ios
tags: [iOS, UIKit, 自定义控件, 界面开发, Core Animation]
author: "iOS开发工程师"
description: "深入探讨iOS UI编程的高级技术，包括自定义控件开发、Core Animation应用、界面性能优化等核心内容，帮助开发者构建流畅美观的用户界面。"
---

iOS应用的用户界面是用户体验的核心，优秀的UI设计和流畅的交互能够显著提升应用的用户满意度。本文将深入探讨iOS UI编程的高级技术，包括自定义控件开发、Core Animation应用、界面性能优化等关键内容。

## UIKit框架核心架构

### UIView层次结构与渲染机制

**1. UIView基础架构**

```objc
@interface CustomView : UIView
@property (nonatomic, strong) CALayer *customLayer;
@property (nonatomic, assign) CGRect contentFrame;
@property (nonatomic, strong) UIColor *borderColor;
@property (nonatomic, assign) CGFloat borderWidth;

- (void)setupView;
- (void)updateLayout;
- (void)performCustomDrawing;
@end

@implementation CustomView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupView];
    }
    return self;
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super initWithCoder:coder];
    if (self) {
        [self setupView];
    }
    return self;
}

- (void)setupView {
    // 1. 设置基本属性
    self.backgroundColor = [UIColor clearColor];
    self.clipsToBounds = YES;
    
    // 2. 创建自定义图层
    _customLayer = [CALayer layer];
    _customLayer.backgroundColor = [UIColor lightGrayColor].CGColor;
    _customLayer.cornerRadius = 8.0;
    _customLayer.masksToBounds = YES;
    [self.layer addSublayer:_customLayer];
    
    // 3. 设置默认值
    _borderColor = [UIColor blackColor];
    _borderWidth = 1.0;
    
    // 4. 添加观察者
    [self addObserver:self forKeyPath:@"bounds" options:NSKeyValueObservingOptionNew context:nil];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey,id> *)change
                       context:(void *)context {
    if ([keyPath isEqualToString:@"bounds"]) {
        [self updateLayout];
    }
}

- (void)updateLayout {
    // 更新自定义图层的frame
    CGRect bounds = self.bounds;
    _customLayer.frame = CGRectInset(bounds, _borderWidth, _borderWidth);
    
    // 更新内容区域
    _contentFrame = CGRectInset(bounds, _borderWidth + 10, _borderWidth + 10);
    
    // 触发重绘
    [self setNeedsDisplay];
}

- (void)drawRect:(CGRect)rect {
    [super drawRect:rect];
    [self performCustomDrawing];
}

- (void)performCustomDrawing {
    CGContextRef context = UIGraphicsGetCurrentContext();
    if (!context) return;
    
    // 1. 绘制边框
    CGContextSetStrokeColorWithColor(context, _borderColor.CGColor);
    CGContextSetLineWidth(context, _borderWidth);
    CGContextStrokeRect(context, CGRectInset(self.bounds, _borderWidth/2, _borderWidth/2));
    
    // 2. 绘制自定义内容
    CGContextSetFillColorWithColor(context, [UIColor blueColor].CGColor);
    CGFloat centerX = CGRectGetMidX(_contentFrame);
    CGFloat centerY = CGRectGetMidY(_contentFrame);
    CGFloat radius = MIN(CGRectGetWidth(_contentFrame), CGRectGetHeight(_contentFrame)) / 4;
    
    CGContextFillEllipseInRect(context, CGRectMake(centerX - radius, centerY - radius, radius * 2, radius * 2));
}

- (void)setBorderColor:(UIColor *)borderColor {
    _borderColor = borderColor;
    [self setNeedsDisplay];
}

- (void)setBorderWidth:(CGFloat)borderWidth {
    _borderWidth = borderWidth;
    [self updateLayout];
}

- (void)dealloc {
    [self removeObserver:self forKeyPath:@"bounds"];
}

@end
```

## Core Animation深度应用

### 图层动画系统

**1. 基础动画管理器**

```objc
@interface AnimationManager : NSObject
@property (nonatomic, weak) CALayer *targetLayer;

- (instancetype)initWithLayer:(CALayer *)layer;
- (void)performBasicAnimation:(NSString *)keyPath
                    fromValue:(id)fromValue
                      toValue:(id)toValue
                     duration:(NSTimeInterval)duration;
- (void)performKeyframeAnimation:(NSString *)keyPath
                          values:(NSArray *)values
                        keyTimes:(NSArray *)keyTimes
                        duration:(NSTimeInterval)duration;
- (void)performSpringAnimation:(NSString *)keyPath
                     fromValue:(id)fromValue
                       toValue:(id)toValue
                      duration:(NSTimeInterval)duration
                       damping:(CGFloat)damping
                     stiffness:(CGFloat)stiffness;
- (void)performGroupAnimation:(NSArray<CAAnimation *> *)animations
                     duration:(NSTimeInterval)duration;
@end

@implementation AnimationManager

- (instancetype)initWithLayer:(CALayer *)layer {
    self = [super init];
    if (self) {
        _targetLayer = layer;
    }
    return self;
}

- (void)performBasicAnimation:(NSString *)keyPath
                    fromValue:(id)fromValue
                      toValue:(id)toValue
                     duration:(NSTimeInterval)duration {
    
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:keyPath];
    animation.fromValue = fromValue;
    animation.toValue = toValue;
    animation.duration = duration;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    
    // 设置动画完成回调
    animation.delegate = self;
    
    [_targetLayer addAnimation:animation forKey:keyPath];
    
    // 更新模型层的值
    [_targetLayer setValue:toValue forKeyPath:keyPath];
}

- (void)performKeyframeAnimation:(NSString *)keyPath
                          values:(NSArray *)values
                        keyTimes:(NSArray *)keyTimes
                        duration:(NSTimeInterval)duration {
    
    CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:keyPath];
    animation.values = values;
    animation.keyTimes = keyTimes;
    animation.duration = duration;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    
    [_targetLayer addAnimation:animation forKey:[NSString stringWithFormat:@"%@_keyframe", keyPath]];
    
    // 更新到最终值
    if (values.count > 0) {
        [_targetLayer setValue:values.lastObject forKeyPath:keyPath];
    }
}

- (void)performSpringAnimation:(NSString *)keyPath
                     fromValue:(id)fromValue
                       toValue:(id)toValue
                      duration:(NSTimeInterval)duration
                       damping:(CGFloat)damping
                     stiffness:(CGFloat)stiffness {
    
    CASpringAnimation *animation = [CASpringAnimation animationWithKeyPath:keyPath];
    animation.fromValue = fromValue;
    animation.toValue = toValue;
    animation.duration = duration;
    animation.damping = damping;
    animation.stiffness = stiffness;
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    
    [_targetLayer addAnimation:animation forKey:[NSString stringWithFormat:@"%@_spring", keyPath]];
    
    // 更新模型层的值
    [_targetLayer setValue:toValue forKeyPath:keyPath];
}

- (void)performGroupAnimation:(NSArray<CAAnimation *> *)animations
                     duration:(NSTimeInterval)duration {
    
    CAAnimationGroup *group = [CAAnimationGroup animation];
    group.animations = animations;
    group.duration = duration;
    group.fillMode = kCAFillModeForwards;
    group.removedOnCompletion = NO;
    
    [_targetLayer addAnimation:group forKey:@"groupAnimation"];
}

#pragma mark - CAAnimationDelegate

- (void)animationDidStart:(CAAnimation *)anim {
    NSLog(@"Animation started");
}

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag {
    NSLog(@"Animation finished: %@", flag ? @"YES" : @"NO");
}

@end
```

**2. 复杂动画序列**

```objc
@interface AnimationSequence : NSObject
@property (nonatomic, strong) NSMutableArray<NSDictionary *> *animationSteps;
@property (nonatomic, weak) CALayer *targetLayer;
@property (nonatomic, assign) NSInteger currentStepIndex;
@property (nonatomic, copy) void(^completionBlock)(void);

- (instancetype)initWithLayer:(CALayer *)layer;
- (void)addAnimationStep:(NSString *)keyPath
               fromValue:(id)fromValue
                 toValue:(id)toValue
                duration:(NSTimeInterval)duration
                   delay:(NSTimeInterval)delay;
- (void)addCustomAnimationStep:(CAAnimation *)animation
                         delay:(NSTimeInterval)delay;
- (void)executeSequenceWithCompletion:(void(^)(void))completion;
- (void)pauseSequence;
- (void)resumeSequence;
- (void)stopSequence;
@end

@implementation AnimationSequence

- (instancetype)initWithLayer:(CALayer *)layer {
    self = [super init];
    if (self) {
        _targetLayer = layer;
        _animationSteps = [NSMutableArray array];
        _currentStepIndex = 0;
    }
    return self;
}

- (void)addAnimationStep:(NSString *)keyPath
               fromValue:(id)fromValue
                 toValue:(id)toValue
                duration:(NSTimeInterval)duration
                   delay:(NSTimeInterval)delay {
    
    NSDictionary *step = @{
        @"type": @"basic",
        @"keyPath": keyPath,
        @"fromValue": fromValue ?: [NSNull null],
        @"toValue": toValue,
        @"duration": @(duration),
        @"delay": @(delay)
    };
    
    [_animationSteps addObject:step];
}

- (void)addCustomAnimationStep:(CAAnimation *)animation
                         delay:(NSTimeInterval)delay {
    
    NSDictionary *step = @{
        @"type": @"custom",
        @"animation": animation,
        @"delay": @(delay)
    };
    
    [_animationSteps addObject:step];
}

- (void)executeSequenceWithCompletion:(void(^)(void))completion {
    _completionBlock = completion;
    _currentStepIndex = 0;
    [self executeNextStep];
}

- (void)executeNextStep {
    if (_currentStepIndex >= _animationSteps.count) {
        // 序列完成
        if (_completionBlock) {
            _completionBlock();
        }
        return;
    }
    
    NSDictionary *step = _animationSteps[_currentStepIndex];
    NSTimeInterval delay = [step[@"delay"] doubleValue];
    
    // 延迟执行
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self performAnimationStep:step];
    });
}

- (void)performAnimationStep:(NSDictionary *)step {
    NSString *type = step[@"type"];
    
    if ([type isEqualToString:@"basic"]) {
        [self performBasicAnimationStep:step];
    } else if ([type isEqualToString:@"custom"]) {
        [self performCustomAnimationStep:step];
    }
}

- (void)performBasicAnimationStep:(NSDictionary *)step {
    NSString *keyPath = step[@"keyPath"];
    id fromValue = step[@"fromValue"];
    id toValue = step[@"toValue"];
    NSTimeInterval duration = [step[@"duration"] doubleValue];
    
    if ([fromValue isKindOfClass:[NSNull class]]) {
        fromValue = [_targetLayer valueForKeyPath:keyPath];
    }
    
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:keyPath];
    animation.fromValue = fromValue;
    animation.toValue = toValue;
    animation.duration = duration;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    animation.delegate = self;
    
    [_targetLayer addAnimation:animation forKey:[NSString stringWithFormat:@"step_%ld", (long)_currentStepIndex]];
    
    // 更新模型层的值
    [_targetLayer setValue:toValue forKeyPath:keyPath];
}

- (void)performCustomAnimationStep:(NSDictionary *)step {
    CAAnimation *animation = step[@"animation"];
    animation.delegate = self;
    
    [_targetLayer addAnimation:animation forKey:[NSString stringWithFormat:@"custom_step_%ld", (long)_currentStepIndex]];
}

- (void)pauseSequence {
    CFTimeInterval pausedTime = [_targetLayer convertTime:CACurrentMediaTime() fromLayer:nil];
    _targetLayer.speed = 0.0;
    _targetLayer.timeOffset = pausedTime;
}

- (void)resumeSequence {
    CFTimeInterval pausedTime = _targetLayer.timeOffset;
    _targetLayer.speed = 1.0;
    _targetLayer.timeOffset = 0.0;
    _targetLayer.beginTime = 0.0;
    CFTimeInterval timeSincePause = [_targetLayer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    _targetLayer.beginTime = timeSincePause;
}

- (void)stopSequence {
    [_targetLayer removeAllAnimations];
    _currentStepIndex = 0;
}

#pragma mark - CAAnimationDelegate

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag {
    if (flag) {
        _currentStepIndex++;
        [self executeNextStep];
    }
}

@end
```

### 高级视觉效果

**1. 粒子系统**

```objc
@interface ParticleEmitterView : UIView
@property (nonatomic, strong) CAEmitterLayer *emitterLayer;
@property (nonatomic, strong) CAEmitterCell *emitterCell;

// 发射器属性
@property (nonatomic, assign) CGPoint emitterPosition;
@property (nonatomic, assign) CGSize emitterSize;
@property (nonatomic, assign) NSString *emitterShape;
@property (nonatomic, assign) NSString *emitterMode;

// 粒子属性
@property (nonatomic, strong) UIImage *particleImage;
@property (nonatomic, assign) CGFloat birthRate;
@property (nonatomic, assign) CGFloat lifetime;
@property (nonatomic, assign) CGFloat velocity;
@property (nonatomic, assign) CGFloat velocityRange;
@property (nonatomic, assign) CGFloat emissionRange;
@property (nonatomic, assign) CGFloat scale;
@property (nonatomic, assign) CGFloat scaleRange;
@property (nonatomic, assign) CGFloat alphaSpeed;

- (void)startEmitting;
- (void)stopEmitting;
- (void)createFireworksEffect;
- (void)createSnowEffect;
- (void)createRainEffect;
@end

@implementation ParticleEmitterView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupEmitter];
    }
    return self;
}

- (void)setupEmitter {
    // 1. 创建发射器图层
    _emitterLayer = [CAEmitterLayer layer];
    _emitterLayer.frame = self.bounds;
    [self.layer addSublayer:_emitterLayer];
    
    // 2. 设置默认发射器属性
    _emitterPosition = CGPointMake(CGRectGetMidX(self.bounds), CGRectGetMidY(self.bounds));
    _emitterSize = CGSizeMake(50, 50);
    _emitterShape = kCAEmitterLayerPoint;
    _emitterMode = kCAEmitterLayerOutline;
    
    // 3. 创建粒子单元
    _emitterCell = [CAEmitterCell emitterCell];
    
    // 4. 设置默认粒子属性
    _birthRate = 50;
    _lifetime = 3.0;
    _velocity = 100;
    _velocityRange = 50;
    _emissionRange = M_PI * 2;
    _scale = 1.0;
    _scaleRange = 0.5;
    _alphaSpeed = -0.5;
    
    [self updateEmitterProperties];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    _emitterLayer.frame = self.bounds;
    [self updateEmitterProperties];
}

- (void)updateEmitterProperties {
    // 更新发射器属性
    _emitterLayer.emitterPosition = _emitterPosition;
    _emitterLayer.emitterSize = _emitterSize;
    _emitterLayer.emitterShape = _emitterShape;
    _emitterLayer.emitterMode = _emitterMode;
    
    // 更新粒子属性
    _emitterCell.birthRate = _birthRate;
    _emitterCell.lifetime = _lifetime;
    _emitterCell.velocity = _velocity;
    _emitterCell.velocityRange = _velocityRange;
    _emitterCell.emissionRange = _emissionRange;
    _emitterCell.scale = _scale;
    _emitterCell.scaleRange = _scaleRange;
    _emitterCell.alphaSpeed = _alphaSpeed;
    
    if (_particleImage) {
        _emitterCell.contents = (id)_particleImage.CGImage;
    }
    
    _emitterLayer.emitterCells = @[_emitterCell];
}

- (void)startEmitting {
    _emitterLayer.birthRate = 1.0;
}

- (void)stopEmitting {
    _emitterLayer.birthRate = 0.0;
}

- (void)createFireworksEffect {
    // 烟花效果配置
    _emitterPosition = CGPointMake(CGRectGetMidX(self.bounds), CGRectGetMaxY(self.bounds));
    _emitterShape = kCAEmitterLayerPoint;
    _emitterMode = kCAEmitterLayerOutline;
    
    _emitterCell.birthRate = 100;
    _emitterCell.lifetime = 2.0;
    _emitterCell.velocity = 200;
    _emitterCell.velocityRange = 100;
    _emitterCell.emissionRange = M_PI / 4;
    _emitterCell.yAcceleration = 150; // 重力效果
    _emitterCell.scale = 0.1;
    _emitterCell.scaleRange = 0.05;
    _emitterCell.alphaSpeed = -1.0;
    
    // 设置颜色变化
    _emitterCell.color = [UIColor redColor].CGColor;
    _emitterCell.redRange = 1.0;
    _emitterCell.greenRange = 1.0;
    _emitterCell.blueRange = 1.0;
    
    [self updateEmitterProperties];
    [self startEmitting];
}

- (void)createSnowEffect {
    // 雪花效果配置
    _emitterPosition = CGPointMake(CGRectGetMidX(self.bounds), 0);
    _emitterSize = CGSizeMake(CGRectGetWidth(self.bounds), 0);
    _emitterShape = kCAEmitterLayerLine;
    _emitterMode = kCAEmitterLayerSurface;
    
    _emitterCell.birthRate = 20;
    _emitterCell.lifetime = 10.0;
    _emitterCell.velocity = 50;
    _emitterCell.velocityRange = 20;
    _emitterCell.emissionRange = M_PI / 8;
    _emitterCell.yAcceleration = 30;
    _emitterCell.scale = 0.05;
    _emitterCell.scaleRange = 0.02;
    _emitterCell.alphaSpeed = 0;
    
    // 设置旋转
    _emitterCell.spin = M_PI / 2;
    _emitterCell.spinRange = M_PI;
    
    _emitterCell.color = [UIColor whiteColor].CGColor;
    
    [self updateEmitterProperties];
    [self startEmitting];
}

- (void)createRainEffect {
    // 雨滴效果配置
    _emitterPosition = CGPointMake(CGRectGetMidX(self.bounds), 0);
    _emitterSize = CGSizeMake(CGRectGetWidth(self.bounds), 0);
    _emitterShape = kCAEmitterLayerLine;
    _emitterMode = kCAEmitterLayerSurface;
    
    _emitterCell.birthRate = 100;
    _emitterCell.lifetime = 3.0;
    _emitterCell.velocity = 300;
    _emitterCell.velocityRange = 50;
    _emitterCell.emissionRange = M_PI / 16;
    _emitterCell.yAcceleration = 100;
    _emitterCell.scale = 0.02;
    _emitterCell.scaleRange = 0.01;
    _emitterCell.alphaSpeed = 0;
    
    _emitterCell.color = [UIColor blueColor].CGColor;
    
    [self updateEmitterProperties];
    [self startEmitting];
}

// 属性设置器
- (void)setEmitterPosition:(CGPoint)emitterPosition {
    _emitterPosition = emitterPosition;
    [self updateEmitterProperties];
}

- (void)setParticleImage:(UIImage *)particleImage {
    _particleImage = particleImage;
    [self updateEmitterProperties];
}

- (void)setBirthRate:(CGFloat)birthRate {
    _birthRate = birthRate;
    [self updateEmitterProperties];
}

@end
```

## 界面性能优化

### 渲染性能监控

```objc
@interface UIPerformanceMonitor : NSObject
@property (nonatomic, strong) CADisplayLink *displayLink;
@property (nonatomic, assign) CFTimeInterval lastTimestamp;
@property (nonatomic, assign) NSInteger frameCount;
@property (nonatomic, assign) CGFloat currentFPS;
@property (nonatomic, strong) NSMutableArray<NSNumber *> *fpsHistory;
@property (nonatomic, copy) void(^fpsUpdateBlock)(CGFloat fps);

+ (instancetype)sharedMonitor;
- (void)startMonitoring;
- (void)stopMonitoring;
- (void)setFPSUpdateBlock:(void(^)(CGFloat fps))updateBlock;
- (CGFloat)averageFPS;
- (CGFloat)minimumFPS;
- (void)logPerformanceReport;
@end

@implementation UIPerformanceMonitor

+ (instancetype)sharedMonitor {
    static UIPerformanceMonitor *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[UIPerformanceMonitor alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _fpsHistory = [NSMutableArray array];
        _frameCount = 0;
        _currentFPS = 0;
    }
    return self;
}

- (void)startMonitoring {
    if (_displayLink) {
        [self stopMonitoring];
    }
    
    _displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkTick:)];
    _displayLink.preferredFramesPerSecond = 60;
    [_displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
    
    _lastTimestamp = 0;
    _frameCount = 0;
}

- (void)stopMonitoring {
    [_displayLink invalidate];
    _displayLink = nil;
}

- (void)displayLinkTick:(CADisplayLink *)displayLink {
    if (_lastTimestamp == 0) {
        _lastTimestamp = displayLink.timestamp;
        return;
    }
    
    _frameCount++;
    
    CFTimeInterval deltaTime = displayLink.timestamp - _lastTimestamp;
    
    if (deltaTime >= 1.0) {
        // 计算FPS
        _currentFPS = _frameCount / deltaTime;
        
        // 记录FPS历史
        [_fpsHistory addObject:@(_currentFPS)];
        if (_fpsHistory.count > 60) { // 保持最近60秒的数据
            [_fpsHistory removeObjectAtIndex:0];
        }
        
        // 回调更新
        if (_fpsUpdateBlock) {
            _fpsUpdateBlock(_currentFPS);
        }
        
        // 重置计数器
        _frameCount = 0;
        _lastTimestamp = displayLink.timestamp;
    }
}

- (void)setFPSUpdateBlock:(void(^)(CGFloat fps))updateBlock {
    _fpsUpdateBlock = updateBlock;
}

- (CGFloat)averageFPS {
    if (_fpsHistory.count == 0) return 0;
    
    CGFloat total = 0;
    for (NSNumber *fps in _fpsHistory) {
        total += fps.floatValue;
    }
    return total / _fpsHistory.count;
}

- (CGFloat)minimumFPS {
    if (_fpsHistory.count == 0) return 0;
    
    CGFloat minimum = CGFLOAT_MAX;
    for (NSNumber *fps in _fpsHistory) {
        minimum = MIN(minimum, fps.floatValue);
    }
    return minimum;
}

- (void)logPerformanceReport {
    NSLog(@"=== UI Performance Report ===");
    NSLog(@"Current FPS: %.1f", _currentFPS);
    NSLog(@"Average FPS: %.1f", [self averageFPS]);
    NSLog(@"Minimum FPS: %.1f", [self minimumFPS]);
    NSLog(@"Sample Count: %lu", (unsigned long)_fpsHistory.count);
    NSLog(@"==============================");
}

@end
```

### 视图优化策略

```objc
@interface ViewOptimizer : NSObject

+ (void)optimizeView:(UIView *)view;
+ (void)optimizeScrollView:(UIScrollView *)scrollView;
+ (void)optimizeTableView:(UITableView *)tableView;
+ (void)optimizeImageView:(UIImageView *)imageView;
+ (void)enableAsyncDrawing:(UIView *)view;
+ (void)optimizeLayerProperties:(CALayer *)layer;
@end

@implementation ViewOptimizer

+ (void)optimizeView:(UIView *)view {
    if (!view) return;
    
    // 1. 设置不透明属性
    if (view.backgroundColor && ![view.backgroundColor isEqual:[UIColor clearColor]]) {
        view.opaque = YES;
    }
    
    // 2. 优化图层属性
    [self optimizeLayerProperties:view.layer];
    
    // 3. 递归优化子视图
    for (UIView *subview in view.subviews) {
        [self optimizeView:subview];
    }
}

+ (void)optimizeScrollView:(UIScrollView *)scrollView {
    if (!scrollView) return;
    
    // 1. 基础优化
    [self optimizeView:scrollView];
    
    // 2. 滚动视图特定优化
    scrollView.delaysContentTouches = NO;
    scrollView.canCancelContentTouches = YES;
    
    // 3. 如果是分页滚动，优化预加载
    if (scrollView.pagingEnabled) {
        scrollView.decelerationRate = UIScrollViewDecelerationRateFast;
    }
}

+ (void)optimizeTableView:(UITableView *)tableView {
    if (!tableView) return;
    
    // 1. 基础优化
    [self optimizeScrollView:tableView];
    
    // 2. 表格视图特定优化
    tableView.estimatedRowHeight = 44; // 设置估算行高
    tableView.rowHeight = UITableViewAutomaticDimension;
    
    // 3. 分隔线优化
    if (tableView.separatorStyle != UITableViewCellSeparatorStyleNone) {
        tableView.separatorInset = UIEdgeInsetsZero;
        tableView.layoutMargins = UIEdgeInsetsZero;
    }
}

+ (void)optimizeImageView:(UIImageView *)imageView {
    if (!imageView) return;
    
    // 1. 基础优化
    [self optimizeView:imageView];
    
    // 2. 图片视图特定优化
    if (imageView.image) {
        // 检查图片尺寸是否合适
        CGSize imageSize = imageView.image.size;
        CGSize viewSize = imageView.bounds.size;
        
        if (imageSize.width > viewSize.width * 2 || imageSize.height > viewSize.height * 2) {
            NSLog(@"Warning: Image size (%@) is much larger than view size (%@)", 
                  NSStringFromCGSize(imageSize), NSStringFromCGSize(viewSize));
        }
        
        // 设置内容模式
        if (imageView.contentMode == UIViewContentModeScaleToFill) {
            imageView.contentMode = UIViewContentModeScaleAspectFit;
        }
    }
}

+ (void)enableAsyncDrawing:(UIView *)view {
    if (!view) return;
    
    // 启用异步绘制
    view.layer.drawsAsynchronously = YES;
    
    // 对于复杂视图，启用光栅化
    if (view.subviews.count > 5 || view.layer.shadowOpacity > 0) {
        view.layer.shouldRasterize = YES;
        view.layer.rasterizationScale = [UIScreen mainScreen].scale;
    }
}

+ (void)optimizeLayerProperties:(CALayer *)layer {
    if (!layer) return;
    
    // 1. 优化边界设置
    if (layer.cornerRadius > 0) {
        layer.masksToBounds = YES;
    }
    
    // 2. 优化阴影设置
    if (layer.shadowOpacity > 0) {
        // 使用阴影路径提高性能
        if (!layer.shadowPath) {
            layer.shadowPath = [UIBezierPath bezierPathWithRoundedRect:layer.bounds 
                                                          cornerRadius:layer.cornerRadius].CGPath;
        }
        
        // 启用光栅化
        layer.shouldRasterize = YES;
        layer.rasterizationScale = [UIScreen mainScreen].scale;
    }
    
    // 3. 优化边框设置
    if (layer.borderWidth > 0) {
        // 确保边框颜色已设置
        if (!layer.borderColor) {
            layer.borderColor = [UIColor blackColor].CGColor;
        }
    }
    
    // 4. 递归优化子图层
    for (CALayer *sublayer in layer.sublayers) {
        [self optimizeLayerProperties:sublayer];
    }
}

@end
```

## 响应式布局与适配

### 自适应布局管理器

```objc
@interface AdaptiveLayoutManager : NSObject
@property (nonatomic, weak) UIView *containerView;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSArray<NSLayoutConstraint *> *> *constraintGroups;

- (instancetype)initWithContainerView:(UIView *)containerView;
- (void)addConstraintGroup:(NSString *)groupName constraints:(NSArray<NSLayoutConstraint *> *)constraints;
- (void)activateConstraintGroup:(NSString *)groupName;
- (void)deactivateConstraintGroup:(NSString *)groupName;
- (void)updateLayoutForTraitCollection:(UITraitCollection *)traitCollection;
- (void)setupResponsiveLayout;
@end

@implementation AdaptiveLayoutManager

- (instancetype)initWithContainerView:(UIView *)containerView {
    self = [super init];
    if (self) {
        _containerView = containerView;
        _constraintGroups = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)addConstraintGroup:(NSString *)groupName constraints:(NSArray<NSLayoutConstraint *> *)constraints {
    _constraintGroups[groupName] = constraints;
}

- (void)activateConstraintGroup:(NSString *)groupName {
    NSArray<NSLayoutConstraint *> *constraints = _constraintGroups[groupName];
    if (constraints) {
        [NSLayoutConstraint activateConstraints:constraints];
    }
}

- (void)deactivateConstraintGroup:(NSString *)groupName {
    NSArray<NSLayoutConstraint *> *constraints = _constraintGroups[groupName];
    if (constraints) {
        [NSLayoutConstraint deactivateConstraints:constraints];
    }
}

- (void)updateLayoutForTraitCollection:(UITraitCollection *)traitCollection {
    // 先停用所有约束组
    for (NSString *groupName in _constraintGroups.allKeys) {
        [self deactivateConstraintGroup:groupName];
    }
    
    // 根据特征集合激活相应的约束组
    if (traitCollection.horizontalSizeClass == UIUserInterfaceSizeClassCompact) {
        // 紧凑宽度（iPhone竖屏）
        [self activateConstraintGroup:@"compact"];
    } else if (traitCollection.horizontalSizeClass == UIUserInterfaceSizeClassRegular) {
        // 常规宽度（iPad或iPhone横屏）
        [self activateConstraintGroup:@"regular"];
    }
    
    if (traitCollection.verticalSizeClass == UIUserInterfaceSizeClassCompact) {
        // 紧凑高度（iPhone横屏）
        [self activateConstraintGroup:@"compactHeight"];
    } else if (traitCollection.verticalSizeClass == UIUserInterfaceSizeClassRegular) {
        // 常规高度（iPhone竖屏或iPad）
        [self activateConstraintGroup:@"regularHeight"];
    }
    
    // 动画更新布局
    [UIView animateWithDuration:0.3 animations:^{
        [self.containerView layoutIfNeeded];
    }];
}

- (void)setupResponsiveLayout {
    // 这个方法应该在具体的视图控制器中实现
    // 这里提供一个示例
    
    UIView *contentView = [[UIView alloc] init];
    contentView.backgroundColor = [UIColor lightGrayColor];
    contentView.translatesAutoresizingMaskIntoConstraints = NO;
    [_containerView addSubview:contentView];
    
    // 紧凑宽度约束（iPhone竖屏）
    NSArray *compactConstraints = @[
        [contentView.leadingAnchor constraintEqualToAnchor:_containerView.leadingAnchor constant:16],
        [contentView.trailingAnchor constraintEqualToAnchor:_containerView.trailingAnchor constant:-16],
        [contentView.topAnchor constraintEqualToAnchor:_containerView.safeAreaLayoutGuide.topAnchor constant:20],
        [contentView.heightAnchor constraintEqualToConstant:200]
    ];
    
    // 常规宽度约束（iPad或iPhone横屏）
    NSArray *regularConstraints = @[
        [contentView.centerXAnchor constraintEqualToAnchor:_containerView.centerXAnchor],
        [contentView.widthAnchor constraintEqualToConstant:400],
        [contentView.topAnchor constraintEqualToAnchor:_containerView.safeAreaLayoutGuide.topAnchor constant:40],
        [contentView.heightAnchor constraintEqualToConstant:300]
    ];
    
    [self addConstraintGroup:@"compact" constraints:compactConstraints];
    [self addConstraintGroup:@"regular" constraints:regularConstraints];
}

@end
```

## UI编程最佳实践

### 设计原则

1. **单一职责原则**：每个自定义控件只负责一个特定的功能
2. **可复用性**：设计通用的、可配置的控件组件
3. **性能优先**：优化渲染性能，避免不必要的重绘
4. **响应式设计**：支持不同屏幕尺寸和方向
5. **可访问性**：确保控件支持VoiceOver等辅助功能

### 开发建议

1. **合理使用Auto Layout**：避免约束冲突，优化约束性能
2. **图层优化**：合理设置图层属性，避免离屏渲染
3. **内存管理**：及时释放不需要的视图和图层
4. **动画性能**：使用Core Animation进行高性能动画
5. **测试验证**：在不同设备和系统版本上测试界面效果

### 调试技巧

1. **视图调试器**：使用Xcode的视图调试器检查视图层次
2. **性能监控**：监控FPS和渲染性能
3. **约束调试**：使用约束调试工具解决布局问题
4. **内存分析**：使用Instruments分析内存使用
5. **动画调试**：使用Core Animation工具调试动画性能

## 总结

iOS UI编程是一个复杂而精细的领域，需要深入理解UIKit框架、Core Animation、性能优化等多个方面。通过掌握自定义控件开发、高级动画技术、性能优化策略和响应式布局，开发者可以创建出优秀的用户界面。

随着iOS系统的不断发展，新的UI技术和设计理念也在不断涌现。开发者应该持续学习新技术，关注用户体验，并在实践中不断优化和改进自己的UI编程技能。良好的UI设计不仅能提升应用的视觉效果，更能显著改善用户的使用体验。

**2. 视图层次管理**

```objc
@interface ViewHierarchyManager : NSObject
@property (nonatomic, weak) UIView *containerView;

- (instancetype)initWithContainerView:(UIView *)containerView;
- (void)addSubview:(UIView *)subview withConstraints:(NSDictionary *)constraints;
- (void)removeSubview:(UIView *)subview;
- (void)bringSubviewToFront:(UIView *)subview;
- (void)sendSubviewToBack:(UIView *)subview;
- (void)optimizeViewHierarchy;
@end

@implementation ViewHierarchyManager

- (instancetype)initWithContainerView:(UIView *)containerView {
    self = [super init];
    if (self) {
        _containerView = containerView;
    }
    return self;
}

- (void)addSubview:(UIView *)subview withConstraints:(NSDictionary *)constraints {
    if (!subview || !self.containerView) return;
    
    // 1. 添加子视图
    [self.containerView addSubview:subview];
    subview.translatesAutoresizingMaskIntoConstraints = NO;
    
    // 2. 应用约束
    NSMutableArray *constraintArray = [NSMutableArray array];
    
    // 处理水平约束
    if (constraints[@"leading"]) {
        NSLayoutConstraint *leading = [NSLayoutConstraint constraintWithItem:subview
                                                                   attribute:NSLayoutAttributeLeading
                                                                   relatedBy:NSLayoutRelationEqual
                                                                      toItem:self.containerView
                                                                   attribute:NSLayoutAttributeLeading
                                                                  multiplier:1.0
                                                                    constant:[constraints[@"leading"] floatValue]];
        [constraintArray addObject:leading];
    }
    
    if (constraints[@"trailing"]) {
        NSLayoutConstraint *trailing = [NSLayoutConstraint constraintWithItem:subview
                                                                    attribute:NSLayoutAttributeTrailing
                                                                    relatedBy:NSLayoutRelationEqual
                                                                       toItem:self.containerView
                                                                    attribute:NSLayoutAttributeTrailing
                                                                   multiplier:1.0
                                                                     constant:-[constraints[@"trailing"] floatValue]];
        [constraintArray addObject:trailing];
    }
    
    // 处理垂直约束
    if (constraints[@"top"]) {
        NSLayoutConstraint *top = [NSLayoutConstraint constraintWithItem:subview
                                                               attribute:NSLayoutAttributeTop
                                                               relatedBy:NSLayoutRelationEqual
                                                                  toItem:self.containerView
                                                               attribute:NSLayoutAttributeTop
                                                              multiplier:1.0
                                                                constant:[constraints[@"top"] floatValue]];
        [constraintArray addObject:top];
    }
    
    if (constraints[@"bottom"]) {
        NSLayoutConstraint *bottom = [NSLayoutConstraint constraintWithItem:subview
                                                                  attribute:NSLayoutAttributeBottom
                                                                  relatedBy:NSLayoutRelationEqual
                                                                     toItem:self.containerView
                                                                  attribute:NSLayoutAttributeBottom
                                                                 multiplier:1.0
                                                                   constant:-[constraints[@"bottom"] floatValue]];
        [constraintArray addObject:bottom];
    }
    
    // 处理尺寸约束
    if (constraints[@"width"]) {
        NSLayoutConstraint *width = [NSLayoutConstraint constraintWithItem:subview
                                                                 attribute:NSLayoutAttributeWidth
                                                                 relatedBy:NSLayoutRelationEqual
                                                                    toItem:nil
                                                                 attribute:NSLayoutAttributeNotAnAttribute
                                                                multiplier:1.0
                                                                  constant:[constraints[@"width"] floatValue]];
        [constraintArray addObject:width];
    }
    
    if (constraints[@"height"]) {
        NSLayoutConstraint *height = [NSLayoutConstraint constraintWithItem:subview
                                                                  attribute:NSLayoutAttributeHeight
                                                                  relatedBy:NSLayoutRelationEqual
                                                                     toItem:nil
                                                                  attribute:NSLayoutAttributeNotAnAttribute
                                                                 multiplier:1.0
                                                                   constant:[constraints[@"height"] floatValue]];
        [constraintArray addObject:height];
    }
    
    // 3. 激活约束
    [NSLayoutConstraint activateConstraints:constraintArray];
    
    // 4. 设置视图标识
    subview.tag = self.containerView.subviews.count;
}

- (void)removeSubview:(UIView *)subview {
    if (!subview) return;
    
    // 1. 移除约束
    [subview removeFromSuperview];
    
    // 2. 重新排列其他子视图的tag
    [self reorderSubviewTags];
}

- (void)reorderSubviewTags {
    NSArray *subviews = self.containerView.subviews;
    for (NSInteger i = 0; i < subviews.count; i++) {
        UIView *subview = subviews[i];
        subview.tag = i + 1;
    }
}

- (void)bringSubviewToFront:(UIView *)subview {
    if (!subview || subview.superview != self.containerView) return;
    
    [self.containerView bringSubviewToFront:subview];
    [self reorderSubviewTags];
}

- (void)sendSubviewToBack:(UIView *)subview {
    if (!subview || subview.superview != self.containerView) return;
    
    [self.containerView sendSubviewToBack:subview];
    [self reorderSubviewTags];
}

- (void)optimizeViewHierarchy {
    // 1. 移除隐藏的视图
    NSMutableArray *viewsToRemove = [NSMutableArray array];
    for (UIView *subview in self.containerView.subviews) {
        if (subview.hidden && subview.alpha == 0) {
            [viewsToRemove addObject:subview];
        }
    }
    
    for (UIView *view in viewsToRemove) {
        [self removeSubview:view];
    }
    
    // 2. 合并相似的视图
    [self mergeSimilarViews];
    
    // 3. 优化图层设置
    [self optimizeLayerSettings];
}

- (void)mergeSimilarViews {
    // 查找可以合并的相似视图
    NSMutableArray *similarGroups = [NSMutableArray array];
    
    for (UIView *subview in self.containerView.subviews) {
        if ([subview isKindOfClass:[UILabel class]]) {
            UILabel *label = (UILabel *)subview;
            
            // 查找相同样式的标签
            NSMutableArray *similarLabels = [NSMutableArray arrayWithObject:label];
            
            for (UIView *otherSubview in self.containerView.subviews) {
                if (otherSubview != subview && [otherSubview isKindOfClass:[UILabel class]]) {
                    UILabel *otherLabel = (UILabel *)otherSubview;
                    
                    if ([self areLabelsStyleSimilar:label other:otherLabel]) {
                        [similarLabels addObject:otherLabel];
                    }
                }
            }
            
            if (similarLabels.count > 1) {
                [similarGroups addObject:similarLabels];
            }
        }
    }
    
    // 处理相似视图组
    for (NSArray *group in similarGroups) {
        [self processSimilarViewGroup:group];
    }
}

- (BOOL)areLabelsStyleSimilar:(UILabel *)label1 other:(UILabel *)label2 {
    return [label1.font isEqual:label2.font] &&
           [label1.textColor isEqual:label2.textColor] &&
           label1.textAlignment == label2.textAlignment;
}

- (void)processSimilarViewGroup:(NSArray *)group {
    // 这里可以实现具体的合并逻辑
    // 例如：创建一个容器视图来包含相似的视图
    NSLog(@"Found %lu similar views that could be optimized", (unsigned long)group.count);
}

- (void)optimizeLayerSettings {
    for (UIView *subview in self.containerView.subviews) {
        // 1. 优化不透明设置
        if (subview.backgroundColor && ![subview.backgroundColor isEqual:[UIColor clearColor]]) {
            subview.opaque = YES;
        }
        
        // 2. 优化光栅化设置
        if (subview.layer.shadowOpacity > 0 || subview.layer.cornerRadius > 0) {
            subview.layer.shouldRasterize = YES;
            subview.layer.rasterizationScale = [UIScreen mainScreen].scale;
        }
        
        // 3. 优化边界设置
        if (subview.layer.cornerRadius > 0) {
            subview.layer.masksToBounds = YES;
        }
    }
}

@end
```

## 自定义控件开发

### 高级按钮控件

```objc
@interface AdvancedButton : UIControl
@property (nonatomic, strong) UILabel *titleLabel;
@property (nonatomic, strong) UIImageView *iconImageView;
@property (nonatomic, strong) CAGradientLayer *gradientLayer;
@property (nonatomic, strong) CAShapeLayer *borderLayer;

// 外观属性
@property (nonatomic, strong) UIColor *normalBackgroundColor;
@property (nonatomic, strong) UIColor *highlightedBackgroundColor;
@property (nonatomic, strong) UIColor *disabledBackgroundColor;
@property (nonatomic, assign) CGFloat cornerRadius;
@property (nonatomic, assign) CGFloat borderWidth;
@property (nonatomic, strong) UIColor *borderColor;

// 渐变属性
@property (nonatomic, strong) NSArray<UIColor *> *gradientColors;
@property (nonatomic, assign) CGPoint gradientStartPoint;
@property (nonatomic, assign) CGPoint gradientEndPoint;

// 图标属性
@property (nonatomic, strong) UIImage *iconImage;
@property (nonatomic, assign) CGFloat iconSize;
@property (nonatomic, assign) CGFloat iconTitleSpacing;

// 动画属性
@property (nonatomic, assign) NSTimeInterval animationDuration;
@property (nonatomic, assign) BOOL enablePressAnimation;
@property (nonatomic, assign) CGFloat pressAnimationScale;

- (void)setTitle:(NSString *)title forState:(UIControlState)state;
- (void)setTitleColor:(UIColor *)color forState:(UIControlState)state;
- (void)setBackgroundColor:(UIColor *)backgroundColor forState:(UIControlState)state;
- (void)updateAppearance;
@end

@implementation AdvancedButton

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupButton];
    }
    return self;
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super initWithCoder:coder];
    if (self) {
        [self setupButton];
    }
    return self;
}

- (void)setupButton {
    // 1. 设置默认属性
    _cornerRadius = 8.0;
    _borderWidth = 1.0;
    _borderColor = [UIColor clearColor];
    _iconSize = 24.0;
    _iconTitleSpacing = 8.0;
    _animationDuration = 0.15;
    _enablePressAnimation = YES;
    _pressAnimationScale = 0.95;
    
    // 2. 设置默认颜色
    _normalBackgroundColor = [UIColor systemBlueColor];
    _highlightedBackgroundColor = [UIColor systemBlueColor];
    _disabledBackgroundColor = [UIColor systemGrayColor];
    
    // 3. 创建渐变图层
    _gradientLayer = [CAGradientLayer layer];
    _gradientLayer.cornerRadius = _cornerRadius;
    _gradientLayer.masksToBounds = YES;
    [self.layer insertSublayer:_gradientLayer atIndex:0];
    
    // 4. 创建边框图层
    _borderLayer = [CAShapeLayer layer];
    _borderLayer.fillColor = [UIColor clearColor].CGColor;
    _borderLayer.strokeColor = _borderColor.CGColor;
    _borderLayer.lineWidth = _borderWidth;
    [self.layer addSublayer:_borderLayer];
    
    // 5. 创建标题标签
    _titleLabel = [[UILabel alloc] init];
    _titleLabel.textAlignment = NSTextAlignmentCenter;
    _titleLabel.font = [UIFont systemFontOfSize:16 weight:UIFontWeightMedium];
    _titleLabel.textColor = [UIColor whiteColor];
    _titleLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self addSubview:_titleLabel];
    
    // 6. 创建图标视图
    _iconImageView = [[UIImageView alloc] init];
    _iconImageView.contentMode = UIViewContentModeScaleAspectFit;
    _iconImageView.translatesAutoresizingMaskIntoConstraints = NO;
    [self addSubview:_iconImageView];
    
    // 7. 设置约束
    [self setupConstraints];
    
    // 8. 添加触摸事件
    [self addTarget:self action:@selector(touchDown:) forControlEvents:UIControlEventTouchDown];
    [self addTarget:self action:@selector(touchUp:) forControlEvents:UIControlEventTouchUpInside];
    [self addTarget:self action:@selector(touchUp:) forControlEvents:UIControlEventTouchUpOutside];
    [self addTarget:self action:@selector(touchUp:) forControlEvents:UIControlEventTouchCancel];
}

- (void)setupConstraints {
    // 图标约束
    [NSLayoutConstraint activateConstraints:@[
        [_iconImageView.leadingAnchor constraintEqualToAnchor:self.leadingAnchor constant:16],
        [_iconImageView.centerYAnchor constraintEqualToAnchor:self.centerYAnchor],
        [_iconImageView.widthAnchor constraintEqualToConstant:_iconSize],
        [_iconImageView.heightAnchor constraintEqualToConstant:_iconSize]
    ]];
    
    // 标题约束
    [NSLayoutConstraint activateConstraints:@[
        [_titleLabel.leadingAnchor constraintEqualToAnchor:_iconImageView.trailingAnchor constant:_iconTitleSpacing],
        [_titleLabel.trailingAnchor constraintEqualToAnchor:self.trailingAnchor constant:-16],
        [_titleLabel.centerYAnchor constraintEqualToAnchor:self.centerYAnchor]
    ]];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    
    // 更新渐变图层frame
    _gradientLayer.frame = self.bounds;
    
    // 更新边框图层路径
    UIBezierPath *borderPath = [UIBezierPath bezierPathWithRoundedRect:self.bounds cornerRadius:_cornerRadius];
    _borderLayer.path = borderPath.CGPath;
    
    // 更新外观
    [self updateAppearance];
}

- (void)updateAppearance {
    // 1. 更新渐变
    if (_gradientColors && _gradientColors.count > 0) {
        NSMutableArray *cgColors = [NSMutableArray array];
        for (UIColor *color in _gradientColors) {
            [cgColors addObject:(id)color.CGColor];
        }
        _gradientLayer.colors = cgColors;
        _gradientLayer.startPoint = _gradientStartPoint;
        _gradientLayer.endPoint = _gradientEndPoint;
        _gradientLayer.hidden = NO;
    } else {
        _gradientLayer.hidden = YES;
        // 使用纯色背景
        UIColor *backgroundColor;
        if (!self.enabled) {
            backgroundColor = _disabledBackgroundColor;
        } else if (self.highlighted) {
            backgroundColor = _highlightedBackgroundColor;
        } else {
            backgroundColor = _normalBackgroundColor;
        }
        self.backgroundColor = backgroundColor;
    }
    
    // 2. 更新圆角
    _gradientLayer.cornerRadius = _cornerRadius;
    self.layer.cornerRadius = _cornerRadius;
    
    // 3. 更新边框
    _borderLayer.strokeColor = _borderColor.CGColor;
    _borderLayer.lineWidth = _borderWidth;
    
    // 4. 更新图标显示
    _iconImageView.hidden = (_iconImage == nil);
    if (_iconImage) {
        _iconImageView.image = _iconImage;
    }
}

- (void)setTitle:(NSString *)title forState:(UIControlState)state {
    if (state == UIControlStateNormal) {
        _titleLabel.text = title;
    }
}

- (void)setTitleColor:(UIColor *)color forState:(UIControlState)state {
    if (state == UIControlStateNormal) {
        _titleLabel.textColor = color;
    }
}

- (void)setBackgroundColor:(UIColor *)backgroundColor forState:(UIControlState)state {
    switch (state) {
        case UIControlStateNormal:
            _normalBackgroundColor = backgroundColor;
            break;
        case UIControlStateHighlighted:
            _highlightedBackgroundColor = backgroundColor;
            break;
        case UIControlStateDisabled:
            _disabledBackgroundColor = backgroundColor;
            break;
        default:
            break;
    }
    [self updateAppearance];
}

- (void)setGradientColors:(NSArray<UIColor *> *)gradientColors {
    _gradientColors = gradientColors;
    [self updateAppearance];
}

- (void)setCornerRadius:(CGFloat)cornerRadius {
    _cornerRadius = cornerRadius;
    [self updateAppearance];
}

- (void)setBorderWidth:(CGFloat)borderWidth {
    _borderWidth = borderWidth;
    [self updateAppearance];
}

- (void)setBorderColor:(UIColor *)borderColor {
    _borderColor = borderColor;
    [self updateAppearance];
}

- (void)setIconImage:(UIImage *)iconImage {
    _iconImage = iconImage;
    [self updateAppearance];
}

- (void)touchDown:(AdvancedButton *)sender {
    if (_enablePressAnimation) {
        [UIView animateWithDuration:_animationDuration animations:^{
            self.transform = CGAffineTransformMakeScale(self.pressAnimationScale, self.pressAnimationScale);
        }];
    }
}

- (void)touchUp:(AdvancedButton *)sender {
    if (_enablePressAnimation) {
        [UIView animateWithDuration:_animationDuration animations:^{
            self.transform = CGAffineTransformIdentity;
        }];
    }
}

@end
```

### 自定义进度指示器

```objc
@interface CircularProgressView : UIView
@property (nonatomic, assign) CGFloat progress; // 0.0 - 1.0
@property (nonatomic, strong) UIColor *progressColor;
@property (nonatomic, strong) UIColor *trackColor;
@property (nonatomic, assign) CGFloat lineWidth;
@property (nonatomic, assign) BOOL showPercentageLabel;
@property (nonatomic, strong) UILabel *percentageLabel;
@property (nonatomic, strong) CAShapeLayer *trackLayer;
@property (nonatomic, strong) CAShapeLayer *progressLayer;

- (void)setProgress:(CGFloat)progress animated:(BOOL)animated;
- (void)setProgress:(CGFloat)progress animated:(BOOL)animated duration:(NSTimeInterval)duration;
@end

@implementation CircularProgressView

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setupProgressView];
    }
    return self;
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super initWithCoder:coder];
    if (self) {
        [self setupProgressView];
    }
    return self;
}

- (void)setupProgressView {
    // 1. 设置默认属性
    _progress = 0.0;
    _progressColor = [UIColor systemBlueColor];
    _trackColor = [UIColor systemGray5Color];
    _lineWidth = 8.0;
    _showPercentageLabel = YES;
    
    self.backgroundColor = [UIColor clearColor];
    
    // 2. 创建轨道图层
    _trackLayer = [CAShapeLayer layer];
    _trackLayer.fillColor = [UIColor clearColor].CGColor;
    _trackLayer.strokeColor = _trackColor.CGColor;
    _trackLayer.lineWidth = _lineWidth;
    _trackLayer.lineCap = kCALineCapRound;
    [self.layer addSublayer:_trackLayer];
    
    // 3. 创建进度图层
    _progressLayer = [CAShapeLayer layer];
    _progressLayer.fillColor = [UIColor clearColor].CGColor;
    _progressLayer.strokeColor = _progressColor.CGColor;
    _progressLayer.lineWidth = _lineWidth;
    _progressLayer.lineCap = kCALineCapRound;
    _progressLayer.strokeEnd = 0.0;
    [self.layer addSublayer:_progressLayer];
    
    // 4. 创建百分比标签
    _percentageLabel = [[UILabel alloc] init];
    _percentageLabel.textAlignment = NSTextAlignmentCenter;
    _percentageLabel.font = [UIFont systemFontOfSize:16 weight:UIFontWeightMedium];
    _percentageLabel.textColor = [UIColor labelColor];
    _percentageLabel.text = @"0%";
    _percentageLabel.translatesAutoresizingMaskIntoConstraints = NO;
    [self addSubview:_percentageLabel];
    
    // 5. 设置约束
    [NSLayoutConstraint activateConstraints:@[
        [_percentageLabel.centerXAnchor constraintEqualToAnchor:self.centerXAnchor],
        [_percentageLabel.centerYAnchor constraintEqualToAnchor:self.centerYAnchor]
    ]];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    [self updatePaths];
}

- (void)updatePaths {
    CGPoint center = CGPointMake(CGRectGetMidX(self.bounds), CGRectGetMidY(self.bounds));
    CGFloat radius = (MIN(CGRectGetWidth(self.bounds), CGRectGetHeight(self.bounds)) - _lineWidth) / 2;
    
    // 创建圆形路径（从顶部开始，顺时针）
    UIBezierPath *path = [UIBezierPath bezierPathWithArcCenter:center
                                                        radius:radius
                                                    startAngle:-M_PI_2
                                                      endAngle:3 * M_PI_2
                                                     clockwise:YES];
    
    _trackLayer.path = path.CGPath;
    _progressLayer.path = path.CGPath;
}

- (void)setProgress:(CGFloat)progress {
    [self setProgress:progress animated:NO];
}

- (void)setProgress:(CGFloat)progress animated:(BOOL)animated {
    [self setProgress:progress animated:animated duration:0.3];
}

- (void)setProgress:(CGFloat)progress animated:(BOOL)animated duration:(NSTimeInterval)duration {
    // 限制进度值范围
    progress = MAX(0.0, MIN(1.0, progress));
    
    CGFloat oldProgress = _progress;
    _progress = progress;
    
    if (animated) {
        // 创建进度动画
        CABasicAnimation *progressAnimation = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
        progressAnimation.fromValue = @(oldProgress);
        progressAnimation.toValue = @(progress);
        progressAnimation.duration = duration;
        progressAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
        
        [_progressLayer addAnimation:progressAnimation forKey:@"progressAnimation"];
        
        // 创建百分比标签动画
        if (_showPercentageLabel) {
            CADisplayLink *displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(updatePercentageLabel:)];
            displayLink.preferredFramesPerSecond = 60;
            [displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
            
            // 存储动画信息
            objc_setAssociatedObject(displayLink, "startProgress", @(oldProgress), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
            objc_setAssociatedObject(displayLink, "endProgress", @(progress), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
            objc_setAssociatedObject(displayLink, "startTime", @(CACurrentMediaTime()), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
            objc_setAssociatedObject(displayLink, "duration", @(duration), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        }
    } else {
        _progressLayer.strokeEnd = progress;
        [self updatePercentageLabelWithProgress:progress];
    }
}

- (void)updatePercentageLabel:(CADisplayLink *)displayLink {
    NSNumber *startProgressNum = objc_getAssociatedObject(displayLink, "startProgress");
    NSNumber *endProgressNum = objc_getAssociatedObject(displayLink, "endProgress");
    NSNumber *startTimeNum = objc_getAssociatedObject(displayLink, "startTime");
    NSNumber *durationNum = objc_getAssociatedObject(displayLink, "duration");
    
    CGFloat startProgress = startProgressNum.floatValue;
    CGFloat endProgress = endProgressNum.floatValue;
    CFTimeInterval startTime = startTimeNum.doubleValue;
    NSTimeInterval duration = durationNum.doubleValue;
    
    CFTimeInterval currentTime = CACurrentMediaTime();
    CGFloat elapsed = (currentTime - startTime) / duration;
    
    if (elapsed >= 1.0) {
        // 动画完成
        [self updatePercentageLabelWithProgress:endProgress];
        [displayLink invalidate];
    } else {
        // 计算当前进度
        CGFloat currentProgress = startProgress + (endProgress - startProgress) * elapsed;
        [self updatePercentageLabelWithProgress:currentProgress];
    }
}

- (void)updatePercentageLabelWithProgress:(CGFloat)progress {
    if (_showPercentageLabel) {
        NSInteger percentage = (NSInteger)(progress * 100);
        _percentageLabel.text = [NSString stringWithFormat:@"%ld%%", (long)percentage];
    }
}

- (void)setProgressColor:(UIColor *)progressColor {
    _progressColor = progressColor;
    _progressLayer.strokeColor = progressColor.CGColor;
}

- (void)setTrackColor:(UIColor *)trackColor {
    _trackColor = trackColor;
    _trackLayer.strokeColor = trackColor.CGColor;
}

- (void)setLineWidth:(CGFloat)lineWidth {
    _lineWidth = lineWidth;
    _trackLayer.lineWidth = lineWidth;
    _progressLayer.lineWidth = lineWidth;
    [self updatePaths];
}

- (void)setShowPercentageLabel:(BOOL)showPercentageLabel {
    _showPercentageLabel = showPercentageLabel;
    _percentageLabel.hidden = !showPercentageLabel;
}

@end
```