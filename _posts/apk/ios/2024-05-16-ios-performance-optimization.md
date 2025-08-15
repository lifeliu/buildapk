---
layout: post
title: "iOS性能优化深度解析：内存管理、渲染优化与启动加速实战指南"
date: 2024-05-16
categories: ios
tags: [iOS, Performance, Memory, Rendering, Launch]
author: "iOS技术专家"
description: "深入探讨iOS应用性能优化的核心技术，包括内存管理策略、UI渲染优化、应用启动加速、网络性能优化等关键领域，提供实用的优化方案和最佳实践。"
---

在iOS应用开发中，性能优化是确保用户体验的关键因素。本文将深入探讨iOS性能优化的各个方面，从内存管理到渲染优化，从启动加速到网络性能，为开发者提供全面的优化策略和实践指南。

## 目录
1. [性能优化概述](#性能优化概述)
2. [内存管理优化](#内存管理优化)
3. [UI渲染性能优化](#UI渲染性能优化)
4. [应用启动优化](#应用启动优化)
5. [网络性能优化](#网络性能优化)
6. [电池续航优化](#电池续航优化)
7. [性能监控与分析](#性能监控与分析)
8. [性能优化最佳实践](#性能优化最佳实践)

## 性能优化概述

### 性能指标体系

```objc
@interface PerformanceMetrics : NSObject

@property (nonatomic, assign) NSTimeInterval launchTime;
@property (nonatomic, assign) CGFloat memoryUsage;
@property (nonatomic, assign) CGFloat cpuUsage;
@property (nonatomic, assign) NSInteger fps;
@property (nonatomic, assign) NSTimeInterval networkLatency;
@property (nonatomic, assign) CGFloat batteryDrain;

- (void)startMonitoring;
- (void)stopMonitoring;
- (NSDictionary *)generateReport;
- (BOOL)isPerformanceAcceptable;

@end

@implementation PerformanceMetrics

- (void)startMonitoring {
    [self startLaunchTimeMonitoring];
    [self startMemoryMonitoring];
    [self startCPUMonitoring];
    [self startFPSMonitoring];
    [self startNetworkMonitoring];
    [self startBatteryMonitoring];
}

- (void)startLaunchTimeMonitoring {
    static NSTimeInterval appLaunchTime;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        appLaunchTime = [[NSDate date] timeIntervalSince1970];
    });
    
    self.launchTime = [[NSDate date] timeIntervalSince1970] - appLaunchTime;
}

- (void)startMemoryMonitoring {
    struct mach_task_basic_info info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
    
    if (kerr == KERN_SUCCESS) {
        self.memoryUsage = info.resident_size / (1024.0 * 1024.0); // MB
    }
}

- (void)startCPUMonitoring {
    thread_array_t threadList;
    mach_msg_type_number_t threadCount;
    
    kern_return_t kernelReturn = task_threads(mach_task_self(), &threadList, &threadCount);
    if (kernelReturn != KERN_SUCCESS) {
        return;
    }
    
    CGFloat totalCPU = 0;
    for (int i = 0; i < threadCount; i++) {
        thread_info_data_t threadInfo;
        thread_basic_info_t threadBasicInfo;
        mach_msg_type_number_t threadInfoCount = THREAD_INFO_MAX;
        
        kernelReturn = thread_info(threadList[i], THREAD_BASIC_INFO, (thread_info_t)threadInfo, &threadInfoCount);
        if (kernelReturn != KERN_SUCCESS) {
            continue;
        }
        
        threadBasicInfo = (thread_basic_info_t)threadInfo;
        if (!(threadBasicInfo->flags & TH_FLAGS_IDLE)) {
            totalCPU += threadBasicInfo->cpu_usage / (float)TH_USAGE_SCALE * 100.0;
        }
    }
    
    vm_deallocate(mach_task_self(), (vm_offset_t)threadList, threadCount * sizeof(thread_t));
    self.cpuUsage = totalCPU;
}

- (void)startFPSMonitoring {
    static CADisplayLink *displayLink;
    static NSInteger frameCount = 0;
    static NSTimeInterval lastTimestamp = 0;
    
    if (!displayLink) {
        displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkTick:)];
        [displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
    }
}

- (void)displayLinkTick:(CADisplayLink *)link {
    static NSInteger frameCount = 0;
    static NSTimeInterval lastTimestamp = 0;
    
    frameCount++;
    if (lastTimestamp == 0) {
        lastTimestamp = link.timestamp;
        return;
    }
    
    NSTimeInterval interval = link.timestamp - lastTimestamp;
    if (interval >= 1.0) {
        self.fps = frameCount / interval;
        frameCount = 0;
        lastTimestamp = link.timestamp;
    }
}

- (NSDictionary *)generateReport {
    return @{
        @"launch_time": @(self.launchTime),
        @"memory_usage_mb": @(self.memoryUsage),
        @"cpu_usage_percent": @(self.cpuUsage),
        @"fps": @(self.fps),
        @"network_latency_ms": @(self.networkLatency * 1000),
        @"battery_drain_percent": @(self.batteryDrain)
    };
}

- (BOOL)isPerformanceAcceptable {
    return self.launchTime < 3.0 &&
           self.memoryUsage < 200.0 &&
           self.cpuUsage < 80.0 &&
           self.fps >= 55 &&
           self.networkLatency < 0.5;
}

@end
```

## UI渲染性能优化

### 渲染性能监控

```objc
@interface RenderingPerformanceMonitor : NSObject

@property (nonatomic, strong) CADisplayLink *displayLink;
@property (nonatomic, assign) NSTimeInterval lastFrameTimestamp;
@property (nonatomic, assign) NSInteger frameCount;
@property (nonatomic, assign) NSInteger droppedFrames;
@property (nonatomic, strong) NSMutableArray<NSNumber *> *frameTimes;
@property (nonatomic, copy) void(^performanceCallback)(NSDictionary *metrics);

- (void)startMonitoring;
- (void)stopMonitoring;
- (void)setPerformanceCallback:(void(^)(NSDictionary *metrics))callback;
- (NSDictionary *)getCurrentMetrics;

@end

@implementation RenderingPerformanceMonitor

- (instancetype)init {
    self = [super init];
    if (self) {
        _frameTimes = [NSMutableArray array];
    }
    return self;
}

- (void)startMonitoring {
    if (self.displayLink) {
        [self stopMonitoring];
    }
    
    self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkTick:)];
    [self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
    
    self.lastFrameTimestamp = 0;
    self.frameCount = 0;
    self.droppedFrames = 0;
    [self.frameTimes removeAllObjects];
}

- (void)stopMonitoring {
    [self.displayLink invalidate];
    self.displayLink = nil;
}

- (void)displayLinkTick:(CADisplayLink *)link {
    if (self.lastFrameTimestamp == 0) {
        self.lastFrameTimestamp = link.timestamp;
        return;
    }
    
    NSTimeInterval frameTime = link.timestamp - self.lastFrameTimestamp;
    self.lastFrameTimestamp = link.timestamp;
    
    [self.frameTimes addObject:@(frameTime)];
    
    // 保持最近100帧的数据
    if (self.frameTimes.count > 100) {
        [self.frameTimes removeObjectAtIndex:0];
    }
    
    self.frameCount++;
    
    // 检测掉帧（假设60fps，每帧应该是16.67ms）
    NSTimeInterval expectedFrameTime = 1.0 / 60.0;
    if (frameTime > expectedFrameTime * 1.5) {
        self.droppedFrames++;
    }
    
    // 每秒回调一次性能数据
    if (self.frameCount % 60 == 0 && self.performanceCallback) {
        self.performanceCallback([self getCurrentMetrics]);
    }
}

- (NSDictionary *)getCurrentMetrics {
    if (self.frameTimes.count == 0) {
        return @{};
    }
    
    // 计算平均帧时间
    NSTimeInterval totalFrameTime = 0;
    NSTimeInterval minFrameTime = INFINITY;
    NSTimeInterval maxFrameTime = 0;
    
    for (NSNumber *frameTime in self.frameTimes) {
        NSTimeInterval time = frameTime.doubleValue;
        totalFrameTime += time;
        minFrameTime = MIN(minFrameTime, time);
        maxFrameTime = MAX(maxFrameTime, time);
    }
    
    NSTimeInterval avgFrameTime = totalFrameTime / self.frameTimes.count;
    CGFloat fps = 1.0 / avgFrameTime;
    CGFloat dropRate = (CGFloat)self.droppedFrames / self.frameCount * 100;
    
    return @{
        @"fps": @(fps),
        @"avg_frame_time_ms": @(avgFrameTime * 1000),
        @"min_frame_time_ms": @(minFrameTime * 1000),
        @"max_frame_time_ms": @(maxFrameTime * 1000),
        @"dropped_frames": @(self.droppedFrames),
        @"total_frames": @(self.frameCount),
        @"drop_rate_percent": @(dropRate)
    };
}

@end
```

### 视图层次优化器

```objc
@interface ViewHierarchyOptimizer : NSObject

+ (void)optimizeView:(UIView *)view;
+ (void)analyzeViewHierarchy:(UIView *)rootView;
+ (NSArray<NSDictionary *> *)identifyPerformanceIssues:(UIView *)view;
+ (void)applyOptimizations:(UIView *)view withIssues:(NSArray<NSDictionary *> *)issues;

@end

@implementation ViewHierarchyOptimizer

+ (void)optimizeView:(UIView *)view {
    NSArray<NSDictionary *> *issues = [self identifyPerformanceIssues:view];
    [self applyOptimizations:view withIssues:issues];
}

+ (void)analyzeViewHierarchy:(UIView *)rootView {
    NSMutableString *analysis = [NSMutableString stringWithString:@"View Hierarchy Analysis:\n"];
    [self analyzeViewRecursively:rootView depth:0 analysis:analysis];
    NSLog(@"%@", analysis);
}

+ (void)analyzeViewRecursively:(UIView *)view depth:(NSInteger)depth analysis:(NSMutableString *)analysis {
    NSString *indent = [@"" stringByPaddingToLength:depth * 2 withString:@" " startingAtIndex:0];
    
    [analysis appendFormat:@"%@%@ (frame: %@, subviews: %lu)\n",
     indent,
     NSStringFromClass([view class]),
     NSStringFromCGRect(view.frame),
     (unsigned long)view.subviews.count];
    
    // 检查潜在问题
    if (view.layer.shadowOpacity > 0 && view.layer.shadowPath == nil) {
        [analysis appendFormat:@"%@  ⚠️ Shadow without shadowPath\n", indent];
    }
    
    if (view.layer.cornerRadius > 0 && !view.layer.masksToBounds) {
        [analysis appendFormat:@"%@  ⚠️ Corner radius without masksToBounds\n", indent];
    }
    
    if (view.alpha < 1.0 && view.subviews.count > 0) {
        [analysis appendFormat:@"%@  ⚠️ Transparent view with subviews\n", indent];
    }
    
    for (UIView *subview in view.subviews) {
        [self analyzeViewRecursively:subview depth:depth + 1 analysis:analysis];
    }
}

+ (NSArray<NSDictionary *> *)identifyPerformanceIssues:(UIView *)view {
    NSMutableArray<NSDictionary *> *issues = [NSMutableArray array];
    [self identifyIssuesRecursively:view issues:issues];
    return [issues copy];
}

+ (void)identifyIssuesRecursively:(UIView *)view issues:(NSMutableArray<NSDictionary *> *)issues {
    // 检查阴影性能问题
    if (view.layer.shadowOpacity > 0 && view.layer.shadowPath == nil) {
        [issues addObject:@{
            @"view": view,
            @"issue": @"shadow_without_path",
            @"severity": @"high",
            @"description": @"Shadow rendering without shadowPath causes performance issues"
        }];
    }
    
    // 检查圆角性能问题
    if (view.layer.cornerRadius > 0 && !view.layer.masksToBounds && view.backgroundColor != nil) {
        [issues addObject:@{
            @"view": view,
            @"issue": @"corner_radius_without_masking",
            @"severity": @"medium",
            @"description": @"Corner radius without proper masking may cause rendering issues"
        }];
    }
    
    // 检查透明度问题
    if (view.alpha < 1.0 && view.subviews.count > 5) {
        [issues addObject:@{
            @"view": view,
            @"issue": @"transparent_view_with_many_subviews",
            @"severity": @"medium",
            @"description": @"Transparent view with many subviews causes expensive blending"
        }];
    }
    
    // 检查视图层次深度
    NSInteger depth = [self calculateViewDepth:view];
    if (depth > 10) {
        [issues addObject:@{
            @"view": view,
            @"issue": @"deep_view_hierarchy",
            @"severity": @"low",
            @"description": [NSString stringWithFormat:@"View hierarchy depth (%ld) is too deep", (long)depth]
        }];
    }
    
    // 检查隐藏视图
    if (view.hidden && view.subviews.count > 0) {
        [issues addObject:@{
            @"view": view,
            @"issue": @"hidden_view_with_subviews",
            @"severity": @"low",
            @"description": @"Hidden view still maintains subview hierarchy"
        }];
    }
    
    for (UIView *subview in view.subviews) {
        [self identifyIssuesRecursively:subview issues:issues];
    }
}

+ (NSInteger)calculateViewDepth:(UIView *)view {
    NSInteger maxDepth = 0;
    for (UIView *subview in view.subviews) {
        NSInteger subviewDepth = [self calculateViewDepth:subview];
        maxDepth = MAX(maxDepth, subviewDepth);
    }
    return maxDepth + 1;
}

+ (void)applyOptimizations:(UIView *)view withIssues:(NSArray<NSDictionary *> *)issues {
    for (NSDictionary *issue in issues) {
        UIView *problemView = issue[@"view"];
        NSString *issueType = issue[@"issue"];
        
        if ([issueType isEqualToString:@"shadow_without_path"]) {
            // 为阴影添加路径
            problemView.layer.shadowPath = [UIBezierPath bezierPathWithRect:problemView.bounds].CGPath;
            NSLog(@"Applied shadowPath optimization to view: %@", NSStringFromClass([problemView class]));
        }
        else if ([issueType isEqualToString:@"corner_radius_without_masking"]) {
            // 启用masksToBounds
            problemView.layer.masksToBounds = YES;
            NSLog(@"Applied masksToBounds optimization to view: %@", NSStringFromClass([problemView class]));
        }
        else if ([issueType isEqualToString:@"transparent_view_with_many_subviews"]) {
            // 建议使用shouldRasterize
            problemView.layer.shouldRasterize = YES;
            problemView.layer.rasterizationScale = [UIScreen mainScreen].scale;
            NSLog(@"Applied rasterization optimization to view: %@", NSStringFromClass([problemView class]));
        }
    }
}

@end
```

### 图像渲染优化

```objc
@interface ImageRenderingOptimizer : NSObject

+ (UIImage *)optimizeImage:(UIImage *)image forSize:(CGSize)targetSize;
+ (UIImage *)resizeImage:(UIImage *)image toSize:(CGSize)newSize;
+ (UIImage *)compressImage:(UIImage *)image quality:(CGFloat)quality;
+ (void)preloadImagesAsync:(NSArray<NSString *> *)imageNames completion:(void(^)(NSDictionary<NSString *, UIImage *> *images))completion;
+ (UIImage *)createThumbnail:(UIImage *)image size:(CGSize)thumbnailSize;

@end

@implementation ImageRenderingOptimizer

+ (UIImage *)optimizeImage:(UIImage *)image forSize:(CGSize)targetSize {
    if (!image) return nil;
    
    // 如果图像已经是目标大小，直接返回
    if (CGSizeEqualToSize(image.size, targetSize)) {
        return image;
    }
    
    // 计算缩放比例
    CGFloat scaleX = targetSize.width / image.size.width;
    CGFloat scaleY = targetSize.height / image.size.height;
    CGFloat scale = MIN(scaleX, scaleY);
    
    // 如果不需要缩放，直接返回
    if (scale >= 1.0) {
        return image;
    }
    
    // 计算新的大小
    CGSize newSize = CGSizeMake(image.size.width * scale, image.size.height * scale);
    
    return [self resizeImage:image toSize:newSize];
}

+ (UIImage *)resizeImage:(UIImage *)image toSize:(CGSize)newSize {
    if (!image) return nil;
    
    UIGraphicsBeginImageContextWithOptions(newSize, NO, 0.0);
    [image drawInRect:CGRectMake(0, 0, newSize.width, newSize.height)];
    UIImage *resizedImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return resizedImage;
}

+ (UIImage *)compressImage:(UIImage *)image quality:(CGFloat)quality {
    if (!image) return nil;
    
    NSData *imageData = UIImageJPEGRepresentation(image, quality);
    return [UIImage imageWithData:imageData];
}

+ (void)preloadImagesAsync:(NSArray<NSString *> *)imageNames completion:(void(^)(NSDictionary<NSString *, UIImage *> *images))completion {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSMutableDictionary<NSString *, UIImage *> *loadedImages = [NSMutableDictionary dictionary];
        
        for (NSString *imageName in imageNames) {
            UIImage *image = [UIImage imageNamed:imageName];
            if (image) {
                loadedImages[imageName] = image;
            }
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion([loadedImages copy]);
            }
        });
    });
}

+ (UIImage *)createThumbnail:(UIImage *)image size:(CGSize)thumbnailSize {
    if (!image) return nil;
    
    CGFloat scale = [UIScreen mainScreen].scale;
    CGSize scaledSize = CGSizeMake(thumbnailSize.width * scale, thumbnailSize.height * scale);
    
    UIGraphicsBeginImageContextWithOptions(thumbnailSize, YES, scale);
    
    // 设置背景色
    [[UIColor whiteColor] setFill];
    UIRectFill(CGRectMake(0, 0, thumbnailSize.width, thumbnailSize.height));
    
    // 计算图像绘制区域
    CGFloat imageAspect = image.size.width / image.size.height;
    CGFloat thumbnailAspect = thumbnailSize.width / thumbnailSize.height;
    
    CGRect drawRect;
    if (imageAspect > thumbnailAspect) {
        // 图像更宽，以高度为准
        CGFloat drawWidth = thumbnailSize.height * imageAspect;
        drawRect = CGRectMake((thumbnailSize.width - drawWidth) / 2, 0, drawWidth, thumbnailSize.height);
    } else {
        // 图像更高，以宽度为准
        CGFloat drawHeight = thumbnailSize.width / imageAspect;
        drawRect = CGRectMake(0, (thumbnailSize.height - drawHeight) / 2, thumbnailSize.width, drawHeight);
    }
    
    [image drawInRect:drawRect];
    
    UIImage *thumbnail = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return thumbnail;
}

@end
```

## 应用启动优化

### 启动时间分析器

```objc
@interface LaunchTimeAnalyzer : NSObject

@property (nonatomic, strong) NSMutableArray<NSDictionary *> *launchPhases;
@property (nonatomic, assign) NSTimeInterval appStartTime;
@property (nonatomic, assign) NSTimeInterval mainStartTime;
@property (nonatomic, assign) NSTimeInterval firstViewControllerTime;
@property (nonatomic, assign) NSTimeInterval firstFrameTime;

+ (instancetype)sharedAnalyzer;
- (void)recordPhase:(NSString *)phaseName;
- (void)recordMainStart;
- (void)recordFirstViewController;
- (void)recordFirstFrame;
- (NSDictionary *)generateLaunchReport;
- (void)optimizeLaunchSequence;

@end

@implementation LaunchTimeAnalyzer

+ (instancetype)sharedAnalyzer {
    static LaunchTimeAnalyzer *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[LaunchTimeAnalyzer alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _launchPhases = [NSMutableArray array];
        _appStartTime = [[NSDate date] timeIntervalSince1970];
    }
    return self;
}

- (void)recordPhase:(NSString *)phaseName {
    NSTimeInterval currentTime = [[NSDate date] timeIntervalSince1970];
    NSTimeInterval elapsedTime = currentTime - self.appStartTime;
    
    [self.launchPhases addObject:@{
        @"phase": phaseName,
        @"timestamp": @(currentTime),
        @"elapsed_time": @(elapsedTime)
    }];
    
    NSLog(@"Launch Phase: %@ at %.3fs", phaseName, elapsedTime);
}

- (void)recordMainStart {
    self.mainStartTime = [[NSDate date] timeIntervalSince1970];
    [self recordPhase:@"main_start"];
}

- (void)recordFirstViewController {
    self.firstViewControllerTime = [[NSDate date] timeIntervalSince1970];
    [self recordPhase:@"first_view_controller"];
}

- (void)recordFirstFrame {
    self.firstFrameTime = [[NSDate date] timeIntervalSince1970];
    [self recordPhase:@"first_frame"];
}

- (NSDictionary *)generateLaunchReport {
    NSTimeInterval totalLaunchTime = self.firstFrameTime - self.appStartTime;
    NSTimeInterval preMainTime = self.mainStartTime - self.appStartTime;
    NSTimeInterval mainToFirstVC = self.firstViewControllerTime - self.mainStartTime;
    NSTimeInterval firstVCToFirstFrame = self.firstFrameTime - self.firstViewControllerTime;
    
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    report[@"total_launch_time"] = @(totalLaunchTime);
    report[@"pre_main_time"] = @(preMainTime);
    report[@"main_to_first_vc_time"] = @(mainToFirstVC);
    report[@"first_vc_to_first_frame_time"] = @(firstVCToFirstFrame);
    report[@"phases"] = [self.launchPhases copy];
    
    // 性能评级
    NSString *performanceRating;
    if (totalLaunchTime < 1.0) {
        performanceRating = @"Excellent";
    } else if (totalLaunchTime < 2.0) {
        performanceRating = @"Good";
    } else if (totalLaunchTime < 3.0) {
        performanceRating = @"Fair";
    } else {
        performanceRating = @"Poor";
    }
    
    report[@"performance_rating"] = performanceRating;
    
    // 优化建议
    NSMutableArray *suggestions = [NSMutableArray array];
    
    if (preMainTime > 0.4) {
        [suggestions addObject:@"Reduce pre-main time by optimizing dynamic library loading and +load methods"];
    }
    
    if (mainToFirstVC > 1.0) {
        [suggestions addObject:@"Optimize main thread initialization and reduce synchronous operations"];
    }
    
    if (firstVCToFirstFrame > 0.5) {
        [suggestions addObject:@"Optimize first view controller's viewDidLoad and initial rendering"];
    }
    
    report[@"optimization_suggestions"] = [suggestions copy];
    
    return [report copy];
}

- (void)optimizeLaunchSequence {
    // 延迟非关键初始化
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
        [self performNonCriticalInitialization];
    });
    
    // 预热关键路径
    [self prewarmCriticalPaths];
}

- (void)performNonCriticalInitialization {
    // 在后台线程执行非关键初始化
    // 例如：分析SDK初始化、第三方库初始化等
    NSLog(@"Performing non-critical initialization in background");
}

- (void)prewarmCriticalPaths {
    // 预热关键代码路径
    // 例如：预加载关键资源、预创建重要对象等
    NSLog(@"Prewarming critical paths");
}

@end
```

### 启动优化管理器

```objc
@interface LaunchOptimizationManager : NSObject

@property (nonatomic, strong) NSMutableArray<void(^)(void)> *deferredTasks;
@property (nonatomic, strong) NSMutableArray<void(^)(void)> *backgroundTasks;
@property (nonatomic, assign) BOOL hasCompletedLaunch;

+ (instancetype)sharedManager;
- (void)addDeferredTask:(void(^)(void))task;
- (void)addBackgroundTask:(void(^)(void))task;
- (void)executeDeferredTasks;
- (void)executeBackgroundTasks;
- (void)markLaunchCompleted;

@end

@implementation LaunchOptimizationManager

+ (instancetype)sharedManager {
    static LaunchOptimizationManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[LaunchOptimizationManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _deferredTasks = [NSMutableArray array];
        _backgroundTasks = [NSMutableArray array];
        _hasCompletedLaunch = NO;
    }
    return self;
}

- (void)addDeferredTask:(void(^)(void))task {
    if (!task) return;
    
    if (self.hasCompletedLaunch) {
        // 如果启动已完成，立即执行
        dispatch_async(dispatch_get_main_queue(), task);
    } else {
        // 否则添加到延迟任务列表
        [self.deferredTasks addObject:task];
    }
}

- (void)addBackgroundTask:(void(^)(void))task {
    if (!task) return;
    
    [self.backgroundTasks addObject:task];
    
    // 立即在后台执行
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), task);
}

- (void)executeDeferredTasks {
    if (self.deferredTasks.count == 0) return;
    
    NSLog(@"Executing %lu deferred tasks", (unsigned long)self.deferredTasks.count);
    
    // 分批执行延迟任务，避免阻塞主线程
    [self executeDeferredTasksBatch:0];
}

- (void)executeDeferredTasksBatch:(NSInteger)startIndex {
    const NSInteger batchSize = 3;
    NSInteger endIndex = MIN(startIndex + batchSize, self.deferredTasks.count);
    
    for (NSInteger i = startIndex; i < endIndex; i++) {
        void(^task)(void) = self.deferredTasks[i];
        task();
    }
    
    if (endIndex < self.deferredTasks.count) {
        // 继续执行下一批
        dispatch_async(dispatch_get_main_queue(), ^{
            [self executeDeferredTasksBatch:endIndex];
        });
    } else {
        // 所有任务执行完毕
        [self.deferredTasks removeAllObjects];
        NSLog(@"All deferred tasks completed");
    }
}

- (void)executeBackgroundTasks {
    NSLog(@"Executing %lu background tasks", (unsigned long)self.backgroundTasks.count);
    
    for (void(^task)(void) in self.backgroundTasks) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), task);
    }
    
    [self.backgroundTasks removeAllObjects];
}

- (void)markLaunchCompleted {
    self.hasCompletedLaunch = YES;
    
    // 启动完成后，执行延迟任务
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self executeDeferredTasks];
    });
}

@end
```

## 网络性能优化

### 网络请求优化器

```objc
@interface NetworkOptimizer : NSObject

@property (nonatomic, strong) NSURLSessionConfiguration *optimizedConfiguration;
@property (nonatomic, strong) NSURLSession *optimizedSession;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSData *> *responseCache;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSNumber *> *requestMetrics;

+ (instancetype)sharedOptimizer;
- (void)configureOptimizedSession;
- (NSURLSessionDataTask *)optimizedRequestWithURL:(NSURL *)url completion:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completion;
- (void)enableRequestCompression;
- (void)configureConnectionPooling;
- (NSDictionary *)getNetworkMetrics;

@end

@implementation NetworkOptimizer

+ (instancetype)sharedOptimizer {
    static NetworkOptimizer *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[NetworkOptimizer alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _responseCache = [NSMutableDictionary dictionary];
        _requestMetrics = [NSMutableDictionary dictionary];
        [self configureOptimizedSession];
    }
    return self;
}

- (void)configureOptimizedSession {
    self.optimizedConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
    
    // 配置缓存策略
    NSURLCache *cache = [[NSURLCache alloc] initWithMemoryCapacity:10 * 1024 * 1024  // 10MB
                                                      diskCapacity:50 * 1024 * 1024  // 50MB
                                                          diskPath:nil];
    self.optimizedConfiguration.URLCache = cache;
    self.optimizedConfiguration.requestCachePolicy = NSURLRequestReturnCacheDataElseLoad;
    
    // 配置连接参数
    self.optimizedConfiguration.timeoutIntervalForRequest = 30.0;
    self.optimizedConfiguration.timeoutIntervalForResource = 60.0;
    self.optimizedConfiguration.HTTPMaximumConnectionsPerHost = 6;
    
    // 启用HTTP/2
    self.optimizedConfiguration.HTTPShouldUsePipelining = YES;
    
    // 配置压缩
    [self enableRequestCompression];
    
    // 配置连接池
    [self configureConnectionPooling];
    
    self.optimizedSession = [NSURLSession sessionWithConfiguration:self.optimizedConfiguration];
}

- (void)enableRequestCompression {
    // 设置Accept-Encoding头
    NSMutableDictionary *headers = [self.optimizedConfiguration.HTTPAdditionalHeaders mutableCopy] ?: [NSMutableDictionary dictionary];
    headers[@"Accept-Encoding"] = @"gzip, deflate, br";
    headers[@"Accept"] = @"application/json, text/plain, */*";
    self.optimizedConfiguration.HTTPAdditionalHeaders = [headers copy];
}

- (void)configureConnectionPooling {
    // 启用连接复用
    self.optimizedConfiguration.HTTPShouldSetCookies = YES;
    self.optimizedConfiguration.HTTPCookieAcceptPolicy = NSHTTPCookieAcceptPolicyAlways;
    
    // 配置TLS设置
    self.optimizedConfiguration.TLSMinimumSupportedProtocol = kTLSProtocol12;
    self.optimizedConfiguration.TLSMaximumSupportedProtocol = kTLSProtocol13;
}

- (NSURLSessionDataTask *)optimizedRequestWithURL:(NSURL *)url completion:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completion {
    NSString *cacheKey = url.absoluteString;
    
    // 检查内存缓存
    NSData *cachedData = self.responseCache[cacheKey];
    if (cachedData) {
        NSLog(@"Returning cached response for: %@", url.absoluteString);
        if (completion) {
            completion(cachedData, nil, nil);
        }
        return nil;
    }
    
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    
    // 添加性能优化头
    [request setValue:@"keep-alive" forHTTPHeaderField:@"Connection"];
    [request setValue:@"gzip, deflate, br" forHTTPHeaderField:@"Accept-Encoding"];
    
    NSDate *startTime = [NSDate date];
    
    NSURLSessionDataTask *task = [self.optimizedSession dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        NSTimeInterval requestTime = [[NSDate date] timeIntervalSinceDate:startTime];
        
        // 记录请求指标
        self.requestMetrics[cacheKey] = @(requestTime);
        
        if (data && !error) {
            // 缓存响应
            self.responseCache[cacheKey] = data;
            
            // 限制缓存大小
            if (self.responseCache.count > 100) {
                NSArray *keys = [self.responseCache.allKeys sortedArrayUsingComparator:^NSComparisonResult(NSString *key1, NSString *key2) {
                    NSNumber *time1 = self.requestMetrics[key1];
                    NSNumber *time2 = self.requestMetrics[key2];
                    return [time1 compare:time2];
                }];
                
                // 移除最旧的缓存项
                for (NSInteger i = 0; i < 20; i++) {
                    NSString *keyToRemove = keys[i];
                    [self.responseCache removeObjectForKey:keyToRemove];
                    [self.requestMetrics removeObjectForKey:keyToRemove];
                }
            }
        }
        
        NSLog(@"Request completed in %.3fs: %@", requestTime, url.absoluteString);
        
        if (completion) {
            completion(data, response, error);
        }
    }];
    
    [task resume];
    return task;
}

- (NSDictionary *)getNetworkMetrics {
    NSArray *requestTimes = [self.requestMetrics.allValues sortedArrayUsingComparator:^NSComparisonResult(NSNumber *time1, NSNumber *time2) {
        return [time1 compare:time2];
    }];
    
    if (requestTimes.count == 0) {
        return @{};
    }
    
    NSTimeInterval totalTime = 0;
    for (NSNumber *time in requestTimes) {
        totalTime += time.doubleValue;
    }
    
    NSTimeInterval averageTime = totalTime / requestTimes.count;
    NSTimeInterval minTime = [requestTimes.firstObject doubleValue];
    NSTimeInterval maxTime = [requestTimes.lastObject doubleValue];
    NSTimeInterval medianTime = [requestTimes[requestTimes.count / 2] doubleValue];
    
    return @{
        @"total_requests": @(requestTimes.count),
        @"average_time": @(averageTime),
        @"min_time": @(minTime),
        @"max_time": @(maxTime),
        @"median_time": @(medianTime),
        @"cache_hit_rate": @((double)self.responseCache.count / requestTimes.count * 100)
    };
}

@end
```

## 电池性能优化

### 电池使用监控器

```objc
@interface BatteryOptimizer : NSObject

@property (nonatomic, assign) float initialBatteryLevel;
@property (nonatomic, strong) NSDate *monitoringStartTime;
@property (nonatomic, strong) NSMutableArray<NSDictionary *> *batteryReadings;
@property (nonatomic, strong) NSTimer *monitoringTimer;

+ (instancetype)sharedOptimizer;
- (void)startBatteryMonitoring;
- (void)stopBatteryMonitoring;
- (NSDictionary *)getBatteryUsageReport;
- (void)optimizeBackgroundTasks;
- (void)optimizeLocationServices;
- (void)optimizeNetworkUsage;

@end

@implementation BatteryOptimizer

+ (instancetype)sharedOptimizer {
    static BatteryOptimizer *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[BatteryOptimizer alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _batteryReadings = [NSMutableArray array];
        
        // 启用电池监控
        [UIDevice currentDevice].batteryMonitoringEnabled = YES;
        
        // 监听电池状态变化
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(batteryStateChanged:)
                                                     name:UIDeviceBatteryStateDidChangeNotification
                                                   object:nil];
        
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(batteryLevelChanged:)
                                                     name:UIDeviceBatteryLevelDidChangeNotification
                                                   object:nil];
    }
    return self;
}

- (void)startBatteryMonitoring {
    self.initialBatteryLevel = [UIDevice currentDevice].batteryLevel;
    self.monitoringStartTime = [NSDate date];
    [self.batteryReadings removeAllObjects];
    
    // 每分钟记录一次电池状态
    self.monitoringTimer = [NSTimer scheduledTimerWithTimeInterval:60.0
                                                            target:self
                                                          selector:@selector(recordBatteryReading)
                                                          userInfo:nil
                                                           repeats:YES];
    
    NSLog(@"Battery monitoring started. Initial level: %.1f%%", self.initialBatteryLevel * 100);
}

- (void)stopBatteryMonitoring {
    [self.monitoringTimer invalidate];
    self.monitoringTimer = nil;
    
    NSLog(@"Battery monitoring stopped");
}

- (void)recordBatteryReading {
    UIDevice *device = [UIDevice currentDevice];
    NSTimeInterval elapsedTime = [[NSDate date] timeIntervalSinceDate:self.monitoringStartTime];
    
    NSDictionary *reading = @{
        @"timestamp": [NSDate date],
        @"elapsed_time": @(elapsedTime),
        @"battery_level": @(device.batteryLevel),
        @"battery_state": @(device.batteryState),
        @"low_power_mode": @([[NSProcessInfo processInfo] isLowPowerModeEnabled])
    };
    
    [self.batteryReadings addObject:reading];
    
    // 检查电池消耗率
    if (self.batteryReadings.count > 1) {
        NSDictionary *previousReading = self.batteryReadings[self.batteryReadings.count - 2];
        float previousLevel = [previousReading[@"battery_level"] floatValue];
        float currentLevel = device.batteryLevel;
        float drainRate = (previousLevel - currentLevel) * 100; // 每分钟消耗百分比
        
        if (drainRate > 2.0) { // 每分钟消耗超过2%
            NSLog(@"⚠️ High battery drain detected: %.1f%% per minute", drainRate);
            [self optimizeForHighDrain];
        }
    }
}

- (void)batteryStateChanged:(NSNotification *)notification {
    UIDevice *device = [UIDevice currentDevice];
    NSString *stateString;
    
    switch (device.batteryState) {
        case UIDeviceBatteryStateUnknown:
            stateString = @"Unknown";
            break;
        case UIDeviceBatteryStateUnplugged:
            stateString = @"Unplugged";
            break;
        case UIDeviceBatteryStateCharging:
            stateString = @"Charging";
            break;
        case UIDeviceBatteryStateFull:
            stateString = @"Full";
            break;
    }
    
    NSLog(@"Battery state changed to: %@", stateString);
}

- (void)batteryLevelChanged:(NSNotification *)notification {
    UIDevice *device = [UIDevice currentDevice];
    NSLog(@"Battery level changed to: %.1f%%", device.batteryLevel * 100);
    
    if (device.batteryLevel < 0.2) { // 电量低于20%
        [self enablePowerSavingMode];
    }
}

- (NSDictionary *)getBatteryUsageReport {
    if (self.batteryReadings.count < 2) {
        return @{@"error": @"Insufficient data for report"};
    }
    
    NSDictionary *firstReading = self.batteryReadings.firstObject;
    NSDictionary *lastReading = self.batteryReadings.lastObject;
    
    float initialLevel = [firstReading[@"battery_level"] floatValue];
    float finalLevel = [lastReading[@"battery_level"] floatValue];
    NSTimeInterval totalTime = [lastReading[@"elapsed_time"] doubleValue];
    
    float totalDrain = (initialLevel - finalLevel) * 100;
    float drainPerHour = totalDrain / (totalTime / 3600.0);
    
    // 计算平均消耗率
    NSMutableArray *drainRates = [NSMutableArray array];
    for (NSInteger i = 1; i < self.batteryReadings.count; i++) {
        NSDictionary *current = self.batteryReadings[i];
        NSDictionary *previous = self.batteryReadings[i-1];
        
        float currentLevel = [current[@"battery_level"] floatValue];
        float previousLevel = [previous[@"battery_level"] floatValue];
        float drain = (previousLevel - currentLevel) * 100;
        
        [drainRates addObject:@(drain)];
    }
    
    // 计算统计数据
    float totalDrainRate = 0;
    float maxDrainRate = 0;
    for (NSNumber *rate in drainRates) {
        float rateValue = rate.floatValue;
        totalDrainRate += rateValue;
        maxDrainRate = MAX(maxDrainRate, rateValue);
    }
    
    float averageDrainRate = totalDrainRate / drainRates.count;
    
    return @{
        @"monitoring_duration_hours": @(totalTime / 3600.0),
        @"initial_battery_level": @(initialLevel * 100),
        @"final_battery_level": @(finalLevel * 100),
        @"total_drain_percent": @(totalDrain),
        @"drain_per_hour_percent": @(drainPerHour),
        @"average_drain_rate_per_minute": @(averageDrainRate),
        @"max_drain_rate_per_minute": @(maxDrainRate),
        @"readings_count": @(self.batteryReadings.count)
    };
}

- (void)optimizeForHighDrain {
    NSLog(@"Applying battery optimization strategies...");
    
    [self optimizeBackgroundTasks];
    [self optimizeLocationServices];
    [self optimizeNetworkUsage];
}

- (void)optimizeBackgroundTasks {
    // 减少后台任务频率
    NSLog(@"Optimizing background tasks for battery saving");
    
    // 示例：减少后台刷新频率
    [[UIApplication sharedApplication] setMinimumBackgroundFetchInterval:UIApplicationBackgroundFetchIntervalNever];
}

- (void)optimizeLocationServices {
    // 优化位置服务设置
    NSLog(@"Optimizing location services for battery saving");
    
    // 示例：降低位置精度要求
    // 这里需要根据具体的CLLocationManager实例进行配置
}

- (void)optimizeNetworkUsage {
    // 优化网络使用
    NSLog(@"Optimizing network usage for battery saving");
    
    // 示例：减少网络请求频率，启用更积极的缓存策略
}

- (void)enablePowerSavingMode {
    NSLog(@"Enabling power saving mode");
    
    // 应用级别的省电模式
    [self optimizeBackgroundTasks];
    [self optimizeLocationServices];
    [self optimizeNetworkUsage];
    
    // 降低动画和视觉效果
    [UIView setAnimationsEnabled:NO];
    
    // 减少屏幕亮度（如果应用有控制权限）
    // [[UIScreen mainScreen] setBrightness:0.3];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    [self.monitoringTimer invalidate];
}

@end
```

## 性能监控与分析

### 综合性能监控器

```objc
@interface PerformanceMonitor : NSObject

@property (nonatomic, strong) PerformanceMetrics *metrics;
@property (nonatomic, strong) RenderingPerformanceMonitor *renderingMonitor;
@property (nonatomic, strong) MemoryLeakDetector *memoryDetector;
@property (nonatomic, strong) NetworkOptimizer *networkOptimizer;
@property (nonatomic, strong) BatteryOptimizer *batteryOptimizer;
@property (nonatomic, strong) NSTimer *reportTimer;

+ (instancetype)sharedMonitor;
- (void)startComprehensiveMonitoring;
- (void)stopComprehensiveMonitoring;
- (NSDictionary *)generateComprehensiveReport;
- (void)exportReportToFile:(NSString *)filePath;
- (void)setPerformanceThresholds:(NSDictionary *)thresholds;

@end

@implementation PerformanceMonitor

+ (instancetype)sharedMonitor {
    static PerformanceMonitor *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[PerformanceMonitor alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _metrics = [[PerformanceMetrics alloc] init];
        _renderingMonitor = [[RenderingPerformanceMonitor alloc] init];
        _memoryDetector = [[MemoryLeakDetector alloc] init];
        _networkOptimizer = [NetworkOptimizer sharedOptimizer];
        _batteryOptimizer = [BatteryOptimizer sharedOptimizer];
    }
    return self;
}

- (void)startComprehensiveMonitoring {
    NSLog(@"Starting comprehensive performance monitoring...");
    
    // 启动各个监控组件
    [self.metrics startMonitoring];
    [self.renderingMonitor startMonitoring];
    [self.memoryDetector startDetection];
    [self.batteryOptimizer startBatteryMonitoring];
    
    // 设置渲染性能回调
    [self.renderingMonitor setPerformanceCallback:^(NSDictionary *metrics) {
        NSLog(@"Rendering Performance: FPS=%.1f, Drop Rate=%.1f%%",
              [metrics[@"fps"] floatValue],
              [metrics[@"drop_rate_percent"] floatValue]);
        
        // 检查性能阈值
        if ([metrics[@"fps"] floatValue] < 50.0) {
            NSLog(@"⚠️ Low FPS detected, consider optimization");
        }
    }];
    
    // 定期生成报告
    self.reportTimer = [NSTimer scheduledTimerWithTimeInterval:300.0 // 5分钟
                                                        target:self
                                                      selector:@selector(generatePeriodicReport)
                                                      userInfo:nil
                                                       repeats:YES];
}

- (void)stopComprehensiveMonitoring {
    NSLog(@"Stopping comprehensive performance monitoring...");
    
    [self.metrics stopMonitoring];
    [self.renderingMonitor stopMonitoring];
    [self.memoryDetector stopDetection];
    [self.batteryOptimizer stopBatteryMonitoring];
    
    [self.reportTimer invalidate];
    self.reportTimer = nil;
}

- (void)generatePeriodicReport {
    NSDictionary *report = [self generateComprehensiveReport];
    NSLog(@"Periodic Performance Report: %@", report);
    
    // 检查关键指标
    NSDictionary *memory = report[@"memory"];
    if ([memory[@"memory_usage_mb"] floatValue] > 200.0) {
        NSLog(@"⚠️ High memory usage detected: %.1f MB", [memory[@"memory_usage_mb"] floatValue]);
    }
    
    NSDictionary *battery = report[@"battery"];
    if ([battery[@"drain_per_hour_percent"] floatValue] > 10.0) {
        NSLog(@"⚠️ High battery drain detected: %.1f%% per hour", [battery[@"drain_per_hour_percent"] floatValue]);
    }
}

- (NSDictionary *)generateComprehensiveReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    // 基础性能指标
    report[@"timestamp"] = [NSDate date];
    report[@"app_version"] = [[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleShortVersionString"];
    report[@"device_model"] = [[UIDevice currentDevice] model];
    report[@"ios_version"] = [[UIDevice currentDevice] systemVersion];
    
    // 内存性能
    report[@"memory"] = [self.metrics getMemoryMetrics];
    
    // CPU性能
    report[@"cpu"] = [self.metrics getCPUMetrics];
    
    // 渲染性能
    report[@"rendering"] = [self.renderingMonitor getCurrentMetrics];
    
    // 网络性能
    report[@"network"] = [self.networkOptimizer getNetworkMetrics];
    
    // 电池性能
    report[@"battery"] = [self.batteryOptimizer getBatteryUsageReport];
    
    // 内存泄漏检测
    report[@"memory_leaks"] = [self.memoryDetector getLeakReport];
    
    // 启动性能
    report[@"launch_performance"] = [[LaunchTimeAnalyzer sharedAnalyzer] generateLaunchReport];
    
    // 计算综合性能评分
    report[@"performance_score"] = [self calculatePerformanceScore:report];
    
    return [report copy];
}

- (NSNumber *)calculatePerformanceScore:(NSDictionary *)report {
    float score = 100.0; // 满分100
    
    // 内存评分 (权重: 25%)
    NSDictionary *memory = report[@"memory"];
    float memoryUsage = [memory[@"memory_usage_mb"] floatValue];
    if (memoryUsage > 300) score -= 25;
    else if (memoryUsage > 200) score -= 15;
    else if (memoryUsage > 100) score -= 5;
    
    // CPU评分 (权重: 20%)
    NSDictionary *cpu = report[@"cpu"];
    float cpuUsage = [cpu[@"cpu_usage_percent"] floatValue];
    if (cpuUsage > 80) score -= 20;
    else if (cpuUsage > 60) score -= 10;
    else if (cpuUsage > 40) score -= 5;
    
    // 渲染评分 (权重: 25%)
    NSDictionary *rendering = report[@"rendering"];
    float fps = [rendering[@"fps"] floatValue];
    if (fps < 30) score -= 25;
    else if (fps < 45) score -= 15;
    else if (fps < 55) score -= 5;
    
    // 电池评分 (权重: 20%)
    NSDictionary *battery = report[@"battery"];
    if (![battery[@"error"] isKindOfClass:[NSNull class]] && battery[@"error"]) {
        // 电池数据不足，不扣分
    } else {
        float drainRate = [battery[@"drain_per_hour_percent"] floatValue];
        if (drainRate > 20) score -= 20;
        else if (drainRate > 15) score -= 10;
        else if (drainRate > 10) score -= 5;
    }
    
    // 内存泄漏评分 (权重: 10%)
    NSDictionary *leaks = report[@"memory_leaks"];
    NSInteger leakCount = [leaks[@"potential_leaks"] integerValue];
    if (leakCount > 10) score -= 10;
    else if (leakCount > 5) score -= 5;
    else if (leakCount > 0) score -= 2;
    
    return @(MAX(0, score));
}

- (void)exportReportToFile:(NSString *)filePath {
    NSDictionary *report = [self generateComprehensiveReport];
    
    NSError *error;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:report
                                                       options:NSJSONWritingPrettyPrinted
                                                         error:&error];
    
    if (error) {
        NSLog(@"Error serializing performance report: %@", error.localizedDescription);
        return;
    }
    
    BOOL success = [jsonData writeToFile:filePath atomically:YES];
    if (success) {
        NSLog(@"Performance report exported to: %@", filePath);
    } else {
        NSLog(@"Failed to export performance report to: %@", filePath);
    }
}

- (void)setPerformanceThresholds:(NSDictionary *)thresholds {
    // 设置性能阈值，用于自动优化
    NSLog(@"Performance thresholds updated: %@", thresholds);
}

@end
```

## 性能优化最佳实践

### 开发阶段优化建议

1. **代码层面优化**
   - 避免在主线程执行耗时操作
   - 合理使用异步编程和GCD
   - 优化算法复杂度，避免嵌套循环
   - 使用懒加载和延迟初始化
   - 避免频繁的对象创建和销毁

2. **内存管理优化**
   - 正确使用ARC，避免循环引用
   - 及时释放不需要的资源
   - 使用弱引用打破循环引用
   - 合理使用autorelease pool
   - 监控内存使用情况

3. **UI渲染优化**
   - 避免复杂的视图层次结构
   - 合理使用图层属性和效果
   - 优化图像加载和缓存
   - 使用异步绘制技术
   - 避免频繁的布局计算

4. **网络优化**
   - 实现智能缓存策略
   - 使用数据压缩和增量更新
   - 合并网络请求，减少请求次数
   - 实现离线功能和数据同步
   - 优化网络超时和重试机制

5. **启动优化**
   - 延迟非关键功能的初始化
   - 优化应用启动流程
   - 减少启动时的同步操作
   - 预加载关键资源
   - 使用启动画面优化用户体验

### 性能监控策略

1. **持续监控**
   - 集成性能监控工具
   - 设置性能指标阈值
   - 建立性能回归检测机制
   - 定期生成性能报告
   - 跟踪性能趋势变化

2. **问题诊断**
   - 使用Instruments进行深度分析
   - 实现自定义性能指标收集
   - 建立性能问题分类体系
   - 制定性能优化优先级
   - 验证优化效果

## 总结

iOS性能优化是一个系统性工程，需要从多个维度进行综合考虑：

### 关键要点

1. **全面监控**：建立完善的性能监控体系，实时跟踪应用性能状况
2. **预防为主**：在开发阶段就考虑性能因素，避免后期大规模重构
3. **数据驱动**：基于真实数据进行优化决策，避免过度优化
4. **用户体验**：始终以提升用户体验为目标，平衡性能和功能
5. **持续改进**：建立性能优化的长效机制，持续跟踪和改进

### 优化收益

通过系统性的性能优化，可以实现：
- **启动时间**减少50-70%
- **内存使用**降低30-50%
- **CPU占用**减少20-40%
- **电池续航**提升15-30%
- **用户体验**显著改善

性能优化是iOS开发中的核心技能，需要开发者具备深入的技术理解和丰富的实践经验。通过本文介绍的方法和工具，开发者可以构建高性能、用户体验优秀的iOS应用。

### 性能基准测试框架

```objc
@interface PerformanceBenchmark : NSObject

@property (nonatomic, strong) NSMutableArray<NSDictionary *> *benchmarkResults;
@property (nonatomic, strong) PerformanceMetrics *metrics;

- (void)runBenchmarkSuite;
- (void)benchmarkMemoryAllocation;
- (void)benchmarkUIRendering;
- (void)benchmarkNetworkRequests;
- (void)benchmarkDatabaseOperations;
- (NSDictionary *)analyzeBenchmarkResults;

@end

@implementation PerformanceBenchmark

- (instancetype)init {
    self = [super init];
    if (self) {
        _benchmarkResults = [NSMutableArray array];
        _metrics = [[PerformanceMetrics alloc] init];
    }
    return self;
}

- (void)runBenchmarkSuite {
    NSLog(@"Starting Performance Benchmark Suite...");
    
    [self benchmarkMemoryAllocation];
    [self benchmarkUIRendering];
    [self benchmarkNetworkRequests];
    [self benchmarkDatabaseOperations];
    
    NSDictionary *analysis = [self analyzeBenchmarkResults];
    NSLog(@"Benchmark Analysis: %@", analysis);
}

- (void)benchmarkMemoryAllocation {
    NSTimeInterval startTime = [[NSDate date] timeIntervalSince1970];
    
    // 测试大量对象分配
    NSMutableArray *objects = [NSMutableArray array];
    for (NSInteger i = 0; i < 100000; i++) {
        NSDictionary *object = @{
            @"id": @(i),
            @"data": [NSString stringWithFormat:@"test_data_%ld", (long)i],
            @"timestamp": [NSDate date]
        };
        [objects addObject:object];
    }
    
    NSTimeInterval endTime = [[NSDate date] timeIntervalSince1970];
    
    [self.benchmarkResults addObject:@{
        @"test": @"memory_allocation",
        @"duration": @(endTime - startTime),
        @"objects_count": @(objects.count),
        @"memory_usage": @(self.metrics.memoryUsage)
    }];
    
    // 清理内存
    objects = nil;
}

- (void)benchmarkUIRendering {
    NSTimeInterval startTime = [[NSDate date] timeIntervalSince1970];
    
    // 创建复杂视图层次结构
    UIView *containerView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 375, 667)];
    
    for (NSInteger i = 0; i < 1000; i++) {
        UIView *subview = [[UIView alloc] initWithFrame:CGRectMake(i % 375, i / 375, 50, 50)];
        subview.backgroundColor = [UIColor colorWithRed:arc4random_uniform(256)/255.0
                                                  green:arc4random_uniform(256)/255.0
                                                   blue:arc4random_uniform(256)/255.0
                                                  alpha:1.0];
        subview.layer.cornerRadius = 25;
        subview.layer.shadowOffset = CGSizeMake(2, 2);
        subview.layer.shadowOpacity = 0.5;
        [containerView addSubview:subview];
    }
    
    // 强制渲染
    [containerView.layer renderInContext:UIGraphicsGetCurrentContext()];
    
    NSTimeInterval endTime = [[NSDate date] timeIntervalSince1970];
    
    [self.benchmarkResults addObject:@{
        @"test": @"ui_rendering",
        @"duration": @(endTime - startTime),
        @"subviews_count": @(containerView.subviews.count),
        @"fps": @(self.metrics.fps)
    }];
}

- (void)benchmarkNetworkRequests {
    NSTimeInterval startTime = [[NSDate date] timeIntervalSince1970];
    
    dispatch_group_t group = dispatch_group_create();
    NSInteger requestCount = 10;
    
    for (NSInteger i = 0; i < requestCount; i++) {
        dispatch_group_enter(group);
        
        NSURL *url = [NSURL URLWithString:@"https://httpbin.org/delay/1"];
        NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithURL:url
                                                                 completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            dispatch_group_leave(group);
        }];
        [task resume];
    }
    
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    
    NSTimeInterval endTime = [[NSDate date] timeIntervalSince1970];
    
    [self.benchmarkResults addObject:@{
        @"test": @"network_requests",
        @"duration": @(endTime - startTime),
        @"requests_count": @(requestCount),
        @"average_latency": @((endTime - startTime) / requestCount)
    }];
}

- (void)benchmarkDatabaseOperations {
    NSTimeInterval startTime = [[NSDate date] timeIntervalSince1970];
    
    // 模拟数据库操作
    NSMutableArray *database = [NSMutableArray array];
    
    // 插入操作
    for (NSInteger i = 0; i < 10000; i++) {
        NSDictionary *record = @{
            @"id": @(i),
            @"name": [NSString stringWithFormat:@"User_%ld", (long)i],
            @"email": [NSString stringWithFormat:@"user%ld@example.com", (long)i],
            @"created_at": [NSDate date]
        };
        [database addObject:record];
    }
    
    // 查询操作
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"id > %@", @5000];
    NSArray *filteredResults = [database filteredArrayUsingPredicate:predicate];
    
    // 更新操作
    for (NSMutableDictionary *record in database) {
        if ([record[@"id"] integerValue] % 2 == 0) {
            record[@"updated_at"] = [NSDate date];
        }
    }
    
    NSTimeInterval endTime = [[NSDate date] timeIntervalSince1970];
    
    [self.benchmarkResults addObject:@{
        @"test": @"database_operations",
        @"duration": @(endTime - startTime),
        @"records_count": @(database.count),
        @"filtered_count": @(filteredResults.count)
    }];
}

- (NSDictionary *)analyzeBenchmarkResults {
    NSMutableDictionary *analysis = [NSMutableDictionary dictionary];
    
    for (NSDictionary *result in self.benchmarkResults) {
        NSString *testName = result[@"test"];
        NSNumber *duration = result[@"duration"];
        
        analysis[testName] = @{
            @"duration_seconds": duration,
            @"performance_rating": [self ratePerformance:duration.doubleValue forTest:testName]
        };
    }
    
    return [analysis copy];
}

- (NSString *)ratePerformance:(NSTimeInterval)duration forTest:(NSString *)testName {
    NSDictionary *thresholds = @{
        @"memory_allocation": @0.5,
        @"ui_rendering": @0.1,
        @"network_requests": @15.0,
        @"database_operations": @1.0
    };
    
    NSNumber *threshold = thresholds[testName];
    if (!threshold) return @"unknown";
    
    if (duration <= threshold.doubleValue * 0.5) {
        return @"excellent";
    } else if (duration <= threshold.doubleValue) {
        return @"good";
    } else if (duration <= threshold.doubleValue * 2) {
        return @"fair";
    } else {
        return @"poor";
    }
}

@end
```

## 内存管理优化

### 内存泄漏检测与修复

```objc
@interface MemoryLeakDetector : NSObject

@property (nonatomic, strong) NSMutableSet<NSValue *> *trackedObjects;
@property (nonatomic, strong) NSMutableDictionary<NSValue *, NSString *> *objectStackTraces;
@property (nonatomic, assign) BOOL isEnabled;

+ (instancetype)sharedDetector;
- (void)startTracking;
- (void)stopTracking;
- (void)trackObject:(id)object withStackTrace:(NSString *)stackTrace;
- (void)untrackObject:(id)object;
- (NSArray<NSDictionary *> *)detectLeaks;
- (void)generateLeakReport;

@end

@implementation MemoryLeakDetector

+ (instancetype)sharedDetector {
    static MemoryLeakDetector *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[MemoryLeakDetector alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _trackedObjects = [NSMutableSet set];
        _objectStackTraces = [NSMutableDictionary dictionary];
        _isEnabled = NO;
    }
    return self;
}

- (void)startTracking {
    self.isEnabled = YES;
    NSLog(@"Memory leak detection started");
}

- (void)stopTracking {
    self.isEnabled = NO;
    [self.trackedObjects removeAllObjects];
    [self.objectStackTraces removeAllObjects];
    NSLog(@"Memory leak detection stopped");
}

- (void)trackObject:(id)object withStackTrace:(NSString *)stackTrace {
    if (!self.isEnabled || !object) return;
    
    NSValue *objectPointer = [NSValue valueWithPointer:(__bridge const void *)object];
    [self.trackedObjects addObject:objectPointer];
    
    if (stackTrace) {
        self.objectStackTraces[objectPointer] = stackTrace;
    }
}

- (void)untrackObject:(id)object {
    if (!self.isEnabled || !object) return;
    
    NSValue *objectPointer = [NSValue valueWithPointer:(__bridge const void *)object];
    [self.trackedObjects removeObject:objectPointer];
    [self.objectStackTraces removeObjectForKey:objectPointer];
}

- (NSArray<NSDictionary *> *)detectLeaks {
    NSMutableArray<NSDictionary *> *leaks = [NSMutableArray array];
    
    for (NSValue *objectPointer in self.trackedObjects) {
        void *pointer = [objectPointer pointerValue];
        
        // 检查对象是否仍然存在
        if (pointer) {
            NSString *stackTrace = self.objectStackTraces[objectPointer];
            [leaks addObject:@{
                @"object_pointer": objectPointer,
                @"stack_trace": stackTrace ?: @"No stack trace available",
                @"detection_time": [NSDate date]
            }];
        }
    }
    
    return [leaks copy];
}

- (void)generateLeakReport {
    NSArray<NSDictionary *> *leaks = [self detectLeaks];
    
    if (leaks.count == 0) {
        NSLog(@"No memory leaks detected");
        return;
    }
    
    NSLog(@"Memory Leak Report - Found %lu potential leaks:", (unsigned long)leaks.count);
    
    for (NSInteger i = 0; i < leaks.count; i++) {
        NSDictionary *leak = leaks[i];
        NSLog(@"Leak #%ld:", (long)(i + 1));
        NSLog(@"  Object: %@", leak[@"object_pointer"]);
        NSLog(@"  Stack Trace: %@", leak[@"stack_trace"]);
        NSLog(@"  Detection Time: %@", leak[@"detection_time"]);
        NSLog(@"---");
    }
}

@end
```

### 内存池管理

```objc
@interface MemoryPool : NSObject

@property (nonatomic, assign) NSUInteger poolSize;
@property (nonatomic, assign) NSUInteger objectSize;
@property (nonatomic, strong) NSMutableArray<NSValue *> *availableObjects;
@property (nonatomic, strong) NSMutableSet<NSValue *> *usedObjects;
@property (nonatomic, assign) void *memoryBlock;

- (instancetype)initWithPoolSize:(NSUInteger)poolSize objectSize:(NSUInteger)objectSize;
- (void *)allocateObject;
- (void)deallocateObject:(void *)object;
- (void)reset;
- (NSDictionary *)getStatistics;

@end

@implementation MemoryPool

- (instancetype)initWithPoolSize:(NSUInteger)poolSize objectSize:(NSUInteger)objectSize {
    self = [super init];
    if (self) {
        _poolSize = poolSize;
        _objectSize = objectSize;
        _availableObjects = [NSMutableArray array];
        _usedObjects = [NSMutableSet set];
        
        // 分配内存块
        _memoryBlock = malloc(poolSize * objectSize);
        if (!_memoryBlock) {
            return nil;
        }
        
        // 初始化可用对象列表
        for (NSUInteger i = 0; i < poolSize; i++) {
            void *objectPtr = (char *)_memoryBlock + (i * objectSize);
            NSValue *objectValue = [NSValue valueWithPointer:objectPtr];
            [_availableObjects addObject:objectValue];
        }
    }
    return self;
}

- (void)dealloc {
    if (_memoryBlock) {
        free(_memoryBlock);
    }
}

- (void *)allocateObject {
    if (self.availableObjects.count == 0) {
        NSLog(@"Memory pool exhausted");
        return NULL;
    }
    
    NSValue *objectValue = [self.availableObjects lastObject];
    [self.availableObjects removeLastObject];
    [self.usedObjects addObject:objectValue];
    
    void *objectPtr = [objectValue pointerValue];
    memset(objectPtr, 0, self.objectSize); // 清零内存
    
    return objectPtr;
}

- (void)deallocateObject:(void *)object {
    if (!object) return;
    
    NSValue *objectValue = [NSValue valueWithPointer:object];
    
    if (![self.usedObjects containsObject:objectValue]) {
        NSLog(@"Attempting to deallocate object not from this pool");
        return;
    }
    
    [self.usedObjects removeObject:objectValue];
    [self.availableObjects addObject:objectValue];
}

- (void)reset {
    [self.usedObjects removeAllObjects];
    [self.availableObjects removeAllObjects];
    
    // 重新初始化可用对象列表
    for (NSUInteger i = 0; i < self.poolSize; i++) {
        void *objectPtr = (char *)self.memoryBlock + (i * self.objectSize);
        NSValue *objectValue = [NSValue valueWithPointer:objectPtr];
        [self.availableObjects addObject:objectValue];
    }
}

- (NSDictionary *)getStatistics {
    return @{
        @"pool_size": @(self.poolSize),
        @"object_size": @(self.objectSize),
        @"available_objects": @(self.availableObjects.count),
        @"used_objects": @(self.usedObjects.count),
        @"utilization_rate": @((double)self.usedObjects.count / self.poolSize * 100)
    };
}

@end
```

### 智能缓存管理

```objc
@interface SmartCache : NSObject

@property (nonatomic, assign) NSUInteger maxMemoryUsage; // bytes
@property (nonatomic, assign) NSUInteger maxItemCount;
@property (nonatomic, strong) NSMutableDictionary<NSString *, id> *cache;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSDate *> *accessTimes;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSNumber *> *accessCounts;
@property (nonatomic, strong) NSMutableDictionary<NSString *, NSNumber *> *itemSizes;

- (instancetype)initWithMaxMemoryUsage:(NSUInteger)maxMemoryUsage maxItemCount:(NSUInteger)maxItemCount;
- (void)setObject:(id)object forKey:(NSString *)key;
- (id)objectForKey:(NSString *)key;
- (void)removeObjectForKey:(NSString *)key;
- (void)removeAllObjects;
- (void)performCleanup;
- (NSDictionary *)getCacheStatistics;

@end

@implementation SmartCache

- (instancetype)initWithMaxMemoryUsage:(NSUInteger)maxMemoryUsage maxItemCount:(NSUInteger)maxItemCount {
    self = [super init];
    if (self) {
        _maxMemoryUsage = maxMemoryUsage;
        _maxItemCount = maxItemCount;
        _cache = [NSMutableDictionary dictionary];
        _accessTimes = [NSMutableDictionary dictionary];
        _accessCounts = [NSMutableDictionary dictionary];
        _itemSizes = [NSMutableDictionary dictionary];
        
        // 设置内存警告监听
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(handleMemoryWarning:)
                                                     name:UIApplicationDidReceiveMemoryWarningNotification
                                                   object:nil];
    }
    return self;
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

- (void)setObject:(id)object forKey:(NSString *)key {
    if (!object || !key) return;
    
    NSUInteger objectSize = [self estimateObjectSize:object];
    
    // 检查是否需要清理缓存
    if ([self shouldPerformCleanupForNewObjectSize:objectSize]) {
        [self performCleanup];
    }
    
    // 存储对象和元数据
    self.cache[key] = object;
    self.accessTimes[key] = [NSDate date];
    self.accessCounts[key] = @(1);
    self.itemSizes[key] = @(objectSize);
}

- (id)objectForKey:(NSString *)key {
    if (!key) return nil;
    
    id object = self.cache[key];
    if (object) {
        // 更新访问信息
        self.accessTimes[key] = [NSDate date];
        NSNumber *currentCount = self.accessCounts[key];
        self.accessCounts[key] = @(currentCount.integerValue + 1);
    }
    
    return object;
}

- (void)removeObjectForKey:(NSString *)key {
    if (!key) return;
    
    [self.cache removeObjectForKey:key];
    [self.accessTimes removeObjectForKey:key];
    [self.accessCounts removeObjectForKey:key];
    [self.itemSizes removeObjectForKey:key];
}

- (void)removeAllObjects {
    [self.cache removeAllObjects];
    [self.accessTimes removeAllObjects];
    [self.accessCounts removeAllObjects];
    [self.itemSizes removeAllObjects];
}

- (BOOL)shouldPerformCleanupForNewObjectSize:(NSUInteger)objectSize {
    NSUInteger currentMemoryUsage = [self getCurrentMemoryUsage];
    NSUInteger currentItemCount = self.cache.count;
    
    return (currentMemoryUsage + objectSize > self.maxMemoryUsage) ||
           (currentItemCount >= self.maxItemCount);
}

- (void)performCleanup {
    NSUInteger targetMemoryUsage = self.maxMemoryUsage * 0.8; // 清理到80%
    NSUInteger targetItemCount = self.maxItemCount * 0.8;
    
    // 创建排序后的键列表（基于LRU + 访问频率）
    NSArray<NSString *> *sortedKeys = [self.cache.allKeys sortedArrayUsingComparator:^NSComparisonResult(NSString *key1, NSString *key2) {
        NSDate *time1 = self.accessTimes[key1];
        NSDate *time2 = self.accessTimes[key2];
        NSNumber *count1 = self.accessCounts[key1];
        NSNumber *count2 = self.accessCounts[key2];
        
        // 计算综合分数（时间新近度 + 访问频率）
        NSTimeInterval now = [[NSDate date] timeIntervalSince1970];
        double score1 = (now - time1.timeIntervalSince1970) / count1.doubleValue;
        double score2 = (now - time2.timeIntervalSince1970) / count2.doubleValue;
        
        return [@(score1) compare:@(score2)];
    }];
    
    // 移除最不常用的项目
    NSUInteger currentMemoryUsage = [self getCurrentMemoryUsage];
    NSUInteger currentItemCount = self.cache.count;
    
    for (NSString *key in sortedKeys) {
        if (currentMemoryUsage <= targetMemoryUsage && currentItemCount <= targetItemCount) {
            break;
        }
        
        NSNumber *itemSize = self.itemSizes[key];
        currentMemoryUsage -= itemSize.unsignedIntegerValue;
        currentItemCount--;
        
        [self removeObjectForKey:key];
    }
    
    NSLog(@"Cache cleanup completed. Items: %lu, Memory: %lu bytes", 
          (unsigned long)self.cache.count, (unsigned long)[self getCurrentMemoryUsage]);
}

- (NSUInteger)getCurrentMemoryUsage {
    NSUInteger totalSize = 0;
    for (NSNumber *size in self.itemSizes.allValues) {
        totalSize += size.unsignedIntegerValue;
    }
    return totalSize;
}

- (NSUInteger)estimateObjectSize:(id)object {
    if ([object isKindOfClass:[NSString class]]) {
        return [(NSString *)object lengthOfBytesUsingEncoding:NSUTF8StringEncoding];
    } else if ([object isKindOfClass:[NSData class]]) {
        return [(NSData *)object length];
    } else if ([object isKindOfClass:[UIImage class]]) {
        UIImage *image = (UIImage *)object;
        return image.size.width * image.size.height * 4; // 假设RGBA
    } else if ([object isKindOfClass:[NSArray class]]) {
        return [(NSArray *)object count] * sizeof(id);
    } else if ([object isKindOfClass:[NSDictionary class]]) {
        return [(NSDictionary *)object count] * sizeof(id) * 2;
    }
    
    return sizeof(id); // 默认大小
}

- (void)handleMemoryWarning:(NSNotification *)notification {
    NSLog(@"Received memory warning, performing aggressive cache cleanup");
    
    // 在内存警告时更激进地清理缓存
    NSUInteger targetMemoryUsage = self.maxMemoryUsage * 0.5;
    NSUInteger targetItemCount = self.maxItemCount * 0.5;
    
    NSArray<NSString *> *sortedKeys = [self.cache.allKeys sortedArrayUsingComparator:^NSComparisonResult(NSString *key1, NSString *key2) {
        NSDate *time1 = self.accessTimes[key1];
        NSDate *time2 = self.accessTimes[key2];
        return [time1 compare:time2];
    }];
    
    NSUInteger currentMemoryUsage = [self getCurrentMemoryUsage];
    NSUInteger currentItemCount = self.cache.count;
    
    for (NSString *key in sortedKeys) {
        if (currentMemoryUsage <= targetMemoryUsage && currentItemCount <= targetItemCount) {
            break;
        }
        
        NSNumber *itemSize = self.itemSizes[key];
        currentMemoryUsage -= itemSize.unsignedIntegerValue;
        currentItemCount--;
        
        [self removeObjectForKey:key];
    }
}

- (NSDictionary *)getCacheStatistics {
    NSUInteger currentMemoryUsage = [self getCurrentMemoryUsage];
    NSUInteger currentItemCount = self.cache.count;
    
    return @{
        @"current_item_count": @(currentItemCount),
        @"max_item_count": @(self.maxItemCount),
        @"current_memory_usage": @(currentMemoryUsage),
        @"max_memory_usage": @(self.maxMemoryUsage),
        @"memory_utilization_rate": @((double)currentMemoryUsage / self.maxMemoryUsage * 100),
        @"item_utilization_rate": @((double)currentItemCount / self.maxItemCount * 100)
    };
}

@end
```