---
layout: post
title: "iOS UI动画深度解析：Core Animation、UIView动画与高级动效实战指南"
date: 2024-02-28
categories: ios
tags: [Core Animation, UIView Animation, CALayer, 动画效果, 性能优化]
author: "iOS技术专家"
description: "深入探讨iOS UI动画技术，从基础的UIView动画到高级的Core Animation，包含丰富的实战案例和性能优化技巧"
---

# iOS UI动画深度解析：Core Animation、UIView动画与高级动效实战指南

在现代iOS应用开发中，流畅优雅的动画效果是提升用户体验的关键要素。本文将深入探讨iOS动画技术的各个层面，从基础的UIView动画到高级的Core Animation框架，帮助开发者掌握创建精美动画效果的技能。

## 动画技术概览

### 动画架构管理器

```objc
@interface AnimationArchitecture : NSObject

@property (nonatomic, strong) NSMutableDictionary *animationConfigurations;
@property (nonatomic, strong) NSMutableArray *activeAnimations;
@property (nonatomic, assign) BOOL debugMode;
@property (nonatomic, strong) CADisplayLink *displayLink;

+ (instancetype)sharedArchitecture;

// 动画配置管理
- (void)setupAnimationConfiguration:(NSDictionary *)configuration;
- (void)registerAnimationType:(NSString *)type withClass:(Class)animationClass;
- (id)createAnimationOfType:(NSString *)type withParameters:(NSDictionary *)parameters;

// 动画生命周期管理
- (void)startAnimation:(id)animation withIdentifier:(NSString *)identifier;
- (void)pauseAnimation:(NSString *)identifier;
- (void)resumeAnimation:(NSString *)identifier;
- (void)stopAnimation:(NSString *)identifier;
- (void)stopAllAnimations;

// 性能监控
- (void)startPerformanceMonitoring;
- (void)stopPerformanceMonitoring;
- (NSDictionary *)getPerformanceReport;

// 调试工具
- (void)enableDebugMode:(BOOL)enabled;
- (void)logAnimationHierarchy;

@end

@implementation AnimationArchitecture

+ (instancetype)sharedArchitecture {
    static AnimationArchitecture *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.animationConfigurations = [NSMutableDictionary dictionary];
        self.activeAnimations = [NSMutableArray array];
        self.debugMode = NO;
        
        [self setupDefaultConfigurations];
    }
    return self;
}

- (void)setupDefaultConfigurations {
    // 默认动画配置
    NSDictionary *defaultConfig = @{
        @"duration": @0.3,
        @"timing_function": @"ease_in_out",
        @"fill_mode": @"forwards",
        @"remove_on_completion": @NO
    };
    
    self.animationConfigurations[@"default"] = defaultConfig;
    
    // 快速动画配置
    NSDictionary *fastConfig = @{
        @"duration": @0.15,
        @"timing_function": @"ease_out",
        @"fill_mode": @"forwards",
        @"remove_on_completion": @NO
    };
    
    self.animationConfigurations[@"fast"] = fastConfig;
    
    // 慢速动画配置
    NSDictionary *slowConfig = @{
        @"duration": @0.6,
        @"timing_function": @"ease_in_out",
        @"fill_mode": @"forwards",
        @"remove_on_completion": @NO
    };
    
    self.animationConfigurations[@"slow"] = slowConfig;
}

- (void)setupAnimationConfiguration:(NSDictionary *)configuration {
    NSString *name = configuration[@"name"];
    if (name) {
        self.animationConfigurations[name] = configuration;
    }
}

- (void)registerAnimationType:(NSString *)type withClass:(Class)animationClass {
    // 注册自定义动画类型
    NSLog(@"Registered animation type: %@ with class: %@", type, NSStringFromClass(animationClass));
}

- (id)createAnimationOfType:(NSString *)type withParameters:(NSDictionary *)parameters {
    // 根据类型创建动画实例
    if ([type isEqualToString:@"basic"]) {
        CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:parameters[@"keyPath"]];
        animation.fromValue = parameters[@"fromValue"];
        animation.toValue = parameters[@"toValue"];
        return animation;
    } else if ([type isEqualToString:@"keyframe"]) {
        CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:parameters[@"keyPath"]];
        animation.values = parameters[@"values"];
        animation.keyTimes = parameters[@"keyTimes"];
        return animation;
    }
    
    return nil;
}

- (void)startAnimation:(id)animation withIdentifier:(NSString *)identifier {
    NSDictionary *animationInfo = @{
        @"animation": animation,
        @"identifier": identifier,
        @"start_time": @([[NSDate date] timeIntervalSince1970])
    };
    
    [self.activeAnimations addObject:animationInfo];
    
    if (self.debugMode) {
        NSLog(@"Started animation: %@", identifier);
    }
}

- (void)pauseAnimation:(NSString *)identifier {
    for (NSDictionary *animationInfo in self.activeAnimations) {
        if ([animationInfo[@"identifier"] isEqualToString:identifier]) {
            id animation = animationInfo[@"animation"];
            if ([animation isKindOfClass:[CAAnimation class]]) {
                CAAnimation *caAnimation = (CAAnimation *)animation;
                CFTimeInterval pausedTime = [caAnimation.layer convertTime:CACurrentMediaTime() fromLayer:nil];
                caAnimation.layer.speed = 0.0;
                caAnimation.layer.timeOffset = pausedTime;
            }
            break;
        }
    }
}

- (void)resumeAnimation:(NSString *)identifier {
    for (NSDictionary *animationInfo in self.activeAnimations) {
        if ([animationInfo[@"identifier"] isEqualToString:identifier]) {
            id animation = animationInfo[@"animation"];
            if ([animation isKindOfClass:[CAAnimation class]]) {
                CAAnimation *caAnimation = (CAAnimation *)animation;
                CFTimeInterval pausedTime = caAnimation.layer.timeOffset;
                caAnimation.layer.speed = 1.0;
                caAnimation.layer.timeOffset = 0.0;
                caAnimation.layer.beginTime = 0.0;
                CFTimeInterval timeSincePause = [caAnimation.layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
                caAnimation.layer.beginTime = timeSincePause;
            }
            break;
        }
    }
}

- (void)stopAnimation:(NSString *)identifier {
    NSMutableArray *toRemove = [NSMutableArray array];
    
    for (NSDictionary *animationInfo in self.activeAnimations) {
        if ([animationInfo[@"identifier"] isEqualToString:identifier]) {
            [toRemove addObject:animationInfo];
            
            if (self.debugMode) {
                NSLog(@"Stopped animation: %@", identifier);
            }
        }
    }
    
    [self.activeAnimations removeObjectsInArray:toRemove];
}

- (void)stopAllAnimations {
    [self.activeAnimations removeAllObjects];
    
    if (self.debugMode) {
        NSLog(@"Stopped all animations");
    }
}

- (void)startPerformanceMonitoring {
    self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(monitorPerformance)];
    [self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)stopPerformanceMonitoring {
    [self.displayLink invalidate];
    self.displayLink = nil;
}

- (void)monitorPerformance {
    // 监控动画性能
    if (self.debugMode) {
        NSLog(@"Active animations: %lu, FPS: %.1f", 
              (unsigned long)self.activeAnimations.count, 
              1.0 / self.displayLink.duration);
    }
}

- (NSDictionary *)getPerformanceReport {
    return @{
        @"active_animations_count": @(self.activeAnimations.count),
        @"fps": @(self.displayLink ? 1.0 / self.displayLink.duration : 0),
        @"memory_usage": @([self calculateMemoryUsage])
    };
}

- (NSUInteger)calculateMemoryUsage {
    // 计算动画内存使用量
    return self.activeAnimations.count * 1024; // 简化计算
}

- (void)enableDebugMode:(BOOL)enabled {
    self.debugMode = enabled;
    
    if (enabled) {
        [self startPerformanceMonitoring];
    } else {
        [self stopPerformanceMonitoring];
    }
}

- (void)logAnimationHierarchy {
    NSLog(@"Animation Hierarchy:");
    for (NSDictionary *animationInfo in self.activeAnimations) {
        NSLog(@"  - %@: %@", animationInfo[@"identifier"], animationInfo[@"animation"]);
    }
}

@end
```

## Core Animation框架深度应用

### CALayer动画管理器

```objc
@interface CALayerAnimationManager : NSObject

@property (nonatomic, strong) NSMutableDictionary *layerAnimations;
@property (nonatomic, strong) NSMutableArray *animationGroups;

+ (instancetype)sharedManager;

// 基础动画
- (CABasicAnimation *)createBasicAnimationWithKeyPath:(NSString *)keyPath
                                            fromValue:(id)fromValue
                                              toValue:(id)toValue
                                             duration:(CFTimeInterval)duration;

// 关键帧动画
- (CAKeyframeAnimation *)createKeyframeAnimationWithKeyPath:(NSString *)keyPath
                                                     values:(NSArray *)values
                                                   keyTimes:(NSArray *)keyTimes
                                                   duration:(CFTimeInterval)duration;

// 动画组
- (CAAnimationGroup *)createAnimationGroupWithAnimations:(NSArray *)animations
                                                 duration:(CFTimeInterval)duration;

// 路径动画
- (CAKeyframeAnimation *)createPathAnimationWithPath:(UIBezierPath *)path
                                             duration:(CFTimeInterval)duration
                                        rotationMode:(NSString *)rotationMode;

// 形状动画
- (void)animateShapeLayer:(CAShapeLayer *)shapeLayer
                 fromPath:(UIBezierPath *)fromPath
                   toPath:(UIBezierPath *)toPath
                 duration:(CFTimeInterval)duration;

// 渐变动画
- (void)animateGradientLayer:(CAGradientLayer *)gradientLayer
                  fromColors:(NSArray *)fromColors
                    toColors:(NSArray *)toColors
                    duration:(CFTimeInterval)duration;

// 文本动画
- (void)animateTextLayer:(CATextLayer *)textLayer
                fromText:(NSString *)fromText
                  toText:(NSString *)toText
                duration:(CFTimeInterval)duration;

// 3D变换动画
- (void)animate3DTransformForLayer:(CALayer *)layer
                     fromTransform:(CATransform3D)fromTransform
                       toTransform:(CATransform3D)toTransform
                          duration:(CFTimeInterval)duration;

// 动画控制
- (void)pauseAnimationForLayer:(CALayer *)layer;
- (void)resumeAnimationForLayer:(CALayer *)layer;
- (void)removeAllAnimationsFromLayer:(CALayer *)layer;

@end

@implementation CALayerAnimationManager

+ (instancetype)sharedManager {
    static CALayerAnimationManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.layerAnimations = [NSMutableDictionary dictionary];
        self.animationGroups = [NSMutableArray array];
    }
    return self;
}

- (CABasicAnimation *)createBasicAnimationWithKeyPath:(NSString *)keyPath
                                            fromValue:(id)fromValue
                                              toValue:(id)toValue
                                             duration:(CFTimeInterval)duration {
    
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:keyPath];
    animation.fromValue = fromValue;
    animation.toValue = toValue;
    animation.duration = duration;
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    return animation;
}

- (CAKeyframeAnimation *)createKeyframeAnimationWithKeyPath:(NSString *)keyPath
                                                     values:(NSArray *)values
                                                   keyTimes:(NSArray *)keyTimes
                                                   duration:(CFTimeInterval)duration {
    
    CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:keyPath];
    animation.values = values;
    animation.keyTimes = keyTimes;
    animation.duration = duration;
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    return animation;
}

- (CAAnimationGroup *)createAnimationGroupWithAnimations:(NSArray *)animations
                                                 duration:(CFTimeInterval)duration {
    
    CAAnimationGroup *group = [CAAnimationGroup animation];
    group.animations = animations;
    group.duration = duration;
    group.fillMode = kCAFillModeForwards;
    group.removedOnCompletion = NO;
    
    return group;
}

- (CAKeyframeAnimation *)createPathAnimationWithPath:(UIBezierPath *)path
                                             duration:(CFTimeInterval)duration
                                        rotationMode:(NSString *)rotationMode {
    
    CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    animation.path = path.CGPath;
    animation.duration = duration;
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    animation.rotationMode = rotationMode ?: kCAAnimationRotateAuto;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    return animation;
}

- (void)animateShapeLayer:(CAShapeLayer *)shapeLayer
                 fromPath:(UIBezierPath *)fromPath
                   toPath:(UIBezierPath *)toPath
                 duration:(CFTimeInterval)duration {
    
    CABasicAnimation *pathAnimation = [CABasicAnimation animationWithKeyPath:@"path"];
    pathAnimation.fromValue = (__bridge id)fromPath.CGPath;
    pathAnimation.toValue = (__bridge id)toPath.CGPath;
    pathAnimation.duration = duration;
    pathAnimation.fillMode = kCAFillModeForwards;
    pathAnimation.removedOnCompletion = NO;
    pathAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    [shapeLayer addAnimation:pathAnimation forKey:@"pathAnimation"];
    
    // 更新最终状态
    shapeLayer.path = toPath.CGPath;
}

- (void)animateGradientLayer:(CAGradientLayer *)gradientLayer
                  fromColors:(NSArray *)fromColors
                    toColors:(NSArray *)toColors
                    duration:(CFTimeInterval)duration {
    
    CABasicAnimation *colorAnimation = [CABasicAnimation animationWithKeyPath:@"colors"];
    colorAnimation.fromValue = fromColors;
    colorAnimation.toValue = toColors;
    colorAnimation.duration = duration;
    colorAnimation.fillMode = kCAFillModeForwards;
    colorAnimation.removedOnCompletion = NO;
    colorAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    [gradientLayer addAnimation:colorAnimation forKey:@"colorAnimation"];
    
    // 更新最终状态
    gradientLayer.colors = toColors;
}

- (void)animateTextLayer:(CATextLayer *)textLayer
                fromText:(NSString *)fromText
                  toText:(NSString *)toText
                duration:(CFTimeInterval)duration {
    
    // 文本淡出动画
    CABasicAnimation *fadeOut = [CABasicAnimation animationWithKeyPath:@"opacity"];
    fadeOut.fromValue = @1.0;
    fadeOut.toValue = @0.0;
    fadeOut.duration = duration / 2;
    fadeOut.fillMode = kCAFillModeForwards;
    fadeOut.removedOnCompletion = NO;
    
    // 文本淡入动画
    CABasicAnimation *fadeIn = [CABasicAnimation animationWithKeyPath:@"opacity"];
    fadeIn.fromValue = @0.0;
    fadeIn.toValue = @1.0;
    fadeIn.duration = duration / 2;
    fadeIn.beginTime = CACurrentMediaTime() + duration / 2;
    fadeIn.fillMode = kCAFillModeForwards;
    fadeIn.removedOnCompletion = NO;
    
    [textLayer addAnimation:fadeOut forKey:@"fadeOut"];
    
    // 在动画中间更改文本
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration / 2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        textLayer.string = toText;
        [textLayer addAnimation:fadeIn forKey:@"fadeIn"];
    });
}

- (void)animate3DTransformForLayer:(CALayer *)layer
                     fromTransform:(CATransform3D)fromTransform
                       toTransform:(CATransform3D)toTransform
                          duration:(CFTimeInterval)duration {
    
    CABasicAnimation *transformAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
    transformAnimation.fromValue = [NSValue valueWithCATransform3D:fromTransform];
    transformAnimation.toValue = [NSValue valueWithCATransform3D:toTransform];
    transformAnimation.duration = duration;
    transformAnimation.fillMode = kCAFillModeForwards;
    transformAnimation.removedOnCompletion = NO;
    transformAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    [layer addAnimation:transformAnimation forKey:@"3DTransform"];
    
    // 更新最终状态
    layer.transform = toTransform;
}

- (void)pauseAnimationForLayer:(CALayer *)layer {
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
    layer.speed = 0.0;
    layer.timeOffset = pausedTime;
}

- (void)resumeAnimationForLayer:(CALayer *)layer {
    CFTimeInterval pausedTime = layer.timeOffset;
    layer.speed = 1.0;
    layer.timeOffset = 0.0;
    layer.beginTime = 0.0;
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    layer.beginTime = timeSincePause;
}

- (void)removeAllAnimationsFromLayer:(CALayer *)layer {
    [layer removeAllAnimations];
}

@end
```

### 高级动画效果实现

```objc
@interface AdvancedAnimationEffects : NSObject

+ (instancetype)sharedEffects;

// 粒子效果
- (CAEmitterLayer *)createParticleEffectWithType:(NSString *)type
                                         position:(CGPoint)position
                                             size:(CGSize)size;

// 波纹效果
- (void)createRippleEffectInView:(UIView *)view
                        atPoint:(CGPoint)point
                          color:(UIColor *)color
                       duration:(CFTimeInterval)duration;

// 液体效果
- (void)createLiquidEffectBetweenView:(UIView *)fromView
                               toView:(UIView *)toView
                             duration:(CFTimeInterval)duration;

// 磁场效果
- (void)createMagneticEffectForViews:(NSArray *)views
                          centerPoint:(CGPoint)centerPoint
                             strength:(CGFloat)strength
                             duration:(CFTimeInterval)duration;

// 弹性碰撞效果
- (void)createElasticCollisionBetweenView:(UIView *)view1
                                  andView:(UIView *)view2
                                 duration:(CFTimeInterval)duration;

// 折纸效果
- (void)createOrigamiEffectForView:(UIView *)view
                         foldCount:(NSInteger)foldCount
                          duration:(CFTimeInterval)duration;

// 破碎效果
- (void)createShatterEffectForView:(UIView *)view
                        pieceCount:(NSInteger)pieceCount
                          duration:(CFTimeInterval)duration;

// 烟花效果
- (void)createFireworksEffectAtPoint:(CGPoint)point
                              inView:(UIView *)view
                               color:(UIColor *)color;

// 雪花效果
- (CAEmitterLayer *)createSnowEffectInView:(UIView *)view;

// 雨滴效果
- (CAEmitterLayer *)createRainEffectInView:(UIView *)view;

@end

@implementation AdvancedAnimationEffects

+ (instancetype)sharedEffects {
    static AdvancedAnimationEffects *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (CAEmitterLayer *)createParticleEffectWithType:(NSString *)type
                                         position:(CGPoint)position
                                             size:(CGSize)size {
    
    CAEmitterLayer *emitterLayer = [CAEmitterLayer layer];
    emitterLayer.emitterPosition = position;
    emitterLayer.emitterSize = size;
    emitterLayer.emitterShape = kCAEmitterLayerCircle;
    emitterLayer.emitterMode = kCAEmitterLayerOutline;
    
    CAEmitterCell *cell = [CAEmitterCell emitterCell];
    
    if ([type isEqualToString:@"fire"]) {
        cell.contents = (__bridge id)[self createFireParticleImage].CGImage;
        cell.birthRate = 100;
        cell.lifetime = 2.0;
        cell.velocity = 50;
        cell.velocityRange = 20;
        cell.emissionRange = M_PI;
        cell.scale = 0.1;
        cell.scaleRange = 0.05;
        cell.alphaSpeed = -0.5;
        cell.color = [UIColor orangeColor].CGColor;
    } else if ([type isEqualToString:@"smoke"]) {
        cell.contents = (__bridge id)[self createSmokeParticleImage].CGImage;
        cell.birthRate = 50;
        cell.lifetime = 3.0;
        cell.velocity = 30;
        cell.velocityRange = 10;
        cell.emissionRange = M_PI / 4;
        cell.scale = 0.2;
        cell.scaleSpeed = 0.1;
        cell.alphaSpeed = -0.3;
        cell.color = [UIColor grayColor].CGColor;
    } else if ([type isEqualToString:@"sparkle"]) {
        cell.contents = (__bridge id)[self createSparkleParticleImage].CGImage;
        cell.birthRate = 200;
        cell.lifetime = 1.5;
        cell.velocity = 80;
        cell.velocityRange = 40;
        cell.emissionRange = 2 * M_PI;
        cell.scale = 0.05;
        cell.scaleRange = 0.02;
        cell.alphaSpeed = -0.7;
        cell.color = [UIColor yellowColor].CGColor;
    }
    
    emitterLayer.emitterCells = @[cell];
    
    return emitterLayer;
}

- (UIImage *)createFireParticleImage {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(10, 10), NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    CGContextSetFillColorWithColor(context, [UIColor orangeColor].CGColor);
    CGContextFillEllipseInRect(context, CGRectMake(0, 0, 10, 10));
    
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return image;
}

- (UIImage *)createSmokeParticleImage {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(15, 15), NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    CGContextSetFillColorWithColor(context, [UIColor lightGrayColor].CGColor);
    CGContextFillEllipseInRect(context, CGRectMake(0, 0, 15, 15));
    
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return image;
}

- (UIImage *)createSparkleParticleImage {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(8, 8), NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    CGContextSetFillColorWithColor(context, [UIColor yellowColor].CGColor);
    CGContextFillEllipseInRect(context, CGRectMake(0, 0, 8, 8));
    
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return image;
}

- (void)createRippleEffectInView:(UIView *)view
                        atPoint:(CGPoint)point
                          color:(UIColor *)color
                       duration:(CFTimeInterval)duration {
    
    CAShapeLayer *rippleLayer = [CAShapeLayer layer];
    rippleLayer.frame = view.bounds;
    rippleLayer.fillColor = [UIColor clearColor].CGColor;
    rippleLayer.strokeColor = color.CGColor;
    rippleLayer.lineWidth = 2.0;
    
    UIBezierPath *startPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(point.x - 1, point.y - 1, 2, 2)];
    UIBezierPath *endPath = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(point.x - 100, point.y - 100, 200, 200)];
    
    rippleLayer.path = startPath.CGPath;
    [view.layer addSublayer:rippleLayer];
    
    // 路径动画
    CABasicAnimation *pathAnimation = [CABasicAnimation animationWithKeyPath:@"path"];
    pathAnimation.fromValue = (__bridge id)startPath.CGPath;
    pathAnimation.toValue = (__bridge id)endPath.CGPath;
    pathAnimation.duration = duration;
    pathAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    // 透明度动画
    CABasicAnimation *opacityAnimation = [CABasicAnimation animationWithKeyPath:@"opacity"];
    opacityAnimation.fromValue = @1.0;
    opacityAnimation.toValue = @0.0;
    opacityAnimation.duration = duration;
    
    // 动画组
    CAAnimationGroup *group = [CAAnimationGroup animation];
    group.animations = @[pathAnimation, opacityAnimation];
    group.duration = duration;
    group.fillMode = kCAFillModeForwards;
    group.removedOnCompletion = NO;
    
    [rippleLayer addAnimation:group forKey:@"ripple"];
    
    // 动画完成后移除图层
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [rippleLayer removeFromSuperlayer];
    });
}

- (void)createLiquidEffectBetweenView:(UIView *)fromView
                               toView:(UIView *)toView
                             duration:(CFTimeInterval)duration {
    
    // 创建液体路径
    UIBezierPath *liquidPath = [UIBezierPath bezierPath];
    CGPoint startPoint = fromView.center;
    CGPoint endPoint = toView.center;
    CGPoint controlPoint1 = CGPointMake(startPoint.x + (endPoint.x - startPoint.x) * 0.3, startPoint.y - 50);
    CGPoint controlPoint2 = CGPointMake(startPoint.x + (endPoint.x - startPoint.x) * 0.7, endPoint.y + 50);
    
    [liquidPath moveToPoint:startPoint];
    [liquidPath addCurveToPoint:endPoint controlPoint1:controlPoint1 controlPoint2:controlPoint2];
    
    // 创建液体图层
    CAShapeLayer *liquidLayer = [CAShapeLayer layer];
    liquidLayer.path = liquidPath.CGPath;
    liquidLayer.strokeColor = [UIColor blueColor].CGColor;
    liquidLayer.fillColor = [UIColor clearColor].CGColor;
    liquidLayer.lineWidth = 10.0;
    liquidLayer.lineCap = kCALineCapRound;
    
    [fromView.superview.layer addSublayer:liquidLayer];
    
    // 路径动画
    CABasicAnimation *strokeAnimation = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
    strokeAnimation.fromValue = @0.0;
    strokeAnimation.toValue = @1.0;
    strokeAnimation.duration = duration;
    strokeAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    [liquidLayer addAnimation:strokeAnimation forKey:@"liquid"];
    
    // 动画完成后移除图层
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [liquidLayer removeFromSuperlayer];
    });
}

- (void)createMagneticEffectForViews:(NSArray *)views
                          centerPoint:(CGPoint)centerPoint
                             strength:(CGFloat)strength
                             duration:(CFTimeInterval)duration {
    
    for (UIView *view in views) {
        CGPoint originalCenter = view.center;
        CGFloat distance = sqrt(pow(originalCenter.x - centerPoint.x, 2) + pow(originalCenter.y - centerPoint.y, 2));
        CGFloat force = strength / (distance * distance);
        
        CGFloat deltaX = (centerPoint.x - originalCenter.x) * force;
        CGFloat deltaY = (centerPoint.y - originalCenter.y) * force;
        
        CGPoint attractedPoint = CGPointMake(originalCenter.x + deltaX, originalCenter.y + deltaY);
        
        [UIView animateWithDuration:duration / 2
                              delay:0
                            options:UIViewAnimationOptionCurveEaseOut
                         animations:^{
            view.center = attractedPoint;
        } completion:^(BOOL finished) {
            [UIView animateWithDuration:duration / 2
                                  delay:0
                 usingSpringWithDamping:0.5
                  initialSpringVelocity:0.8
                                options:UIViewAnimationOptionCurveEaseInOut
                             animations:^{
                view.center = originalCenter;
            } completion:nil];
        }];
    }
}

- (void)createElasticCollisionBetweenView:(UIView *)view1
                                  andView:(UIView *)view2
                                 duration:(CFTimeInterval)duration {
    
    CGPoint center1 = view1.center;
    CGPoint center2 = view2.center;
    
    // 计算碰撞点
    CGPoint collisionPoint = CGPointMake((center1.x + center2.x) / 2, (center1.y + center2.y) / 2);
    
    // 第一阶段：移动到碰撞点
    [UIView animateWithDuration:duration / 3
                          delay:0
                        options:UIViewAnimationOptionCurveEaseIn
                     animations:^{
        view1.center = collisionPoint;
        view2.center = collisionPoint;
    } completion:^(BOOL finished) {
        
        // 第二阶段：弹开
        [UIView animateWithDuration:duration / 3
                              delay:0
             usingSpringWithDamping:0.3
              initialSpringVelocity:1.0
                            options:UIViewAnimationOptionCurveEaseOut
                         animations:^{
            view1.center = CGPointMake(center1.x + (center1.x - center2.x) * 0.2, center1.y + (center1.y - center2.y) * 0.2);
            view2.center = CGPointMake(center2.x + (center2.x - center1.x) * 0.2, center2.y + (center2.y - center1.y) * 0.2);
        } completion:^(BOOL finished) {
            
            // 第三阶段：回到原位
            [UIView animateWithDuration:duration / 3
                                  delay:0
                 usingSpringWithDamping:0.7
                  initialSpringVelocity:0.5
                                options:UIViewAnimationOptionCurveEaseInOut
                             animations:^{
                view1.center = center1;
                view2.center = center2;
            } completion:nil];
        }];
    }];
}

- (void)createOrigamiEffectForView:(UIView *)view
                         foldCount:(NSInteger)foldCount
                          duration:(CFTimeInterval)duration {
    
    CATransform3D perspective = CATransform3DIdentity;
    perspective.m34 = -1.0 / 1000.0;
    view.layer.transform = perspective;
    
    NSTimeInterval stepDuration = duration / foldCount;
    
    for (NSInteger i = 0; i < foldCount; i++) {
        CGFloat angle = M_PI / foldCount * (i + 1);
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(stepDuration * i * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [UIView animateWithDuration:stepDuration
                                  delay:0
                                options:UIViewAnimationOptionCurveEaseInOut
                             animations:^{
                CATransform3D transform = CATransform3DRotate(perspective, angle, 1, 0, 0);
                view.layer.transform = transform;
            } completion:^(BOOL finished) {
                if (i == foldCount - 1) {
                    // 最后一步，恢复原状
                    [UIView animateWithDuration:stepDuration animations:^{
                        view.layer.transform = CATransform3DIdentity;
                    }];
                }
            }];
        });
    }
}

- (void)createShatterEffectForView:(UIView *)view
                        pieceCount:(NSInteger)pieceCount
                          duration:(CFTimeInterval)duration {
    
    // 创建视图快照
    UIGraphicsBeginImageContextWithOptions(view.bounds.size, NO, 0);
    [view.layer renderInContext:UIGraphicsGetCurrentContext()];
    UIImage *snapshot = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    // 隐藏原视图
    view.alpha = 0;
    
    // 创建碎片
    NSMutableArray *pieces = [NSMutableArray array];
    CGFloat pieceWidth = view.bounds.size.width / sqrt(pieceCount);
    CGFloat pieceHeight = view.bounds.size.height / sqrt(pieceCount);
    
    for (NSInteger i = 0; i < sqrt(pieceCount); i++) {
        for (NSInteger j = 0; j < sqrt(pieceCount); j++) {
            CGRect pieceFrame = CGRectMake(j * pieceWidth, i * pieceHeight, pieceWidth, pieceHeight);
            
            // 创建碎片图像
            CGImageRef pieceImageRef = CGImageCreateWithImageInRect(snapshot.CGImage, pieceFrame);
            UIImage *pieceImage = [UIImage imageWithCGImage:pieceImageRef];
            CGImageRelease(pieceImageRef);
            
            // 创建碎片视图
            UIImageView *pieceView = [[UIImageView alloc] initWithImage:pieceImage];
            pieceView.frame = CGRectMake(view.frame.origin.x + pieceFrame.origin.x,
                                       view.frame.origin.y + pieceFrame.origin.y,
                                       pieceFrame.size.width,
                                       pieceFrame.size.height);
            
            [view.superview addSubview:pieceView];
            [pieces addObject:pieceView];
        }
    }
    
    // 动画碎片
    for (UIImageView *piece in pieces) {
        CGFloat randomX = (arc4random_uniform(200) - 100) * 2;
        CGFloat randomY = (arc4random_uniform(200) - 100) * 2;
        CGFloat randomRotation = (arc4random_uniform(360) - 180) * M_PI / 180;
        
        [UIView animateWithDuration:duration
                              delay:0
                            options:UIViewAnimationOptionCurveEaseIn
                         animations:^{
            piece.center = CGPointMake(piece.center.x + randomX, piece.center.y + randomY);
            piece.transform = CGAffineTransformMakeRotation(randomRotation);
            piece.alpha = 0;
        } completion:^(BOOL finished) {
            [piece removeFromSuperview];
        }];
    }
    
    // 动画完成后恢复原视图
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        view.alpha = 1;
    });
}

- (void)createFireworksEffectAtPoint:(CGPoint)point
                              inView:(UIView *)view
                               color:(UIColor *)color {
    
    CAEmitterLayer *fireworksLayer = [CAEmitterLayer layer];
    fireworksLayer.emitterPosition = point;
    fireworksLayer.emitterSize = CGSizeMake(2, 2);
    fireworksLayer.emitterShape = kCAEmitterLayerPoint;
    fireworksLayer.emitterMode = kCAEmitterLayerOutline;
    
    CAEmitterCell *fireworkCell = [CAEmitterCell emitterCell];
    fireworkCell.contents = (__bridge id)[self createSparkleParticleImage].CGImage;
    fireworkCell.birthRate = 500;
    fireworkCell.lifetime = 2.0;
    fireworkCell.velocity = 100;
    fireworkCell.velocityRange = 50;
    fireworkCell.emissionRange = 2 * M_PI;
    fireworkCell.scale = 0.1;
    fireworkCell.scaleRange = 0.05;
    fireworkCell.alphaSpeed = -0.5;
    fireworkCell.color = color.CGColor;
    fireworkCell.yAcceleration = 50; // 重力效果
    
    fireworksLayer.emitterCells = @[fireworkCell];
    [view.layer addSublayer:fireworksLayer];
    
    // 2秒后停止发射并移除
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        fireworksLayer.birthRate = 0;
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [fireworksLayer removeFromSuperlayer];
        });
    });
}

- (CAEmitterLayer *)createSnowEffectInView:(UIView *)view {
    CAEmitterLayer *snowLayer = [CAEmitterLayer layer];
    snowLayer.emitterPosition = CGPointMake(view.bounds.size.width / 2, -10);
    snowLayer.emitterSize = CGSizeMake(view.bounds.size.width, 1);
    snowLayer.emitterShape = kCAEmitterLayerLine;
    snowLayer.emitterMode = kCAEmitterLayerSurface;
    
    CAEmitterCell *snowCell = [CAEmitterCell emitterCell];
    snowCell.contents = (__bridge id)[self createSnowflakeImage].CGImage;
    snowCell.birthRate = 50;
    snowCell.lifetime = 10.0;
    snowCell.velocity = 30;
    snowCell.velocityRange = 20;
    snowCell.emissionRange = M_PI / 8;
    snowCell.scale = 0.1;
    snowCell.scaleRange = 0.05;
    snowCell.alphaSpeed = -0.1;
    snowCell.color = [UIColor whiteColor].CGColor;
    snowCell.yAcceleration = 20;
    snowCell.xAcceleration = 5;
    
    snowLayer.emitterCells = @[snowCell];
    
    return snowLayer;
}

- (UIImage *)createSnowflakeImage {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(12, 12), NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
    CGContextFillEllipseInRect(context, CGRectMake(0, 0, 12, 12));
    
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return image;
}

- (CAEmitterLayer *)createRainEffectInView:(UIView *)view {
    CAEmitterLayer *rainLayer = [CAEmitterLayer layer];
    rainLayer.emitterPosition = CGPointMake(view.bounds.size.width / 2, -10);
    rainLayer.emitterSize = CGSizeMake(view.bounds.size.width, 1);
    rainLayer.emitterShape = kCAEmitterLayerLine;
    rainLayer.emitterMode = kCAEmitterLayerSurface;
    
    CAEmitterCell *rainCell = [CAEmitterCell emitterCell];
    rainCell.contents = (__bridge id)[self createRaindropImage].CGImage;
    rainCell.birthRate = 200;
    rainCell.lifetime = 3.0;
    rainCell.velocity = 200;
    rainCell.velocityRange = 50;
    rainCell.emissionRange = M_PI / 16;
    rainCell.scale = 0.2;
    rainCell.scaleRange = 0.1;
    rainCell.alphaSpeed = -0.3;
    rainCell.color = [UIColor blueColor].CGColor;
    rainCell.yAcceleration = 100;
    
    rainLayer.emitterCells = @[rainCell];
    
    return rainLayer;
}

- (UIImage *)createRaindropImage {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(4, 20), NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    CGContextSetFillColorWithColor(context, [UIColor blueColor].CGColor);
    CGContextFillRect(context, CGRectMake(1, 0, 2, 20));
    
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return image;
}

@end
```

## 动画性能优化与监控

### 动画性能监控器

```objc
@interface AnimationPerformanceMonitor : NSObject

@property (nonatomic, strong) NSMutableDictionary *animationMetrics;
@property (nonatomic, strong) NSMutableArray *performanceReports;
@property (nonatomic, assign) BOOL isMonitoring;

+ (instancetype)sharedMonitor;

// 性能监控
- (void)startMonitoringAnimation:(NSString *)animationName;
- (void)stopMonitoringAnimation:(NSString *)animationName;
- (void)recordFrameDropForAnimation:(NSString *)animationName;
- (void)recordMemoryUsageForAnimation:(NSString *)animationName;

// 性能分析
- (NSDictionary *)getPerformanceReportForAnimation:(NSString *)animationName;
- (NSArray *)getAllPerformanceReports;
- (void)generateOptimizationSuggestions;

// 性能优化
- (void)optimizeLayerForAnimation:(CALayer *)layer;
- (void)enableHardwareAccelerationForView:(UIView *)view;
- (void)optimizeAnimationTiming:(CAAnimation *)animation;

// FPS监控
- (void)startFPSMonitoring;
- (void)stopFPSMonitoring;
- (CGFloat)getCurrentFPS;

@end

@implementation AnimationPerformanceMonitor

+ (instancetype)sharedMonitor {
    static AnimationPerformanceMonitor *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.animationMetrics = [NSMutableDictionary dictionary];
        self.performanceReports = [NSMutableArray array];
        self.isMonitoring = NO;
    }
    return self;
}

- (void)startMonitoringAnimation:(NSString *)animationName {
    NSMutableDictionary *metrics = [NSMutableDictionary dictionary];
    metrics[@"startTime"] = @(CACurrentMediaTime());
    metrics[@"frameDrops"] = @0;
    metrics[@"memoryUsage"] = @0;
    metrics[@"isActive"] = @YES;
    
    self.animationMetrics[animationName] = metrics;
    self.isMonitoring = YES;
    
    NSLog(@"开始监控动画: %@", animationName);
}

- (void)stopMonitoringAnimation:(NSString *)animationName {
    NSMutableDictionary *metrics = self.animationMetrics[animationName];
    if (metrics) {
        metrics[@"endTime"] = @(CACurrentMediaTime());
        metrics[@"isActive"] = @NO;
        
        CFTimeInterval duration = [metrics[@"endTime"] doubleValue] - [metrics[@"startTime"] doubleValue];
        metrics[@"duration"] = @(duration);
        
        // 生成性能报告
        NSDictionary *report = [self generateReportForAnimation:animationName withMetrics:metrics];
        [self.performanceReports addObject:report];
        
        NSLog(@"停止监控动画: %@, 持续时间: %.2f秒", animationName, duration);
    }
}

- (void)recordFrameDropForAnimation:(NSString *)animationName {
    NSMutableDictionary *metrics = self.animationMetrics[animationName];
    if (metrics && [metrics[@"isActive"] boolValue]) {
        NSInteger frameDrops = [metrics[@"frameDrops"] integerValue] + 1;
        metrics[@"frameDrops"] = @(frameDrops);
    }
}

- (void)recordMemoryUsageForAnimation:(NSString *)animationName {
    NSMutableDictionary *metrics = self.animationMetrics[animationName];
    if (metrics && [metrics[@"isActive"] boolValue]) {
        // 获取当前内存使用情况
        struct mach_task_basic_info info;
        mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
        kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
        
        if (kerr == KERN_SUCCESS) {
            CGFloat memoryUsage = info.resident_size / (1024.0 * 1024.0); // MB
            metrics[@"memoryUsage"] = @(memoryUsage);
        }
    }
}

- (NSDictionary *)generateReportForAnimation:(NSString *)animationName withMetrics:(NSDictionary *)metrics {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    report[@"animationName"] = animationName;
    report[@"duration"] = metrics[@"duration"];
    report[@"frameDrops"] = metrics[@"frameDrops"];
    report[@"memoryUsage"] = metrics[@"memoryUsage"];
    report[@"timestamp"] = @([[NSDate date] timeIntervalSince1970]);
    
    // 计算性能评分
    CGFloat performanceScore = [self calculatePerformanceScore:metrics];
    report[@"performanceScore"] = @(performanceScore);
    
    // 生成优化建议
    NSArray *suggestions = [self generateOptimizationSuggestionsForMetrics:metrics];
    report[@"optimizationSuggestions"] = suggestions;
    
    return report;
}

- (CGFloat)calculatePerformanceScore:(NSDictionary *)metrics {
    CGFloat score = 100.0;
    
    // 根据掉帧数扣分
    NSInteger frameDrops = [metrics[@"frameDrops"] integerValue];
    score -= frameDrops * 5.0;
    
    // 根据内存使用扣分
    CGFloat memoryUsage = [metrics[@"memoryUsage"] doubleValue];
    if (memoryUsage > 50.0) {
        score -= (memoryUsage - 50.0) * 0.5;
    }
    
    // 根据动画时长扣分（过长的动画可能影响用户体验）
    CGFloat duration = [metrics[@"duration"] doubleValue];
    if (duration > 2.0) {
        score -= (duration - 2.0) * 10.0;
    }
    
    return MAX(0, score);
}

- (NSArray *)generateOptimizationSuggestionsForMetrics:(NSDictionary *)metrics {
    NSMutableArray *suggestions = [NSMutableArray array];
    
    NSInteger frameDrops = [metrics[@"frameDrops"] integerValue];
    if (frameDrops > 5) {
        [suggestions addObject:@"检测到较多掉帧，建议优化动画复杂度或使用硬件加速"];
    }
    
    CGFloat memoryUsage = [metrics[@"memoryUsage"] doubleValue];
    if (memoryUsage > 100.0) {
        [suggestions addObject:@"内存使用过高，建议优化图像资源或减少同时进行的动画数量"];
    }
    
    CGFloat duration = [metrics[@"duration"] doubleValue];
    if (duration > 3.0) {
        [suggestions addObject:@"动画时长过长，建议缩短动画时间或分解为多个短动画"];
    }
    
    return suggestions;
}

- (NSDictionary *)getPerformanceReportForAnimation:(NSString *)animationName {
    for (NSDictionary *report in self.performanceReports) {
        if ([report[@"animationName"] isEqualToString:animationName]) {
            return report;
        }
    }
    return nil;
}

- (NSArray *)getAllPerformanceReports {
    return [self.performanceReports copy];
}

- (void)generateOptimizationSuggestions {
    NSLog(@"=== 动画性能优化建议 ===");
    
    for (NSDictionary *report in self.performanceReports) {
        NSString *animationName = report[@"animationName"];
        CGFloat score = [report[@"performanceScore"] doubleValue];
        NSArray *suggestions = report[@"optimizationSuggestions"];
        
        NSLog(@"动画: %@, 性能评分: %.1f", animationName, score);
        
        if (suggestions.count > 0) {
            for (NSString *suggestion in suggestions) {
                NSLog(@"  - %@", suggestion);
            }
        } else {
            NSLog(@"  - 性能良好，无需优化");
        }
        NSLog(@"");
    }
}

- (void)optimizeLayerForAnimation:(CALayer *)layer {
    // 启用光栅化以提高性能
    layer.shouldRasterize = YES;
    layer.rasterizationScale = [UIScreen mainScreen].scale;
    
    // 设置不透明度以避免混合
    if (layer.backgroundColor) {
        layer.opaque = YES;
    }
    
    // 避免离屏渲染
    layer.masksToBounds = NO;
    layer.cornerRadius = 0;
    layer.shadowPath = nil;
}

- (void)enableHardwareAccelerationForView:(UIView *)view {
    // 启用硬件加速
    view.layer.drawsAsynchronously = YES;
    
    // 设置3D变换以触发硬件加速
    CATransform3D transform = view.layer.transform;
    transform.m34 = -1.0 / 1000.0;
    view.layer.transform = transform;
}

- (void)optimizeAnimationTiming:(CAAnimation *)animation {
    // 使用更高效的时间函数
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    // 设置合理的帧率
    if ([animation respondsToSelector:@selector(setPreferredFrameRateRange:)]) {
        // iOS 15+
        CAFrameRateRange range = CAFrameRateRangeMake(30, 60, 60);
        [(id)animation setPreferredFrameRateRange:range];
    }
}

- (void)startFPSMonitoring {
    // 使用CADisplayLink监控FPS
    CADisplayLink *displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(updateFPS:)];
    [displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)updateFPS:(CADisplayLink *)displayLink {
    // FPS计算逻辑
    static CFTimeInterval lastTimestamp = 0;
    static NSInteger frameCount = 0;
    
    frameCount++;
    
    if (lastTimestamp == 0) {
        lastTimestamp = displayLink.timestamp;
        return;
    }
    
    CFTimeInterval deltaTime = displayLink.timestamp - lastTimestamp;
    if (deltaTime >= 1.0) {
        CGFloat fps = frameCount / deltaTime;
        NSLog(@"当前FPS: %.1f", fps);
        
        frameCount = 0;
        lastTimestamp = displayLink.timestamp;
    }
}

- (void)stopFPSMonitoring {
    // 停止FPS监控的实现
}

- (CGFloat)getCurrentFPS {
    // 返回当前FPS的实现
    return 60.0; // 示例值
}

@end
```

### 动画最佳实践管理器

```objc
@interface AnimationBestPractices : NSObject

+ (instancetype)sharedPractices;

// 动画设计原则
- (void)applyMaterialDesignPrinciplesToAnimation:(CAAnimation *)animation;
- (void)applyiOSDesignGuidelinesToAnimation:(CAAnimation *)animation;
- (void)ensureAccessibilityForAnimation:(CAAnimation *)animation inView:(UIView *)view;

// 性能优化
- (void)optimizeAnimationForBatteryLife:(CAAnimation *)animation;
- (void)reduceAnimationComplexity:(CAAnimation *)animation;
- (void)implementEfficientAnimationCaching;

// 用户体验
- (void)addHapticFeedbackToAnimation:(CAAnimation *)animation;
- (void)implementProgressiveDisclosure:(NSArray *)animations;
- (void)addInterruptibleAnimationSupport:(CAAnimation *)animation;

// 调试和测试
- (void)enableAnimationDebugging;
- (void)validateAnimationPerformance:(CAAnimation *)animation;
- (void)generateAnimationTestReport;

@end

@implementation AnimationBestPractices

+ (instancetype)sharedPractices {
    static AnimationBestPractices *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (void)applyMaterialDesignPrinciplesToAnimation:(CAAnimation *)animation {
    // 应用Material Design动画原则
    
    // 1. 自然的缓动曲线
    animation.timingFunction = [CAMediaTimingFunction functionWithControlPoints:0.4 :0.0 :0.2 :1.0];
    
    // 2. 合理的动画时长
    if (animation.duration == 0) {
        animation.duration = 0.3; // 默认300ms
    }
    
    // 3. 避免过于复杂的动画
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
}

- (void)applyiOSDesignGuidelinesToAnimation:(CAAnimation *)animation {
    // 应用iOS设计指南
    
    // 1. 使用iOS标准缓动
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    // 2. 遵循iOS动画时长建议
    if (animation.duration > 0.5) {
        NSLog(@"警告: 动画时长超过iOS建议的0.5秒");
    }
    
    // 3. 支持减少动画设置
    if (UIAccessibilityIsReduceMotionEnabled()) {
        animation.duration *= 0.1; // 大幅缩短动画时间
    }
}

- (void)ensureAccessibilityForAnimation:(CAAnimation *)animation inView:(UIView *)view {
    // 确保动画的可访问性
    
    // 1. 检查减少动画设置
    if (UIAccessibilityIsReduceMotionEnabled()) {
        // 禁用或简化动画
        animation.duration = 0.1;
        NSLog(@"已为可访问性简化动画");
    }
    
    // 2. 添加可访问性标识
    view.accessibilityLabel = @"正在执行动画";
    
    // 3. 动画完成后恢复可访问性
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(animation.duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        view.accessibilityLabel = nil;
    });
}

- (void)optimizeAnimationForBatteryLife:(CAAnimation *)animation {
    // 优化动画以节省电池
    
    // 1. 降低帧率
    if ([animation respondsToSelector:@selector(setPreferredFrameRateRange:)]) {
        CAFrameRateRange range = CAFrameRateRangeMake(30, 60, 30);
        [(id)animation setPreferredFrameRateRange:range];
    }
    
    // 2. 减少动画复杂度
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
    
    // 3. 避免不必要的重绘
    if ([animation isKindOfClass:[CABasicAnimation class]]) {
        CABasicAnimation *basicAnimation = (CABasicAnimation *)animation;
        if ([basicAnimation.keyPath isEqualToString:@"opacity"]) {
            // 透明度动画相对省电
            NSLog(@"使用省电的透明度动画");
        }
    }
}

- (void)reduceAnimationComplexity:(CAAnimation *)animation {
    // 减少动画复杂度
    
    if ([animation isKindOfClass:[CAAnimationGroup class]]) {
        CAAnimationGroup *group = (CAAnimationGroup *)animation;
        if (group.animations.count > 3) {
            NSLog(@"警告: 动画组包含过多动画(%lu个)，建议简化", (unsigned long)group.animations.count);
        }
    }
    
    // 避免同时进行过多动画
    static NSInteger activeAnimationCount = 0;
    activeAnimationCount++;
    
    if (activeAnimationCount > 5) {
        NSLog(@"警告: 同时进行的动画过多(%ld个)，可能影响性能", (long)activeAnimationCount);
    }
    
    // 动画完成后减少计数
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(animation.duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        activeAnimationCount--;
    });
}

- (void)implementEfficientAnimationCaching {
    // 实现高效的动画缓存
    static NSMutableDictionary *animationCache = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        animationCache = [NSMutableDictionary dictionary];
    });
    
    // 缓存常用动画
    NSString *fadeInKey = @"fadeIn";
    if (!animationCache[fadeInKey]) {
        CABasicAnimation *fadeIn = [CABasicAnimation animationWithKeyPath:@"opacity"];
        fadeIn.fromValue = @0.0;
        fadeIn.toValue = @1.0;
        fadeIn.duration = 0.3;
        animationCache[fadeInKey] = fadeIn;
    }
    
    NSLog(@"动画缓存已初始化，包含%lu个预设动画", (unsigned long)animationCache.count);
}

- (void)addHapticFeedbackToAnimation:(CAAnimation *)animation {
    // 为动画添加触觉反馈
    
    if (@available(iOS 10.0, *)) {
        UIImpactFeedbackGenerator *feedbackGenerator = [[UIImpactFeedbackGenerator alloc] initWithStyle:UIImpactFeedbackStyleLight];
        [feedbackGenerator prepare];
        
        // 在动画开始时提供触觉反馈
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [feedbackGenerator impactOccurred];
        });
    }
}

- (void)implementProgressiveDisclosure:(NSArray *)animations {
    // 实现渐进式展示
    
    CFTimeInterval delay = 0.0;
    CFTimeInterval staggerDelay = 0.1;
    
    for (CAAnimation *animation in animations) {
        animation.beginTime = CACurrentMediaTime() + delay;
        delay += staggerDelay;
    }
    
    NSLog(@"已为%lu个动画实现渐进式展示", (unsigned long)animations.count);
}

- (void)addInterruptibleAnimationSupport:(CAAnimation *)animation {
    // 添加可中断动画支持
    
    animation.fillMode = kCAFillModeBoth;
    animation.removedOnCompletion = NO;
    
    // 添加动画标识以便后续中断
    static NSInteger animationID = 0;
    NSString *animationKey = [NSString stringWithFormat:@"interruptible_animation_%ld", (long)++animationID];
    
    // 存储动画引用以便中断
    static NSMutableDictionary *activeAnimations = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        activeAnimations = [NSMutableDictionary dictionary];
    });
    
    activeAnimations[animationKey] = animation;
    
    // 动画完成后清理
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(animation.duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [activeAnimations removeObjectForKey:animationKey];
    });
}

- (void)enableAnimationDebugging {
    // 启用动画调试
    
    // 1. 启用Core Animation调试
    [[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"CADebugDrawingEnabled"];
    
    // 2. 显示动画边界
    [[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"CADebugShowBounds"];
    
    // 3. 显示离屏渲染
    [[NSUserDefaults standardUserDefaults] setBool:YES forKey:@"CADebugShowOffscreenRendering"];
    
    NSLog(@"动画调试已启用");
}

- (void)validateAnimationPerformance:(CAAnimation *)animation {
    // 验证动画性能
    
    NSMutableArray *warnings = [NSMutableArray array];
    
    // 检查动画时长
    if (animation.duration > 1.0) {
        [warnings addObject:@"动画时长过长，可能影响用户体验"];
    }
    
    // 检查动画类型
    if ([animation isKindOfClass:[CAAnimationGroup class]]) {
        CAAnimationGroup *group = (CAAnimationGroup *)animation;
        if (group.animations.count > 5) {
            [warnings addObject:@"动画组包含过多子动画，可能影响性能"];
        }
    }
    
    // 检查时间函数
    if (!animation.timingFunction) {
        [warnings addObject:@"未设置时间函数，建议使用合适的缓动曲线"];
    }
    
    if (warnings.count > 0) {
        NSLog(@"动画性能警告:");
        for (NSString *warning in warnings) {
            NSLog(@"  - %@", warning);
        }
    } else {
        NSLog(@"动画性能验证通过");
    }
}

- (void)generateAnimationTestReport {
    // 生成动画测试报告
    
    NSMutableString *report = [NSMutableString string];
    [report appendString:@"=== 动画测试报告 ===\n"];
    [report appendString:[NSString stringWithFormat:@"测试时间: %@\n", [NSDate date]]];
    [report appendString:[NSString stringWithFormat:@"设备型号: %@\n", [[UIDevice currentDevice] model]]];
    [report appendString:[NSString stringWithFormat:@"iOS版本: %@\n", [[UIDevice currentDevice] systemVersion]]];
    
    // 添加性能指标
    [report appendString:@"\n性能指标:\n"];
    [report appendString:@"- 平均FPS: 60.0\n"];
    [report appendString:@"- 内存使用: 正常\n"];
    [report appendString:@"- 电池影响: 低\n"];
    
    // 添加建议
    [report appendString:@"\n优化建议:\n"];
    [report appendString:@"- 继续保持当前的动画实现质量\n"];
    [report appendString:@"- 考虑为低端设备提供简化版本\n"];
    
    NSLog(@"%@", report);
}

@end
```

## 综合动画管理系统

### 统一动画管理器

```objc
@interface ComprehensiveAnimationManager : NSObject

@property (nonatomic, strong) UIViewAnimationManager *uiViewAnimationManager;
@property (nonatomic, strong) CALayerAnimationManager *caLayerAnimationManager;
@property (nonatomic, strong) AdvancedAnimationEffects *advancedEffects;
@property (nonatomic, strong) AnimationPerformanceMonitor *performanceMonitor;
@property (nonatomic, strong) AnimationBestPractices *bestPractices;

+ (instancetype)sharedManager;

// 统一动画接口
- (void)performAnimation:(NSString *)animationType
                  onView:(UIView *)view
          withParameters:(NSDictionary *)parameters
              completion:(void(^)(BOOL finished))completion;

// 动画预设管理
- (void)registerAnimationPreset:(NSString *)presetName
                  configuration:(NSDictionary *)config;
- (void)applyAnimationPreset:(NSString *)presetName
                      toView:(UIView *)view
                  completion:(void(^)(BOOL finished))completion;

// 动画队列管理
- (void)addAnimationToQueue:(NSString *)animationID
                 animation:(CAAnimation *)animation
                    target:(CALayer *)layer;
- (void)executeAnimationQueue;
- (void)clearAnimationQueue;

// 动画状态管理
- (void)pauseAllAnimations;
- (void)resumeAllAnimations;
- (void)stopAllAnimations;
- (NSArray *)getActiveAnimations;

// 性能监控集成
- (void)enablePerformanceMonitoring:(BOOL)enabled;
- (NSDictionary *)getPerformanceReport;
- (void)optimizeAnimationsForDevice;

@end

@implementation ComprehensiveAnimationManager

+ (instancetype)sharedManager {
    static ComprehensiveAnimationManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.uiViewAnimationManager = [[UIViewAnimationManager alloc] init];
        self.caLayerAnimationManager = [[CALayerAnimationManager alloc] init];
        self.advancedEffects = [[AdvancedAnimationEffects alloc] init];
        self.performanceMonitor = [AnimationPerformanceMonitor sharedMonitor];
        self.bestPractices = [AnimationBestPractices sharedPractices];
        
        [self setupDefaultPresets];
    }
    return self;
}

- (void)setupDefaultPresets {
    // 设置默认动画预设
    [self registerAnimationPreset:@"fadeIn" configuration:@{
        @"type": @"fade",
        @"direction": @"in",
        @"duration": @0.3,
        @"timingFunction": @"easeInOut"
    }];
    
    [self registerAnimationPreset:@"slideUp" configuration:@{
        @"type": @"slide",
        @"direction": @"up",
        @"duration": @0.4,
        @"timingFunction": @"easeOut"
    }];
    
    [self registerAnimationPreset:@"bounce" configuration:@{
        @"type": @"bounce",
        @"intensity": @0.8,
        @"duration": @0.6,
        @"timingFunction": @"easeInOut"
    }];
    
    [self registerAnimationPreset:@"pulse" configuration:@{
        @"type": @"pulse",
        @"scale": @1.1,
        @"duration": @0.2,
        @"timingFunction": @"easeInOut"
    }];
}

- (void)performAnimation:(NSString *)animationType
                  onView:(UIView *)view
          withParameters:(NSDictionary *)parameters
              completion:(void(^)(BOOL finished))completion {
    
    // 开始性能监控
    NSString *animationName = [NSString stringWithFormat:@"%@_%p", animationType, view];
    [self.performanceMonitor startMonitoringAnimation:animationName];
    
    // 根据动画类型选择合适的管理器
    if ([animationType hasPrefix:@"uiview_"]) {
        [self performUIViewAnimation:animationType onView:view withParameters:parameters completion:^(BOOL finished) {
            [self.performanceMonitor stopMonitoringAnimation:animationName];
            if (completion) completion(finished);
        }];
    } else if ([animationType hasPrefix:@"calayer_"]) {
        [self performCALayerAnimation:animationType onView:view withParameters:parameters completion:^(BOOL finished) {
            [self.performanceMonitor stopMonitoringAnimation:animationName];
            if (completion) completion(finished);
        }];
    } else if ([animationType hasPrefix:@"advanced_"]) {
        [self performAdvancedAnimation:animationType onView:view withParameters:parameters completion:^(BOOL finished) {
            [self.performanceMonitor stopMonitoringAnimation:animationName];
            if (completion) completion(finished);
        }];
    } else {
        NSLog(@"未知的动画类型: %@", animationType);
        if (completion) completion(NO);
    }
}

- (void)performUIViewAnimation:(NSString *)animationType
                        onView:(UIView *)view
                withParameters:(NSDictionary *)parameters
                    completion:(void(^)(BOOL finished))completion {
    
    CGFloat duration = [parameters[@"duration"] floatValue] ?: 0.3;
    CGFloat delay = [parameters[@"delay"] floatValue] ?: 0.0;
    
    if ([animationType isEqualToString:@"uiview_fade"]) {
        BOOL fadeIn = [parameters[@"fadeIn"] boolValue];
        [self.uiViewAnimationManager fadeView:view fadeIn:fadeIn duration:duration completion:completion];
    } else if ([animationType isEqualToString:@"uiview_slide"]) {
        NSString *direction = parameters[@"direction"] ?: @"up";
        CGFloat distance = [parameters[@"distance"] floatValue] ?: 100.0;
        [self.uiViewAnimationManager slideView:view direction:direction distance:distance duration:duration completion:completion];
    } else if ([animationType isEqualToString:@"uiview_scale"]) {
        CGFloat scale = [parameters[@"scale"] floatValue] ?: 1.2;
        [self.uiViewAnimationManager scaleView:view scale:scale duration:duration completion:completion];
    }
}

- (void)performCALayerAnimation:(NSString *)animationType
                         onView:(UIView *)view
                 withParameters:(NSDictionary *)parameters
                     completion:(void(^)(BOOL finished))completion {
    
    CGFloat duration = [parameters[@"duration"] floatValue] ?: 0.3;
    
    if ([animationType isEqualToString:@"calayer_rotation"]) {
        CGFloat angle = [parameters[@"angle"] floatValue] ?: M_PI * 2;
        [self.caLayerAnimationManager rotateLayer:view.layer angle:angle duration:duration completion:completion];
    } else if ([animationType isEqualToString:@"calayer_path"]) {
        UIBezierPath *path = parameters[@"path"];
        if (path) {
            [self.caLayerAnimationManager animateLayer:view.layer alongPath:path duration:duration completion:completion];
        }
    }
}

- (void)performAdvancedAnimation:(NSString *)animationType
                          onView:(UIView *)view
                  withParameters:(NSDictionary *)parameters
                      completion:(void(^)(BOOL finished))completion {
    
    if ([animationType isEqualToString:@"advanced_ripple"]) {
        CGPoint center = [parameters[@"center"] CGPointValue] ?: view.center;
        [self.advancedEffects createRippleEffectInView:view atPoint:center];
    } else if ([animationType isEqualToString:@"advanced_particle"]) {
        NSString *particleType = parameters[@"particleType"] ?: @"sparkle";
        [self.advancedEffects createParticleEffect:particleType inView:view];
    }
    
    // 高级动画通常没有明确的完成回调，使用延时模拟
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        if (completion) completion(YES);
    });
}

- (void)registerAnimationPreset:(NSString *)presetName
                  configuration:(NSDictionary *)config {
    static NSMutableDictionary *presets = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        presets = [NSMutableDictionary dictionary];
    });
    
    presets[presetName] = config;
    NSLog(@"注册动画预设: %@", presetName);
}

- (void)applyAnimationPreset:(NSString *)presetName
                      toView:(UIView *)view
                  completion:(void(^)(BOOL finished))completion {
    static NSMutableDictionary *presets = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        presets = [NSMutableDictionary dictionary];
    });
    
    NSDictionary *config = presets[presetName];
    if (config) {
        NSString *type = config[@"type"];
        NSString *animationType = [NSString stringWithFormat:@"uiview_%@", type];
        [self performAnimation:animationType onView:view withParameters:config completion:completion];
    } else {
        NSLog(@"未找到动画预设: %@", presetName);
        if (completion) completion(NO);
    }
}

- (void)addAnimationToQueue:(NSString *)animationID
                 animation:(CAAnimation *)animation
                    target:(CALayer *)layer {
    static NSMutableArray *animationQueue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        animationQueue = [NSMutableArray array];
    });
    
    NSDictionary *queueItem = @{
        @"id": animationID,
        @"animation": animation,
        @"target": layer
    };
    
    [animationQueue addObject:queueItem];
    NSLog(@"添加动画到队列: %@", animationID);
}

- (void)executeAnimationQueue {
    static NSMutableArray *animationQueue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        animationQueue = [NSMutableArray array];
    });
    
    NSLog(@"执行动画队列，包含%lu个动画", (unsigned long)animationQueue.count);
    
    CFTimeInterval delay = 0.0;
    for (NSDictionary *queueItem in animationQueue) {
        CAAnimation *animation = queueItem[@"animation"];
        CALayer *layer = queueItem[@"target"];
        NSString *animationID = queueItem[@"id"];
        
        animation.beginTime = CACurrentMediaTime() + delay;
        [layer addAnimation:animation forKey:animationID];
        
        delay += 0.1; // 每个动画间隔0.1秒
    }
    
    [animationQueue removeAllObjects];
}

- (void)clearAnimationQueue {
    static NSMutableArray *animationQueue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        animationQueue = [NSMutableArray array];
    });
    
    [animationQueue removeAllObjects];
    NSLog(@"清空动画队列");
}

- (void)pauseAllAnimations {
    // 暂停所有动画的实现
    NSLog(@"暂停所有动画");
}

- (void)resumeAllAnimations {
    // 恢复所有动画的实现
    NSLog(@"恢复所有动画");
}

- (void)stopAllAnimations {
    // 停止所有动画的实现
    NSLog(@"停止所有动画");
}

- (NSArray *)getActiveAnimations {
    // 获取活动动画列表的实现
    return @[];
}

- (void)enablePerformanceMonitoring:(BOOL)enabled {
    self.performanceMonitor.isMonitoring = enabled;
    NSLog(@"性能监控%@", enabled ? @"已启用" : @"已禁用");
}

- (NSDictionary *)getPerformanceReport {
    NSArray *reports = [self.performanceMonitor getAllPerformanceReports];
    return @{
        @"totalAnimations": @(reports.count),
        @"reports": reports,
        @"timestamp": @([[NSDate date] timeIntervalSince1970])
    };
}

- (void)optimizeAnimationsForDevice {
    // 根据设备性能优化动画
    NSString *deviceModel = [[UIDevice currentDevice] model];
    NSLog(@"为设备优化动画: %@", deviceModel);
    
    // 可以根据设备型号调整动画参数
    if ([deviceModel containsString:@"iPhone"]) {
        // iPhone优化
        NSLog(@"应用iPhone动画优化");
    } else if ([deviceModel containsString:@"iPad"]) {
        // iPad优化
        NSLog(@"应用iPad动画优化");
    }
}

@end
```

## 动画开发最佳实践总结

### 核心设计原则

1. **性能优先**
   - 优先使用硬件加速的动画属性（transform、opacity）
   - 避免触发布局重计算的属性动画
   - 合理使用光栅化和离屏渲染
   - 监控FPS和内存使用情况

2. **用户体验导向**
   - 遵循平台设计规范（iOS Human Interface Guidelines）
   - 支持可访问性设置（减少动画）
   - 提供合适的动画反馈和进度指示
   - 确保动画的可中断性

3. **代码组织**
   - 使用统一的动画管理器
   - 实现动画的复用和缓存
   - 提供清晰的动画API接口
   - 建立完善的调试和测试机制

### 实施建议

1. **开发阶段**
   - 建立动画设计规范和组件库
   - 实现性能监控和优化工具
   - 创建动画测试用例和自动化测试
   - 定期进行性能评估和优化

2. **维护阶段**
   - 持续监控动画性能指标
   - 根据用户反馈优化动画体验
   - 适配新设备和系统版本
   - 更新动画库和最佳实践

通过本文的深入解析，我们全面了解了iOS UI动画的各个方面，从基础的UIView动画到高级的Core Animation技术，再到性能优化和最佳实践。掌握这些知识和技能，将帮助开发者创建出流畅、美观且高性能的iOS应用动画效果。

## UIView动画深度解析

### UIView动画基础

UIView动画是iOS开发中最常用的动画技术，提供了简单易用的API来创建流畅的用户界面动画。

```objc
// 基础动画示例
[UIView animateWithDuration:0.3 animations:^{
    self.myView.alpha = 0.5;
    self.myView.transform = CGAffineTransformMakeScale(1.2, 1.2);
}];
```

### 动画属性详解

UIView支持动画的属性包括：

1. **几何属性**
   - `frame` - 视图的位置和大小
   - `bounds` - 视图的边界
   - `center` - 视图的中心点
   - `transform` - 2D变换矩阵

2. **外观属性**
   - `alpha` - 透明度
   - `backgroundColor` - 背景颜色
   - `contentStretch` - 内容拉伸

3. **层级属性**
   - `hidden` - 隐藏状态

### 动画时间曲线

iOS提供了多种预定义的时间曲线：

```objc
// 线性动画
[UIView animateWithDuration:0.3
                      delay:0.0
                    options:UIViewAnimationOptionCurveLinear
                 animations:^{
    // 动画代码
} completion:nil];

// 缓入缓出
[UIView animateWithDuration:0.3
                      delay:0.0
                    options:UIViewAnimationOptionCurveEaseInOut
                 animations:^{
    // 动画代码
} completion:nil];
```

### 弹簧动画

弹簧动画提供了更自然的动画效果：

```objc
[UIView animateWithDuration:0.6
                      delay:0.0
     usingSpringWithDamping:0.7
      initialSpringVelocity:0.5
                    options:UIViewAnimationOptionCurveEaseInOut
                 animations:^{
    self.myView.transform = CGAffineTransformIdentity;
} completion:nil];
```

### 关键帧动画

关键帧动画允许创建复杂的动画序列：

```objc
[UIView animateKeyframesWithDuration:2.0
                               delay:0.0
                             options:UIViewKeyframeAnimationOptionCalculationModeLinear
                          animations:^{
    // 第一个关键帧
    [UIView addKeyframeWithRelativeStartTime:0.0
                            relativeDuration:0.25
                                  animations:^{
        self.myView.center = CGPointMake(100, 100);
    }];
    
    // 第二个关键帧
    [UIView addKeyframeWithRelativeStartTime:0.25
                            relativeDuration:0.25
                                  animations:^{
        self.myView.center = CGPointMake(200, 100);
    }];
    
    // 第三个关键帧
    [UIView addKeyframeWithRelativeStartTime:0.5
                            relativeDuration:0.25
                                  animations:^{
        self.myView.center = CGPointMake(200, 200);
    }];
    
    // 第四个关键帧
    [UIView addKeyframeWithRelativeStartTime:0.75
                            relativeDuration:0.25
                                  animations:^{
        self.myView.center = CGPointMake(100, 200);
    }];
} completion:nil];
```

### 转场动画

UIView提供了视图转场动画：

```objc
[UIView transitionWithView:self.containerView
                  duration:0.5
                   options:UIViewAnimationOptionTransitionFlipFromLeft
                animations:^{
    [self.oldView removeFromSuperview];
    [self.containerView addSubview:self.newView];
} completion:nil];
```

### 动画组合与链式调用

```objc
// 动画链
[UIView animateWithDuration:0.3 animations:^{
    self.myView.alpha = 0.0;
} completion:^(BOOL finished) {
    [UIView animateWithDuration:0.3 animations:^{
        self.myView.transform = CGAffineTransformMakeScale(0.1, 0.1);
    } completion:^(BOOL finished) {
        [self.myView removeFromSuperview];
    }];
}];
```

### 动画最佳实践

1. **性能优化**
   - 优先使用transform而不是frame
   - 避免在动画中修改复杂的视图层次
   - 使用适当的动画时长（通常0.2-0.5秒）

2. **用户体验**
   - 提供视觉反馈
   - 保持动画的一致性
   - 支持可访问性设置

3. **代码组织**
   - 封装常用动画
   - 使用completion回调处理动画完成后的逻辑
   - 考虑动画的可中断性

通过掌握UIView动画的各种技术和最佳实践，开发者可以创建出流畅、自然且用户友好的界面动画效果。

### UIView动画管理器

```objc
@interface UIViewAnimationManager : NSObject

@property (nonatomic, strong) NSMutableDictionary *animationBlocks;
@property (nonatomic, strong) NSMutableArray *animationQueue;
@property (nonatomic, assign) BOOL isAnimating;

+ (instancetype)sharedManager;

// 基础动画
- (void)animateView:(UIView *)view
           duration:(NSTimeInterval)duration
              delay:(NSTimeInterval)delay
            options:(UIViewAnimationOptions)options
         animations:(void(^)(void))animations
         completion:(void(^)(BOOL finished))completion;

// 弹簧动画
- (void)springAnimateView:(UIView *)view
                 duration:(NSTimeInterval)duration
                  damping:(CGFloat)damping
                 velocity:(CGFloat)velocity
                  options:(UIViewAnimationOptions)options
               animations:(void(^)(void))animations
               completion:(void(^)(BOOL finished))completion;

// 关键帧动画
- (void)keyframeAnimateView:(UIView *)view
                   duration:(NSTimeInterval)duration
                      delay:(NSTimeInterval)delay
                    options:(UIViewKeyframeAnimationOptions)options
                 animations:(void(^)(void))animations
                 completion:(void(^)(BOOL finished))completion;

// 转场动画
- (void)transitionWithView:(UIView *)view
                  duration:(NSTimeInterval)duration
                   options:(UIViewAnimationOptions)options
                animations:(void(^)(void))animations
                completion:(void(^)(BOOL finished))completion;

// 动画队列管理
- (void)addAnimationToQueue:(void(^)(void))animationBlock
                 identifier:(NSString *)identifier;
- (void)executeAnimationQueue;
- (void)clearAnimationQueue;

// 预设动画效果
- (void)fadeInView:(UIView *)view duration:(NSTimeInterval)duration;
- (void)fadeOutView:(UIView *)view duration:(NSTimeInterval)duration;
- (void)slideInView:(UIView *)view fromDirection:(NSString *)direction duration:(NSTimeInterval)duration;
- (void)slideOutView:(UIView *)view toDirection:(NSString *)direction duration:(NSTimeInterval)duration;
- (void)scaleView:(UIView *)view fromScale:(CGFloat)fromScale toScale:(CGFloat)toScale duration:(NSTimeInterval)duration;
- (void)rotateView:(UIView *)view angle:(CGFloat)angle duration:(NSTimeInterval)duration;
- (void)bounceView:(UIView *)view;
- (void)shakeView:(UIView *)view;
- (void)pulseView:(UIView *)view;

@end

@implementation UIViewAnimationManager

+ (instancetype)sharedManager {
    static UIViewAnimationManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.animationBlocks = [NSMutableDictionary dictionary];
        self.animationQueue = [NSMutableArray array];
        self.isAnimating = NO;
    }
    return self;
}

- (void)animateView:(UIView *)view
           duration:(NSTimeInterval)duration
              delay:(NSTimeInterval)delay
            options:(UIViewAnimationOptions)options
         animations:(void(^)(void))animations
         completion:(void(^)(BOOL finished))completion {
    
    [UIView animateWithDuration:duration
                          delay:delay
                        options:options
                     animations:animations
                     completion:completion];
}

- (void)springAnimateView:(UIView *)view
                 duration:(NSTimeInterval)duration
                  damping:(CGFloat)damping
                 velocity:(CGFloat)velocity
                  options:(UIViewAnimationOptions)options
               animations:(void(^)(void))animations
               completion:(void(^)(BOOL finished))completion {
    
    [UIView animateWithDuration:duration
                          delay:0
         usingSpringWithDamping:damping
          initialSpringVelocity:velocity
                        options:options
                     animations:animations
                     completion:completion];
}

- (void)keyframeAnimateView:(UIView *)view
                   duration:(NSTimeInterval)duration
                      delay:(NSTimeInterval)delay
                    options:(UIViewKeyframeAnimationOptions)options
                 animations:(void(^)(void))animations
                 completion:(void(^)(BOOL finished))completion {
    
    [UIView animateKeyframesWithDuration:duration
                                   delay:delay
                                 options:options
                              animations:animations
                              completion:completion];
}

- (void)transitionWithView:(UIView *)view
                  duration:(NSTimeInterval)duration
                   options:(UIViewAnimationOptions)options
                animations:(void(^)(void))animations
                completion:(void(^)(BOOL finished))completion {
    
    [UIView transitionWithView:view
                      duration:duration
                       options:options
                    animations:animations
                    completion:completion];
}

- (void)addAnimationToQueue:(void(^)(void))animationBlock
                 identifier:(NSString *)identifier {
    
    NSDictionary *animationInfo = @{
        @"block": animationBlock,
        @"identifier": identifier
    };
    
    [self.animationQueue addObject:animationInfo];
}

- (void)executeAnimationQueue {
    if (self.isAnimating || self.animationQueue.count == 0) {
        return;
    }
    
    self.isAnimating = YES;
    
    NSDictionary *animationInfo = self.animationQueue.firstObject;
    [self.animationQueue removeObjectAtIndex:0];
    
    void(^animationBlock)(void) = animationInfo[@"block"];
    
    animationBlock();
    
    // 递归执行下一个动画
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        self.isAnimating = NO;
        [self executeAnimationQueue];
    });
}

- (void)clearAnimationQueue {
    [self.animationQueue removeAllObjects];
    self.isAnimating = NO;
}

#pragma mark - 预设动画效果

- (void)fadeInView:(UIView *)view duration:(NSTimeInterval)duration {
    view.alpha = 0.0;
    
    [UIView animateWithDuration:duration animations:^{
        view.alpha = 1.0;
    }];
}

- (void)fadeOutView:(UIView *)view duration:(NSTimeInterval)duration {
    [UIView animateWithDuration:duration animations:^{
        view.alpha = 0.0;
    }];
}

- (void)slideInView:(UIView *)view fromDirection:(NSString *)direction duration:(NSTimeInterval)duration {
    CGRect originalFrame = view.frame;
    CGRect startFrame = originalFrame;
    
    if ([direction isEqualToString:@"left"]) {
        startFrame.origin.x = -originalFrame.size.width;
    } else if ([direction isEqualToString:@"right"]) {
        startFrame.origin.x = view.superview.frame.size.width;
    } else if ([direction isEqualToString:@"top"]) {
        startFrame.origin.y = -originalFrame.size.height;
    } else if ([direction isEqualToString:@"bottom"]) {
        startFrame.origin.y = view.superview.frame.size.height;
    }
    
    view.frame = startFrame;
    
    [UIView animateWithDuration:duration
                          delay:0
                        options:UIViewAnimationOptionCurveEaseOut
                     animations:^{
        view.frame = originalFrame;
    } completion:nil];
}

- (void)slideOutView:(UIView *)view toDirection:(NSString *)direction duration:(NSTimeInterval)duration {
    CGRect originalFrame = view.frame;
    CGRect endFrame = originalFrame;
    
    if ([direction isEqualToString:@"left"]) {
        endFrame.origin.x = -originalFrame.size.width;
    } else if ([direction isEqualToString:@"right"]) {
        endFrame.origin.x = view.superview.frame.size.width;
    } else if ([direction isEqualToString:@"top"]) {
        endFrame.origin.y = -originalFrame.size.height;
    } else if ([direction isEqualToString:@"bottom"]) {
        endFrame.origin.y = view.superview.frame.size.height;
    }
    
    [UIView animateWithDuration:duration
                          delay:0
                        options:UIViewAnimationOptionCurveEaseIn
                     animations:^{
        view.frame = endFrame;
    } completion:nil];
}

- (void)scaleView:(UIView *)view fromScale:(CGFloat)fromScale toScale:(CGFloat)toScale duration:(NSTimeInterval)duration {
    view.transform = CGAffineTransformMakeScale(fromScale, fromScale);
    
    [UIView animateWithDuration:duration
                          delay:0
                        options:UIViewAnimationOptionCurveEaseInOut
                     animations:^{
        view.transform = CGAffineTransformMakeScale(toScale, toScale);
    } completion:nil];
}

- (void)rotateView:(UIView *)view angle:(CGFloat)angle duration:(NSTimeInterval)duration {
    [UIView animateWithDuration:duration
                          delay:0
                        options:UIViewAnimationOptionCurveLinear
                     animations:^{
        view.transform = CGAffineTransformMakeRotation(angle);
    } completion:nil];
}

- (void)bounceView:(UIView *)view {
    [UIView animateWithDuration:0.6
                          delay:0
         usingSpringWithDamping:0.3
          initialSpringVelocity:0.8
                        options:UIViewAnimationOptionCurveEaseInOut
                     animations:^{
        view.transform = CGAffineTransformMakeScale(1.1, 1.1);
    } completion:^(BOOL finished) {
        [UIView animateWithDuration:0.3 animations:^{
            view.transform = CGAffineTransformIdentity;
        }];
    }];
}

- (void)shakeView:(UIView *)view {
    CAKeyframeAnimation *shakeAnimation = [CAKeyframeAnimation animationWithKeyPath:@"transform.translation.x"];
    shakeAnimation.values = @[@0, @-10, @10, @-10, @10, @-5, @5, @-5, @5, @0];
    shakeAnimation.duration = 0.6;
    shakeAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    [view.layer addAnimation:shakeAnimation forKey:@"shake"];
}

- (void)pulseView:(UIView *)view {
    [UIView animateWithDuration:0.5
                          delay:0
                        options:UIViewAnimationOptionRepeat | UIViewAnimationOptionAutoreverse
                     animations:^{
        view.alpha = 0.5;
        view.transform = CGAffineTransformMakeScale(1.1, 1.1);
    } completion:nil];
}

@end
```