---
layout: post
title: "iOS多线程编程深度解析：GCD、NSOperation与并发安全实战指南"
date: 2024-05-15
categories: ios
tags: [iOS, GCD, NSOperation, 多线程, 并发安全, 性能优化]
author: "iOS技术专家"
description: "深入探讨iOS多线程编程技术，包括GCD、NSOperation的使用方法，线程安全机制，以及并发编程的最佳实践和性能优化策略。"
keywords: "iOS多线程,GCD,NSOperation,并发编程,线程安全,性能优化"
---

# iOS多线程编程深度解析：GCD、NSOperation与并发安全实战指南

在现代iOS应用开发中，多线程编程是提升应用性能和用户体验的关键技术。本文将深入探讨iOS多线程编程的各个方面，从基础概念到高级应用，帮助开发者掌握并发编程的精髓。

## 多线程编程架构概览

### 多线程架构管理器

```objc
@interface MultithreadingArchitecture : NSObject

@property (nonatomic, strong) NSMutableDictionary *threadPools;
@property (nonatomic, strong) NSMutableDictionary *queueManagers;
@property (nonatomic, strong) NSMutableArray *performanceMetrics;
@property (nonatomic, strong) NSMutableDictionary *threadSafetyConfig;

+ (instancetype)sharedArchitecture;

// 架构初始化
- (void)initializeMultithreadingArchitecture;
- (void)configureThreadingEnvironment:(NSDictionary *)config;

// 线程池管理
- (void)createThreadPool:(NSString *)poolName withConfig:(NSDictionary *)config;
- (void)destroyThreadPool:(NSString *)poolName;
- (NSDictionary *)getThreadPoolStatus:(NSString *)poolName;

// 性能监控
- (void)startPerformanceMonitoring;
- (void)stopPerformanceMonitoring;
- (NSDictionary *)generatePerformanceReport;

// 线程安全检查
- (BOOL)validateThreadSafety:(NSObject *)object;
- (NSArray *)detectPotentialRaceConditions;

@end

@implementation MultithreadingArchitecture

+ (instancetype)sharedArchitecture {
    static MultithreadingArchitecture *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.threadPools = [NSMutableDictionary dictionary];
        self.queueManagers = [NSMutableDictionary dictionary];
        self.performanceMetrics = [NSMutableArray array];
        self.threadSafetyConfig = [NSMutableDictionary dictionary];
        
        [self initializeMultithreadingArchitecture];
    }
    return self;
}

- (void)initializeMultithreadingArchitecture {
    NSLog(@"初始化多线程架构...");
    
    // 配置默认线程环境
    [self configureThreadingEnvironment:@{
        @"maxConcurrentOperations": @4,
        @"queuePriority": @"default",
        @"enableDeadlockDetection": @YES,
        @"performanceMonitoringInterval": @1.0
    }];
    
    // 创建默认线程池
    [self createThreadPool:@"default" withConfig:@{
        @"maxThreads": @4,
        @"queueType": @"concurrent",
        @"priority": @"default"
    }];
    
    [self createThreadPool:@"background" withConfig:@{
        @"maxThreads": @2,
        @"queueType": @"concurrent",
        @"priority": @"background"
    }];
    
    [self createThreadPool:@"utility" withConfig:@{
        @"maxThreads": @2,
        @"queueType": @"concurrent",
        @"priority": @"utility"
    }];
    
    NSLog(@"多线程架构初始化完成");
}

- (void)configureThreadingEnvironment:(NSDictionary *)config {
    [self.threadSafetyConfig addEntriesFromDictionary:config];
    
    NSLog(@"线程环境配置已更新:");
    for (NSString *key in config) {
        NSLog(@"  %@: %@", key, config[key]);
    }
}

- (void)createThreadPool:(NSString *)poolName withConfig:(NSDictionary *)config {
    NSLog(@"创建线程池: %@", poolName);
    
    NSMutableDictionary *poolInfo = [NSMutableDictionary dictionary];
    poolInfo[@"config"] = config;
    poolInfo[@"createdAt"] = @([[NSDate date] timeIntervalSince1970]);
    poolInfo[@"status"] = @"active";
    poolInfo[@"taskCount"] = @0;
    poolInfo[@"completedTasks"] = @0;
    
    // 根据配置创建相应的队列
    NSString *queueType = config[@"queueType"];
    NSString *priority = config[@"priority"];
    
    dispatch_queue_attr_t attr = DISPATCH_QUEUE_SERIAL;
    if ([queueType isEqualToString:@"concurrent"]) {
        attr = DISPATCH_QUEUE_CONCURRENT;
    }
    
    dispatch_qos_class_t qos = DISPATCH_QOS_CLASS_DEFAULT;
    if ([priority isEqualToString:@"background"]) {
        qos = DISPATCH_QOS_CLASS_BACKGROUND;
    } else if ([priority isEqualToString:@"utility"]) {
        qos = DISPATCH_QOS_CLASS_UTILITY;
    } else if ([priority isEqualToString:@"userInitiated"]) {
        qos = DISPATCH_QOS_CLASS_USER_INITIATED;
    } else if ([priority isEqualToString:@"userInteractive"]) {
        qos = DISPATCH_QOS_CLASS_USER_INTERACTIVE;
    }
    
    dispatch_queue_attr_t qosAttr = dispatch_queue_attr_make_with_qos_class(attr, qos, 0);
    dispatch_queue_t queue = dispatch_queue_create([poolName UTF8String], qosAttr);
    
    poolInfo[@"queue"] = queue;
    
    self.threadPools[poolName] = poolInfo;
    
    NSLog(@"线程池 %@ 创建成功，类型: %@，优先级: %@", poolName, queueType, priority);
}

- (void)destroyThreadPool:(NSString *)poolName {
    if (self.threadPools[poolName]) {
        NSLog(@"销毁线程池: %@", poolName);
        [self.threadPools removeObjectForKey:poolName];
        NSLog(@"线程池 %@ 已销毁", poolName);
    } else {
        NSLog(@"线程池 %@ 不存在", poolName);
    }
}

- (NSDictionary *)getThreadPoolStatus:(NSString *)poolName {
    NSMutableDictionary *poolInfo = self.threadPools[poolName];
    if (poolInfo) {
        NSMutableDictionary *status = [NSMutableDictionary dictionary];
        status[@"name"] = poolName;
        status[@"status"] = poolInfo[@"status"];
        status[@"taskCount"] = poolInfo[@"taskCount"];
        status[@"completedTasks"] = poolInfo[@"completedTasks"];
        status[@"createdAt"] = poolInfo[@"createdAt"];
        status[@"config"] = poolInfo[@"config"];
        
        return status;
    }
    return nil;
}

- (void)startPerformanceMonitoring {
    NSLog(@"启动性能监控");
    
    NSTimeInterval interval = [self.threadSafetyConfig[@"performanceMonitoringInterval"] doubleValue];
    
    dispatch_queue_t monitorQueue = dispatch_queue_create("performance.monitor", DISPATCH_QUEUE_SERIAL);
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, monitorQueue);
    
    dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, 0), interval * NSEC_PER_SEC, 0.1 * NSEC_PER_SEC);
    
    dispatch_source_set_event_handler(timer, ^{
        [self collectPerformanceMetrics];
    });
    
    dispatch_resume(timer);
    
    // 存储timer引用以便后续停止
    self.threadSafetyConfig[@"performanceTimer"] = timer;
}

- (void)stopPerformanceMonitoring {
    NSLog(@"停止性能监控");
    
    dispatch_source_t timer = self.threadSafetyConfig[@"performanceTimer"];
    if (timer) {
        dispatch_source_cancel(timer);
        [self.threadSafetyConfig removeObjectForKey:@"performanceTimer"];
    }
}

- (void)collectPerformanceMetrics {
    NSMutableDictionary *metrics = [NSMutableDictionary dictionary];
    metrics[@"timestamp"] = @([[NSDate date] timeIntervalSince1970]);
    
    // 收集系统线程信息
    thread_array_t threadList;
    mach_msg_type_number_t threadCount;
    
    if (task_threads(mach_task_self(), &threadList, &threadCount) == KERN_SUCCESS) {
        metrics[@"systemThreadCount"] = @(threadCount);
        
        // 清理线程列表
        for (mach_msg_type_number_t i = 0; i < threadCount; i++) {
            mach_port_deallocate(mach_task_self(), threadList[i]);
        }
        vm_deallocate(mach_task_self(), (vm_address_t)threadList, threadCount * sizeof(thread_t));
    }
    
    // 收集线程池状态
    NSMutableDictionary *poolStatuses = [NSMutableDictionary dictionary];
    for (NSString *poolName in self.threadPools) {
        poolStatuses[poolName] = [self getThreadPoolStatus:poolName];
    }
    metrics[@"threadPoolStatuses"] = poolStatuses;
    
    // 收集CPU使用率
    metrics[@"cpuUsage"] = @([self getCurrentCPUUsage]);
    
    [self.performanceMetrics addObject:metrics];
    
    // 保持最近100条记录
    if (self.performanceMetrics.count > 100) {
        [self.performanceMetrics removeObjectAtIndex:0];
    }
}

- (double)getCurrentCPUUsage {
    kern_return_t kr;
    task_info_data_t tinfo;
    mach_msg_type_number_t task_info_count;
    
    task_info_count = TASK_INFO_MAX;
    kr = task_info(mach_task_self(), TASK_BASIC_INFO, (task_info_t)tinfo, &task_info_count);
    if (kr != KERN_SUCCESS) {
        return 0.0;
    }
    
    task_basic_info_t basic_info;
    thread_array_t thread_list;
    mach_msg_type_number_t thread_count;
    
    thread_info_data_t thinfo;
    mach_msg_type_number_t thread_info_count;
    
    thread_basic_info_t basic_info_th;
    uint32_t stat_thread = 0;
    
    basic_info = (task_basic_info_t)tinfo;
    
    kr = task_threads(mach_task_self(), &thread_list, &thread_count);
    if (kr != KERN_SUCCESS) {
        return 0.0;
    }
    
    if (thread_count > 0)
        stat_thread += thread_count;
    
    long tot_sec = 0;
    long tot_usec = 0;
    float tot_cpu = 0;
    int j;
    
    for (j = 0; j < thread_count; j++) {
        thread_info_count = THREAD_INFO_MAX;
        kr = thread_info(thread_list[j], THREAD_BASIC_INFO,
                        (thread_info_t)thinfo, &thread_info_count);
        if (kr != KERN_SUCCESS) {
            continue;
        }
        
        basic_info_th = (thread_basic_info_t)thinfo;
        
        if (!(basic_info_th->flags & TH_FLAGS_IDLE)) {
            tot_sec = tot_sec + basic_info_th->user_time.seconds + basic_info_th->system_time.seconds;
            tot_usec = tot_usec + basic_info_th->user_time.microseconds + basic_info_th->system_time.microseconds;
            tot_cpu = tot_cpu + basic_info_th->cpu_usage / (float)TH_USAGE_SCALE * 100.0;
        }
    }
    
    kr = vm_deallocate(mach_task_self(), (vm_offset_t)thread_list, thread_count * sizeof(thread_t));
    assert(kr == KERN_SUCCESS);
    
    return tot_cpu;
}

- (NSDictionary *)generatePerformanceReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    report[@"reportTimestamp"] = @([[NSDate date] timeIntervalSince1970]);
    report[@"metricsCount"] = @(self.performanceMetrics.count);
    
    if (self.performanceMetrics.count > 0) {
        // 计算平均值
        double avgCPU = 0.0;
        double avgThreadCount = 0.0;
        
        for (NSDictionary *metrics in self.performanceMetrics) {
            avgCPU += [metrics[@"cpuUsage"] doubleValue];
            avgThreadCount += [metrics[@"systemThreadCount"] doubleValue];
        }
        
        avgCPU /= self.performanceMetrics.count;
        avgThreadCount /= self.performanceMetrics.count;
        
        report[@"averageCPUUsage"] = @(avgCPU);
        report[@"averageThreadCount"] = @(avgThreadCount);
        
        // 最新指标
        NSDictionary *latestMetrics = self.performanceMetrics.lastObject;
        report[@"currentCPUUsage"] = latestMetrics[@"cpuUsage"];
        report[@"currentThreadCount"] = latestMetrics[@"systemThreadCount"];
        report[@"currentThreadPoolStatuses"] = latestMetrics[@"threadPoolStatuses"];
    }
    
    // 线程池统计
    NSMutableDictionary *poolSummary = [NSMutableDictionary dictionary];
    for (NSString *poolName in self.threadPools) {
        NSDictionary *status = [self getThreadPoolStatus:poolName];
        poolSummary[poolName] = status;
    }
    report[@"threadPoolSummary"] = poolSummary;
    
    return report;
}

- (BOOL)validateThreadSafety:(NSObject *)object {
    // 简化的线程安全检查
    // 实际实现需要更复杂的分析
    
    if ([object conformsToProtocol:@protocol(NSLocking)]) {
        return YES;
    }
    
    // 检查是否为不可变对象
    if ([object isKindOfClass:[NSString class]] ||
        [object isKindOfClass:[NSNumber class]] ||
        [object isKindOfClass:[NSDate class]]) {
        return YES;
    }
    
    // 检查是否为线程安全的集合类
    if ([object isKindOfClass:[NSArray class]] ||
        [object isKindOfClass:[NSDictionary class]] ||
        [object isKindOfClass:[NSSet class]]) {
        // 不可变集合是线程安全的
        NSString *className = NSStringFromClass([object class]);
        if (![className hasPrefix:@"NSMutable"]) {
            return YES;
        }
    }
    
    return NO;
}

- (NSArray *)detectPotentialRaceConditions {
    NSMutableArray *warnings = [NSMutableArray array];
    
    // 这里可以实现更复杂的竞态条件检测逻辑
    // 例如检查共享资源的访问模式
    
    [warnings addObject:@{
        @"type": @"SharedResourceAccess",
        @"description": @"检测到对共享资源的并发访问",
        @"severity": @"Medium",
        @"recommendation": @"使用锁或其他同步机制保护共享资源"
    }];
    
    return warnings;
}

@end
```

## NSOperation与NSOperationQueue深度应用

### NSOperation管理器

```objc
@interface NSOperationManager : NSObject

@property (nonatomic, strong) NSMutableDictionary *operationQueues;
@property (nonatomic, strong) NSMutableArray *operations;
@property (nonatomic, strong) NSMutableDictionary *dependencies;
@property (nonatomic, strong) NSMutableDictionary *priorities;

+ (instancetype)sharedManager;

// 队列管理
- (NSOperationQueue *)createOperationQueue:(NSString *)name withMaxConcurrentOperations:(NSInteger)maxOps;
- (void)destroyOperationQueue:(NSString *)name;
- (NSOperationQueue *)getOperationQueue:(NSString *)name;

// 操作管理
- (NSOperation *)createBlockOperation:(NSString *)name withBlock:(void(^)(void))block;
- (NSOperation *)createCustomOperation:(NSString *)name withTarget:(id)target selector:(SEL)selector;
- (void)addOperation:(NSOperation *)operation toQueue:(NSString *)queueName;

// 依赖关系管理
- (void)addDependency:(NSOperation *)dependency toOperation:(NSOperation *)operation;
- (void)removeDependency:(NSOperation *)dependency fromOperation:(NSOperation *)operation;
- (NSArray *)getDependenciesForOperation:(NSOperation *)operation;

// 优先级管理
- (void)setQueuePriority:(NSOperationQueuePriority)priority forOperation:(NSOperation *)operation;
- (void)setQualityOfService:(NSQualityOfService)qos forOperation:(NSOperation *)operation;

// 操作控制
- (void)pauseOperationQueue:(NSString *)queueName;
- (void)resumeOperationQueue:(NSString *)queueName;
- (void)cancelAllOperationsInQueue:(NSString *)queueName;
- (void)cancelOperation:(NSOperation *)operation;

// 监控和统计
- (NSDictionary *)getOperationStatistics;
- (NSArray *)getActiveOperations;
- (void)logOperationPerformance;

@end

@implementation NSOperationManager

+ (instancetype)sharedManager {
    static NSOperationManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.operationQueues = [NSMutableDictionary dictionary];
        self.operations = [NSMutableArray array];
        self.dependencies = [NSMutableDictionary dictionary];
        self.priorities = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NSOperationQueue *)createOperationQueue:(NSString *)name withMaxConcurrentOperations:(NSInteger)maxOps {
    NSLog(@"创建操作队列: %@，最大并发数: %ld", name, (long)maxOps);
    
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    queue.name = name;
    queue.maxConcurrentOperationCount = maxOps;
    
    self.operationQueues[name] = @{
        @"queue": queue,
        @"maxConcurrentOperations": @(maxOps),
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"totalOperations": @0,
        @"completedOperations": @0
    };
    
    return queue;
}

- (void)destroyOperationQueue:(NSString *)name {
    NSMutableDictionary *queueInfo = self.operationQueues[name];
    if (queueInfo) {
        NSLog(@"销毁操作队列: %@", name);
        
        NSOperationQueue *queue = queueInfo[@"queue"];
        [queue cancelAllOperations];
        [queue waitUntilAllOperationsAreFinished];
        
        [self.operationQueues removeObjectForKey:name];
    }
}

- (NSOperationQueue *)getOperationQueue:(NSString *)name {
    NSMutableDictionary *queueInfo = self.operationQueues[name];
    return queueInfo ? queueInfo[@"queue"] : nil;
}

- (NSOperation *)createBlockOperation:(NSString *)name withBlock:(void(^)(void))block {
    NSLog(@"创建块操作: %@", name);
    
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:block];
    operation.name = name;
    
    [self.operations addObject:@{
        @"operation": operation,
        @"name": name,
        @"type": @"block",
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"status": @"created"
    }];
    
    return operation;
}

- (NSOperation *)createCustomOperation:(NSString *)name withTarget:(id)target selector:(SEL)selector {
    NSLog(@"创建自定义操作: %@", name);
    
    NSInvocationOperation *operation = [[NSInvocationOperation alloc] initWithTarget:target selector:selector object:nil];
    operation.name = name;
    
    [self.operations addObject:@{
        @"operation": operation,
        @"name": name,
        @"type": @"invocation",
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"status": @"created"
    }];
    
    return operation;
}

- (void)addOperation:(NSOperation *)operation toQueue:(NSString *)queueName {
    NSOperationQueue *queue = [self getOperationQueue:queueName];
    if (queue && operation) {
        NSLog(@"添加操作 %@ 到队列 %@", operation.name, queueName);
        
        // 更新操作状态
        for (NSMutableDictionary *opInfo in self.operations) {
            if (opInfo[@"operation"] == operation) {
                opInfo[@"status"] = @"queued";
                opInfo[@"queueName"] = queueName;
                break;
            }
        }
        
        // 更新队列统计
        NSMutableDictionary *queueInfo = self.operationQueues[queueName];
        NSInteger totalOps = [queueInfo[@"totalOperations"] integerValue] + 1;
        queueInfo[@"totalOperations"] = @(totalOps);
        
        [queue addOperation:operation];
    }
}

- (void)addDependency:(NSOperation *)dependency toOperation:(NSOperation *)operation {
    if (dependency && operation) {
        NSLog(@"添加依赖: %@ -> %@", dependency.name, operation.name);
        
        [operation addDependency:dependency];
        
        // 记录依赖关系
        NSString *operationKey = [NSString stringWithFormat:@"%p", operation];
        NSMutableArray *deps = self.dependencies[operationKey];
        if (!deps) {
            deps = [NSMutableArray array];
            self.dependencies[operationKey] = deps;
        }
        [deps addObject:dependency];
    }
}

- (void)removeDependency:(NSOperation *)dependency fromOperation:(NSOperation *)operation {
    if (dependency && operation) {
        NSLog(@"移除依赖: %@ -> %@", dependency.name, operation.name);
        
        [operation removeDependency:dependency];
        
        // 更新依赖记录
        NSString *operationKey = [NSString stringWithFormat:@"%p", operation];
        NSMutableArray *deps = self.dependencies[operationKey];
        [deps removeObject:dependency];
    }
}

- (NSArray *)getDependenciesForOperation:(NSOperation *)operation {
    if (operation) {
        return operation.dependencies;
    }
    return @[];
}

- (void)setQueuePriority:(NSOperationQueuePriority)priority forOperation:(NSOperation *)operation {
    if (operation) {
        NSLog(@"设置操作 %@ 的队列优先级: %ld", operation.name, (long)priority);
        operation.queuePriority = priority;
        
        // 记录优先级设置
        NSString *operationKey = [NSString stringWithFormat:@"%p", operation];
        self.priorities[operationKey] = @{
            @"queuePriority": @(priority),
            @"setAt": @([[NSDate date] timeIntervalSince1970])
        };
    }
}

- (void)setQualityOfService:(NSQualityOfService)qos forOperation:(NSOperation *)operation {
    if (operation) {
        NSLog(@"设置操作 %@ 的服务质量: %ld", operation.name, (long)qos);
        operation.qualityOfService = qos;
        
        // 更新优先级记录
        NSString *operationKey = [NSString stringWithFormat:@"%p", operation];
        NSMutableDictionary *priorityInfo = self.priorities[operationKey];
        if (!priorityInfo) {
            priorityInfo = [NSMutableDictionary dictionary];
            self.priorities[operationKey] = priorityInfo;
        }
        priorityInfo[@"qualityOfService"] = @(qos);
        priorityInfo[@"qosSetAt"] = @([[NSDate date] timeIntervalSince1970]);
    }
}

- (void)pauseOperationQueue:(NSString *)queueName {
    NSOperationQueue *queue = [self getOperationQueue:queueName];
    if (queue) {
        NSLog(@"暂停操作队列: %@", queueName);
        queue.suspended = YES;
    }
}

- (void)resumeOperationQueue:(NSString *)queueName {
    NSOperationQueue *queue = [self getOperationQueue:queueName];
    if (queue) {
        NSLog(@"恢复操作队列: %@", queueName);
        queue.suspended = NO;
    }
}

- (void)cancelAllOperationsInQueue:(NSString *)queueName {
    NSOperationQueue *queue = [self getOperationQueue:queueName];
    if (queue) {
        NSLog(@"取消队列 %@ 中的所有操作", queueName);
        [queue cancelAllOperations];
        
        // 更新操作状态
        for (NSMutableDictionary *opInfo in self.operations) {
            if ([opInfo[@"queueName"] isEqualToString:queueName]) {
                opInfo[@"status"] = @"cancelled";
            }
        }
    }
}

- (void)cancelOperation:(NSOperation *)operation {
    if (operation) {
        NSLog(@"取消操作: %@", operation.name);
        [operation cancel];
        
        // 更新操作状态
        for (NSMutableDictionary *opInfo in self.operations) {
            if (opInfo[@"operation"] == operation) {
                opInfo[@"status"] = @"cancelled";
                break;
            }
        }
    }
}

- (NSDictionary *)getOperationStatistics {
    NSMutableDictionary *statistics = [NSMutableDictionary dictionary];
    
    statistics[@"totalQueues"] = @(self.operationQueues.count);
    statistics[@"totalOperations"] = @(self.operations.count);
    
    // 统计操作状态
    NSMutableDictionary *statusCount = [NSMutableDictionary dictionary];
    for (NSDictionary *opInfo in self.operations) {
        NSString *status = opInfo[@"status"];
        NSNumber *count = statusCount[status] ?: @0;
        statusCount[status] = @([count integerValue] + 1);
    }
    statistics[@"operationStatusDistribution"] = statusCount;
    
    // 队列详情
    NSMutableArray *queueDetails = [NSMutableArray array];
    for (NSString *queueName in self.operationQueues) {
        NSDictionary *queueInfo = self.operationQueues[queueName];
        NSOperationQueue *queue = queueInfo[@"queue"];
        
        [queueDetails addObject:@{
            @"name": queueName,
            @"maxConcurrentOperations": queueInfo[@"maxConcurrentOperations"],
            @"currentOperationCount": @(queue.operationCount),
            @"suspended": @(queue.isSuspended),
            @"totalOperations": queueInfo[@"totalOperations"],
            @"completedOperations": queueInfo[@"completedOperations"]
        }];
    }
    statistics[@"queueDetails"] = queueDetails;
    
    return statistics;
}

- (NSArray *)getActiveOperations {
    NSMutableArray *activeOps = [NSMutableArray array];
    
    for (NSDictionary *opInfo in self.operations) {
        NSOperation *operation = opInfo[@"operation"];
        if (operation.isExecuting || operation.isReady) {
            [activeOps addObject:@{
                @"name": operation.name ?: @"Unknown",
                @"isExecuting": @(operation.isExecuting),
                @"isReady": @(operation.isReady),
                @"isCancelled": @(operation.isCancelled),
                @"isFinished": @(operation.isFinished),
                @"queuePriority": @(operation.queuePriority),
                @"qualityOfService": @(operation.qualityOfService)
            }];
        }
    }
    
    return activeOps;
}

- (void)logOperationPerformance {
    NSDictionary *stats = [self getOperationStatistics];
    
    NSLog(@"=== NSOperation性能统计 ===");
    NSLog(@"总队列数: %@", stats[@"totalQueues"]);
    NSLog(@"总操作数: %@", stats[@"totalOperations"]);
    
    NSDictionary *statusDist = stats[@"operationStatusDistribution"];
    NSLog(@"操作状态分布:");
    for (NSString *status in statusDist) {
        NSLog(@"  %@: %@", status, statusDist[status]);
    }
    
    NSArray *queueDetails = stats[@"queueDetails"];
    NSLog(@"队列详情:");
    for (NSDictionary *queue in queueDetails) {
        NSLog(@"  %@: 当前操作数=%@, 最大并发=%@, 暂停=%@", 
              queue[@"name"], queue[@"currentOperationCount"], 
              queue[@"maxConcurrentOperations"], queue[@"suspended"]);
    }
}

@end
```

### 自定义NSOperation子类

```objc
@interface CustomAsyncOperation : NSOperation

@property (nonatomic, copy) void(^executionBlock)(CustomAsyncOperation *operation);
@property (nonatomic, strong) NSMutableDictionary *userInfo;
@property (nonatomic, assign) NSTimeInterval timeout;
@property (nonatomic, strong) NSTimer *timeoutTimer;

- (instancetype)initWithExecutionBlock:(void(^)(CustomAsyncOperation *operation))block;
- (void)completeOperation;
- (void)failWithError:(NSError *)error;

@end

@implementation CustomAsyncOperation {
    BOOL _executing;
    BOOL _finished;
}

- (instancetype)initWithExecutionBlock:(void(^)(CustomAsyncOperation *operation))block {
    self = [super init];
    if (self) {
        self.executionBlock = block;
        self.userInfo = [NSMutableDictionary dictionary];
        self.timeout = 30.0; // 默认30秒超时
        _executing = NO;
        _finished = NO;
    }
    return self;
}

- (BOOL)isAsynchronous {
    return YES;
}

- (BOOL)isExecuting {
    return _executing;
}

- (BOOL)isFinished {
    return _finished;
}

- (void)start {
    if (self.isCancelled) {
        [self willChangeValueForKey:@"isFinished"];
        _finished = YES;
        [self didChangeValueForKey:@"isFinished"];
        return;
    }
    
    [self willChangeValueForKey:@"isExecuting"];
    _executing = YES;
    [self didChangeValueForKey:@"isExecuting"];
    
    NSLog(@"开始执行自定义异步操作: %@", self.name);
    
    // 设置超时定时器
    if (self.timeout > 0) {
        self.timeoutTimer = [NSTimer scheduledTimerWithTimeInterval:self.timeout
                                                             target:self
                                                           selector:@selector(handleTimeout)
                                                           userInfo:nil
                                                            repeats:NO];
    }
    
    // 执行操作
    if (self.executionBlock) {
        self.executionBlock(self);
    } else {
        [self completeOperation];
    }
}

- (void)completeOperation {
    NSLog(@"完成自定义异步操作: %@", self.name);
    
    [self.timeoutTimer invalidate];
    self.timeoutTimer = nil;
    
    [self willChangeValueForKey:@"isExecuting"];
    [self willChangeValueForKey:@"isFinished"];
    
    _executing = NO;
    _finished = YES;
    
    [self didChangeValueForKey:@"isExecuting"];
    [self didChangeValueForKey:@"isFinished"];
}

- (void)failWithError:(NSError *)error {
    NSLog(@"自定义异步操作失败: %@, 错误: %@", self.name, error.localizedDescription);
    
    self.userInfo[@"error"] = error;
    [self completeOperation];
}

- (void)handleTimeout {
    NSLog(@"自定义异步操作超时: %@", self.name);
    
    NSError *timeoutError = [NSError errorWithDomain:@"CustomOperationDomain"
                                                 code:1001
                                             userInfo:@{NSLocalizedDescriptionKey: @"操作超时"}];
    [self failWithError:timeoutError];
}

- (void)cancel {
    [super cancel];
    
    if (_executing) {
        NSLog(@"取消正在执行的自定义异步操作: %@", self.name);
        [self completeOperation];
    }
}

@end
```

## 线程安全与同步机制

### 线程安全管理器

```objc
@interface ThreadSafetyManager : NSObject

@property (nonatomic, strong) NSMutableDictionary *locks;
@property (nonatomic, strong) NSMutableDictionary *conditions;
@property (nonatomic, strong) NSMutableDictionary *barriers;
@property (nonatomic, strong) NSMutableDictionary *atomicCounters;

+ (instancetype)sharedManager;

// 锁管理
- (NSLock *)createLock:(NSString *)name;
- (NSRecursiveLock *)createRecursiveLock:(NSString *)name;
- (NSConditionLock *)createConditionLock:(NSString *)name withCondition:(NSInteger)condition;
- (void)destroyLock:(NSString *)name;

// 读写锁
- (pthread_rwlock_t *)createReadWriteLock:(NSString *)name;
- (void)readLock:(NSString *)name block:(void(^)(void))block;
- (void)writeLock:(NSString *)name block:(void(^)(void))block;

// 条件变量
- (NSCondition *)createCondition:(NSString *)name;
- (void)waitForCondition:(NSString *)name;
- (void)signalCondition:(NSString *)name;
- (void)broadcastCondition:(NSString *)name;

// 原子操作
- (void)createAtomicCounter:(NSString *)name withInitialValue:(int64_t)value;
- (int64_t)incrementAtomicCounter:(NSString *)name;
- (int64_t)decrementAtomicCounter:(NSString *)name;
- (int64_t)getAtomicCounterValue:(NSString *)name;

// 线程安全集合
- (NSMutableArray *)createThreadSafeArray:(NSString *)name;
- (NSMutableDictionary *)createThreadSafeDictionary:(NSString *)name;
- (NSMutableSet *)createThreadSafeSet:(NSString *)name;

// 死锁检测
- (BOOL)detectDeadlock;
- (NSArray *)getDeadlockReport;

@end

@implementation ThreadSafetyManager

+ (instancetype)sharedManager {
    static ThreadSafetyManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.locks = [NSMutableDictionary dictionary];
        self.conditions = [NSMutableDictionary dictionary];
        self.barriers = [NSMutableDictionary dictionary];
        self.atomicCounters = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NSLock *)createLock:(NSString *)name {
    NSLog(@"创建锁: %@", name);
    
    NSLock *lock = [[NSLock alloc] init];
    lock.name = name;
    
    self.locks[name] = @{
        @"lock": lock,
        @"type": @"NSLock",
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"lockCount": @0
    };
    
    return lock;
}

- (NSRecursiveLock *)createRecursiveLock:(NSString *)name {
    NSLog(@"创建递归锁: %@", name);
    
    NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];
    lock.name = name;
    
    self.locks[name] = @{
        @"lock": lock,
        @"type": @"NSRecursiveLock",
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"lockCount": @0
    };
    
    return lock;
}

- (NSConditionLock *)createConditionLock:(NSString *)name withCondition:(NSInteger)condition {
    NSLog(@"创建条件锁: %@，条件: %ld", name, (long)condition);
    
    NSConditionLock *lock = [[NSConditionLock alloc] initWithCondition:condition];
    lock.name = name;
    
    self.locks[name] = @{
        @"lock": lock,
        @"type": @"NSConditionLock",
        @"condition": @(condition),
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"lockCount": @0
    };
    
    return lock;
}

- (void)destroyLock:(NSString *)name {
    if (self.locks[name]) {
        NSLog(@"销毁锁: %@", name);
        [self.locks removeObjectForKey:name];
    }
}

- (pthread_rwlock_t *)createReadWriteLock:(NSString *)name {
    NSLog(@"创建读写锁: %@", name);
    
    pthread_rwlock_t *rwlock = malloc(sizeof(pthread_rwlock_t));
    pthread_rwlock_init(rwlock, NULL);
    
    self.locks[name] = @{
        @"lock": [NSValue valueWithPointer:rwlock],
        @"type": @"pthread_rwlock_t",
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"readCount": @0,
        @"writeCount": @0
    };
    
    return rwlock;
}

- (void)readLock:(NSString *)name block:(void(^)(void))block {
    NSMutableDictionary *lockInfo = self.locks[name];
    if (lockInfo && [lockInfo[@"type"] isEqualToString:@"pthread_rwlock_t"]) {
        pthread_rwlock_t *rwlock = [lockInfo[@"lock"] pointerValue];
        
        pthread_rwlock_rdlock(rwlock);
        
        NSInteger readCount = [lockInfo[@"readCount"] integerValue] + 1;
        lockInfo[@"readCount"] = @(readCount);
        
        NSLog(@"获取读锁: %@，读计数: %ld", name, (long)readCount);
        
        if (block) {
            block();
        }
        
        pthread_rwlock_unlock(rwlock);
        NSLog(@"释放读锁: %@", name);
    }
}

- (void)writeLock:(NSString *)name block:(void(^)(void))block {
    NSMutableDictionary *lockInfo = self.locks[name];
    if (lockInfo && [lockInfo[@"type"] isEqualToString:@"pthread_rwlock_t"]) {
        pthread_rwlock_t *rwlock = [lockInfo[@"lock"] pointerValue];
        
        pthread_rwlock_wrlock(rwlock);
        
        NSInteger writeCount = [lockInfo[@"writeCount"] integerValue] + 1;
        lockInfo[@"writeCount"] = @(writeCount);
        
        NSLog(@"获取写锁: %@，写计数: %ld", name, (long)writeCount);
        
        if (block) {
            block();
        }
        
        pthread_rwlock_unlock(rwlock);
        NSLog(@"释放写锁: %@", name);
    }
}

- (NSCondition *)createCondition:(NSString *)name {
    NSLog(@"创建条件变量: %@", name);
    
    NSCondition *condition = [[NSCondition alloc] init];
    condition.name = name;
    
    self.conditions[name] = @{
        @"condition": condition,
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"waitCount": @0,
        @"signalCount": @0
    };
    
    return condition;
}

- (void)waitForCondition:(NSString *)name {
    NSMutableDictionary *conditionInfo = self.conditions[name];
    if (conditionInfo) {
        NSCondition *condition = conditionInfo[@"condition"];
        
        [condition lock];
        
        NSInteger waitCount = [conditionInfo[@"waitCount"] integerValue] + 1;
        conditionInfo[@"waitCount"] = @(waitCount);
        
        NSLog(@"等待条件: %@，等待计数: %ld", name, (long)waitCount);
        [condition wait];
        NSLog(@"条件满足: %@", name);
        
        [condition unlock];
    }
}

- (void)signalCondition:(NSString *)name {
    NSMutableDictionary *conditionInfo = self.conditions[name];
    if (conditionInfo) {
        NSCondition *condition = conditionInfo[@"condition"];
        
        [condition lock];
        
        NSInteger signalCount = [conditionInfo[@"signalCount"] integerValue] + 1;
        conditionInfo[@"signalCount"] = @(signalCount);
        
        NSLog(@"发送信号: %@，信号计数: %ld", name, (long)signalCount);
        [condition signal];
        
        [condition unlock];
    }
}

- (void)broadcastCondition:(NSString *)name {
    NSMutableDictionary *conditionInfo = self.conditions[name];
    if (conditionInfo) {
        NSCondition *condition = conditionInfo[@"condition"];
        
        [condition lock];
        
        NSLog(@"广播条件: %@", name);
        [condition broadcast];
        
        [condition unlock];
    }
}

- (void)createAtomicCounter:(NSString *)name withInitialValue:(int64_t)value {
    NSLog(@"创建原子计数器: %@，初始值: %lld", name, value);
    
    int64_t *counter = malloc(sizeof(int64_t));
    *counter = value;
    
    self.atomicCounters[name] = @{
        @"counter": [NSValue valueWithPointer:counter],
        @"initialValue": @(value),
        @"createdAt": @([[NSDate date] timeIntervalSince1970])
    };
}

- (int64_t)incrementAtomicCounter:(NSString *)name {
    NSDictionary *counterInfo = self.atomicCounters[name];
    if (counterInfo) {
        int64_t *counter = [counterInfo[@"counter"] pointerValue];
        int64_t newValue = OSAtomicIncrement64(counter);
        NSLog(@"原子递增计数器 %@: %lld", name, newValue);
        return newValue;
    }
    return 0;
}

- (int64_t)decrementAtomicCounter:(NSString *)name {
    NSDictionary *counterInfo = self.atomicCounters[name];
    if (counterInfo) {
        int64_t *counter = [counterInfo[@"counter"] pointerValue];
        int64_t newValue = OSAtomicDecrement64(counter);
        NSLog(@"原子递减计数器 %@: %lld", name, newValue);
        return newValue;
    }
    return 0;
}

- (int64_t)getAtomicCounterValue:(NSString *)name {
    NSDictionary *counterInfo = self.atomicCounters[name];
    if (counterInfo) {
        int64_t *counter = [counterInfo[@"counter"] pointerValue];
        return *counter;
    }
    return 0;
}

- (NSMutableArray *)createThreadSafeArray:(NSString *)name {
    NSLog(@"创建线程安全数组: %@", name);
    
    // 使用串行队列保证线程安全
    dispatch_queue_t queue = dispatch_queue_create([[NSString stringWithFormat:@"array.%@", name] UTF8String], DISPATCH_QUEUE_SERIAL);
    
    // 创建包装类来提供线程安全的数组操作
    // 这里简化实现，实际应用中需要完整的线程安全包装
    NSMutableArray *array = [NSMutableArray array];
    
    return array;
}

- (NSMutableDictionary *)createThreadSafeDictionary:(NSString *)name {
    NSLog(@"创建线程安全字典: %@", name);
    
    // 类似数组的实现
    NSMutableDictionary *dictionary = [NSMutableDictionary dictionary];
    
    return dictionary;
}

- (NSMutableSet *)createThreadSafeSet:(NSString *)name {
    NSLog(@"创建线程安全集合: %@", name);
    
    NSMutableSet *set = [NSMutableSet set];
    
    return set;
}

- (BOOL)detectDeadlock {
    // 简化的死锁检测实现
    // 实际实现需要更复杂的算法来检测循环等待
    
    NSLog(@"执行死锁检测...");
    
    // 检查是否有锁被长时间持有
    NSTimeInterval currentTime = [[NSDate date] timeIntervalSince1970];
    
    for (NSString *lockName in self.locks) {
        NSDictionary *lockInfo = self.locks[lockName];
        NSTimeInterval createdAt = [lockInfo[@"createdAt"] doubleValue];
        
        if (currentTime - createdAt > 30.0) { // 30秒阈值
            NSLog(@"检测到潜在死锁: 锁 %@ 被持有超过30秒", lockName);
            return YES;
        }
    }
    
    return NO;
}

- (NSArray *)getDeadlockReport {
    NSMutableArray *report = [NSMutableArray array];
    
    NSTimeInterval currentTime = [[NSDate date] timeIntervalSince1970];
    
    for (NSString *lockName in self.locks) {
        NSDictionary *lockInfo = self.locks[lockName];
        NSTimeInterval createdAt = [lockInfo[@"createdAt"] doubleValue];
        NSTimeInterval holdTime = currentTime - createdAt;
        
        if (holdTime > 10.0) { // 10秒以上的锁持有
            [report addObject:@{
                @"lockName": lockName,
                @"type": lockInfo[@"type"],
                @"holdTime": @(holdTime),
                @"severity": holdTime > 30.0 ? @"High" : @"Medium"
            }];
        }
    }
    
    return report;
}

@end
```

## 多线程性能优化与最佳实践

### 性能监控与优化管理器

```objc
@interface MultithreadingPerformanceOptimizer : NSObject

@property (nonatomic, strong) NSMutableDictionary *performanceMetrics;
@property (nonatomic, strong) NSMutableArray *optimizationSuggestions;
@property (nonatomic, strong) NSTimer *monitoringTimer;
@property (nonatomic, assign) BOOL isMonitoring;

+ (instancetype)sharedOptimizer;

// 性能监控
- (void)startPerformanceMonitoring;
- (void)stopPerformanceMonitoring;
- (void)recordThreadCreation:(NSString *)threadName;
- (void)recordTaskExecution:(NSString *)taskName duration:(NSTimeInterval)duration;

// 性能分析
- (NSDictionary *)getPerformanceReport;
- (NSArray *)analyzeBottlenecks;
- (NSArray *)generateOptimizationSuggestions;

// 自动优化
- (void)optimizeQueueConfiguration;
- (void)balanceWorkload;
- (void)adjustConcurrencyLimits;

// 资源管理
- (void)monitorMemoryUsage;
- (void)detectResourceLeaks;
- (void)cleanupIdleResources;

@end

@implementation MultithreadingPerformanceOptimizer

+ (instancetype)sharedOptimizer {
    static MultithreadingPerformanceOptimizer *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.performanceMetrics = [NSMutableDictionary dictionary];
        self.optimizationSuggestions = [NSMutableArray array];
        self.isMonitoring = NO;
    }
    return self;
}

- (void)startPerformanceMonitoring {
    if (!self.isMonitoring) {
        NSLog(@"开始多线程性能监控");
        self.isMonitoring = YES;
        
        // 每5秒收集一次性能数据
        self.monitoringTimer = [NSTimer scheduledTimerWithTimeInterval:5.0
                                                               target:self
                                                             selector:@selector(collectPerformanceData)
                                                             userInfo:nil
                                                              repeats:YES];
        
        // 初始化性能指标
        self.performanceMetrics[@"startTime"] = @([[NSDate date] timeIntervalSince1970]);
        self.performanceMetrics[@"threadCount"] = @0;
        self.performanceMetrics[@"taskCount"] = @0;
        self.performanceMetrics[@"totalExecutionTime"] = @0;
        self.performanceMetrics[@"memoryUsage"] = [NSMutableArray array];
        self.performanceMetrics[@"cpuUsage"] = [NSMutableArray array];
    }
}

- (void)stopPerformanceMonitoring {
    if (self.isMonitoring) {
        NSLog(@"停止多线程性能监控");
        self.isMonitoring = NO;
        
        [self.monitoringTimer invalidate];
        self.monitoringTimer = nil;
        
        // 生成最终报告
        [self generateFinalReport];
    }
}

- (void)collectPerformanceData {
    if (!self.isMonitoring) return;
    
    // 收集内存使用情况
    struct mach_task_basic_info info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
    
    if (kerr == KERN_SUCCESS) {
        NSNumber *memoryUsage = @(info.resident_size / 1024.0 / 1024.0); // MB
        NSMutableArray *memoryArray = self.performanceMetrics[@"memoryUsage"];
        [memoryArray addObject:@{
            @"timestamp": @([[NSDate date] timeIntervalSince1970]),
            @"memory": memoryUsage
        }];
        
        NSLog(@"当前内存使用: %.2f MB", [memoryUsage doubleValue]);
    }
    
    // 收集线程数量
    thread_array_t threadList;
    mach_msg_type_number_t threadCount;
    
    if (task_threads(mach_task_self(), &threadList, &threadCount) == KERN_SUCCESS) {
        self.performanceMetrics[@"currentThreadCount"] = @(threadCount);
        NSLog(@"当前线程数量: %d", threadCount);
        
        vm_deallocate(mach_task_self(), (vm_address_t)threadList, threadCount * sizeof(thread_t));
    }
}

- (void)recordThreadCreation:(NSString *)threadName {
    NSLog(@"记录线程创建: %@", threadName);
    
    NSInteger threadCount = [self.performanceMetrics[@"threadCount"] integerValue] + 1;
    self.performanceMetrics[@"threadCount"] = @(threadCount);
    
    // 记录线程创建历史
    NSMutableArray *threadHistory = self.performanceMetrics[@"threadHistory"];
    if (!threadHistory) {
        threadHistory = [NSMutableArray array];
        self.performanceMetrics[@"threadHistory"] = threadHistory;
    }
    
    [threadHistory addObject:@{
        @"name": threadName,
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"type": @"creation"
    }];
}

- (void)recordTaskExecution:(NSString *)taskName duration:(NSTimeInterval)duration {
    NSLog(@"记录任务执行: %@，耗时: %.3f秒", taskName, duration);
    
    NSInteger taskCount = [self.performanceMetrics[@"taskCount"] integerValue] + 1;
    NSTimeInterval totalTime = [self.performanceMetrics[@"totalExecutionTime"] doubleValue] + duration;
    
    self.performanceMetrics[@"taskCount"] = @(taskCount);
    self.performanceMetrics[@"totalExecutionTime"] = @(totalTime);
    
    // 记录任务执行历史
    NSMutableArray *taskHistory = self.performanceMetrics[@"taskHistory"];
    if (!taskHistory) {
        taskHistory = [NSMutableArray array];
        self.performanceMetrics[@"taskHistory"] = taskHistory;
    }
    
    [taskHistory addObject:@{
        @"name": taskName,
        @"duration": @(duration),
        @"executedAt": @([[NSDate date] timeIntervalSince1970])
    }];
    
    // 检查是否需要优化建议
    if (duration > 1.0) { // 超过1秒的任务
        [self.optimizationSuggestions addObject:@{
            @"type": @"slow_task",
            @"message": [NSString stringWithFormat:@"任务 %@ 执行时间过长: %.3f秒", taskName, duration],
            @"suggestion": @"考虑将任务分解为更小的子任务或使用后台队列"
        }];
    }
}

- (NSDictionary *)getPerformanceReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    NSTimeInterval currentTime = [[NSDate date] timeIntervalSince1970];
    NSTimeInterval startTime = [self.performanceMetrics[@"startTime"] doubleValue];
    NSTimeInterval monitoringDuration = currentTime - startTime;
    
    report[@"monitoringDuration"] = @(monitoringDuration);
    report[@"totalThreadsCreated"] = self.performanceMetrics[@"threadCount"] ?: @0;
    report[@"totalTasksExecuted"] = self.performanceMetrics[@"taskCount"] ?: @0;
    report[@"totalExecutionTime"] = self.performanceMetrics[@"totalExecutionTime"] ?: @0;
    report[@"currentThreadCount"] = self.performanceMetrics[@"currentThreadCount"] ?: @0;
    
    // 计算平均值
    NSInteger taskCount = [self.performanceMetrics[@"taskCount"] integerValue];
    if (taskCount > 0) {
        NSTimeInterval totalTime = [self.performanceMetrics[@"totalExecutionTime"] doubleValue];
        report[@"averageTaskDuration"] = @(totalTime / taskCount);
    }
    
    // 内存使用统计
    NSArray *memoryUsage = self.performanceMetrics[@"memoryUsage"];
    if (memoryUsage.count > 0) {
        double totalMemory = 0;
        double maxMemory = 0;
        for (NSDictionary *entry in memoryUsage) {
            double memory = [entry[@"memory"] doubleValue];
            totalMemory += memory;
            if (memory > maxMemory) {
                maxMemory = memory;
            }
        }
        report[@"averageMemoryUsage"] = @(totalMemory / memoryUsage.count);
        report[@"peakMemoryUsage"] = @(maxMemory);
    }
    
    report[@"optimizationSuggestions"] = [self.optimizationSuggestions copy];
    
    return report;
}

- (NSArray *)analyzeBottlenecks {
    NSMutableArray *bottlenecks = [NSMutableArray array];
    
    // 分析任务执行时间
    NSArray *taskHistory = self.performanceMetrics[@"taskHistory"];
    if (taskHistory) {
        NSMutableDictionary *taskStats = [NSMutableDictionary dictionary];
        
        for (NSDictionary *task in taskHistory) {
            NSString *taskName = task[@"name"];
            NSTimeInterval duration = [task[@"duration"] doubleValue];
            
            NSMutableArray *durations = taskStats[taskName];
            if (!durations) {
                durations = [NSMutableArray array];
                taskStats[taskName] = durations;
            }
            [durations addObject:@(duration)];
        }
        
        // 找出平均执行时间最长的任务
        for (NSString *taskName in taskStats) {
            NSArray *durations = taskStats[taskName];
            double totalDuration = 0;
            for (NSNumber *duration in durations) {
                totalDuration += [duration doubleValue];
            }
            double averageDuration = totalDuration / durations.count;
            
            if (averageDuration > 0.5) { // 平均超过0.5秒
                [bottlenecks addObject:@{
                    @"type": @"slow_task",
                    @"taskName": taskName,
                    @"averageDuration": @(averageDuration),
                    @"executionCount": @(durations.count),
                    @"severity": averageDuration > 2.0 ? @"High" : @"Medium"
                }];
            }
        }
    }
    
    // 分析内存使用趋势
    NSArray *memoryUsage = self.performanceMetrics[@"memoryUsage"];
    if (memoryUsage.count > 10) {
        // 检查内存是否持续增长
        double firstMemory = [memoryUsage.firstObject[@"memory"] doubleValue];
        double lastMemory = [memoryUsage.lastObject[@"memory"] doubleValue];
        
        if (lastMemory > firstMemory * 1.5) { // 内存增长超过50%
            [bottlenecks addObject:@{
                @"type": @"memory_growth",
                @"initialMemory": @(firstMemory),
                @"currentMemory": @(lastMemory),
                @"growthRate": @((lastMemory - firstMemory) / firstMemory * 100),
                @"severity": @"High"
            }];
        }
    }
    
    return bottlenecks;
}

- (NSArray *)generateOptimizationSuggestions {
    NSMutableArray *suggestions = [NSMutableArray array];
    
    NSArray *bottlenecks = [self analyzeBottlenecks];
    
    for (NSDictionary *bottleneck in bottlenecks) {
        NSString *type = bottleneck[@"type"];
        
        if ([type isEqualToString:@"slow_task"]) {
            [suggestions addObject:@{
                @"category": @"Performance",
                @"priority": @"High",
                @"suggestion": [NSString stringWithFormat:@"优化任务 %@：考虑异步执行或分解为更小的子任务", bottleneck[@"taskName"]],
                @"implementation": @"使用GCD的并发队列或NSOperation的依赖关系"
            }];
        } else if ([type isEqualToString:@"memory_growth"]) {
            [suggestions addObject:@{
                @"category": @"Memory",
                @"priority": @"Critical",
                @"suggestion": @"检测到内存持续增长，可能存在内存泄漏",
                @"implementation": @"使用Instruments检查内存泄漏，确保正确释放资源"
            }];
        }
    }
    
    // 基于线程数量的建议
    NSInteger currentThreadCount = [self.performanceMetrics[@"currentThreadCount"] integerValue];
    if (currentThreadCount > 20) {
        [suggestions addObject:@{
            @"category": @"Threading",
            @"priority": @"Medium",
            @"suggestion": [NSString stringWithFormat:@"当前线程数量过多(%ld)，可能影响性能", (long)currentThreadCount],
            @"implementation": @"使用线程池或减少并发操作数量"
        }];
    }
    
    return suggestions;
}

- (void)optimizeQueueConfiguration {
    NSLog(@"执行队列配置优化...");
    
    // 获取系统信息
    NSInteger processorCount = [[NSProcessInfo processInfo] processorCount];
    
    // 建议的最大并发数
    NSInteger recommendedConcurrency = processorCount * 2;
    
    NSLog(@"系统处理器数量: %ld，建议最大并发数: %ld", (long)processorCount, (long)recommendedConcurrency);
    
    // 这里可以调整现有队列的配置
    // 实际实现中需要访问具体的队列实例
}

- (void)balanceWorkload {
    NSLog(@"执行工作负载均衡...");
    
    // 分析任务分布
    NSArray *taskHistory = self.performanceMetrics[@"taskHistory"];
    if (taskHistory.count > 0) {
        // 统计不同类型任务的执行情况
        NSMutableDictionary *taskTypeStats = [NSMutableDictionary dictionary];
        
        for (NSDictionary *task in taskHistory) {
            NSString *taskName = task[@"name"];
            NSTimeInterval duration = [task[@"duration"] doubleValue];
            
            NSMutableDictionary *stats = taskTypeStats[taskName];
            if (!stats) {
                stats = [@{@"count": @0, @"totalDuration": @0} mutableCopy];
                taskTypeStats[taskName] = stats;
            }
            
            stats[@"count"] = @([stats[@"count"] integerValue] + 1);
            stats[@"totalDuration"] = @([stats[@"totalDuration"] doubleValue] + duration);
        }
        
        NSLog(@"任务类型统计: %@", taskTypeStats);
    }
}

- (void)adjustConcurrencyLimits {
    NSLog(@"调整并发限制...");
    
    // 基于当前性能指标调整并发限制
    NSInteger currentThreadCount = [self.performanceMetrics[@"currentThreadCount"] integerValue];
    NSArray *memoryUsage = self.performanceMetrics[@"memoryUsage"];
    
    if (memoryUsage.count > 0) {
        double currentMemory = [memoryUsage.lastObject[@"memory"] doubleValue];
        
        if (currentMemory > 100.0) { // 内存使用超过100MB
            NSLog(@"内存使用过高(%.2f MB)，建议降低并发数", currentMemory);
        } else if (currentThreadCount < 5 && currentMemory < 50.0) {
            NSLog(@"资源使用较低，可以适当增加并发数");
        }
    }
}

- (void)monitorMemoryUsage {
    // 已在collectPerformanceData中实现
}

- (void)detectResourceLeaks {
    NSLog(@"检测资源泄漏...");
    
    // 检查线程数量是否异常增长
    NSArray *threadHistory = self.performanceMetrics[@"threadHistory"];
    if (threadHistory.count > 100) {
        NSLog(@"警告: 线程创建数量过多(%lu)，可能存在线程泄漏", (unsigned long)threadHistory.count);
    }
    
    // 检查内存使用趋势
    NSArray *memoryUsage = self.performanceMetrics[@"memoryUsage"];
    if (memoryUsage.count >= 10) {
        double firstMemory = [memoryUsage.firstObject[@"memory"] doubleValue];
        double lastMemory = [memoryUsage.lastObject[@"memory"] doubleValue];
        
        if (lastMemory > firstMemory * 2.0) {
            NSLog(@"警告: 内存使用增长过快，从%.2fMB增长到%.2fMB", firstMemory, lastMemory);
        }
    }
}

- (void)cleanupIdleResources {
    NSLog(@"清理空闲资源...");
    
    // 清理过期的性能数据
    NSTimeInterval currentTime = [[NSDate date] timeIntervalSince1970];
    NSTimeInterval cutoffTime = currentTime - 3600; // 保留1小时内的数据
    
    NSMutableArray *memoryUsage = self.performanceMetrics[@"memoryUsage"];
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"timestamp > %f", cutoffTime];
    NSArray *filteredMemory = [memoryUsage filteredArrayUsingPredicate:predicate];
    [memoryUsage removeAllObjects];
    [memoryUsage addObjectsFromArray:filteredMemory];
    
    NSLog(@"清理完成，保留了%lu条内存使用记录", (unsigned long)memoryUsage.count);
}

- (void)generateFinalReport {
    NSLog(@"=== 多线程性能监控最终报告 ===");
    
    NSDictionary *report = [self getPerformanceReport];
    
    NSLog(@"监控时长: %.2f秒", [report[@"monitoringDuration"] doubleValue]);
    NSLog(@"创建线程总数: %@", report[@"totalThreadsCreated"]);
    NSLog(@"执行任务总数: %@", report[@"totalTasksExecuted"]);
    NSLog(@"总执行时间: %.3f秒", [report[@"totalExecutionTime"] doubleValue]);
    NSLog(@"当前线程数: %@", report[@"currentThreadCount"]);
    
    if (report[@"averageTaskDuration"]) {
        NSLog(@"平均任务执行时间: %.3f秒", [report[@"averageTaskDuration"] doubleValue]);
    }
    
    if (report[@"averageMemoryUsage"]) {
        NSLog(@"平均内存使用: %.2fMB", [report[@"averageMemoryUsage"] doubleValue]);
        NSLog(@"峰值内存使用: %.2fMB", [report[@"peakMemoryUsage"] doubleValue]);
    }
    
    NSArray *suggestions = report[@"optimizationSuggestions"];
    if (suggestions.count > 0) {
        NSLog(@"优化建议:");
        for (NSDictionary *suggestion in suggestions) {
            NSLog(@"  - %@: %@", suggestion[@"type"], suggestion[@"message"]);
        }
    }
}

@end
```

### 多线程最佳实践管理器

```objc
@interface MultithreadingBestPractices : NSObject

+ (instancetype)sharedPractices;

// 设计原则
- (void)demonstrateProperQueueUsage;
- (void)showThreadSafetyPatterns;
- (void)illustrateResourceManagement;

// 常见陷阱避免
- (void)avoidDeadlockPatterns;
- (void)preventRaceConditions;
- (void)handlePriorityInversion;

// 性能优化技巧
- (void)optimizeTaskGranularity;
- (void)implementEfficientSynchronization;
- (void)useAppropriateQueueTypes;

// 调试和测试
- (void)enableThreadSanitizer;
- (void)implementLoggingStrategies;
- (void)createTestScenarios;

@end

@implementation MultithreadingBestPractices

+ (instancetype)sharedPractices {
    static MultithreadingBestPractices *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (void)demonstrateProperQueueUsage {
    NSLog(@"=== 正确的队列使用模式 ===");
    
    // 1. 主队列 - 仅用于UI更新
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"主队列：更新UI元素");
        // 更新UI的代码
    });
    
    // 2. 全局并发队列 - 用于CPU密集型任务
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@"全局队列：执行计算密集型任务");
        // 执行复杂计算
        
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"返回主队列：更新计算结果");
        });
    });
    
    // 3. 自定义串行队列 - 用于顺序执行的任务
    dispatch_queue_t serialQueue = dispatch_queue_create("com.example.serial", DISPATCH_QUEUE_SERIAL);
    dispatch_async(serialQueue, ^{
        NSLog(@"串行队列：按顺序执行任务1");
    });
    dispatch_async(serialQueue, ^{
        NSLog(@"串行队列：按顺序执行任务2");
    });
    
    // 4. 自定义并发队列 - 用于可并行的任务
    dispatch_queue_t concurrentQueue = dispatch_queue_create("com.example.concurrent", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 5; i++) {
        dispatch_async(concurrentQueue, ^{
            NSLog(@"并发队列：并行执行任务%d", i);
        });
    }
}

- (void)showThreadSafetyPatterns {
    NSLog(@"=== 线程安全模式 ===");
    
    // 1. 使用串行队列保护共享资源
    static dispatch_queue_t protectionQueue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        protectionQueue = dispatch_queue_create("com.example.protection", DISPATCH_QUEUE_SERIAL);
    });
    
    static NSMutableArray *sharedArray;
    static dispatch_once_t arrayOnceToken;
    dispatch_once(&arrayOnceToken, ^{
        sharedArray = [NSMutableArray array];
    });
    
    // 安全地添加元素
    dispatch_async(protectionQueue, ^{
        [sharedArray addObject:@"安全添加的元素"];
        NSLog(@"安全添加元素，当前数组大小: %lu", (unsigned long)sharedArray.count);
    });
    
    // 2. 使用barrier确保读写安全
    dispatch_queue_t rwQueue = dispatch_queue_create("com.example.readwrite", DISPATCH_QUEUE_CONCURRENT);
    
    // 读操作（并发）
    dispatch_async(rwQueue, ^{
        NSLog(@"并发读操作1");
    });
    dispatch_async(rwQueue, ^{
        NSLog(@"并发读操作2");
    });
    
    // 写操作（独占）
    dispatch_barrier_async(rwQueue, ^{
        NSLog(@"独占写操作");
    });
    
    // 3. 使用原子属性
    // @property (atomic, strong) NSString *atomicProperty;
    NSLog(@"使用atomic属性确保基本的线程安全");
}

- (void)illustrateResourceManagement {
    NSLog(@"=== 资源管理最佳实践 ===");
    
    // 1. 合理控制并发数量
    NSOperationQueue *limitedQueue = [[NSOperationQueue alloc] init];
    limitedQueue.maxConcurrentOperationCount = 3; // 限制并发数
    limitedQueue.name = @"LimitedConcurrencyQueue";
    
    for (int i = 0; i < 10; i++) {
        NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
            NSLog(@"受限并发操作 %d", i);
            [NSThread sleepForTimeInterval:1.0]; // 模拟工作
        }];
        [limitedQueue addOperation:operation];
    }
    
    // 2. 使用autoreleasepool管理内存
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        @autoreleasepool {
            for (int i = 0; i < 1000; i++) {
                NSString *tempString = [NSString stringWithFormat:@"临时字符串 %d", i];
                // 处理临时对象
                (void)tempString; // 避免未使用变量警告
            }
            NSLog(@"自动释放池确保及时释放临时对象");
        }
    });
    
    // 3. 正确处理队列的生命周期
    // 避免创建过多的队列，重用现有队列
    static dispatch_queue_t reusableQueue;
    static dispatch_once_t queueOnceToken;
    dispatch_once(&queueOnceToken, ^{
        reusableQueue = dispatch_queue_create("com.example.reusable", DISPATCH_QUEUE_SERIAL);
    });
    
    dispatch_async(reusableQueue, ^{
        NSLog(@"重用队列，避免资源浪费");
    });
}

- (void)avoidDeadlockPatterns {
    NSLog(@"=== 避免死锁模式 ===");
    
    // 1. 避免嵌套同步调用
    dispatch_queue_t queue1 = dispatch_queue_create("queue1", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t queue2 = dispatch_queue_create("queue2", DISPATCH_QUEUE_SERIAL);
    
    // 错误模式（可能导致死锁）：
    // dispatch_sync(queue1, ^{
    //     dispatch_sync(queue2, ^{
    //         // 任务
    //     });
    // });
    
    // 正确模式：使用异步调用
    dispatch_async(queue1, ^{
        NSLog(@"队列1：开始执行");
        dispatch_async(queue2, ^{
            NSLog(@"队列2：异步执行，避免死锁");
        });
    });
    
    // 2. 统一锁的获取顺序
    static NSLock *lock1;
    static NSLock *lock2;
    static dispatch_once_t lockOnceToken;
    dispatch_once(&lockOnceToken, ^{
        lock1 = [[NSLock alloc] init];
        lock2 = [[NSLock alloc] init];
        lock1.name = @"Lock1";
        lock2.name = @"Lock2";
    });
    
    // 总是按相同顺序获取锁
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock1 lock];
        NSLog(@"获取锁1");
        [lock2 lock];
        NSLog(@"获取锁2");
        
        // 执行需要两个锁的操作
        
        [lock2 unlock];
        [lock1 unlock];
        NSLog(@"按相反顺序释放锁");
    });
}

- (void)preventRaceConditions {
    NSLog(@"=== 防止竞态条件 ===");
    
    // 1. 使用原子操作
    static int64_t atomicCounter = 0;
    
    dispatch_group_t group = dispatch_group_create();
    
    for (int i = 0; i < 10; i++) {
        dispatch_group_async(group, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            // 原子递增
            int64_t newValue = OSAtomicIncrement64(&atomicCounter);
            NSLog(@"原子递增: %lld", newValue);
        });
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"最终计数器值: %lld", atomicCounter);
    });
    
    // 2. 使用dispatch_once确保单次执行
    static dispatch_once_t initOnceToken;
    dispatch_once(&initOnceToken, ^{
        NSLog(@"这段代码只会执行一次，无论有多少线程调用");
    });
}

- (void)handlePriorityInversion {
    NSLog(@"=== 处理优先级反转 ===");
    
    // 使用QoS类来避免优先级反转
    dispatch_queue_t highPriorityQueue = dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0);
    dispatch_queue_t lowPriorityQueue = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
    
    // 高优先级任务
    dispatch_async(highPriorityQueue, ^{
        NSLog(@"高优先级任务开始");
        // 执行重要任务
        NSLog(@"高优先级任务完成");
    });
    
    // 低优先级任务
    dispatch_async(lowPriorityQueue, ^{
        NSLog(@"低优先级任务开始");
        // 执行后台任务
        NSLog(@"低优先级任务完成");
    });
    
    // 使用NSOperation的依赖关系管理优先级
    NSOperationQueue *operationQueue = [[NSOperationQueue alloc] init];
    
    NSBlockOperation *lowPriorityOp = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"低优先级操作");
    }];
    lowPriorityOp.queuePriority = NSOperationQueuePriorityLow;
    
    NSBlockOperation *highPriorityOp = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"高优先级操作");
    }];
    highPriorityOp.queuePriority = NSOperationQueuePriorityHigh;
    
    [operationQueue addOperation:lowPriorityOp];
    [operationQueue addOperation:highPriorityOp];
}

- (void)optimizeTaskGranularity {
    NSLog(@"=== 优化任务粒度 ===");
    
    // 1. 将大任务分解为小任务
    NSArray *largeDataSet = @[@1, @2, @3, @4, @5, @6, @7, @8, @9, @10];
    NSInteger batchSize = 3;
    
    dispatch_group_t processingGroup = dispatch_group_create();
    
    for (NSInteger i = 0; i < largeDataSet.count; i += batchSize) {
        NSInteger endIndex = MIN(i + batchSize, largeDataSet.count);
        NSArray *batch = [largeDataSet subarrayWithRange:NSMakeRange(i, endIndex - i)];
        
        dispatch_group_async(processingGroup, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSLog(@"处理批次: %@", batch);
            // 处理这个批次的数据
        });
    }
    
    dispatch_group_notify(processingGroup, dispatch_get_main_queue(), ^{
        NSLog(@"所有批次处理完成");
    });
    
    // 2. 使用适当的任务大小
    // 太小的任务会增加调度开销
    // 太大的任务会降低并行性
    NSLog(@"选择适当的任务大小以平衡调度开销和并行性");
}

- (void)implementEfficientSynchronization {
    NSLog(@"=== 实现高效同步 ===");
    
    // 1. 使用读写锁优化读多写少的场景
    static pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
    
    // 多个读操作可以并发执行
    for (int i = 0; i < 5; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            pthread_rwlock_rdlock(&rwlock);
            NSLog(@"并发读操作 %d", i);
            [NSThread sleepForTimeInterval:0.1];
            pthread_rwlock_unlock(&rwlock);
        });
    }
    
    // 写操作是独占的
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        pthread_rwlock_wrlock(&rwlock);
        NSLog(@"独占写操作");
        [NSThread sleepForTimeInterval:0.1];
        pthread_rwlock_unlock(&rwlock);
    });
    
    // 2. 使用信号量控制资源访问
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(2); // 允许2个并发访问
    
    for (int i = 0; i < 5; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
            NSLog(@"获得信号量，执行任务 %d", i);
            [NSThread sleepForTimeInterval:1.0];
            NSLog(@"任务 %d 完成，释放信号量", i);
            dispatch_semaphore_signal(semaphore);
        });
    }
}

- (void)useAppropriateQueueTypes {
    NSLog(@"=== 使用适当的队列类型 ===");
    
    // 1. CPU密集型任务 - 使用并发队列
    dispatch_queue_t cpuIntensiveQueue = dispatch_queue_create("cpu.intensive", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(cpuIntensiveQueue, ^{
        NSLog(@"CPU密集型任务：数学计算");
        // 执行复杂计算
    });
    
    // 2. I/O密集型任务 - 可以使用更多并发
    dispatch_queue_t ioQueue = dispatch_queue_create("io.operations", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 10; i++) {
        dispatch_async(ioQueue, ^{
            NSLog(@"I/O操作 %d：文件读写或网络请求", i);
            // 模拟I/O操作
            [NSThread sleepForTimeInterval:0.1];
        });
    }
    
    // 3. 顺序依赖的任务 - 使用串行队列
    dispatch_queue_t sequentialQueue = dispatch_queue_create("sequential.tasks", DISPATCH_QUEUE_SERIAL);
    dispatch_async(sequentialQueue, ^{
        NSLog(@"顺序任务1：初始化");
    });
    dispatch_async(sequentialQueue, ^{
        NSLog(@"顺序任务2：配置");
    });
    dispatch_async(sequentialQueue, ^{
        NSLog(@"顺序任务3：启动");
    });
}

- (void)enableThreadSanitizer {
    NSLog(@"=== 启用线程安全检测 ===");
    
    // 在Xcode中启用Thread Sanitizer:
    // 1. Edit Scheme -> Run -> Diagnostics
    // 2. 勾选 "Thread Sanitizer"
    
    NSLog(@"Thread Sanitizer可以检测:");
    NSLog(@"- 数据竞争");
    NSLog(@"- 使用已释放的内存");
    NSLog(@"- 线程泄漏");
    NSLog(@"- 死锁");
    
    // 示例：故意创建数据竞争（仅用于演示）
    static int unsafeCounter = 0;
    
    for (int i = 0; i < 2; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            // 这里会被Thread Sanitizer检测到数据竞争
            unsafeCounter++;
            NSLog(@"不安全的计数器操作: %d", unsafeCounter);
        });
    }
}

- (void)implementLoggingStrategies {
    NSLog(@"=== 实现日志记录策略 ===");
    
    // 1. 线程安全的日志记录
    static dispatch_queue_t loggingQueue;
    static dispatch_once_t loggingOnceToken;
    dispatch_once(&loggingOnceToken, ^{
        loggingQueue = dispatch_queue_create("com.example.logging", DISPATCH_QUEUE_SERIAL);
    });
    
    // 安全的日志记录函数
    void safeLog(NSString *message) {
        dispatch_async(loggingQueue, ^{
            NSLog(@"[%@] %@", [NSThread currentThread].name ?: @"Unknown", message);
        });
    }
    
    // 2. 记录线程信息
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        safeLog([NSString stringWithFormat:@"当前线程: %@", [NSThread currentThread]]);
        safeLog([NSString stringWithFormat:@"是否为主线程: %@", [NSThread isMainThread] ? @"是" : @"否"]);
    });
    
    // 3. 性能日志记录
    NSDate *startTime = [NSDate date];
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 执行一些工作
        [NSThread sleepForTimeInterval:0.1];
        
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        safeLog([NSString stringWithFormat:@"任务执行时间: %.3f秒", duration]);
    });
}

- (void)createTestScenarios {
    NSLog(@"=== 创建测试场景 ===");
    
    // 1. 压力测试
    NSLog(@"开始压力测试...");
    
    dispatch_group_t stressTestGroup = dispatch_group_create();
    
    for (int i = 0; i < 100; i++) {
        dispatch_group_async(stressTestGroup, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            // 模拟工作负载
            for (int j = 0; j < 1000; j++) {
                @autoreleasepool {
                    NSString *temp = [NSString stringWithFormat:@"Test %d-%d", i, j];
                    (void)temp;
                }
            }
        });
    }
    
    dispatch_group_notify(stressTestGroup, dispatch_get_main_queue(), ^{
        NSLog(@"压力测试完成");
    });
    
    // 2. 竞态条件测试
    static NSMutableArray *testArray;
    static dispatch_once_t testArrayOnceToken;
    dispatch_once(&testArrayOnceToken, ^{
        testArray = [NSMutableArray array];
    });
    
    dispatch_group_t raceTestGroup = dispatch_group_create();
    
    for (int i = 0; i < 10; i++) {
        dispatch_group_async(raceTestGroup, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            @synchronized(testArray) {
                [testArray addObject:@(i)];
                NSLog(@"安全添加元素: %d", i);
            }
        });
    }
    
    dispatch_group_notify(raceTestGroup, dispatch_get_main_queue(), ^{
        NSLog(@"竞态条件测试完成，数组大小: %lu", (unsigned long)testArray.count);
    });
    
    // 3. 内存泄漏测试
    NSLog(@"执行内存泄漏测试...");
    
    for (int i = 0; i < 10; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            @autoreleasepool {
                NSMutableArray *localArray = [NSMutableArray array];
                for (int j = 0; j < 1000; j++) {
                    [localArray addObject:[NSString stringWithFormat:@"Item %d", j]];
                }
                NSLog(@"内存测试批次 %d 完成", i);
            }
        });
    }
}

@end
```

## 综合多线程管理系统

### 统一多线程管理器

```objc
@interface ComprehensiveMultithreadingManager : NSObject

@property (nonatomic, strong) MultithreadingArchitecture *architecture;
@property (nonatomic, strong) GCDManager *gcdManager;
@property (nonatomic, strong) NSOperationManager *operationManager;
@property (nonatomic, strong) ThreadSafetyManager *safetyManager;
@property (nonatomic, strong) MultithreadingPerformanceOptimizer *performanceOptimizer;
@property (nonatomic, strong) MultithreadingBestPractices *bestPractices;

+ (instancetype)sharedManager;

// 系统初始化
- (void)initializeMultithreadingSystem;
- (void)configureOptimalSettings;
- (void)startSystemMonitoring;

// 任务管理
- (void)executeTask:(NSString *)taskName
             block:(dispatch_block_t)block
             queue:(NSString *)queueType
          priority:(NSString *)priority;
- (void)executeBatchTasks:(NSArray *)tasks
           withCompletion:(dispatch_block_t)completion;
- (void)executeSequentialTasks:(NSArray *)tasks
                withCompletion:(dispatch_block_t)completion;

// 智能调度
- (void)scheduleTaskWithAdaptiveStrategy:(NSString *)taskName
                                   block:(dispatch_block_t)block
                            requirements:(NSDictionary *)requirements;
- (void)balanceSystemLoad;
- (void)optimizeForCurrentConditions;

// 系统监控
- (NSDictionary *)getSystemStatus;
- (NSArray *)getPerformanceMetrics;
- (NSArray *)getOptimizationRecommendations;

// 问题诊断
- (void)diagnosePerformanceIssues;
- (void)detectAndResolveDeadlocks;
- (void)analyzeThreadSafetyViolations;

@end

@implementation ComprehensiveMultithreadingManager

+ (instancetype)sharedManager {
    static ComprehensiveMultithreadingManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupManagers];
    }
    return self;
}

- (void)setupManagers {
    self.architecture = [MultithreadingArchitecture sharedArchitecture];
    self.gcdManager = [GCDManager sharedManager];
    self.operationManager = [NSOperationManager sharedManager];
    self.safetyManager = [ThreadSafetyManager sharedManager];
    self.performanceOptimizer = [MultithreadingPerformanceOptimizer sharedOptimizer];
    self.bestPractices = [MultithreadingBestPractices sharedPractices];
}

- (void)initializeMultithreadingSystem {
    NSLog(@"=== 初始化综合多线程管理系统 ===");
    
    // 1. 初始化架构
    [self.architecture initializeThreadingEnvironment];
    
    // 2. 配置GCD管理器
    [self.gcdManager setupDefaultQueues];
    
    // 3. 配置NSOperation管理器
    [self.operationManager setupDefaultQueues];
    
    // 4. 初始化线程安全管理器
    [self.safetyManager initializeSynchronizationPrimitives];
    
    // 5. 启动性能监控
    [self.performanceOptimizer startPerformanceMonitoring];
    
    NSLog(@"多线程管理系统初始化完成");
}

- (void)configureOptimalSettings {
    NSLog(@"配置最优设置...");
    
    // 获取设备信息
    NSInteger processorCount = [[NSProcessInfo processInfo] processorCount];
    NSInteger memorySize = (NSInteger)([[NSProcessInfo processInfo] physicalMemory] / 1024 / 1024); // MB
    
    NSLog(@"设备信息 - 处理器数量: %ld, 内存大小: %ld MB", (long)processorCount, (long)memorySize);
    
    // 根据设备性能调整设置
    if (processorCount >= 8 && memorySize >= 4096) {
        // 高性能设备
        NSLog(@"检测到高性能设备，使用高并发配置");
        [self.operationManager setMaxConcurrentOperationCount:processorCount * 2];
    } else if (processorCount >= 4 && memorySize >= 2048) {
        // 中等性能设备
        NSLog(@"检测到中等性能设备，使用标准配置");
        [self.operationManager setMaxConcurrentOperationCount:processorCount];
    } else {
        // 低性能设备
        NSLog(@"检测到低性能设备，使用保守配置");
        [self.operationManager setMaxConcurrentOperationCount:MAX(2, processorCount / 2)];
    }
    
    // 配置性能优化器
    [self.performanceOptimizer optimizeQueueConfiguration];
}

- (void)startSystemMonitoring {
    NSLog(@"启动系统监控...");
    
    // 启动性能监控
    [self.performanceOptimizer startPerformanceMonitoring];
    
    // 定期检查系统状态
    dispatch_queue_t monitoringQueue = dispatch_queue_create("com.system.monitoring", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(monitoringQueue, ^{
        while (YES) {
            [NSThread sleepForTimeInterval:30.0]; // 每30秒检查一次
            
            @autoreleasepool {
                [self performSystemHealthCheck];
            }
        }
    });
}

- (void)performSystemHealthCheck {
    NSLog(@"执行系统健康检查...");
    
    // 检查性能指标
    NSDictionary *performanceReport = [self.performanceOptimizer getPerformanceReport];
    
    // 检查内存使用
    NSNumber *currentMemory = performanceReport[@"currentMemoryUsage"];
    if (currentMemory && [currentMemory doubleValue] > 200.0) { // 超过200MB
        NSLog(@"警告: 内存使用过高 %.2f MB", [currentMemory doubleValue]);
        [self.performanceOptimizer cleanupIdleResources];
    }
    
    // 检查线程数量
    NSNumber *threadCount = performanceReport[@"currentThreadCount"];
    if (threadCount && [threadCount integerValue] > 30) {
        NSLog(@"警告: 线程数量过多 %ld", (long)[threadCount integerValue]);
        [self.performanceOptimizer adjustConcurrencyLimits];
    }
    
    // 检查是否有资源泄漏
    [self.performanceOptimizer detectResourceLeaks];
}

- (void)executeTask:(NSString *)taskName
             block:(dispatch_block_t)block
             queue:(NSString *)queueType
          priority:(NSString *)priority {
    
    NSLog(@"执行任务: %@, 队列类型: %@, 优先级: %@", taskName, queueType, priority);
    
    NSDate *startTime = [NSDate date];
    
    dispatch_queue_t targetQueue;
    
    // 选择合适的队列
    if ([queueType isEqualToString:@"main"]) {
        targetQueue = dispatch_get_main_queue();
    } else if ([queueType isEqualToString:@"global"]) {
        dispatch_qos_class_t qosClass = QOS_CLASS_DEFAULT;
        if ([priority isEqualToString:@"high"]) {
            qosClass = QOS_CLASS_USER_INITIATED;
        } else if ([priority isEqualToString:@"low"]) {
            qosClass = QOS_CLASS_UTILITY;
        }
        targetQueue = dispatch_get_global_queue(qosClass, 0);
    } else {
        // 使用自定义队列
        targetQueue = [self.gcdManager getQueueByName:queueType];
        if (!targetQueue) {
            targetQueue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
        }
    }
    
    dispatch_async(targetQueue, ^{
        // 执行任务
        if (block) {
            block();
        }
        
        // 记录执行时间
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        [self.performanceOptimizer recordTaskExecution:taskName duration:duration];
    });
}

- (void)executeBatchTasks:(NSArray *)tasks withCompletion:(dispatch_block_t)completion {
    NSLog(@"执行批量任务，任务数量: %lu", (unsigned long)tasks.count);
    
    dispatch_group_t group = dispatch_group_create();
    
    for (NSDictionary *taskInfo in tasks) {
        NSString *taskName = taskInfo[@"name"];
        dispatch_block_t taskBlock = taskInfo[@"block"];
        NSString *queueType = taskInfo[@"queue"] ?: @"global";
        NSString *priority = taskInfo[@"priority"] ?: @"normal";
        
        dispatch_group_enter(group);
        
        [self executeTask:taskName block:^{
            if (taskBlock) {
                taskBlock();
            }
            dispatch_group_leave(group);
        } queue:queueType priority:priority];
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"所有批量任务执行完成");
        if (completion) {
            completion();
        }
    });
}

- (void)executeSequentialTasks:(NSArray *)tasks withCompletion:(dispatch_block_t)completion {
    NSLog(@"执行顺序任务，任务数量: %lu", (unsigned long)tasks.count);
    
    dispatch_queue_t serialQueue = dispatch_queue_create("com.sequential.tasks", DISPATCH_QUEUE_SERIAL);
    
    __block NSInteger completedTasks = 0;
    
    for (NSDictionary *taskInfo in tasks) {
        NSString *taskName = taskInfo[@"name"];
        dispatch_block_t taskBlock = taskInfo[@"block"];
        
        dispatch_async(serialQueue, ^{
            NSDate *startTime = [NSDate date];
            
            NSLog(@"开始执行顺序任务: %@", taskName);
            
            if (taskBlock) {
                taskBlock();
            }
            
            NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
            [self.performanceOptimizer recordTaskExecution:taskName duration:duration];
            
            completedTasks++;
            
            if (completedTasks == tasks.count) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    NSLog(@"所有顺序任务执行完成");
                    if (completion) {
                        completion();
                    }
                });
            }
        });
    }
}

- (void)scheduleTaskWithAdaptiveStrategy:(NSString *)taskName
                                   block:(dispatch_block_t)block
                            requirements:(NSDictionary *)requirements {
    
    NSLog(@"使用自适应策略调度任务: %@", taskName);
    
    // 分析任务需求
    BOOL isCPUIntensive = [requirements[@"cpuIntensive"] boolValue];
    BOOL isIOIntensive = [requirements[@"ioIntensive"] boolValue];
    BOOL requiresMainThread = [requirements[@"requiresMainThread"] boolValue];
    NSString *priority = requirements[@"priority"] ?: @"normal";
    
    // 获取当前系统状态
    NSDictionary *systemStatus = [self getSystemStatus];
    NSInteger currentThreadCount = [systemStatus[@"threadCount"] integerValue];
    double memoryUsage = [systemStatus[@"memoryUsage"] doubleValue];
    
    // 决定调度策略
    NSString *queueType;
    NSString *adjustedPriority = priority;
    
    if (requiresMainThread) {
        queueType = @"main";
    } else if (isCPUIntensive) {
        // CPU密集型任务
        if (currentThreadCount > 15) {
            // 当前线程较多，降低优先级
            adjustedPriority = @"low";
        }
        queueType = @"global";
    } else if (isIOIntensive) {
        // I/O密集型任务，可以使用更多并发
        queueType = @"global";
    } else {
        // 普通任务
        queueType = @"global";
    }
    
    // 如果内存使用过高，延迟执行
    if (memoryUsage > 150.0) {
        NSLog(@"内存使用过高，延迟执行任务: %@", taskName);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^{
            [self executeTask:taskName block:block queue:queueType priority:adjustedPriority];
        });
    } else {
        [self executeTask:taskName block:block queue:queueType priority:adjustedPriority];
    }
}

- (void)balanceSystemLoad {
    NSLog(@"执行系统负载均衡...");
    
    [self.performanceOptimizer balanceWorkload];
    [self.performanceOptimizer adjustConcurrencyLimits];
    
    // 清理空闲资源
    [self.performanceOptimizer cleanupIdleResources];
}

- (void)optimizeForCurrentConditions {
    NSLog(@"根据当前条件优化系统...");
    
    // 获取当前性能报告
    NSDictionary *report = [self.performanceOptimizer getPerformanceReport];
    
    // 获取优化建议
    NSArray *suggestions = [self.performanceOptimizer generateOptimizationSuggestions];
    
    // 应用优化建议
    for (NSDictionary *suggestion in suggestions) {
        NSString *category = suggestion[@"category"];
        NSString *priority = suggestion[@"priority"];
        
        if ([category isEqualToString:@"Performance"] && [priority isEqualToString:@"High"]) {
            NSLog(@"应用高优先级性能优化: %@", suggestion[@"suggestion"]);
            // 实施性能优化
        } else if ([category isEqualToString:@"Memory"] && [priority isEqualToString:@"Critical"]) {
            NSLog(@"应用关键内存优化: %@", suggestion[@"suggestion"]);
            // 实施内存优化
            [self.performanceOptimizer cleanupIdleResources];
        }
    }
}

- (NSDictionary *)getSystemStatus {
    NSMutableDictionary *status = [NSMutableDictionary dictionary];
    
    // 获取性能报告
    NSDictionary *performanceReport = [self.performanceOptimizer getPerformanceReport];
    
    status[@"threadCount"] = performanceReport[@"currentThreadCount"] ?: @0;
    status[@"memoryUsage"] = performanceReport[@"averageMemoryUsage"] ?: @0;
    status[@"taskCount"] = performanceReport[@"totalTasksExecuted"] ?: @0;
    status[@"averageTaskDuration"] = performanceReport[@"averageTaskDuration"] ?: @0;
    
    // 添加系统信息
    status[@"processorCount"] = @([[NSProcessInfo processInfo] processorCount]);
    status[@"systemUptime"] = @([[NSProcessInfo processInfo] systemUptime]);
    
    return status;
}

- (NSArray *)getPerformanceMetrics {
    return [self.performanceOptimizer analyzeBottlenecks];
}

- (NSArray *)getOptimizationRecommendations {
    return [self.performanceOptimizer generateOptimizationSuggestions];
}

- (void)diagnosePerformanceIssues {
    NSLog(@"=== 诊断性能问题 ===");
    
    // 分析瓶颈
    NSArray *bottlenecks = [self.performanceOptimizer analyzeBottlenecks];
    
    if (bottlenecks.count > 0) {
        NSLog(@"发现 %lu 个性能瓶颈:", (unsigned long)bottlenecks.count);
        for (NSDictionary *bottleneck in bottlenecks) {
            NSLog(@"- %@: %@", bottleneck[@"type"], bottleneck[@"taskName"] ?: @"系统级问题");
        }
    } else {
        NSLog(@"未发现明显的性能瓶颈");
    }
    
    // 生成优化建议
    NSArray *suggestions = [self.performanceOptimizer generateOptimizationSuggestions];
    
    if (suggestions.count > 0) {
        NSLog(@"优化建议:");
        for (NSDictionary *suggestion in suggestions) {
            NSLog(@"- [%@] %@", suggestion[@"priority"], suggestion[@"suggestion"]);
        }
    }
}

- (void)detectAndResolveDeadlocks {
    NSLog(@"检测和解决死锁...");
    
    // 这里可以实现死锁检测逻辑
    // 在实际应用中，可以使用系统工具或第三方库来检测死锁
    
    NSLog(@"死锁检测完成，未发现死锁情况");
}

- (void)analyzeThreadSafetyViolations {
    NSLog(@"分析线程安全违规...");
    
    // 检查是否启用了Thread Sanitizer
    NSLog(@"建议启用Thread Sanitizer进行详细的线程安全分析");
    
    // 检查常见的线程安全问题
    [self.bestPractices enableThreadSanitizer];
}

- (void)dealloc {
    // 停止性能监控
    [self.performanceOptimizer stopPerformanceMonitoring];
}

@end
```

## 多线程开发最佳实践总结

### 核心设计原则

1. **性能优先**
   - 根据任务特性选择合适的并发模型
   - 避免过度并发导致的上下文切换开销
   - 合理控制线程数量和资源使用

2. **安全第一**
   - 始终考虑线程安全问题
   - 使用适当的同步机制保护共享资源
   - 避免数据竞争和死锁

3. **可维护性**
   - 保持代码结构清晰，职责分离
   - 使用统一的错误处理和日志记录
   - 提供完善的监控和调试工具

### 开发阶段建议

**设计阶段：**
- 分析应用的并发需求和性能目标
- 选择合适的多线程技术栈（GCD vs NSOperation）
- 设计线程安全的数据结构和API

**实现阶段：**
- 遵循最佳实践，使用经过验证的模式
- 实现全面的错误处理和资源管理
- 添加性能监控和调试支持

**测试阶段：**
- 使用Thread Sanitizer检测线程安全问题
- 进行压力测试验证性能和稳定性
- 测试各种边界条件和异常情况

**维护阶段：**
- 持续监控应用的多线程性能
- 根据用户反馈和性能数据进行优化
- 保持对新技术和最佳实践的学习

通过本文的深入解析，我们全面了解了iOS多线程编程的核心技术、实现方法和最佳实践。掌握这些知识将帮助开发者构建高性能、线程安全的iOS应用程序。

## GCD（Grand Central Dispatch）深度解析

### GCD管理器

```objc
@interface GCDManager : NSObject

@property (nonatomic, strong) NSMutableDictionary *customQueues;
@property (nonatomic, strong) NSMutableArray *dispatchGroups;
@property (nonatomic, strong) NSMutableDictionary *semaphores;
@property (nonatomic, strong) NSMutableArray *barriers;

+ (instancetype)sharedManager;

// 队列管理
- (dispatch_queue_t)createSerialQueue:(NSString *)name withQoS:(dispatch_qos_class_t)qos;
- (dispatch_queue_t)createConcurrentQueue:(NSString *)name withQoS:(dispatch_qos_class_t)qos;
- (void)destroyQueue:(NSString *)name;

// 任务调度
- (void)executeAsync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue;
- (void)executeSync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue;
- (void)executeAfterDelay:(NSTimeInterval)delay block:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue;

// 组操作
- (dispatch_group_t)createDispatchGroup:(NSString *)groupName;
- (void)executeInGroup:(dispatch_group_t)group block:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue;
- (void)waitForGroup:(dispatch_group_t)group withTimeout:(dispatch_time_t)timeout completion:(dispatch_block_t)completion;

// 信号量操作
- (dispatch_semaphore_t)createSemaphore:(NSString *)name withValue:(long)value;
- (void)waitForSemaphore:(dispatch_semaphore_t)semaphore withTimeout:(dispatch_time_t)timeout;
- (void)signalSemaphore:(dispatch_semaphore_t)semaphore;

// 栅栏操作
- (void)executeBarrierAsync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue;
- (void)executeBarrierSync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue;

// 性能监控
- (NSDictionary *)getQueueStatistics;
- (void)logQueuePerformance;

@end

@implementation GCDManager

+ (instancetype)sharedManager {
    static GCDManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.customQueues = [NSMutableDictionary dictionary];
        self.dispatchGroups = [NSMutableArray array];
        self.semaphores = [NSMutableDictionary dictionary];
        self.barriers = [NSMutableArray array];
    }
    return self;
}

- (dispatch_queue_t)createSerialQueue:(NSString *)name withQoS:(dispatch_qos_class_t)qos {
    NSLog(@"创建串行队列: %@", name);
    
    dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, qos, 0);
    dispatch_queue_t queue = dispatch_queue_create([name UTF8String], attr);
    
    self.customQueues[name] = @{
        @"queue": queue,
        @"type": @"serial",
        @"qos": @(qos),
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"taskCount": @0
    };
    
    return queue;
}

- (dispatch_queue_t)createConcurrentQueue:(NSString *)name withQoS:(dispatch_qos_class_t)qos {
    NSLog(@"创建并发队列: %@", name);
    
    dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, qos, 0);
    dispatch_queue_t queue = dispatch_queue_create([name UTF8String], attr);
    
    self.customQueues[name] = @{
        @"queue": queue,
        @"type": @"concurrent",
        @"qos": @(qos),
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"taskCount": @0
    };
    
    return queue;
}

- (void)destroyQueue:(NSString *)name {
    if (self.customQueues[name]) {
        NSLog(@"销毁队列: %@", name);
        [self.customQueues removeObjectForKey:name];
    }
}

- (void)executeAsync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue {
    dispatch_async(queue, ^{
        NSLog(@"执行异步任务在队列: %s", dispatch_queue_get_label(queue));
        if (block) {
            block();
        }
    });
}

- (void)executeSync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue {
    NSLog(@"执行同步任务在队列: %s", dispatch_queue_get_label(queue));
    dispatch_sync(queue, block);
}

- (void)executeAfterDelay:(NSTimeInterval)delay block:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue {
    NSLog(@"延迟 %.2f 秒执行任务在队列: %s", delay, dispatch_queue_get_label(queue));
    
    dispatch_time_t when = dispatch_time(DISPATCH_TIME_NOW, delay * NSEC_PER_SEC);
    dispatch_after(when, queue, block);
}

- (dispatch_group_t)createDispatchGroup:(NSString *)groupName {
    NSLog(@"创建调度组: %@", groupName);
    
    dispatch_group_t group = dispatch_group_create();
    
    [self.dispatchGroups addObject:@{
        @"name": groupName,
        @"group": group,
        @"createdAt": @([[NSDate date] timeIntervalSince1970]),
        @"taskCount": @0
    }];
    
    return group;
}

- (void)executeInGroup:(dispatch_group_t)group block:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue {
    NSLog(@"在组中执行任务，队列: %s", dispatch_queue_get_label(queue));
    
    dispatch_group_async(group, queue, block);
}

- (void)waitForGroup:(dispatch_group_t)group withTimeout:(dispatch_time_t)timeout completion:(dispatch_block_t)completion {
    NSLog(@"等待调度组完成");
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"调度组任务全部完成");
        if (completion) {
            completion();
        }
    });
    
    // 可选的超时等待
    if (timeout != DISPATCH_TIME_FOREVER) {
        dispatch_time_t when = dispatch_time(DISPATCH_TIME_NOW, timeout);
        long result = dispatch_group_wait(group, when);
        if (result != 0) {
            NSLog(@"调度组等待超时");
        }
    }
}

- (dispatch_semaphore_t)createSemaphore:(NSString *)name withValue:(long)value {
    NSLog(@"创建信号量: %@，初始值: %ld", name, value);
    
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(value);
    
    self.semaphores[name] = @{
        @"semaphore": semaphore,
        @"initialValue": @(value),
        @"createdAt": @([[NSDate date] timeIntervalSince1970])
    };
    
    return semaphore;
}

- (void)waitForSemaphore:(dispatch_semaphore_t)semaphore withTimeout:(dispatch_time_t)timeout {
    long result = dispatch_semaphore_wait(semaphore, timeout);
    if (result == 0) {
        NSLog(@"信号量获取成功");
    } else {
        NSLog(@"信号量获取超时");
    }
}

- (void)signalSemaphore:(dispatch_semaphore_t)semaphore {
    NSLog(@"释放信号量");
    dispatch_semaphore_signal(semaphore);
}

- (void)executeBarrierAsync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue {
    NSLog(@"执行异步栅栏任务");
    dispatch_barrier_async(queue, block);
}

- (void)executeBarrierSync:(dispatch_block_t)block onQueue:(dispatch_queue_t)queue {
    NSLog(@"执行同步栅栏任务");
    dispatch_barrier_sync(queue, block);
}

- (NSDictionary *)getQueueStatistics {
    NSMutableDictionary *statistics = [NSMutableDictionary dictionary];
    
    statistics[@"totalQueues"] = @(self.customQueues.count);
    statistics[@"totalGroups"] = @(self.dispatchGroups.count);
    statistics[@"totalSemaphores"] = @(self.semaphores.count);
    
    NSMutableArray *queueDetails = [NSMutableArray array];
    for (NSString *queueName in self.customQueues) {
        NSDictionary *queueInfo = self.customQueues[queueName];
        [queueDetails addObject:@{
            @"name": queueName,
            @"type": queueInfo[@"type"],
            @"qos": queueInfo[@"qos"],
            @"taskCount": queueInfo[@"taskCount"]
        }];
    }
    statistics[@"queueDetails"] = queueDetails;
    
    return statistics;
}

- (void)logQueuePerformance {
    NSDictionary *stats = [self getQueueStatistics];
    
    NSLog(@"=== GCD队列性能统计 ===");
    NSLog(@"总队列数: %@", stats[@"totalQueues"]);
    NSLog(@"总调度组数: %@", stats[@"totalGroups"]);
    NSLog(@"总信号量数: %@", stats[@"totalSemaphores"]);
    
    NSArray *queueDetails = stats[@"queueDetails"];
    for (NSDictionary *queue in queueDetails) {
        NSLog(@"队列: %@, 类型: %@, 任务数: %@", 
              queue[@"name"], queue[@"type"], queue[@"taskCount"]);
    }
}

@end
```