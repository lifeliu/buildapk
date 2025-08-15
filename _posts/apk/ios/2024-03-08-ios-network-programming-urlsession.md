---
layout: post
title: "iOS网络编程深度解析：URLSession与现代网络架构设计"
date: 2024-03-08 14:20:00 +0800
categories: [iOS, 网络编程]
tags: [iOS, URLSession, 网络架构, HTTP, WebSocket]
author: iOS技术团队
description: "深入探讨iOS网络编程技术，包括URLSession的原理、使用方法和现代网络架构设计模式"
---

## 引言

在移动互联网时代，网络编程是iOS应用开发的核心技能之一。从早期的NSURLConnection到现在的URLSession，苹果不断完善和优化网络编程框架，为开发者提供了强大而灵活的网络访问能力。本文将深入解析URLSession的工作原理、使用方法，并探讨现代iOS应用的网络架构设计模式，帮助开发者构建高效、稳定的网络层。

## iOS网络编程发展历程

### 网络框架演进

**NSURLConnection时代（iOS 2.0 - iOS 8.0）**

NSURLConnection是iOS早期的网络编程框架，提供了基本的HTTP请求功能：

```objc
// NSURLConnection的基本使用（已废弃）
NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:@"https://api.example.com/data"]];
NSURLConnection *connection = [[NSURLConnection alloc] initWithRequest:request delegate:self];
[connection start];

// 实现代理方法
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    // 处理响应
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    // 处理数据
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    // 请求完成
}
```

NSURLConnection的局限性：
1. **线程管理复杂**：需要手动管理运行循环
2. **功能有限**：缺乏现代网络编程所需的高级特性
3. **代码冗长**：需要实现多个代理方法
4. **难以测试**：紧耦合的设计不利于单元测试

**URLSession时代（iOS 7.0至今）**

URLSession是苹果在iOS 7.0引入的现代网络编程框架，提供了更强大、更灵活的网络访问能力。

## URLSession架构深度解析

### URLSession核心组件

**1. URLSession**
会话对象，管理网络请求的生命周期：

```objc
// 共享会话（适用于简单请求）
NSURLSession *sharedSession = [NSURLSession sharedSession];

// 自定义会话配置
NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
config.timeoutIntervalForRequest = 30.0;
config.timeoutIntervalForResource = 60.0;
config.HTTPMaximumConnectionsPerHost = 4;

NSURLSession *customSession = [NSURLSession sessionWithConfiguration:config
                                                            delegate:self
                                                       delegateQueue:nil];
```

**2. URLSessionConfiguration**
会话配置对象，定义会话的行为和策略：

```objc
@interface NetworkConfigurationManager : NSObject
+ (NSURLSessionConfiguration *)createProductionConfiguration;
+ (NSURLSessionConfiguration *)createDebugConfiguration;
+ (NSURLSessionConfiguration *)createBackgroundConfiguration;
@end

@implementation NetworkConfigurationManager

+ (NSURLSessionConfiguration *)createProductionConfiguration {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    
    // 基本配置
    config.timeoutIntervalForRequest = 30.0;
    config.timeoutIntervalForResource = 300.0;
    config.HTTPMaximumConnectionsPerHost = 6;
    config.requestCachePolicy = NSURLRequestUseProtocolCachePolicy;
    
    // 安全配置
    config.TLSMinimumSupportedProtocol = kTLSProtocol12;
    config.HTTPShouldUsePipelining = YES;
    config.HTTPShouldSetCookies = YES;
    config.HTTPCookieAcceptPolicy = NSHTTPCookieAcceptPolicyOnlyFromMainDocumentDomain;
    
    // 网络服务类型
    config.networkServiceType = NSURLNetworkServiceTypeDefault;
    
    // 自定义HTTP头
    config.HTTPAdditionalHeaders = @{
        @"User-Agent": @"MyApp/1.0 (iOS)",
        @"Accept": @"application/json",
        @"Accept-Language": @"en-US,en;q=0.9"
    };
    
    return config;
}

+ (NSURLSessionConfiguration *)createDebugConfiguration {
    NSURLSessionConfiguration *config = [self createProductionConfiguration];
    
    // 调试模式下的特殊配置
    config.timeoutIntervalForRequest = 60.0; // 更长的超时时间
    config.requestCachePolicy = NSURLRequestReloadIgnoringLocalCacheData; // 忽略缓存
    
    return config;
}

+ (NSURLSessionConfiguration *)createBackgroundConfiguration {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:@"com.example.background"];
    
    config.timeoutIntervalForResource = 3600.0; // 1小时
    config.HTTPMaximumConnectionsPerHost = 2;
    config.discretionary = YES; // 允许系统优化网络使用
    
    return config;
}

@end
```

**3. URLSessionTask**
任务对象，代表具体的网络请求：

```objc
// 数据任务（Data Task）
NSURLSessionDataTask *dataTask = [session dataTaskWithURL:url
                                         completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
    // 处理响应
}];

// 下载任务（Download Task）
NSURLSessionDownloadTask *downloadTask = [session downloadTaskWithURL:url
                                                     completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
    // 处理下载完成
}];

// 上传任务（Upload Task）
NSURLSessionUploadTask *uploadTask = [session uploadTaskWithRequest:request
                                                            fromData:data
                                                   completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
    // 处理上传完成
}];
```

### URLSession工作原理

**1. 请求生命周期**

```objc
@interface RequestLifecycleDemo : NSObject <NSURLSessionDelegate, NSURLSessionDataDelegate>
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSMutableData *receivedData;
@end

@implementation RequestLifecycleDemo

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
        _receivedData = [[NSMutableData alloc] init];
    }
    return self;
}

- (void)performRequest {
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
    NSURLSessionDataTask *task = [self.session dataTaskWithURL:url];
    [task resume];
}

#pragma mark - NSURLSessionDelegate

- (void)URLSession:(NSURLSession *)session didBecomeInvalidWithError:(NSError *)error {
    NSLog(@"Session became invalid: %@", error.localizedDescription);
}

- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
                                             completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    // 处理认证挑战
    NSLog(@"Received authentication challenge");
    completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
}

#pragma mark - NSURLSessionDataDelegate

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                 didReceiveResponse:(NSURLResponse *)response
                                  completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler {
    NSLog(@"Received response: %@", response);
    [self.receivedData setLength:0];
    completionHandler(NSURLSessionResponseAllow);
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {
    NSLog(@"Received data: %lu bytes", (unsigned long)data.length);
    [self.receivedData appendData:data];
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    if (error) {
        NSLog(@"Task completed with error: %@", error.localizedDescription);
    } else {
        NSLog(@"Task completed successfully. Total data: %lu bytes", (unsigned long)self.receivedData.length);
        // 处理完整的响应数据
        [self processReceivedData:self.receivedData];
    }
}

- (void)processReceivedData:(NSData *)data {
    // 解析和处理数据
    NSError *error;
    id jsonObject = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
    if (jsonObject) {
        NSLog(@"Parsed JSON: %@", jsonObject);
    } else {
        NSLog(@"JSON parsing error: %@", error.localizedDescription);
    }
}

@end
```

**2. 任务状态管理**

```objc
@interface TaskStateManager : NSObject
@property (nonatomic, strong) NSMutableDictionary<NSNumber *, NSURLSessionTask *> *activeTasks;
@property (nonatomic, strong) dispatch_queue_t taskQueue;
@end

@implementation TaskStateManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _activeTasks = [[NSMutableDictionary alloc] init];
        _taskQueue = dispatch_queue_create("com.example.taskqueue", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (void)addTask:(NSURLSessionTask *)task {
    dispatch_barrier_async(self.taskQueue, ^{
        self.activeTasks[@(task.taskIdentifier)] = task;
        NSLog(@"Added task %lu. Active tasks: %lu", (unsigned long)task.taskIdentifier, (unsigned long)self.activeTasks.count);
    });
}

- (void)removeTask:(NSURLSessionTask *)task {
    dispatch_barrier_async(self.taskQueue, ^{
        [self.activeTasks removeObjectForKey:@(task.taskIdentifier)];
        NSLog(@"Removed task %lu. Active tasks: %lu", (unsigned long)task.taskIdentifier, (unsigned long)self.activeTasks.count);
    });
}

- (void)cancelAllTasks {
    dispatch_barrier_async(self.taskQueue, ^{
        for (NSURLSessionTask *task in self.activeTasks.allValues) {
            [task cancel];
        }
        [self.activeTasks removeAllObjects];
        NSLog(@"Cancelled all tasks");
    });
}

- (NSArray<NSURLSessionTask *> *)getActiveTasksWithState:(NSURLSessionTaskState)state {
    __block NSMutableArray *tasks = [[NSMutableArray alloc] init];
    dispatch_sync(self.taskQueue, ^{
        for (NSURLSessionTask *task in self.activeTasks.allValues) {
            if (task.state == state) {
                [tasks addObject:task];
            }
        }
    });
    return [tasks copy];
}

@end
```

## 现代网络架构设计

### 网络层架构模式

**1. 分层架构设计**

```objc
// 网络层接口定义
@protocol NetworkServiceProtocol <NSObject>
- (void)performRequest:(NSURLRequest *)request
            completion:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completion;
- (void)downloadFileFromURL:(NSURL *)url
                 completion:(void (^)(NSURL *fileURL, NSError *error))completion;
- (void)uploadData:(NSData *)data
             toURL:(NSURL *)url
        completion:(void (^)(NSData *response, NSError *error))completion;
@end

// 网络服务实现
@interface NetworkService : NSObject <NetworkServiceProtocol>
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) TaskStateManager *taskManager;
@end

@implementation NetworkService

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NetworkConfigurationManager createProductionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
        _taskManager = [[TaskStateManager alloc] init];
    }
    return self;
}

- (void)performRequest:(NSURLRequest *)request
            completion:(void (^)(NSData *, NSURLResponse *, NSError *))completion {
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request
                                                 completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(data, response, error);
        });
    }];
    
    [self.taskManager addTask:task];
    [task resume];
}

- (void)downloadFileFromURL:(NSURL *)url
                 completion:(void (^)(NSURL *, NSError *))completion {
    NSURLSessionDownloadTask *task = [self.session downloadTaskWithURL:url
                                                      completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        if (location && !error) {
            // 移动文件到永久位置
            NSURL *documentsURL = [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory
                                                                           inDomains:NSUserDomainMask] firstObject];
            NSURL *destinationURL = [documentsURL URLByAppendingPathComponent:[url lastPathComponent]];
            
            NSError *moveError;
            [[NSFileManager defaultManager] moveItemAtURL:location toURL:destinationURL error:&moveError];
            
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(moveError ? nil : destinationURL, moveError ?: error);
            });
        } else {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, error);
            });
        }
    }];
    
    [self.taskManager addTask:task];
    [task resume];
}

- (void)uploadData:(NSData *)data
             toURL:(NSURL *)url
        completion:(void (^)(NSData *, NSError *))completion {
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/octet-stream" forHTTPHeaderField:@"Content-Type"];
    
    NSURLSessionUploadTask *task = [self.session uploadTaskWithRequest:request
                                                               fromData:data
                                                      completionHandler:^(NSData *responseData, NSURLResponse *response, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(responseData, error);
        });
    }];
    
    [self.taskManager addTask:task];
    [task resume];
}

@end
```

**2. API客户端设计**

```objc
// API端点定义
typedef NS_ENUM(NSInteger, APIEndpoint) {
    APIEndpointUsers,
    APIEndpointPosts,
    APIEndpointComments,
    APIEndpointAuth
};

// API客户端接口
@protocol APIClientProtocol <NSObject>
- (void)fetchUsersWithCompletion:(void (^)(NSArray *users, NSError *error))completion;
- (void)createUser:(NSDictionary *)userData completion:(void (^)(NSDictionary *user, NSError *error))completion;
- (void)updateUser:(NSString *)userID withData:(NSDictionary *)userData completion:(void (^)(NSDictionary *user, NSError *error))completion;
- (void)deleteUser:(NSString *)userID completion:(void (^)(BOOL success, NSError *error))completion;
@end

// API客户端实现
@interface APIClient : NSObject <APIClientProtocol>
@property (nonatomic, strong) NetworkService *networkService;
@property (nonatomic, strong) NSURL *baseURL;
@property (nonatomic, strong) NSString *apiKey;
@end

@implementation APIClient

- (instancetype)initWithBaseURL:(NSURL *)baseURL apiKey:(NSString *)apiKey {
    self = [super init];
    if (self) {
        _baseURL = baseURL;
        _apiKey = apiKey;
        _networkService = [[NetworkService alloc] init];
    }
    return self;
}

- (NSURL *)URLForEndpoint:(APIEndpoint)endpoint {
    NSString *path;
    switch (endpoint) {
        case APIEndpointUsers:
            path = @"users";
            break;
        case APIEndpointPosts:
            path = @"posts";
            break;
        case APIEndpointComments:
            path = @"comments";
            break;
        case APIEndpointAuth:
            path = @"auth";
            break;
    }
    return [self.baseURL URLByAppendingPathComponent:path];
}

- (NSMutableURLRequest *)requestForEndpoint:(APIEndpoint)endpoint method:(NSString *)method {
    NSURL *url = [self URLForEndpoint:endpoint];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = method;
    
    // 添加通用头部
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    [request setValue:[NSString stringWithFormat:@"Bearer %@", self.apiKey] forHTTPHeaderField:@"Authorization"];
    
    return request;
}

- (void)fetchUsersWithCompletion:(void (^)(NSArray *, NSError *))completion {
    NSURLRequest *request = [self requestForEndpoint:APIEndpointUsers method:@"GET"];
    
    [self.networkService performRequest:request completion:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            completion(nil, error);
            return;
        }
        
        NSError *parseError;
        NSArray *users = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        completion(users, parseError);
    }];
}

- (void)createUser:(NSDictionary *)userData completion:(void (^)(NSDictionary *, NSError *))completion {
    NSMutableURLRequest *request = [self requestForEndpoint:APIEndpointUsers method:@"POST"];
    
    NSError *serializationError;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:userData options:0 error:&serializationError];
    if (serializationError) {
        completion(nil, serializationError);
        return;
    }
    
    request.HTTPBody = jsonData;
    
    [self.networkService performRequest:request completion:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            completion(nil, error);
            return;
        }
        
        NSError *parseError;
        NSDictionary *user = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        completion(user, parseError);
    }];
}

- (void)updateUser:(NSString *)userID withData:(NSDictionary *)userData completion:(void (^)(NSDictionary *, NSError *))completion {
    NSURL *url = [[self URLForEndpoint:APIEndpointUsers] URLByAppendingPathComponent:userID];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"PUT";
    
    // 添加头部
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    [request setValue:[NSString stringWithFormat:@"Bearer %@", self.apiKey] forHTTPHeaderField:@"Authorization"];
    
    NSError *serializationError;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:userData options:0 error:&serializationError];
    if (serializationError) {
        completion(nil, serializationError);
        return;
    }
    
    request.HTTPBody = jsonData;
    
    [self.networkService performRequest:request completion:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            completion(nil, error);
            return;
        }
        
        NSError *parseError;
        NSDictionary *user = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        completion(user, parseError);
    }];
}

- (void)deleteUser:(NSString *)userID completion:(void (^)(BOOL, NSError *))completion {
    NSURL *url = [[self URLForEndpoint:APIEndpointUsers] URLByAppendingPathComponent:userID];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"DELETE";
    
    [request setValue:[NSString stringWithFormat:@"Bearer %@", self.apiKey] forHTTPHeaderField:@"Authorization"];
    
    [self.networkService performRequest:request completion:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            completion(NO, error);
            return;
        }
        
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        BOOL success = httpResponse.statusCode >= 200 && httpResponse.statusCode < 300;
        completion(success, nil);
    }];
}

@end
```

### 网络缓存策略

**1. 多级缓存架构**

```objc
@interface NetworkCacheManager : NSObject
@property (nonatomic, strong) NSURLCache *urlCache;
@property (nonatomic, strong) NSCache *memoryCache;
@property (nonatomic, strong) NSString *diskCachePath;
@end

@implementation NetworkCacheManager

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupCaches];
    }
    return self;
}

- (void)setupCaches {
    // 设置URL缓存
    NSUInteger memoryCapacity = 10 * 1024 * 1024; // 10MB
    NSUInteger diskCapacity = 50 * 1024 * 1024;   // 50MB
    
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);
    NSString *cachesDirectory = [paths firstObject];
    NSString *cacheDirectory = [cachesDirectory stringByAppendingPathComponent:@"URLCache"];
    
    self.urlCache = [[NSURLCache alloc] initWithMemoryCapacity:memoryCapacity
                                                  diskCapacity:diskCapacity
                                                      diskPath:cacheDirectory];
    [NSURLCache setSharedURLCache:self.urlCache];
    
    // 设置内存缓存
    self.memoryCache = [[NSCache alloc] init];
    self.memoryCache.countLimit = 100;
    self.memoryCache.totalCostLimit = memoryCapacity;
    
    // 设置磁盘缓存路径
    self.diskCachePath = [cachesDirectory stringByAppendingPathComponent:@"CustomCache"];
    [[NSFileManager defaultManager] createDirectoryAtPath:self.diskCachePath
                              withIntermediateDirectories:YES
                                               attributes:nil
                                                    error:nil];
}

- (void)cacheData:(NSData *)data forKey:(NSString *)key {
    // 内存缓存
    [self.memoryCache setObject:data forKey:key cost:data.length];
    
    // 磁盘缓存
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:[self fileNameForKey:key]];
        [data writeToFile:filePath atomically:YES];
    });
}

- (NSData *)cachedDataForKey:(NSString *)key {
    // 先检查内存缓存
    NSData *data = [self.memoryCache objectForKey:key];
    if (data) {
        return data;
    }
    
    // 检查磁盘缓存
    NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:[self fileNameForKey:key]];
    data = [NSData dataWithContentsOfFile:filePath];
    if (data) {
        // 重新加载到内存缓存
        [self.memoryCache setObject:data forKey:key cost:data.length];
    }
    
    return data;
}

- (void)removeCachedDataForKey:(NSString *)key {
    [self.memoryCache removeObjectForKey:key];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:[self fileNameForKey:key]];
        [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];
    });
}

- (void)clearAllCaches {
    [self.memoryCache removeAllObjects];
    [self.urlCache removeAllCachedResponses];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSArray *files = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:self.diskCachePath error:nil];
        for (NSString *file in files) {
            NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:file];
            [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];
        }
    });
}

- (NSString *)fileNameForKey:(NSString *)key {
    // 使用MD5哈希作为文件名
    const char *cStr = [key UTF8String];
    unsigned char digest[CC_MD5_DIGEST_LENGTH];
    CC_MD5(cStr, (CC_LONG)strlen(cStr), digest);
    
    NSMutableString *output = [NSMutableString stringWithCapacity:CC_MD5_DIGEST_LENGTH * 2];
    for (int i = 0; i < CC_MD5_DIGEST_LENGTH; i++) {
        [output appendFormat:@"%02x", digest[i]];
    }
    
    return output;
}

@end
```

**2. 智能缓存策略**

```objc
@interface SmartCacheStrategy : NSObject
@property (nonatomic, strong) NetworkCacheManager *cacheManager;
@end

@implementation SmartCacheStrategy

- (instancetype)init {
    self = [super init];
    if (self) {
        _cacheManager = [[NetworkCacheManager alloc] init];
    }
    return self;
}

- (NSURLRequest *)requestWithCachePolicy:(NSURLRequest *)originalRequest {
    NSMutableURLRequest *request = [originalRequest mutableCopy];
    
    // 根据网络状况调整缓存策略
    if ([self isNetworkReachable]) {
        if ([self isWiFiConnected]) {
            // WiFi环境：使用默认缓存策略
            request.cachePolicy = NSURLRequestUseProtocolCachePolicy;
        } else {
            // 移动网络：优先使用缓存
            request.cachePolicy = NSURLRequestReturnCacheDataElseLoad;
        }
    } else {
        // 无网络：只使用缓存
        request.cachePolicy = NSURLRequestReturnCacheDataDontLoad;
    }
    
    return request;
}

- (BOOL)shouldCacheResponse:(NSURLResponse *)response data:(NSData *)data {
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    
    // 检查状态码
    if (httpResponse.statusCode < 200 || httpResponse.statusCode >= 300) {
        return NO;
    }
    
    // 检查Content-Type
    NSString *contentType = httpResponse.allHeaderFields[@"Content-Type"];
    NSArray *cacheableTypes = @[@"application/json", @"text/html", @"text/plain", @"image/"];
    
    for (NSString *type in cacheableTypes) {
        if ([contentType hasPrefix:type]) {
            return YES;
        }
    }
    
    return NO;
}

- (NSTimeInterval)cacheExpirationForResponse:(NSURLResponse *)response {
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    
    // 检查Cache-Control头
    NSString *cacheControl = httpResponse.allHeaderFields[@"Cache-Control"];
    if (cacheControl) {
        NSRegularExpression *regex = [NSRegularExpression regularExpressionWithPattern:@"max-age=(\\d+)"
                                                                               options:0
                                                                                 error:nil];
        NSTextCheckingResult *match = [regex firstMatchInString:cacheControl
                                                        options:0
                                                          range:NSMakeRange(0, cacheControl.length)];
        if (match) {
            NSString *maxAge = [cacheControl substringWithRange:[match rangeAtIndex:1]];
            return [maxAge doubleValue];
        }
    }
    
    // 检查Expires头
    NSString *expires = httpResponse.allHeaderFields[@"Expires"];
    if (expires) {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"EEE, dd MMM yyyy HH:mm:ss zzz";
        NSDate *expirationDate = [formatter dateFromString:expires];
        if (expirationDate) {
            return [expirationDate timeIntervalSinceNow];
        }
    }
    
    // 默认缓存时间
    return 3600; // 1小时
}

- (BOOL)isNetworkReachable {
    // 实现网络可达性检查
    // 这里简化处理，实际应该使用Reachability库
    return YES;
}

- (BOOL)isWiFiConnected {
    // 实现WiFi连接检查
    // 这里简化处理，实际应该使用Reachability库
    return YES;
}

@end
```

## 网络安全与认证

### HTTPS与证书验证

```objc
@interface SecureNetworkManager : NSObject <NSURLSessionDelegate>
@property (nonatomic, strong) NSURLSession *secureSession;
@property (nonatomic, strong) NSArray<NSData *> *pinnedCertificates;
@end

@implementation SecureNetworkManager

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupSecureSession];
        [self loadPinnedCertificates];
    }
    return self;
}

- (void)setupSecureSession {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    config.TLSMinimumSupportedProtocol = kTLSProtocol12;
    
    self.secureSession = [NSURLSession sessionWithConfiguration:config
                                                        delegate:self
                                                   delegateQueue:nil];
}

- (void)loadPinnedCertificates {
    NSMutableArray *certificates = [[NSMutableArray alloc] init];
    
    // 加载应用包中的证书文件
    NSArray *certFiles = @[@"api.example.com", @"cdn.example.com"];
    for (NSString *certFile in certFiles) {
        NSString *certPath = [[NSBundle mainBundle] pathForResource:certFile ofType:@"cer"];
        if (certPath) {
            NSData *certData = [NSData dataWithContentsOfFile:certPath];
            if (certData) {
                [certificates addObject:certData];
            }
        }
    }
    
    self.pinnedCertificates = [certificates copy];
}

#pragma mark - NSURLSessionDelegate

- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    NSString *authMethod = challenge.protectionSpace.authenticationMethod;
    
    if ([authMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        // 服务器信任验证
        if ([self validateServerTrust:challenge.protectionSpace.serverTrust
                            forDomain:challenge.protectionSpace.host]) {
            NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        } else {
            completionHandler(NSURLSessionAuthChallengeRejectProtectionSpace, nil);
        }
    } else if ([authMethod isEqualToString:NSURLAuthenticationMethodClientCertificate]) {
        // 客户端证书验证
        NSURLCredential *credential = [self clientCertificateCredential];
        if (credential) {
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        } else {
            completionHandler(NSURLSessionAuthChallengeRejectProtectionSpace, nil);
        }
    } else {
        // 其他认证方法使用默认处理
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
    }
}

- (BOOL)validateServerTrust:(SecTrustRef)serverTrust forDomain:(NSString *)domain {
    // 1. 基本的系统验证
    SecTrustResultType result;
    OSStatus status = SecTrustEvaluate(serverTrust, &result);
    
    if (status != errSecSuccess) {
        return NO;
    }
    
    if (result != kSecTrustResultUnspecified && result != kSecTrustResultProceed) {
        // 如果系统验证失败，检查是否是我们信任的证书
        return [self validateCertificatePinning:serverTrust];
    }
    
    // 2. 证书固定验证
    return [self validateCertificatePinning:serverTrust];
}

- (BOOL)validateCertificatePinning:(SecTrustRef)serverTrust {
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    
    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        NSData *certificateData = (__bridge NSData *)SecCertificateCopyData(certificate);
        
        for (NSData *pinnedCert in self.pinnedCertificates) {
            if ([certificateData isEqualToData:pinnedCert]) {
                return YES;
            }
        }
    }
    
    return NO;
}

- (NSURLCredential *)clientCertificateCredential {
    // 加载客户端证书
    NSString *certPath = [[NSBundle mainBundle] pathForResource:@"client" ofType:@"p12"];
    if (!certPath) {
        return nil;
    }
    
    NSData *certData = [NSData dataWithContentsOfFile:certPath];
    if (!certData) {
        return nil;
    }
    
    CFDataRef inP12Data = (__bridge CFDataRef)certData;
    SecIdentityRef identity;
    SecTrustRef trust;
    
    const void *keys[] = { kSecImportExportPassphrase };
    const void *values[] = { CFSTR("password") }; // 证书密码
    CFDictionaryRef options = CFDictionaryCreate(NULL, keys, values, 1, NULL, NULL);
    
    CFArrayRef items;
    OSStatus status = SecPKCS12Import(inP12Data, options, &items);
    
    if (status == errSecSuccess) {
        CFDictionaryRef identityDict = CFArrayGetValueAtIndex(items, 0);
        identity = (SecIdentityRef)CFDictionaryGetValue(identityDict, kSecImportItemIdentity);
        trust = (SecTrustRef)CFDictionaryGetValue(identityDict, kSecImportItemTrust);
        
        NSURLCredential *credential = [NSURLCredential credentialWithIdentity:identity
                                                                  certificates:nil
                                                                   persistence:NSURLCredentialPersistenceNone];
        
        CFRelease(items);
        CFRelease(options);
        
        return credential;
    }
    
    CFRelease(options);
    return nil;
}

@end
```

### OAuth 2.0认证实现

```objc
@interface OAuth2Manager : NSObject
@property (nonatomic, strong) NSString *clientID;
@property (nonatomic, strong) NSString *clientSecret;
@property (nonatomic, strong) NSString *redirectURI;
@property (nonatomic, strong) NSString *accessToken;
@property (nonatomic, strong) NSString *refreshToken;
@property (nonatomic, strong) NSDate *tokenExpirationDate;
@end

@implementation OAuth2Manager

- (instancetype)initWithClientID:(NSString *)clientID
                    clientSecret:(NSString *)clientSecret
                     redirectURI:(NSString *)redirectURI {
    self = [super init];
    if (self) {
        _clientID = clientID;
        _clientSecret = clientSecret;
        _redirectURI = redirectURI;
        [self loadStoredTokens];
    }
    return self;
}

- (NSURL *)authorizationURL {
    NSURLComponents *components = [NSURLComponents componentsWithString:@"https://api.example.com/oauth/authorize"];
    components.queryItems = @[
        [NSURLQueryItem queryItemWithName:@"client_id" value:self.clientID],
        [NSURLQueryItem queryItemWithName:@"redirect_uri" value:self.redirectURI],
        [NSURLQueryItem queryItemWithName:@"response_type" value:@"code"],
        [NSURLQueryItem queryItemWithName:@"scope" value:@"read write"],
        [NSURLQueryItem queryItemWithName:@"state" value:[self generateState]]
    ];
    return components.URL;
}

- (void)exchangeAuthorizationCode:(NSString *)code
                       completion:(void (^)(BOOL success, NSError *error))completion {
    NSURL *tokenURL = [NSURL URLWithString:@"https://api.example.com/oauth/token"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:tokenURL];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
    
    NSString *bodyString = [NSString stringWithFormat:@"grant_type=authorization_code&code=%@&client_id=%@&client_secret=%@&redirect_uri=%@",
                           code, self.clientID, self.clientSecret, self.redirectURI];
    request.HTTPBody = [bodyString dataUsingEncoding:NSUTF8StringEncoding];
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:request
                                                                 completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(NO, error);
            });
            return;
        }
        
        NSError *parseError;
        NSDictionary *tokenData = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        
        if (parseError) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(NO, parseError);
            });
            return;
        }
        
        [self processTokenResponse:tokenData];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(YES, nil);
        });
    }];
    
    [task resume];
}

- (void)refreshAccessTokenWithCompletion:(void (^)(BOOL success, NSError *error))completion {
    if (!self.refreshToken) {
        NSError *error = [NSError errorWithDomain:@"OAuth2Error"
                                             code:1001
                                         userInfo:@{NSLocalizedDescriptionKey: @"No refresh token available"}];
        completion(NO, error);
        return;
    }
    
    NSURL *tokenURL = [NSURL URLWithString:@"https://api.example.com/oauth/token"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:tokenURL];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
    
    NSString *bodyString = [NSString stringWithFormat:@"grant_type=refresh_token&refresh_token=%@&client_id=%@&client_secret=%@",
                           self.refreshToken, self.clientID, self.clientSecret];
    request.HTTPBody = [bodyString dataUsingEncoding:NSUTF8StringEncoding];
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:request
                                                                 completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(NO, error);
            });
            return;
        }
        
        NSError *parseError;
        NSDictionary *tokenData = [NSJSONSerialization JSONObjectWithData:data options:0 error:&parseError];
        
        if (parseError) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(NO, parseError);
            });
            return;
        }
        
        [self processTokenResponse:tokenData];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(YES, nil);
        });
    }];
    
    [task resume];
}

- (void)processTokenResponse:(NSDictionary *)tokenData {
    self.accessToken = tokenData[@"access_token"];
    self.refreshToken = tokenData[@"refresh_token"];
    
    NSNumber *expiresIn = tokenData[@"expires_in"];
    if (expiresIn) {
        self.tokenExpirationDate = [NSDate dateWithTimeIntervalSinceNow:[expiresIn doubleValue]];
    }
    
    [self saveTokens];
}

- (BOOL)isAccessTokenValid {
    if (!self.accessToken) {
        return NO;
    }
    
    if (self.tokenExpirationDate && [self.tokenExpirationDate timeIntervalSinceNow] <= 60) {
        // Token expires within 60 seconds
        return NO;
    }
    
    return YES;
}

- (void)addAuthorizationToRequest:(NSMutableURLRequest *)request {
    if (self.accessToken) {
        [request setValue:[NSString stringWithFormat:@"Bearer %@", self.accessToken]
       forHTTPHeaderField:@"Authorization"];
    }
}

- (NSString *)generateState {
    NSMutableString *state = [NSMutableString string];
    NSString *letters = @"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    
    for (int i = 0; i < 32; i++) {
        [state appendFormat:@"%C", [letters characterAtIndex:arc4random_uniform((uint32_t)letters.length)]];
    }
    
    return state;
}

- (void)saveTokens {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    [defaults setObject:self.accessToken forKey:@"oauth_access_token"];
    [defaults setObject:self.refreshToken forKey:@"oauth_refresh_token"];
    [defaults setObject:self.tokenExpirationDate forKey:@"oauth_token_expiration"];
    [defaults synchronize];
}

- (void)loadStoredTokens {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    self.accessToken = [defaults stringForKey:@"oauth_access_token"];
    self.refreshToken = [defaults stringForKey:@"oauth_refresh_token"];
    self.tokenExpirationDate = [defaults objectForKey:@"oauth_token_expiration"];
}

- (void)clearTokens {
    self.accessToken = nil;
    self.refreshToken = nil;
    self.tokenExpirationDate = nil;
    
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    [defaults removeObjectForKey:@"oauth_access_token"];
    [defaults removeObjectForKey:@"oauth_refresh_token"];
    [defaults removeObjectForKey:@"oauth_token_expiration"];
    [defaults synchronize];
}

@end
```

## 网络性能优化

### 请求优化策略

```objc
@interface NetworkOptimizer : NSObject
@property (nonatomic, strong) NSOperationQueue *requestQueue;
@property (nonatomic, strong) NSMutableDictionary *pendingRequests;
@property (nonatomic, strong) dispatch_queue_t optimizerQueue;
@end

@implementation NetworkOptimizer

- (instancetype)init {
    self = [super init];
    if (self) {
        _requestQueue = [[NSOperationQueue alloc] init];
        _requestQueue.maxConcurrentOperationCount = 4;
        _pendingRequests = [[NSMutableDictionary alloc] init];
        _optimizerQueue = dispatch_queue_create("com.example.optimizer", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

// 请求去重
- (void)performRequest:(NSURLRequest *)request
            completion:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completion {
    NSString *requestKey = [self keyForRequest:request];
    
    dispatch_barrier_async(self.optimizerQueue, ^{
        NSMutableArray *callbacks = self.pendingRequests[requestKey];
        if (callbacks) {
            // 请求已在进行中，添加回调
            [callbacks addObject:[completion copy]];
            return;
        }
        
        // 新请求，创建回调数组
        callbacks = [[NSMutableArray alloc] init];
        [callbacks addObject:[completion copy]];
        self.pendingRequests[requestKey] = callbacks;
        
        // 执行实际请求
        [self executeRequest:request withKey:requestKey];
    });
}

- (void)executeRequest:(NSURLRequest *)request withKey:(NSString *)requestKey {
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:request
                                                                 completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        dispatch_barrier_async(self.optimizerQueue, ^{
            NSArray *callbacks = self.pendingRequests[requestKey];
            [self.pendingRequests removeObjectForKey:requestKey];
            
            // 调用所有等待的回调
            for (void (^callback)(NSData *, NSURLResponse *, NSError *) in callbacks) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    callback(data, response, error);
                });
            }
        });
    }];
    
    [task resume];
}

- (NSString *)keyForRequest:(NSURLRequest *)request {
    NSMutableString *key = [NSMutableString stringWithString:request.URL.absoluteString];
    [key appendString:request.HTTPMethod ?: @"GET"];
    
    if (request.HTTPBody) {
        [key appendString:[[NSString alloc] initWithData:request.HTTPBody encoding:NSUTF8StringEncoding]];
    }
    
    return key;
}

// 批量请求
- (void)performBatchRequests:(NSArray<NSURLRequest *> *)requests
                  completion:(void (^)(NSArray<NSDictionary *> *results))completion {
    NSMutableArray *results = [[NSMutableArray alloc] init];
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t concurrentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    for (NSUInteger i = 0; i < requests.count; i++) {
        [results addObject:[NSNull null]];
        
        dispatch_group_enter(group);
        [self performRequest:requests[i] completion:^(NSData *data, NSURLResponse *response, NSError *error) {
            dispatch_async(concurrentQueue, ^{
                NSDictionary *result = @{
                    @"data": data ?: [NSNull null],
                    @"response": response ?: [NSNull null],
                    @"error": error ?: [NSNull null]
                };
                
                @synchronized(results) {
                    results[i] = result;
                }
                
                dispatch_group_leave(group);
            });
        }];
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        completion([results copy]);
    });
}

// 请求重试机制
- (void)performRequestWithRetry:(NSURLRequest *)request
                     maxRetries:(NSInteger)maxRetries
                     completion:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completion {
    [self performRequestWithRetry:request currentAttempt:0 maxRetries:maxRetries completion:completion];
}

- (void)performRequestWithRetry:(NSURLRequest *)request
                 currentAttempt:(NSInteger)currentAttempt
                     maxRetries:(NSInteger)maxRetries
                     completion:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completion {
    
    [self performRequest:request completion:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error && currentAttempt < maxRetries) {
            // 计算重试延迟（指数退避）
            NSTimeInterval delay = pow(2, currentAttempt) * 0.5; // 0.5s, 1s, 2s, 4s...
            
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                [self performRequestWithRetry:request
                                currentAttempt:currentAttempt + 1
                                    maxRetries:maxRetries
                                    completion:completion];
            });
        } else {
            completion(data, response, error);
        }
    }];
}

@end
```

### 网络监控与分析

```objc
@interface NetworkMonitor : NSObject
@property (nonatomic, strong) NSMutableArray<NSDictionary *> *requestMetrics;
@property (nonatomic, strong) dispatch_queue_t metricsQueue;
@end

@implementation NetworkMonitor

- (instancetype)init {
    self = [super init];
    if (self) {
        _requestMetrics = [[NSMutableArray alloc] init];
        _metricsQueue = dispatch_queue_create("com.example.metrics", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (void)recordRequestMetrics:(NSURLRequest *)request
                    response:(NSURLResponse *)response
                       error:(NSError *)error
                   startTime:(NSDate *)startTime
                     endTime:(NSDate *)endTime {
    
    NSTimeInterval duration = [endTime timeIntervalSinceDate:startTime];
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    
    NSDictionary *metrics = @{
        @"url": request.URL.absoluteString,
        @"method": request.HTTPMethod ?: @"GET",
        @"statusCode": @(httpResponse.statusCode),
        @"duration": @(duration),
        @"responseSize": @(httpResponse.expectedContentLength),
        @"error": error ? error.localizedDescription : [NSNull null],
        @"timestamp": startTime
    };
    
    dispatch_barrier_async(self.metricsQueue, ^{
        [self.requestMetrics addObject:metrics];
        
        // 保持最近1000条记录
        if (self.requestMetrics.count > 1000) {
            [self.requestMetrics removeObjectsInRange:NSMakeRange(0, self.requestMetrics.count - 1000)];
        }
    });
}

- (NSDictionary *)generatePerformanceReport {
    __block NSArray *metrics;
    dispatch_sync(self.metricsQueue, ^{
        metrics = [self.requestMetrics copy];
    });
    
    if (metrics.count == 0) {
        return @{};
    }
    
    // 计算统计数据
    NSMutableDictionary *report = [[NSMutableDictionary alloc] init];
    
    // 总请求数
    report[@"totalRequests"] = @(metrics.count);
    
    // 成功率
    NSInteger successCount = 0;
    NSInteger errorCount = 0;
    NSMutableArray *durations = [[NSMutableArray alloc] init];
    NSMutableDictionary *statusCodes = [[NSMutableDictionary alloc] init];
    
    for (NSDictionary *metric in metrics) {
        NSNumber *statusCode = metric[@"statusCode"];
        NSNumber *duration = metric[@"duration"];
        
        if (statusCode.integerValue >= 200 && statusCode.integerValue < 300) {
            successCount++;
        } else {
            errorCount++;
        }
        
        [durations addObject:duration];
        
        NSString *statusKey = statusCode.stringValue;
        statusCodes[statusKey] = @([statusCodes[statusKey] integerValue] + 1);
    }
    
    // 成功率
    double successRate = (double)successCount / metrics.count * 100;
    report[@"successRate"] = @(successRate);
    report[@"errorRate"] = @(100 - successRate);
    
    // 响应时间统计
    [durations sortUsingComparator:^NSComparisonResult(NSNumber *obj1, NSNumber *obj2) {
        return [obj1 compare:obj2];
    }];
    
    double averageDuration = [[durations valueForKeyPath:@"@avg.doubleValue"] doubleValue];
    double medianDuration = [durations[durations.count / 2] doubleValue];
    double p95Duration = [durations[(NSUInteger)(durations.count * 0.95)] doubleValue];
    double p99Duration = [durations[(NSUInteger)(durations.count * 0.99)] doubleValue];
    
    report[@"averageResponseTime"] = @(averageDuration);
    report[@"medianResponseTime"] = @(medianDuration);
    report[@"p95ResponseTime"] = @(p95Duration);
    report[@"p99ResponseTime"] = @(p99Duration);
    
    // 状态码分布
    report[@"statusCodeDistribution"] = statusCodes;
    
    return report;
}

- (NSArray *)getSlowRequests:(NSTimeInterval)threshold {
    __block NSArray *metrics;
    dispatch_sync(self.metricsQueue, ^{
        metrics = [self.requestMetrics copy];
    });
    
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"duration > %f", threshold];
    return [metrics filteredArrayUsingPredicate:predicate];
}

- (NSArray *)getFailedRequests {
    __block NSArray *metrics;
    dispatch_sync(self.metricsQueue, ^{
        metrics = [self.requestMetrics copy];
    });
    
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"error != %@", [NSNull null]];
    return [metrics filteredArrayUsingPredicate:predicate];
}

@end
```

## WebSocket实时通信

### WebSocket基础实现

```objc
@protocol WebSocketManagerDelegate <NSObject>
- (void)webSocketDidConnect:(WebSocketManager *)manager;
- (void)webSocketDidDisconnect:(WebSocketManager *)manager error:(NSError *)error;
- (void)webSocket:(WebSocketManager *)manager didReceiveMessage:(NSString *)message;
- (void)webSocket:(WebSocketManager *)manager didReceiveData:(NSData *)data;
@end

@interface WebSocketManager : NSObject <NSURLSessionWebSocketDelegate>
@property (nonatomic, weak) id<WebSocketManagerDelegate> delegate;
@property (nonatomic, strong) NSURLSessionWebSocketTask *webSocketTask;
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, assign) BOOL isConnected;
@property (nonatomic, strong) NSTimer *pingTimer;
@end

@implementation WebSocketManager

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
    }
    return self;
}

- (void)connectToURL:(NSURL *)url {
    if (self.isConnected) {
        [self disconnect];
    }
    
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    self.webSocketTask = [self.session webSocketTaskWithRequest:request];
    [self.webSocketTask resume];
    
    [self startReceiving];
}

- (void)disconnect {
    [self.pingTimer invalidate];
    self.pingTimer = nil;
    
    if (self.webSocketTask) {
        [self.webSocketTask cancelWithCloseCode:NSURLSessionWebSocketCloseCodeNormalClosure reason:nil];
        self.webSocketTask = nil;
    }
    
    self.isConnected = NO;
}

- (void)sendMessage:(NSString *)message {
    if (!self.isConnected) {
        NSLog(@"WebSocket not connected");
        return;
    }
    
    NSURLSessionWebSocketMessage *webSocketMessage = [[NSURLSessionWebSocketMessage alloc] initWithString:message];
    [self.webSocketTask sendMessage:webSocketMessage completionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to send message: %@", error.localizedDescription);
        }
    }];
}

- (void)sendData:(NSData *)data {
    if (!self.isConnected) {
        NSLog(@"WebSocket not connected");
        return;
    }
    
    NSURLSessionWebSocketMessage *webSocketMessage = [[NSURLSessionWebSocketMessage alloc] initWithData:data];
    [self.webSocketTask sendMessage:webSocketMessage completionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to send data: %@", error.localizedDescription);
        }
    }];
}

- (void)startReceiving {
    [self.webSocketTask receiveMessageWithCompletionHandler:^(NSURLSessionWebSocketMessage *message, NSError *error) {
        if (error) {
            NSLog(@"Failed to receive message: %@", error.localizedDescription);
            return;
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            switch (message.type) {
                case NSURLSessionWebSocketMessageTypeString:
                    [self.delegate webSocket:self didReceiveMessage:message.string];
                    break;
                case NSURLSessionWebSocketMessageTypeData:
                    [self.delegate webSocket:self didReceiveData:message.data];
                    break;
            }
        });
        
        // 继续接收下一条消息
        [self startReceiving];
    }];
}

- (void)startPingTimer {
    self.pingTimer = [NSTimer scheduledTimerWithTimeInterval:30.0
                                                      target:self
                                                    selector:@selector(sendPing)
                                                    userInfo:nil
                                                     repeats:YES];
}

- (void)sendPing {
    if (self.isConnected) {
        [self.webSocketTask sendPingWithPongReceiveHandler:^(NSError *error) {
            if (error) {
                NSLog(@"Ping failed: %@", error.localizedDescription);
            }
        }];
    }
}

#pragma mark - NSURLSessionWebSocketDelegate

- (void)URLSession:(NSURLSession *)session webSocketTask:(NSURLSessionWebSocketTask *)webSocketTask didOpenWithProtocol:(NSString *)protocol {
    NSLog(@"WebSocket connected with protocol: %@", protocol);
    self.isConnected = YES;
    [self startPingTimer];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [self.delegate webSocketDidConnect:self];
    });
}

- (void)URLSession:(NSURLSession *)session webSocketTask:(NSURLSessionWebSocketTask *)webSocketTask didCloseWithCode:(NSURLSessionWebSocketCloseCode)closeCode reason:(NSData *)reason {
    NSLog(@"WebSocket closed with code: %ld", (long)closeCode);
    self.isConnected = NO;
    [self.pingTimer invalidate];
    self.pingTimer = nil;
    
    NSString *reasonString = reason ? [[NSString alloc] initWithData:reason encoding:NSUTF8StringEncoding] : @"Unknown";
    NSError *error = [NSError errorWithDomain:@"WebSocketError"
                                         code:closeCode
                                     userInfo:@{NSLocalizedDescriptionKey: reasonString}];
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [self.delegate webSocketDidDisconnect:self error:error];
    });
}

@end
```

## 总结

iOS网络编程是现代移动应用开发的核心技能。URLSession作为苹果提供的现代网络框架，具有强大的功能和灵活的配置选项。通过合理的架构设计、缓存策略、安全措施和性能优化，可以构建出高效、稳定、安全的网络层。

**关键要点：**

1. **URLSession架构**：理解会话、配置和任务的关系
2. **网络架构设计**：采用分层架构，分离关注点
3. **缓存策略**：实现多级缓存，提升用户体验
4. **安全措施**：实施HTTPS、证书固定和OAuth认证
5. **性能优化**：请求去重、批量处理、重试机制
6. **实时通信**：WebSocket实现双向通信

在实际开发中，应该根据应用的具体需求选择合适的技术方案，同时注重代码的可维护性、可测试性和扩展性。随着网络技术的不断发展，开发者需要持续学习和实践，以构建出更优秀的网络层架构。