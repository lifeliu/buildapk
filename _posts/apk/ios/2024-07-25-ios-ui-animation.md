---
layout: post
title: "iOS UI动画深度解析：Core Animation、UIView动画与高级动效实战指南"
date: 2024-07-25
categories: [iOS开发, UI动画, Core Animation]
tags: [iOS, Swift, Objective-C, Core Animation, UIView Animation, 动画效果, 性能优化]
author: "iOS技术专家"
description: "深入探讨iOS UI动画开发的核心技术，包括Core Animation框架、UIView动画系统、高级动效实现和性能优化策略，提供完整的动画开发解决方案。"
keywords: "iOS动画, Core Animation, UIView动画, CALayer, 动画性能, 动效设计, iOS开发"
---

# iOS UI动画深度解析：Core Animation、UIView动画与高级动效实战指南

在现代iOS应用开发中，优秀的UI动画不仅能提升用户体验，还能让应用更加生动和吸引人。本文将深入探讨iOS动画开发的核心技术，从基础的UIView动画到高级的Core Animation框架，为开发者提供全面的动画开发指南。

## 动画系统架构概览

### 动画架构管理器

```objc
@interface AnimationArchitecture : NSObject

@property (nonatomic, strong) NSMutableDictionary *animationManagers;
@property (nonatomic, strong) NSMutableDictionary *performanceMetrics;
@property (nonatomic, assign) BOOL isMonitoring;
@property (nonatomic, strong) NSTimer *monitoringTimer;

+ (instancetype)sharedArchitecture;

// 架构初始化
- (void)initializeArchitecture;
- (void)configureAnimationEnvironment;
- (void)setupPerformanceMonitoring;

// 动画管理器注册
- (void)registerAnimationManager:(id)manager forType:(NSString *)type;
- (id)getAnimationManagerForType:(NSString *)type;
- (void)unregisterAnimationManagerForType:(NSString *)type;

// 性能监控
- (void)startMonitoring;
- (void)stopMonitoring;
- (void)recordAnimationPerformance:(NSString *)animationType duration:(NSTimeInterval)duration frameRate:(CGFloat)frameRate;
- (NSDictionary *)getPerformanceReport;

// 动画优化
- (void)optimizeAnimationPerformance;
- (NSArray *)getOptimizationSuggestions;
- (void)applyOptimizationSettings;

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
        self.animationManagers = [NSMutableDictionary dictionary];
        self.performanceMetrics = [NSMutableDictionary dictionary];
        self.isMonitoring = NO;
    }
    return self;
}

- (void)initializeArchitecture {
    NSLog(@"=== 初始化动画架构 ===");
    
    // 配置动画环境
    [self configureAnimationEnvironment];
    
    // 设置性能监控
    [self setupPerformanceMonitoring];
    
    NSLog(@"动画架构初始化完成");
}

- (void)configureAnimationEnvironment {
    NSLog(@"配置动画环境...");
    
    // 设置全局动画参数
    [CATransaction setAnimationDuration:0.3];
    [CATransaction setAnimationTimingFunction:[CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut]];
    
    // 启用硬件加速
    [[UIApplication sharedApplication] setStatusBarHidden:NO withAnimation:UIStatusBarAnimationNone];
    
    NSLog(@"动画环境配置完成");
}

- (void)setupPerformanceMonitoring {
    NSLog(@"设置性能监控...");
    
    // 初始化性能指标字典
    self.performanceMetrics[@"frameRates"] = [NSMutableArray array];
    self.performanceMetrics[@"animationDurations"] = [NSMutableArray array];
    self.performanceMetrics[@"memoryUsage"] = [NSMutableArray array];
    
    NSLog(@"性能监控设置完成");
}

- (void)registerAnimationManager:(id)manager forType:(NSString *)type {
    NSLog(@"注册动画管理器: %@", type);
    self.animationManagers[type] = manager;
}

- (id)getAnimationManagerForType:(NSString *)type {
    return self.animationManagers[type];
}

- (void)unregisterAnimationManagerForType:(NSString *)type {
    NSLog(@"注销动画管理器: %@", type);
    [self.animationManagers removeObjectForKey:type];
}

- (void)startMonitoring {
    if (self.isMonitoring) {
        return;
    }
    
    NSLog(@"开始性能监控...");
    self.isMonitoring = YES;
    
    // 启动监控定时器
    self.monitoringTimer = [NSTimer scheduledTimerWithTimeInterval:1.0
                                                            target:self
                                                          selector:@selector(collectPerformanceMetrics)
                                                          userInfo:nil
                                                           repeats:YES];
}

- (void)stopMonitoring {
    if (!self.isMonitoring) {
        return;
    }
    
    NSLog(@"停止性能监控...");
    self.isMonitoring = NO;
    
    [self.monitoringTimer invalidate];
    self.monitoringTimer = nil;
}

- (void)collectPerformanceMetrics {
    // 收集当前帧率
    CGFloat currentFrameRate = [self getCurrentFrameRate];
    NSMutableArray *frameRates = self.performanceMetrics[@"frameRates"];
    [frameRates addObject:@(currentFrameRate)];
    
    // 收集内存使用情况
    NSInteger memoryUsage = [self getCurrentMemoryUsage];
    NSMutableArray *memoryUsages = self.performanceMetrics[@"memoryUsage"];
    [memoryUsages addObject:@(memoryUsage)];
    
    // 保持数据量在合理范围内
    if (frameRates.count > 100) {
        [frameRates removeObjectAtIndex:0];
    }
    if (memoryUsages.count > 100) {
        [memoryUsages removeObjectAtIndex:0];
    }
}

- (CGFloat)getCurrentFrameRate {
    // 简化的帧率计算，实际应用中需要更精确的实现
    return 60.0; // 假设60fps
}

- (NSInteger)getCurrentMemoryUsage {
    // 获取当前内存使用情况
    struct mach_task_basic_info info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
    
    if (kerr == KERN_SUCCESS) {
        return info.resident_size;
    }
    return 0;
}

- (void)recordAnimationPerformance:(NSString *)animationType duration:(NSTimeInterval)duration frameRate:(CGFloat)frameRate {
    NSLog(@"记录动画性能: %@ - 时长:%.3f秒, 帧率:%.1ffps", animationType, duration, frameRate);
    
    NSMutableDictionary *animationMetrics = self.performanceMetrics[animationType];
    if (!animationMetrics) {
        animationMetrics = [NSMutableDictionary dictionary];
        animationMetrics[@"durations"] = [NSMutableArray array];
        animationMetrics[@"frameRates"] = [NSMutableArray array];
        self.performanceMetrics[animationType] = animationMetrics;
    }
    
    [animationMetrics[@"durations"] addObject:@(duration)];
    [animationMetrics[@"frameRates"] addObject:@(frameRate)];
}

- (NSDictionary *)getPerformanceReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    for (NSString *animationType in self.performanceMetrics) {
        if ([animationType isEqualToString:@"frameRates"] || [animationType isEqualToString:@"animationDurations"] || [animationType isEqualToString:@"memoryUsage"]) {
            continue;
        }
        
        NSDictionary *metrics = self.performanceMetrics[animationType];
        NSArray *durations = metrics[@"durations"];
        NSArray *frameRates = metrics[@"frameRates"];
        
        if (durations.count > 0) {
            double totalDuration = 0;
            double totalFrameRate = 0;
            
            for (NSNumber *duration in durations) {
                totalDuration += [duration doubleValue];
            }
            
            for (NSNumber *frameRate in frameRates) {
                totalFrameRate += [frameRate doubleValue];
            }
            
            report[animationType] = @{
                @"averageDuration": @(totalDuration / durations.count),
                @"averageFrameRate": @(totalFrameRate / frameRates.count),
                @"totalAnimations": @(durations.count)
            };
        }
    }
    
    return report;
}

- (void)optimizeAnimationPerformance {
    NSLog(@"优化动画性能...");
    
    // 分析性能报告
    NSDictionary *report = [self getPerformanceReport];
    
    for (NSString *animationType in report) {
        NSDictionary *metrics = report[animationType];
        double averageFrameRate = [metrics[@"averageFrameRate"] doubleValue];
        
        if (averageFrameRate < 30.0) {
            NSLog(@"动画类型 %@ 帧率过低: %.1ffps，需要优化", animationType, averageFrameRate);
            // 应用优化策略
        }
    }
    
    NSLog(@"动画性能优化完成");
}

- (NSArray *)getOptimizationSuggestions {
    NSMutableArray *suggestions = [NSMutableArray array];
    NSDictionary *report = [self getPerformanceReport];
    
    for (NSString *animationType in report) {
        NSDictionary *metrics = report[animationType];
        double averageFrameRate = [metrics[@"averageFrameRate"] doubleValue];
        double averageDuration = [metrics[@"averageDuration"] doubleValue];
        
        if (averageFrameRate < 30.0) {
            [suggestions addObject:[NSString stringWithFormat:@"%@ 动画帧率过低，建议减少动画复杂度或使用硬件加速", animationType]];
        }
        
        if (averageDuration > 1.0) {
            [suggestions addObject:[NSString stringWithFormat:@"%@ 动画时长过长，建议缩短动画时间或分解为多个短动画", animationType]];
        }
    }
    
    return suggestions;
}

- (void)applyOptimizationSettings {
    NSLog(@"应用优化设置...");
    
    // 启用图层光栅化
    [CATransaction begin];
    [CATransaction setDisableActions:YES];
    
    // 设置全局动画优化参数
    [CATransaction setAnimationDuration:0.25]; // 缩短默认动画时长
    
    [CATransaction commit];
    
    NSLog(@"优化设置应用完成");
}

- (void)dealloc {
    [self stopMonitoring];
}

@end
```

## Core Animation深度解析

### Core Animation管理器

```objc
@interface CoreAnimationManager : NSObject

@property (nonatomic, weak) AnimationArchitecture *architecture;
@property (nonatomic, strong) NSMutableDictionary *activeAnimations;
@property (nonatomic, strong) NSMutableDictionary *animationGroups;
@property (nonatomic, assign) NSInteger animationCounter;

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

// 转场动画
- (CATransition *)createTransitionWithType:(NSString *)type
                                   subtype:(NSString *)subtype
                                  duration:(CFTimeInterval)duration;

// 弹簧动画
- (CASpringAnimation *)createSpringAnimationWithKeyPath:(NSString *)keyPath
                                              fromValue:(id)fromValue
                                                toValue:(id)toValue
                                                damping:(CGFloat)damping
                                              stiffness:(CGFloat)stiffness
                                                   mass:(CGFloat)mass;

// 动画执行
- (void)addAnimation:(CAAnimation *)animation
             toLayer:(CALayer *)layer
              forKey:(NSString *)key
          completion:(void(^)(BOOL finished))completion;

// 动画控制
- (void)pauseAnimationForLayer:(CALayer *)layer key:(NSString *)key;
- (void)resumeAnimationForLayer:(CALayer *)layer key:(NSString *)key;
- (void)removeAnimationForLayer:(CALayer *)layer key:(NSString *)key;
- (void)removeAllAnimationsForLayer:(CALayer *)layer;

// 动画查询
- (BOOL)isAnimatingLayer:(CALayer *)layer;
- (NSArray *)getAnimationKeysForLayer:(CALayer *)layer;
- (CAAnimation *)getAnimationForLayer:(CALayer *)layer key:(NSString *)key;

@end

@implementation CoreAnimationManager

+ (instancetype)sharedManager {
    static CoreAnimationManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.architecture = [AnimationArchitecture sharedArchitecture];
        self.activeAnimations = [NSMutableDictionary dictionary];
        self.animationGroups = [NSMutableDictionary dictionary];
        self.animationCounter = 0;
        
        // 注册到架构管理器
        [self.architecture registerAnimationManager:self forType:@"CoreAnimation"];
    }
    return self;
}

- (CABasicAnimation *)createBasicAnimationWithKeyPath:(NSString *)keyPath
                                            fromValue:(id)fromValue
                                              toValue:(id)toValue
                                             duration:(CFTimeInterval)duration {
    
    NSLog(@"创建基础动画: %@, 时长: %.3f秒", keyPath, duration);
    
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:keyPath];
    animation.fromValue = fromValue;
    animation.toValue = toValue;
    animation.duration = duration;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    
    return animation;
}

- (CAKeyframeAnimation *)createKeyframeAnimationWithKeyPath:(NSString *)keyPath
                                                     values:(NSArray *)values
                                                   keyTimes:(NSArray *)keyTimes
                                                   duration:(CFTimeInterval)duration {
    
    NSLog(@"创建关键帧动画: %@, 关键帧数: %lu, 时长: %.3f秒", keyPath, (unsigned long)values.count, duration);
    
    CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:keyPath];
    animation.values = values;
    animation.keyTimes = keyTimes;
    animation.duration = duration;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    
    return animation;
}

- (CAAnimationGroup *)createAnimationGroupWithAnimations:(NSArray *)animations
                                                 duration:(CFTimeInterval)duration {
    
    NSLog(@"创建动画组: %lu个动画, 时长: %.3f秒", (unsigned long)animations.count, duration);
    
    CAAnimationGroup *group = [CAAnimationGroup animation];
    group.animations = animations;
    group.duration = duration;
    group.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    group.fillMode = kCAFillModeForwards;
    group.removedOnCompletion = NO;
    
    return group;
}

- (CATransition *)createTransitionWithType:(NSString *)type
                                   subtype:(NSString *)subtype
                                  duration:(CFTimeInterval)duration {
    
    NSLog(@"创建转场动画: %@-%@, 时长: %.3f秒", type, subtype, duration);
    
    CATransition *transition = [CATransition animation];
    transition.type = type;
    transition.subtype = subtype;
    transition.duration = duration;
    transition.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    return transition;
}

- (CASpringAnimation *)createSpringAnimationWithKeyPath:(NSString *)keyPath
                                              fromValue:(id)fromValue
                                                toValue:(id)toValue
                                                damping:(CGFloat)damping
                                              stiffness:(CGFloat)stiffness
                                                   mass:(CGFloat)mass {
    
    NSLog(@"创建弹簧动画: %@, 阻尼: %.2f, 刚度: %.2f, 质量: %.2f", keyPath, damping, stiffness, mass);
    
    CASpringAnimation *animation = [CASpringAnimation animationWithKeyPath:keyPath];
    animation.fromValue = fromValue;
    animation.toValue = toValue;
    animation.damping = damping;
    animation.stiffness = stiffness;
    animation.mass = mass;
    animation.duration = animation.settlingDuration;
    animation.fillMode = kCAFillModeForwards;
    animation.removedOnCompletion = NO;
    
    return animation;
}

- (void)addAnimation:(CAAnimation *)animation
             toLayer:(CALayer *)layer
              forKey:(NSString *)key
          completion:(void(^)(BOOL finished))completion {
    
    NSString *animationId = [self generateAnimationId];
    NSDate *startTime = [NSDate date];
    
    NSLog(@"添加动画到图层: %@, 键: %@", animationId, key);
    
    // 设置动画代理
    animation.delegate = self;
    
    // 记录动画信息
    [self recordAnimationStart:animationId layer:layer animation:animation key:key completion:completion];
    
    // 添加动画到图层
    [layer addAnimation:animation forKey:key];
}

- (void)pauseAnimationForLayer:(CALayer *)layer key:(NSString *)key {
    NSLog(@"暂停图层动画: %@", key);
    
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
    layer.speed = 0.0;
    layer.timeOffset = pausedTime;
}

- (void)resumeAnimationForLayer:(CALayer *)layer key:(NSString *)key {
    NSLog(@"恢复图层动画: %@", key);
    
    CFTimeInterval pausedTime = [layer timeOffset];
    layer.speed = 1.0;
    layer.timeOffset = 0.0;
    layer.beginTime = 0.0;
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    layer.beginTime = timeSincePause;
}

- (void)removeAnimationForLayer:(CALayer *)layer key:(NSString *)key {
    NSLog(@"移除图层动画: %@", key);
    
    [layer removeAnimationForKey:key];
    
    // 清理动画记录
    NSArray *keysToRemove = [NSArray array];
    for (NSString *animationId in self.activeAnimations) {
        NSDictionary *animationInfo = self.activeAnimations[animationId];
        if (animationInfo[@"layer"] == layer && [animationInfo[@"key"] isEqualToString:key]) {
            keysToRemove = [keysToRemove arrayByAddingObject:animationId];
        }
    }
    
    for (NSString *animationId in keysToRemove) {
        [self.activeAnimations removeObjectForKey:animationId];
    }
}

- (void)removeAllAnimationsForLayer:(CALayer *)layer {
    NSLog(@"移除图层所有动画");
    
    [layer removeAllAnimations];
    
    // 清理相关的动画记录
    NSArray *keysToRemove = [NSArray array];
    for (NSString *animationId in self.activeAnimations) {
        NSDictionary *animationInfo = self.activeAnimations[animationId];
        if (animationInfo[@"layer"] == layer) {
            keysToRemove = [keysToRemove arrayByAddingObject:animationId];
        }
    }
    
    for (NSString *key in keysToRemove) {
        [self.activeAnimations removeObjectForKey:key];
    }
}

- (BOOL)isAnimatingLayer:(CALayer *)layer {
    return layer.animationKeys.count > 0;
}

- (NSArray *)getAnimationKeysForLayer:(CALayer *)layer {
    return layer.animationKeys;
}

- (CAAnimation *)getAnimationForLayer:(CALayer *)layer key:(NSString *)key {
    return [layer animationForKey:key];
}

#pragma mark - CAAnimationDelegate

- (void)animationDidStart:(CAAnimation *)anim {
    NSLog(@"Core Animation开始");
}

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag {
    NSLog(@"Core Animation结束: %@", flag ? @"完成" : @"中断");
    
    // 查找对应的动画记录
    NSString *animationIdToRemove = nil;
    void(^completion)(BOOL) = nil;
    
    for (NSString *animationId in self.activeAnimations) {
        NSDictionary *animationInfo = self.activeAnimations[animationId];
        if (animationInfo[@"animation"] == anim) {
            animationIdToRemove = animationId;
            completion = animationInfo[@"completion"];
            
            // 记录性能指标
            NSDate *startTime = animationInfo[@"startTime"];
            NSTimeInterval actualDuration = [[NSDate date] timeIntervalSinceDate:startTime];
            [self.architecture recordAnimationPerformance:@"CoreAnimation"
                                                  duration:actualDuration
                                                 frameRate:60.0];
            break;
        }
    }
    
    // 清理动画记录
    if (animationIdToRemove) {
        [self.activeAnimations removeObjectForKey:animationIdToRemove];
    }
    
    // 执行完成回调
    if (completion) {
        completion(flag);
    }
}

- (NSString *)generateAnimationId {
    self.animationCounter++;
    return [NSString stringWithFormat:@"CoreAnim_%ld", (long)self.animationCounter];
}

- (void)recordAnimationStart:(NSString *)animationId
                       layer:(CALayer *)layer
                   animation:(CAAnimation *)animation
                         key:(NSString *)key
                  completion:(void(^)(BOOL finished))completion {
    
    self.activeAnimations[animationId] = @{
        @"layer": layer,
        @"animation": animation,
        @"key": key ?: @"",
        @"startTime": [NSDate date],
        @"completion": completion ?: ^(BOOL finished) {},
        @"type": @"CoreAnimation"
    };
}

@end
```

### 高级动画效果管理器

```objc
@interface AdvancedAnimationManager : NSObject

@property (nonatomic, weak) AnimationArchitecture *architecture;
@property (nonatomic, strong) CoreAnimationManager *coreAnimationManager;
@property (nonatomic, strong) UIViewAnimationManager *uiViewAnimationManager;

+ (instancetype)sharedManager;

// 复杂动画效果
- (void)createParticleAnimationForLayer:(CALayer *)layer
                           particleType:(NSString *)particleType
                               duration:(NSTimeInterval)duration
                             completion:(void(^)(BOOL finished))completion;

- (void)createMorphingAnimationForLayer:(CALayer *)layer
                              fromPath:(UIBezierPath *)fromPath
                                toPath:(UIBezierPath *)toPath
                              duration:(NSTimeInterval)duration
                            completion:(void(^)(BOOL finished))completion;

- (void)createRippleAnimationForView:(UIView *)view
                          startPoint:(CGPoint)startPoint
                               color:(UIColor *)color
                            duration:(NSTimeInterval)duration
                          completion:(void(^)(BOOL finished))completion;

- (void)createShimmerAnimationForView:(UIView *)view
                             duration:(NSTimeInterval)duration
                           completion:(void(^)(BOOL finished))completion;

// 3D动画效果
- (void)create3DFlipAnimationForView:(UIView *)view
                                axis:(NSString *)axis
                            duration:(NSTimeInterval)duration
                          completion:(void(^)(BOOL finished))completion;

- (void)create3DCubeTransitionFromView:(UIView *)fromView
                                toView:(UIView *)toView
                             direction:(NSString *)direction
                              duration:(NSTimeInterval)duration
                            completion:(void(^)(BOOL finished))completion;

// 物理动画效果
- (void)createGravityAnimationForView:(UIView *)view
                             duration:(NSTimeInterval)duration
                           completion:(void(^)(BOOL finished))completion;

- (void)createCollisionAnimationForViews:(NSArray *)views
                                duration:(NSTimeInterval)duration
                              completion:(void(^)(BOOL finished))completion;

// 动画序列
- (void)createAnimationSequence:(NSArray *)animationBlocks
                       duration:(NSTimeInterval)totalDuration
                     completion:(void(^)(BOOL finished))completion;

- (void)createParallelAnimations:(NSArray *)animationBlocks
                        duration:(NSTimeInterval)duration
                      completion:(void(^)(BOOL finished))completion;

@end

@implementation AdvancedAnimationManager

+ (instancetype)sharedManager {
    static AdvancedAnimationManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.architecture = [AnimationArchitecture sharedArchitecture];
        self.coreAnimationManager = [CoreAnimationManager sharedManager];
        self.uiViewAnimationManager = [UIViewAnimationManager sharedManager];
        
        // 注册到架构管理器
        [self.architecture registerAnimationManager:self forType:@"AdvancedAnimation"];
    }
    return self;
}

- (void)createParticleAnimationForLayer:(CALayer *)layer
                           particleType:(NSString *)particleType
                               duration:(NSTimeInterval)duration
                             completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建粒子动画: %@, 时长: %.3f秒", particleType, duration);
    
    // 创建粒子发射器
    CAEmitterLayer *emitterLayer = [CAEmitterLayer layer];
    emitterLayer.emitterPosition = CGPointMake(layer.bounds.size.width / 2, layer.bounds.size.height / 2);
    emitterLayer.emitterSize = layer.bounds.size;
    emitterLayer.emitterShape = kCAEmitterLayerRectangle;
    emitterLayer.emitterMode = kCAEmitterLayerOutline;
    
    // 创建粒子单元
    CAEmitterCell *cell = [CAEmitterCell emitterCell];
    cell.birthRate = 100;
    cell.lifetime = duration;
    cell.velocity = 50;
    cell.velocityRange = 20;
    cell.emissionRange = M_PI * 2;
    cell.scale = 0.1;
    cell.scaleRange = 0.05;
    
    // 根据粒子类型设置不同的效果
    if ([particleType isEqualToString:@"fire"]) {
        cell.color = [UIColor orangeColor].CGColor;
        cell.redRange = 0.5;
        cell.greenRange = 0.2;
        cell.alphaSpeed = -1.0 / duration;
    } else if ([particleType isEqualToString:@"snow"]) {
        cell.color = [UIColor whiteColor].CGColor;
        cell.velocity = 20;
        cell.yAcceleration = 50;
    } else if ([particleType isEqualToString:@"sparkle"]) {
        cell.color = [UIColor yellowColor].CGColor;
        cell.scale = 0.05;
        cell.scaleSpeed = 0.1;
        cell.alphaSpeed = -2.0 / duration;
    }
    
    // 创建粒子图像
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(10, 10), NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
    CGContextFillEllipseInRect(context, CGRectMake(0, 0, 10, 10));
    UIImage *particleImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    cell.contents = (id)particleImage.CGImage;
    emitterLayer.emitterCells = @[cell];
    
    [layer addSublayer:emitterLayer];
    
    // 设置动画结束后的清理
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [emitterLayer removeFromSuperlayer];
        if (completion) {
            completion(YES);
        }
    });
}

- (void)createMorphingAnimationForLayer:(CALayer *)layer
                              fromPath:(UIBezierPath *)fromPath
                                toPath:(UIBezierPath *)toPath
                              duration:(NSTimeInterval)duration
                            completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建形变动画, 时长: %.3f秒", duration);
    
    if (![layer isKindOfClass:[CAShapeLayer class]]) {
        NSLog(@"警告: 形变动画需要CAShapeLayer");
        if (completion) completion(NO);
        return;
    }
    
    CAShapeLayer *shapeLayer = (CAShapeLayer *)layer;
    
    // 创建路径动画
    CABasicAnimation *pathAnimation = [self.coreAnimationManager createBasicAnimationWithKeyPath:@"path"
                                                                                        fromValue:(id)fromPath.CGPath
                                                                                          toValue:(id)toPath.CGPath
                                                                                         duration:duration];
    
    [self.coreAnimationManager addAnimation:pathAnimation
                                     toLayer:shapeLayer
                                      forKey:@"morphing"
                                  completion:completion];
}

- (void)createRippleAnimationForView:(UIView *)view
                          startPoint:(CGPoint)startPoint
                               color:(UIColor *)color
                            duration:(NSTimeInterval)duration
                          completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建涟漪动画, 起点: (%.1f, %.1f), 时长: %.3f秒", startPoint.x, startPoint.y, duration);
    
    // 创建涟漪图层
    CAShapeLayer *rippleLayer = [CAShapeLayer layer];
    rippleLayer.frame = view.bounds;
    rippleLayer.fillColor = [UIColor clearColor].CGColor;
    rippleLayer.strokeColor = color.CGColor;
    rippleLayer.lineWidth = 2.0;
    
    // 创建圆形路径
    CGFloat maxRadius = MAX(view.bounds.size.width, view.bounds.size.height);
    UIBezierPath *startPath = [UIBezierPath bezierPathWithArcCenter:startPoint radius:0 startAngle:0 endAngle:2*M_PI clockwise:YES];
    UIBezierPath *endPath = [UIBezierPath bezierPathWithArcCenter:startPoint radius:maxRadius startAngle:0 endAngle:2*M_PI clockwise:YES];
    
    rippleLayer.path = startPath.CGPath;
    [view.layer addSublayer:rippleLayer];
    
    // 创建扩散动画
    CABasicAnimation *pathAnimation = [self.coreAnimationManager createBasicAnimationWithKeyPath:@"path"
                                                                                        fromValue:(id)startPath.CGPath
                                                                                          toValue:(id)endPath.CGPath
                                                                                         duration:duration];
    
    // 创建透明度动画
    CABasicAnimation *opacityAnimation = [self.coreAnimationManager createBasicAnimationWithKeyPath:@"opacity"
                                                                                            fromValue:@1.0
                                                                                              toValue:@0.0
                                                                                             duration:duration];
    
    // 创建动画组
    CAAnimationGroup *group = [self.coreAnimationManager createAnimationGroupWithAnimations:@[pathAnimation, opacityAnimation]
                                                                                     duration:duration];
    
    [self.coreAnimationManager addAnimation:group
                                     toLayer:rippleLayer
                                      forKey:@"ripple"
                                  completion:^(BOOL finished) {
                                      [rippleLayer removeFromSuperlayer];
                                      if (completion) {
                                          completion(finished);
                                      }
                                  }];
}

- (void)createShimmerAnimationForView:(UIView *)view
                             duration:(NSTimeInterval)duration
                           completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建闪光动画, 时长: %.3f秒", duration);
    
    // 创建渐变图层
    CAGradientLayer *gradientLayer = [CAGradientLayer layer];
    gradientLayer.frame = view.bounds;
    gradientLayer.colors = @[
        (id)[UIColor clearColor].CGColor,
        (id)[UIColor whiteColor].CGColor,
        (id)[UIColor clearColor].CGColor
    ];
    gradientLayer.locations = @[@0.0, @0.5, @1.0];
    gradientLayer.startPoint = CGPointMake(0, 0.5);
    gradientLayer.endPoint = CGPointMake(1, 0.5);
    
    // 设置遮罩
    view.layer.mask = gradientLayer;
    
    // 创建位置动画
    CABasicAnimation *animation = [self.coreAnimationManager createBasicAnimationWithKeyPath:@"locations"
                                                                                    fromValue:@[@(-0.5), @0.0, @0.5]
                                                                                      toValue:@[@0.5, @1.0, @1.5]
                                                                                     duration:duration];
    animation.repeatCount = INFINITY;
    
    [self.coreAnimationManager addAnimation:animation
                                     toLayer:gradientLayer
                                      forKey:@"shimmer"
                                  completion:completion];
}

- (void)create3DFlipAnimationForView:(UIView *)view
                                axis:(NSString *)axis
                            duration:(NSTimeInterval)duration
                          completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建3D翻转动画, 轴: %@, 时长: %.3f秒", axis, duration);
    
    // 设置3D变换
    CATransform3D transform = CATransform3DIdentity;
    transform.m34 = -1.0 / 1000.0; // 透视效果
    
    if ([axis isEqualToString:@"x"]) {
        transform = CATransform3DRotate(transform, M_PI, 1, 0, 0);
    } else if ([axis isEqualToString:@"y"]) {
        transform = CATransform3DRotate(transform, M_PI, 0, 1, 0);
    } else if ([axis isEqualToString:@"z"]) {
        transform = CATransform3DRotate(transform, M_PI, 0, 0, 1);
    }
    
    // 创建3D变换动画
    CABasicAnimation *flipAnimation = [self.coreAnimationManager createBasicAnimationWithKeyPath:@"transform"
                                                                                        fromValue:[NSValue valueWithCATransform3D:CATransform3DIdentity]
                                                                                          toValue:[NSValue valueWithCATransform3D:transform]
                                                                                         duration:duration];
    
    [self.coreAnimationManager addAnimation:flipAnimation
                                     toLayer:view.layer
                                      forKey:@"3dFlip"
                                  completion:completion];
}

- (void)create3DCubeTransitionFromView:(UIView *)fromView
                                toView:(UIView *)toView
                             direction:(NSString *)direction
                              duration:(NSTimeInterval)duration
                            completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建3D立方体转场, 方向: %@, 时长: %.3f秒", direction, duration);
    
    // 创建容器视图
    UIView *containerView = fromView.superview;
    
    // 设置3D透视
    CATransform3D perspective = CATransform3DIdentity;
    perspective.m34 = -1.0 / 1000.0;
    containerView.layer.sublayerTransform = perspective;
    
    // 设置初始位置
    fromView.layer.anchorPoint = CGPointMake(0.5, 0.5);
    toView.layer.anchorPoint = CGPointMake(0.5, 0.5);
    
    // 根据方向设置变换
    CATransform3D fromTransform, toTransform;
    
    if ([direction isEqualToString:@"left"]) {
        fromTransform = CATransform3DRotate(CATransform3DIdentity, -M_PI_2, 0, 1, 0);
        toTransform = CATransform3DRotate(CATransform3DIdentity, M_PI_2, 0, 1, 0);
        toView.layer.transform = toTransform;
    } else if ([direction isEqualToString:@"right"]) {
        fromTransform = CATransform3DRotate(CATransform3DIdentity, M_PI_2, 0, 1, 0);
        toTransform = CATransform3DRotate(CATransform3DIdentity, -M_PI_2, 0, 1, 0);
        toView.layer.transform = toTransform;
    } else if ([direction isEqualToString:@"up"]) {
        fromTransform = CATransform3DRotate(CATransform3DIdentity, M_PI_2, 1, 0, 0);
        toTransform = CATransform3DRotate(CATransform3DIdentity, -M_PI_2, 1, 0, 0);
        toView.layer.transform = toTransform;
    } else { // down
        fromTransform = CATransform3DRotate(CATransform3DIdentity, -M_PI_2, 1, 0, 0);
        toTransform = CATransform3DRotate(CATransform3DIdentity, M_PI_2, 1, 0, 0);
        toView.layer.transform = toTransform;
    }
    
    [containerView addSubview:toView];
    
    // 执行动画
    [self.uiViewAnimationManager animateView:containerView
                                     duration:duration
                                        delay:0
                                      options:UIViewAnimationOptionCurveEaseInOut
                                   animations:^{
                                       fromView.layer.transform = fromTransform;
                                       toView.layer.transform = CATransform3DIdentity;
                                   }
                                   completion:^(BOOL finished) {
                                       [fromView removeFromSuperview];
                                       containerView.layer.sublayerTransform = CATransform3DIdentity;
                                       if (completion) {
                                           completion(finished);
                                       }
                                   }];
}

- (void)createGravityAnimationForView:(UIView *)view
                             duration:(NSTimeInterval)duration
                           completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建重力动画, 时长: %.3f秒", duration);
    
    // 创建物理动画器
    UIDynamicAnimator *animator = [[UIDynamicAnimator alloc] initWithReferenceView:view.superview];
    
    // 创建重力行为
    UIGravityBehavior *gravity = [[UIGravityBehavior alloc] initWithItems:@[view]];
    gravity.gravityDirection = CGVectorMake(0, 1.0);
    
    // 创建碰撞行为
    UICollisionBehavior *collision = [[UICollisionBehavior alloc] initWithItems:@[view]];
    collision.translatesReferenceBoundsIntoBoundary = YES;
    
    // 创建动态属性
    UIDynamicItemBehavior *itemBehavior = [[UIDynamicItemBehavior alloc] initWithItems:@[view]];
    itemBehavior.elasticity = 0.6;
    itemBehavior.friction = 0.8;
    
    [animator addBehavior:gravity];
    [animator addBehavior:collision];
    [animator addBehavior:itemBehavior];
    
    // 设置结束定时器
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [animator removeAllBehaviors];
        if (completion) {
            completion(YES);
        }
    });
}

- (void)createCollisionAnimationForViews:(NSArray *)views
                                duration:(NSTimeInterval)duration
                              completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建碰撞动画, 视图数量: %lu, 时长: %.3f秒", (unsigned long)views.count, duration);
    
    if (views.count == 0) {
        if (completion) completion(NO);
        return;
    }
    
    UIView *referenceView = [(UIView *)views.firstObject superview];
    UIDynamicAnimator *animator = [[UIDynamicAnimator alloc] initWithReferenceView:referenceView];
    
    // 为每个视图添加随机推力
    for (UIView *view in views) {
        UIPushBehavior *push = [[UIPushBehavior alloc] initWithItems:@[view] mode:UIPushBehaviorModeInstantaneous];
        push.pushDirection = CGVectorMake((arc4random_uniform(200) - 100) / 100.0, (arc4random_uniform(200) - 100) / 100.0);
        push.magnitude = 0.5;
        [animator addBehavior:push];
    }
    
    // 创建碰撞行为
    UICollisionBehavior *collision = [[UICollisionBehavior alloc] initWithItems:views];
    collision.translatesReferenceBoundsIntoBoundary = YES;
    
    // 创建动态属性
    UIDynamicItemBehavior *itemBehavior = [[UIDynamicItemBehavior alloc] initWithItems:views];
    itemBehavior.elasticity = 0.8;
    itemBehavior.friction = 0.5;
    
    [animator addBehavior:collision];
    [animator addBehavior:itemBehavior];
    
    // 设置结束定时器
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [animator removeAllBehaviors];
        if (completion) {
            completion(YES);
        }
    });
}

- (void)createAnimationSequence:(NSArray *)animationBlocks
                       duration:(NSTimeInterval)totalDuration
                     completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建动画序列, 动画数量: %lu, 总时长: %.3f秒", (unsigned long)animationBlocks.count, totalDuration);
    
    if (animationBlocks.count == 0) {
        if (completion) completion(YES);
        return;
    }
    
    NSTimeInterval intervalDuration = totalDuration / animationBlocks.count;
    __block NSInteger currentIndex = 0;
    
    void (^executeNextAnimation)(void) = ^void(void) {
        if (currentIndex >= animationBlocks.count) {
            if (completion) completion(YES);
            return;
        }
        
        void(^animationBlock)(void) = animationBlocks[currentIndex];
        currentIndex++;
        
        if (animationBlock) {
            animationBlock();
        }
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(intervalDuration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            executeNextAnimation();
        });
    };
    
    executeNextAnimation();
}

- (void)createParallelAnimations:(NSArray *)animationBlocks
                        duration:(NSTimeInterval)duration
                      completion:(void(^)(BOOL finished))completion {
    
    NSLog(@"创建并行动画, 动画数量: %lu, 时长: %.3f秒", (unsigned long)animationBlocks.count, duration);
    
    if (animationBlocks.count == 0) {
        if (completion) completion(YES);
        return;
    }
    
    dispatch_group_t group = dispatch_group_create();
    
    for (void(^animationBlock)(void) in animationBlocks) {
        dispatch_group_enter(group);
        
        if (animationBlock) {
            animationBlock();
        }
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            dispatch_group_leave(group);
        });
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        if (completion) {
            completion(YES);
        }
    });
}

@end
```

## UIView动画深度解析

### UIView动画管理器

```objc
@interface UIViewAnimationManager : NSObject

@property (nonatomic, weak) AnimationArchitecture *architecture;
@property (nonatomic, strong) NSMutableDictionary *activeAnimations;
@property (nonatomic, assign) NSInteger animationCounter;

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

// 动画控制
- (void)pauseAnimationsForView:(UIView *)view;
- (void)resumeAnimationsForView:(UIView *)view;
- (void)stopAnimationsForView:(UIView *)view;
- (void)stopAllAnimations;

// 动画状态查询
- (BOOL)isAnimatingView:(UIView *)view;
- (NSArray *)getActiveAnimations;
- (NSDictionary *)getAnimationInfo:(NSString *)animationId;

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
        self.architecture = [AnimationArchitecture sharedArchitecture];
        self.activeAnimations = [NSMutableDictionary dictionary];
        self.animationCounter = 0;
        
        // 注册到架构管理器
        [self.architecture registerAnimationManager:self forType:@"UIViewAnimation"];
    }
    return self;
}

- (void)animateView:(UIView *)view
           duration:(NSTimeInterval)duration
              delay:(NSTimeInterval)delay
            options:(UIViewAnimationOptions)options
         animations:(void(^)(void))animations
         completion:(void(^)(BOOL finished))completion {
    
    NSString *animationId = [self generateAnimationId];
    NSDate *startTime = [NSDate date];
    
    NSLog(@"开始UIView动画: %@, 时长: %.3f秒", animationId, duration);
    
    // 记录动画信息
    [self recordAnimationStart:animationId view:view duration:duration];
    
    [UIView animateWithDuration:duration
                          delay:delay
                        options:options
                     animations:^{
                         if (animations) {
                             animations();
                         }
                     }
                     completion:^(BOOL finished) {
                         NSTimeInterval actualDuration = [[NSDate date] timeIntervalSinceDate:startTime];
                         
                         // 记录性能指标
                         [self.architecture recordAnimationPerformance:@"UIViewBasic"
                                                               duration:actualDuration
                                                              frameRate:60.0];
                         
                         // 清理动画记录
                         [self recordAnimationEnd:animationId];
                         
                         NSLog(@"UIView动画完成: %@, 实际时长: %.3f秒", animationId, actualDuration);
                         
                         if (completion) {
                             completion(finished);
                         }
                     }];
}

- (void)springAnimateView:(UIView *)view
                 duration:(NSTimeInterval)duration
                  damping:(CGFloat)damping
                 velocity:(CGFloat)velocity
                  options:(UIViewAnimationOptions)options
               animations:(void(^)(void))animations
               completion:(void(^)(BOOL finished))completion {
    
    NSString *animationId = [self generateAnimationId];
    NSDate *startTime = [NSDate date];
    
    NSLog(@"开始弹簧动画: %@, 时长: %.3f秒, 阻尼: %.2f, 速度: %.2f", animationId, duration, damping, velocity);
    
    // 记录动画信息
    [self recordAnimationStart:animationId view:view duration:duration];
    
    [UIView animateWithDuration:duration
                          delay:0
         usingSpringWithDamping:damping
          initialSpringVelocity:velocity
                        options:options
                     animations:^{
                         if (animations) {
                             animations();
                         }
                     }
                     completion:^(BOOL finished) {
                         NSTimeInterval actualDuration = [[NSDate date] timeIntervalSinceDate:startTime];
                         
                         // 记录性能指标
                         [self.architecture recordAnimationPerformance:@"UIViewSpring"
                                                               duration:actualDuration
                                                              frameRate:60.0];
                         
                         // 清理动画记录
                         [self recordAnimationEnd:animationId];
                         
                         NSLog(@"弹簧动画完成: %@, 实际时长: %.3f秒", animationId, actualDuration);
                         
                         if (completion) {
                             completion(finished);
                         }
                     }];
}

- (void)keyframeAnimateView:(UIView *)view
                   duration:(NSTimeInterval)duration
                      delay:(NSTimeInterval)delay
                    options:(UIViewKeyframeAnimationOptions)options
                 animations:(void(^)(void))animations
                 completion:(void(^)(BOOL finished))completion {
    
    NSString *animationId = [self generateAnimationId];
    NSDate *startTime = [NSDate date];
    
    NSLog(@"开始关键帧动画: %@, 时长: %.3f秒", animationId, duration);
    
    // 记录动画信息
    [self recordAnimationStart:animationId view:view duration:duration];
    
    [UIView animateKeyframesWithDuration:duration
                                   delay:delay
                                 options:options
                              animations:^{
                                  if (animations) {
                                      animations();
                                  }
                              }
                              completion:^(BOOL finished) {
                                  NSTimeInterval actualDuration = [[NSDate date] timeIntervalSinceDate:startTime];
                                  
                                  // 记录性能指标
                                  [self.architecture recordAnimationPerformance:@"UIViewKeyframe"
                                                                        duration:actualDuration
                                                                       frameRate:60.0];
                                  
                                  // 清理动画记录
                                  [self recordAnimationEnd:animationId];
                                  
                                  NSLog(@"关键帧动画完成: %@, 实际时长: %.3f秒", animationId, actualDuration);
                                  
                                  if (completion) {
                                      completion(finished);
                                  }
                              }];
}

- (void)transitionWithView:(UIView *)view
                  duration:(NSTimeInterval)duration
                   options:(UIViewAnimationOptions)options
                animations:(void(^)(void))animations
                completion:(void(^)(BOOL finished))completion {
    
    NSString *animationId = [self generateAnimationId];
    NSDate *startTime = [NSDate date];
    
    NSLog(@"开始转场动画: %@, 时长: %.3f秒", animationId, duration);
    
    // 记录动画信息
    [self recordAnimationStart:animationId view:view duration:duration];
    
    [UIView transitionWithView:view
                      duration:duration
                       options:options
                    animations:^{
                        if (animations) {
                            animations();
                        }
                    }
                    completion:^(BOOL finished) {
                        NSTimeInterval actualDuration = [[NSDate date] timeIntervalSinceDate:startTime];
                        
                        // 记录性能指标
                        [self.architecture recordAnimationPerformance:@"UIViewTransition"
                                                              duration:actualDuration
                                                             frameRate:60.0];
                        
                        // 清理动画记录
                        [self recordAnimationEnd:animationId];
                        
                        NSLog(@"转场动画完成: %@, 实际时长: %.3f秒", animationId, actualDuration);
                        
                        if (completion) {
                            completion(finished);
                        }
                    }];
}

- (void)pauseAnimationsForView:(UIView *)view {
    NSLog(@"暂停视图动画: %@", view);
    
    CALayer *layer = view.layer;
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
    layer.speed = 0.0;
    layer.timeOffset = pausedTime;
}

- (void)resumeAnimationsForView:(UIView *)view {
    NSLog(@"恢复视图动画: %@", view);
    
    CALayer *layer = view.layer;
    CFTimeInterval pausedTime = [layer timeOffset];
    layer.speed = 1.0;
    layer.timeOffset = 0.0;
    layer.beginTime = 0.0;
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    layer.beginTime = timeSincePause;
}

- (void)stopAnimationsForView:(UIView *)view {
    NSLog(@"停止视图动画: %@", view);
    
    [view.layer removeAllAnimations];
    
    // 清理相关的动画记录
    NSArray *keysToRemove = [NSArray array];
    for (NSString *animationId in self.activeAnimations) {
        NSDictionary *animationInfo = self.activeAnimations[animationId];
        if (animationInfo[@"view"] == view) {
            keysToRemove = [keysToRemove arrayByAddingObject:animationId];
        }
    }
    
    for (NSString *key in keysToRemove) {
        [self.activeAnimations removeObjectForKey:key];
    }
}

- (void)stopAllAnimations {
    NSLog(@"停止所有动画");
    
    for (NSString *animationId in self.activeAnimations) {
        NSDictionary *animationInfo = self.activeAnimations[animationId];
        UIView *view = animationInfo[@"view"];
        if (view) {
            [view.layer removeAllAnimations];
        }
    }
    
    [self.activeAnimations removeAllObjects];
}

- (BOOL)isAnimatingView:(UIView *)view {
    return view.layer.animationKeys.count > 0;
}

- (NSArray *)getActiveAnimations {
    return [self.activeAnimations allKeys];
}

- (NSDictionary *)getAnimationInfo:(NSString *)animationId {
    return self.activeAnimations[animationId];
}

- (NSString *)generateAnimationId {
    self.animationCounter++;
    return [NSString stringWithFormat:@"UIViewAnim_%ld", (long)self.animationCounter];
}

- (void)recordAnimationStart:(NSString *)animationId view:(UIView *)view duration:(NSTimeInterval)duration {
    self.activeAnimations[animationId] = @{
        @"view": view,
        @"duration": @(duration),
        @"startTime": [NSDate date],
        @"type": @"UIView"
    };
}

- (void)recordAnimationEnd:(NSString *)animationId {
    [self.activeAnimations removeObjectForKey:animationId];
}

@end
```