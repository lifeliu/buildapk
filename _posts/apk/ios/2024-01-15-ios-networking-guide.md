---
layout: post
title: "iOS网络编程深度解析：URLSession、网络安全与性能优化实战指南"
date: 2024-01-15
categories: ios
tags: [iOS, Networking, URLSession, Security, Performance, HTTP, HTTPS, WebSocket]
author: "iOS技术专家"
description: "深入探讨iOS网络编程技术，包括URLSession框架使用、网络安全实现、性能优化策略和实际应用案例，帮助开发者构建高效安全的网络层架构。"
keywords: "iOS网络编程,URLSession,网络安全,性能优化,HTTP,HTTPS,WebSocket,网络架构"
---

在现代iOS应用开发中，网络编程是不可或缺的核心技术。本文将深入探讨iOS网络编程的各个方面，从基础的URLSession使用到高级的网络安全和性能优化，为开发者提供全面的网络编程解决方案。

## 网络编程概述

### 网络架构设计

```objc
@interface NetworkArchitecture : NSObject

@property (nonatomic, strong) NSURLSession *defaultSession;
@property (nonatomic, strong) NSURLSession *backgroundSession;
@property (nonatomic, strong) NSURLSession *ephemeralSession;
@property (nonatomic, strong) NSOperationQueue *networkQueue;
@property (nonatomic, strong) dispatch_queue_t callbackQueue;

+ (instancetype)sharedArchitecture;

// 网络层初始化
- (void)setupNetworkArchitecture;
- (void)configureSessionWithConfiguration:(NSURLSessionConfiguration *)configuration;

// 网络状态监控
- (void)startNetworkMonitoring;
- (void)stopNetworkMonitoring;
- (BOOL)isNetworkAvailable;
- (NSString *)getCurrentNetworkType;

// 请求管理
- (NSURLSessionDataTask *)performRequest:(NSURLRequest *)request
                       completionHandler:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;

// 性能监控
- (void)logNetworkMetrics:(NSURLSessionTask *)task;
- (NSDictionary *)getNetworkPerformanceReport;

@end

@implementation NetworkArchitecture

+ (instancetype)sharedArchitecture {
    static NetworkArchitecture *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupNetworkArchitecture];
    }
    return self;
}

- (void)setupNetworkArchitecture {
    // 创建默认会话配置
    NSURLSessionConfiguration *defaultConfig = [NSURLSessionConfiguration defaultSessionConfiguration];
    defaultConfig.timeoutIntervalForRequest = 30.0;
    defaultConfig.timeoutIntervalForResource = 60.0;
    defaultConfig.HTTPMaximumConnectionsPerHost = 6;
    defaultConfig.requestCachePolicy = NSURLRequestUseProtocolCachePolicy;
    
    // 设置HTTP头
    defaultConfig.HTTPAdditionalHeaders = @{
        @"User-Agent": [self getUserAgent],
        @"Accept": @"application/json",
        @"Accept-Language": [[NSLocale preferredLanguages] firstObject],
        @"Accept-Encoding": @"gzip, deflate"
    };
    
    self.defaultSession = [NSURLSession sessionWithConfiguration:defaultConfig
                                                        delegate:self
                                                   delegateQueue:nil];
    
    // 创建后台会话配置
    NSURLSessionConfiguration *backgroundConfig = [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:@"com.app.background"];
    backgroundConfig.discretionary = YES;
    backgroundConfig.sessionSendsLaunchEvents = YES;
    
    self.backgroundSession = [NSURLSession sessionWithConfiguration:backgroundConfig
                                                           delegate:self
                                                      delegateQueue:nil];
    
    // 创建临时会话配置
    NSURLSessionConfiguration *ephemeralConfig = [NSURLSessionConfiguration ephemeralSessionConfiguration];
    ephemeralConfig.URLCache = nil;
    ephemeralConfig.URLCredentialStorage = nil;
    ephemeralConfig.HTTPCookieStorage = nil;
    
    self.ephemeralSession = [NSURLSession sessionWithConfiguration:ephemeralConfig
                                                          delegate:self
                                                     delegateQueue:nil];
    
    // 创建网络队列
    self.networkQueue = [[NSOperationQueue alloc] init];
    self.networkQueue.maxConcurrentOperationCount = 4;
    self.networkQueue.name = @"NetworkQueue";
    
    // 创建回调队列
    self.callbackQueue = dispatch_queue_create("com.app.network.callback", DISPATCH_QUEUE_CONCURRENT);
    
    // 开始网络监控
    [self startNetworkMonitoring];
}

- (NSString *)getUserAgent {
    NSBundle *bundle = [NSBundle mainBundle];
    NSString *appName = [bundle objectForInfoDictionaryKey:@"CFBundleName"];
    NSString *appVersion = [bundle objectForInfoDictionaryKey:@"CFBundleShortVersionString"];
    NSString *buildNumber = [bundle objectForInfoDictionaryKey:@"CFBundleVersion"];
    
    UIDevice *device = [UIDevice currentDevice];
    NSString *deviceModel = device.model;
    NSString *systemVersion = device.systemVersion;
    
    return [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; %@)",
            appName, appVersion, buildNumber, systemVersion, deviceModel];
}

- (void)startNetworkMonitoring {
    // 使用Reachability或Network framework监控网络状态
    // 这里简化实现
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(networkStatusChanged:)
                                                 name:@"NetworkStatusChanged"
                                               object:nil];
}

- (void)stopNetworkMonitoring {
    [[NSNotificationCenter defaultCenter] removeObserver:self
                                                    name:@"NetworkStatusChanged"
                                                  object:nil];
}

- (void)networkStatusChanged:(NSNotification *)notification {
    NSString *networkType = [self getCurrentNetworkType];
    NSLog(@"Network status changed to: %@", networkType);
    
    // 根据网络类型调整配置
    if ([networkType isEqualToString:@"Cellular"]) {
        // 蜂窝网络下减少并发连接数
        self.defaultSession.configuration.HTTPMaximumConnectionsPerHost = 2;
    } else if ([networkType isEqualToString:@"WiFi"]) {
        // WiFi网络下增加并发连接数
        self.defaultSession.configuration.HTTPMaximumConnectionsPerHost = 6;
    }
}

- (BOOL)isNetworkAvailable {
    // 实现网络可用性检查
    return YES; // 简化实现
}

- (NSString *)getCurrentNetworkType {
    // 实现网络类型检测
    return @"WiFi"; // 简化实现
}

- (NSURLSessionDataTask *)performRequest:(NSURLRequest *)request
                       completionHandler:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler {
    
    NSURLSessionDataTask *task = [self.defaultSession dataTaskWithRequest:request
                                                         completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        dispatch_async(self.callbackQueue, ^{
            // 记录网络指标
            [self logNetworkMetrics:task];
            
            // 执行回调
            if (completionHandler) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completionHandler(data, response, error);
                });
            }
        });
    }];
    
    [task resume];
    return task;
}

- (void)logNetworkMetrics:(NSURLSessionTask *)task {
    NSURLSessionTaskMetrics *metrics = task.taskMetrics;
    if (metrics) {
        for (NSURLSessionTaskTransactionMetrics *transactionMetrics in metrics.transactionMetrics) {
            NSTimeInterval totalTime = [transactionMetrics.responseEndDate timeIntervalSinceDate:transactionMetrics.requestStartDate];
            NSTimeInterval dnsTime = [transactionMetrics.domainLookupEndDate timeIntervalSinceDate:transactionMetrics.domainLookupStartDate];
            NSTimeInterval connectTime = [transactionMetrics.connectEndDate timeIntervalSinceDate:transactionMetrics.connectStartDate];
            NSTimeInterval requestTime = [transactionMetrics.responseStartDate timeIntervalSinceDate:transactionMetrics.requestStartDate];
            
            NSLog(@"Network Metrics - Total: %.3fs, DNS: %.3fs, Connect: %.3fs, Request: %.3fs",
                  totalTime, dnsTime, connectTime, requestTime);
        }
    }
}

- (NSDictionary *)getNetworkPerformanceReport {
    // 返回网络性能报告
    return @{
        @"total_requests": @(100),
        @"successful_requests": @(95),
        @"failed_requests": @(5),
        @"average_response_time": @(0.5),
        @"cache_hit_rate": @(0.8)
    };
}

- (NSURLSessionUploadTask *)uploadFile:(NSURL *)fileURL
                                 toURL:(NSString *)URLString
                            parameters:(NSDictionary *)parameters
                               headers:(NSDictionary *)headers
                              progress:(void(^)(NSProgress *progress))progress
                               success:(void(^)(NSURLSessionUploadTask *task, id responseObject))success
                               failure:(void(^)(NSURLSessionUploadTask *task, NSError *error))failure {
    
    // 创建multipart请求
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:URLString]];
    request.HTTPMethod = @"POST";
    
    NSString *boundary = [NSString stringWithFormat:@"Boundary-%@", [[NSUUID UUID] UUIDString]];
    NSString *contentType = [NSString stringWithFormat:@"multipart/form-data; boundary=%@", boundary];
    [request setValue:contentType forHTTPHeaderField:@"Content-Type"];
    
    // 设置自定义头部
    if (headers) {
        for (NSString *key in headers) {
            [request setValue:headers[key] forHTTPHeaderField:key];
        }
    }
    
    // 构建multipart数据
    NSMutableData *bodyData = [NSMutableData data];
    
    // 添加参数
    if (parameters) {
        for (NSString *key in parameters) {
            [bodyData appendData:[[NSString stringWithFormat:@"--%@\r\n", boundary] dataUsingEncoding:NSUTF8StringEncoding]];
            [bodyData appendData:[[NSString stringWithFormat:@"Content-Disposition: form-data; name=\"%@\"\r\n\r\n", key] dataUsingEncoding:NSUTF8StringEncoding]];
            [bodyData appendData:[[NSString stringWithFormat:@"%@\r\n", parameters[key]] dataUsingEncoding:NSUTF8StringEncoding]];
        }
    }
    
    // 添加文件
    NSString *fileName = [fileURL lastPathComponent];
    NSString *mimeType = [self mimeTypeForFileExtension:[fileName pathExtension]];
    
    [bodyData appendData:[[NSString stringWithFormat:@"--%@\r\n", boundary] dataUsingEncoding:NSUTF8StringEncoding]];
    [bodyData appendData:[[NSString stringWithFormat:@"Content-Disposition: form-data; name=\"file\"; filename=\"%@\"\r\n", fileName] dataUsingEncoding:NSUTF8StringEncoding]];
    [bodyData appendData:[[NSString stringWithFormat:@"Content-Type: %@\r\n\r\n", mimeType] dataUsingEncoding:NSUTF8StringEncoding]];
    
    NSData *fileData = [NSData dataWithContentsOfURL:fileURL];
    [bodyData appendData:fileData];
    [bodyData appendData:[@"\r\n" dataUsingEncoding:NSUTF8StringEncoding]];
    
    [bodyData appendData:[[NSString stringWithFormat:@"--%@--\r\n", boundary] dataUsingEncoding:NSUTF8StringEncoding]];
    
    NSURLSessionUploadTask *uploadTask = [self.session uploadTaskWithRequest:request
                                                                     fromData:bodyData
                                                            completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        [self.activeTasks removeObjectForKey:@(uploadTask.taskIdentifier)];
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (failure) {
                    failure(uploadTask, error);
                }
            });
            return;
        }
        
        id responseObject = [self parseResponseData:data response:response error:&error];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                if (failure) {
                    failure(uploadTask, error);
                }
            } else {
                if (success) {
                    success(uploadTask, responseObject);
                }
            }
        });
    }];
    
    // 设置进度回调
    if (progress) {
        [uploadTask addObserver:self forKeyPath:@"countOfBytesSent" options:NSKeyValueObservingOptionNew context:nil];
    }
    
    self.activeTasks[@(uploadTask.taskIdentifier)] = uploadTask;
    [uploadTask resume];
    
    return uploadTask;
}

- (NSURLSessionDownloadTask *)downloadFile:(NSString *)URLString
                               destination:(NSURL *)destinationURL
                                  progress:(void(^)(NSProgress *progress))progress
                                   success:(void(^)(NSURLSessionDownloadTask *task, NSURL *filePath))success
                                   failure:(void(^)(NSURLSessionDownloadTask *task, NSError *error))failure {
    
    NSURL *URL = [NSURL URLWithString:URLString];
    NSURLRequest *request = [NSURLRequest requestWithURL:URL];
    
    NSURLSessionDownloadTask *downloadTask = [self.session downloadTaskWithRequest:request
                                                                  completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        
        [self.activeTasks removeObjectForKey:@(downloadTask.taskIdentifier)];
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (failure) {
                    failure(downloadTask, error);
                }
            });
            return;
        }
        
        // 移动文件到目标位置
        NSError *moveError;
        [[NSFileManager defaultManager] moveItemAtURL:location toURL:destinationURL error:&moveError];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (moveError) {
                if (failure) {
                    failure(downloadTask, moveError);
                }
            } else {
                if (success) {
                    success(downloadTask, destinationURL);
                }
            }
        });
    }];
    
    // 设置进度回调
    if (progress) {
        [downloadTask addObserver:self forKeyPath:@"countOfBytesReceived" options:NSKeyValueObservingOptionNew context:nil];
    }
    
    self.activeTasks[@(downloadTask.taskIdentifier)] = downloadTask;
    [downloadTask resume];
    
    return downloadTask;
}

- (NSString *)mimeTypeForFileExtension:(NSString *)extension {
    NSDictionary *mimeTypes = @{
        @"jpg": @"image/jpeg",
        @"jpeg": @"image/jpeg",
        @"png": @"image/png",
        @"gif": @"image/gif",
        @"pdf": @"application/pdf",
        @"txt": @"text/plain",
        @"mp4": @"video/mp4",
        @"mov": @"video/quicktime",
        @"mp3": @"audio/mpeg",
        @"wav": @"audio/wav"
    };
    
    return mimeTypes[extension.lowercaseString] ?: @"application/octet-stream";
}

@end
```

## 网络安全实现

### SSL/TLS证书验证

```objc
@interface NetworkSecurityManager : NSObject <NSURLSessionDelegate>

@property (nonatomic, strong) NSSet *pinnedCertificates;
@property (nonatomic, strong) NSSet *pinnedPublicKeys;
@property (nonatomic, assign) BOOL allowInvalidCertificates;
@property (nonatomic, assign) BOOL validatesDomainName;

+ (instancetype)sharedManager;

// 证书固定
- (void)pinCertificatesFromBundle;
- (void)pinCertificate:(NSData *)certificateData;
- (void)pinPublicKey:(NSData *)publicKeyData;

// 安全验证
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain;

- (NSData *)publicKeyDataFromCertificate:(SecCertificateRef)certificate;
- (NSData *)publicKeyDataFromTrust:(SecTrustRef)trust;

// 请求加密
- (NSURLRequest *)encryptedRequestFromRequest:(NSURLRequest *)request
                                   withAPIKey:(NSString *)apiKey
                                       secret:(NSString *)secret;

@end

@implementation NetworkSecurityManager

+ (instancetype)sharedManager {
    static NetworkSecurityManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.validatesDomainName = YES;
        self.allowInvalidCertificates = NO;
        [self pinCertificatesFromBundle];
    }
    return self;
}

- (void)pinCertificatesFromBundle {
    NSMutableSet *certificates = [NSMutableSet set];
    NSMutableSet *publicKeys = [NSMutableSet set];
    
    NSBundle *bundle = [NSBundle mainBundle];
    NSArray *certPaths = [bundle pathsForResourcesOfType:@"cer" inDirectory:nil];
    
    for (NSString *certPath in certPaths) {
        NSData *certData = [NSData dataWithContentsOfFile:certPath];
        if (certData) {
            [certificates addObject:certData];
            
            // 提取公钥
            SecCertificateRef certificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certData);
            if (certificate) {
                NSData *publicKeyData = [self publicKeyDataFromCertificate:certificate];
                if (publicKeyData) {
                    [publicKeys addObject:publicKeyData];
                }
                CFRelease(certificate);
            }
        }
    }
    
    self.pinnedCertificates = [certificates copy];
    self.pinnedPublicKeys = [publicKeys copy];
}

- (void)pinCertificate:(NSData *)certificateData {
    NSMutableSet *certificates = [self.pinnedCertificates mutableCopy] ?: [NSMutableSet set];
    [certificates addObject:certificateData];
    self.pinnedCertificates = [certificates copy];
}

- (void)pinPublicKey:(NSData *)publicKeyData {
    NSMutableSet *publicKeys = [self.pinnedPublicKeys mutableCopy] ?: [NSMutableSet set];
    [publicKeys addObject:publicKeyData];
    self.pinnedPublicKeys = [publicKeys copy];
}

- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain {
    
    // 如果允许无效证书，直接返回YES
    if (self.allowInvalidCertificates) {
        return YES;
    }
    
    // 设置验证策略
    SecPolicyRef policy = SecPolicyCreateSSL(true, (__bridge CFStringRef)domain);
    SecTrustSetPolicies(serverTrust, policy);
    
    // 验证证书链
    SecTrustResultType result;
    OSStatus status = SecTrustEvaluate(serverTrust, &result);
    
    if (status != errSecSuccess) {
        CFRelease(policy);
        return NO;
    }
    
    // 检查验证结果
    BOOL isValid = (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);
    
    // 如果基本验证失败，检查是否有固定的证书或公钥
    if (!isValid && (self.pinnedCertificates.count > 0 || self.pinnedPublicKeys.count > 0)) {
        isValid = [self validatePinnedCertificatesForTrust:serverTrust];
    }
    
    CFRelease(policy);
    return isValid;
}

- (BOOL)validatePinnedCertificatesForTrust:(SecTrustRef)serverTrust {
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    
    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        NSData *certificateData = (__bridge NSData *)SecCertificateCopyData(certificate);
        
        // 检查证书固定
        if ([self.pinnedCertificates containsObject:certificateData]) {
            return YES;
        }
        
        // 检查公钥固定
        NSData *publicKeyData = [self publicKeyDataFromCertificate:certificate];
        if (publicKeyData && [self.pinnedPublicKeys containsObject:publicKeyData]) {
            return YES;
        }
    }
    
    return NO;
}

- (NSData *)publicKeyDataFromCertificate:(SecCertificateRef)certificate {
    SecPolicyRef policy = SecPolicyCreateBasicX509();
    SecTrustRef trust;
    OSStatus status = SecTrustCreateWithCertificates(certificate, policy, &trust);
    
    if (status != errSecSuccess) {
        CFRelease(policy);
        return nil;
    }
    
    NSData *publicKeyData = [self publicKeyDataFromTrust:trust];
    
    CFRelease(trust);
    CFRelease(policy);
    
    return publicKeyData;
}

- (NSData *)publicKeyDataFromTrust:(SecTrustRef)trust {
    SecKeyRef publicKey = SecTrustCopyPublicKey(trust);
    if (!publicKey) {
        return nil;
    }
    
    NSData *publicKeyData = (__bridge NSData *)SecKeyCopyExternalRepresentation(publicKey, NULL);
    CFRelease(publicKey);
    
    return publicKeyData;
}

- (NSURLRequest *)encryptedRequestFromRequest:(NSURLRequest *)request
                                   withAPIKey:(NSString *)apiKey
                                       secret:(NSString *)secret {
    
    NSMutableURLRequest *mutableRequest = [request mutableCopy];
    
    // 添加时间戳
    NSString *timestamp = [NSString stringWithFormat:@"%.0f", [[NSDate date] timeIntervalSince1970]];
    [mutableRequest setValue:timestamp forHTTPHeaderField:@"X-Timestamp"];
    
    // 添加随机数
    NSString *nonce = [[NSUUID UUID] UUIDString];
    [mutableRequest setValue:nonce forHTTPHeaderField:@"X-Nonce"];
    
    // 添加API密钥
    [mutableRequest setValue:apiKey forHTTPHeaderField:@"X-API-Key"];
    
    // 生成签名
    NSString *signature = [self generateSignatureForRequest:mutableRequest
                                                  withSecret:secret
                                                   timestamp:timestamp
                                                       nonce:nonce];
    [mutableRequest setValue:signature forHTTPHeaderField:@"X-Signature"];
    
    return [mutableRequest copy];
}

- (NSString *)generateSignatureForRequest:(NSURLRequest *)request
                               withSecret:(NSString *)secret
                                timestamp:(NSString *)timestamp
                                    nonce:(NSString *)nonce {
    
    // 构建签名字符串
    NSString *method = request.HTTPMethod ?: @"GET";
    NSString *url = request.URL.absoluteString;
    NSString *body = @"";
    
    if (request.HTTPBody) {
        body = [[NSString alloc] initWithData:request.HTTPBody encoding:NSUTF8StringEncoding] ?: @"";
    }
    
    NSString *signatureString = [NSString stringWithFormat:@"%@\n%@\n%@\n%@\n%@",
                                method, url, body, timestamp, nonce];
    
    // 使用HMAC-SHA256生成签名
    return [self hmacSHA256:signatureString withKey:secret];
}

- (NSString *)hmacSHA256:(NSString *)data withKey:(NSString *)key {
    const char *cKey = [key cStringUsingEncoding:NSUTF8StringEncoding];
    const char *cData = [data cStringUsingEncoding:NSUTF8StringEncoding];
    
    unsigned char cHMAC[CC_SHA256_DIGEST_LENGTH];
    CCHmac(kCCHmacAlgSHA256, cKey, strlen(cKey), cData, strlen(cData), cHMAC);
    
    NSMutableString *result = [NSMutableString stringWithCapacity:CC_SHA256_DIGEST_LENGTH * 2];
    for (int i = 0; i < CC_SHA256_DIGEST_LENGTH; i++) {
        [result appendFormat:@"%02x", cHMAC[i]];
    }
    
    return result;
}

#pragma mark - NSURLSessionDelegate

- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    NSString *authMethod = challenge.protectionSpace.authenticationMethod;
    
    if ([authMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
        NSString *host = challenge.protectionSpace.host;
        
        if ([self evaluateServerTrust:serverTrust forDomain:host]) {
            NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        } else {
            completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
        }
    } else {
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
    }
}

@end
```

## 网络性能优化

### 性能监控和优化

```objc
@interface NetworkPerformanceOptimizer : NSObject

@property (nonatomic, strong) NSMutableDictionary *requestMetrics;
@property (nonatomic, strong) NSMutableArray *performanceHistory;
@property (nonatomic, assign) NSUInteger maxHistoryCount;

+ (instancetype)sharedOptimizer;

// 性能监控
- (void)startMonitoringRequest:(NSURLRequest *)request withIdentifier:(NSString *)identifier;
- (void)finishMonitoringRequest:(NSString *)identifier
                   withResponse:(NSURLResponse *)response
                          error:(NSError *)error
                        metrics:(NSURLSessionTaskMetrics *)metrics;

// 性能分析
- (NSDictionary *)getPerformanceReport;
- (NSArray *)getSlowRequests;
- (NSDictionary *)getNetworkQualityMetrics;

// 优化建议
- (NSArray *)getOptimizationSuggestions;
- (NSURLRequest *)optimizeRequest:(NSURLRequest *)request;

// 连接池管理
- (void)optimizeConnectionPoolForHost:(NSString *)host;
- (void)preconnectToHost:(NSString *)host;

// 请求合并
- (void)batchRequests:(NSArray *)requests
    completionHandler:(void(^)(NSArray *responses, NSArray *errors))completionHandler;

@end

@implementation NetworkPerformanceOptimizer

+ (instancetype)sharedOptimizer {
    static NetworkPerformanceOptimizer *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.requestMetrics = [NSMutableDictionary dictionary];
        self.performanceHistory = [NSMutableArray array];
        self.maxHistoryCount = 1000;
    }
    return self;
}

- (void)startMonitoringRequest:(NSURLRequest *)request withIdentifier:(NSString *)identifier {
    NSDictionary *requestInfo = @{
        @"identifier": identifier,
        @"url": request.URL.absoluteString,
        @"method": request.HTTPMethod ?: @"GET",
        @"start_time": @([[NSDate date] timeIntervalSince1970]),
        @"request_size": @(request.HTTPBody.length)
    };
    
    self.requestMetrics[identifier] = [requestInfo mutableCopy];
}

- (void)finishMonitoringRequest:(NSString *)identifier
                   withResponse:(NSURLResponse *)response
                          error:(NSError *)error
                        metrics:(NSURLSessionTaskMetrics *)metrics {
    
    NSMutableDictionary *requestInfo = self.requestMetrics[identifier];
    if (!requestInfo) {
        return;
    }
    
    NSTimeInterval endTime = [[NSDate date] timeIntervalSince1970];
    NSTimeInterval startTime = [requestInfo[@"start_time"] doubleValue];
    NSTimeInterval totalTime = endTime - startTime;
    
    requestInfo[@"end_time"] = @(endTime);
    requestInfo[@"total_time"] = @(totalTime);
    requestInfo[@"success"] = @(error == nil);
    
    if (response) {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        requestInfo[@"status_code"] = @(httpResponse.statusCode);
        requestInfo[@"response_size"] = @(httpResponse.expectedContentLength);
    }
    
    if (error) {
        requestInfo[@"error_code"] = @(error.code);
        requestInfo[@"error_domain"] = error.domain;
    }
    
    // 解析详细指标
    if (metrics) {
        [self parseTaskMetrics:metrics forRequest:requestInfo];
    }
    
    // 添加到历史记录
    [self.performanceHistory addObject:[requestInfo copy]];
    
    // 限制历史记录数量
    if (self.performanceHistory.count > self.maxHistoryCount) {
        [self.performanceHistory removeObjectAtIndex:0];
    }
    
    // 移除活动监控
    [self.requestMetrics removeObjectForKey:identifier];
}

- (void)parseTaskMetrics:(NSURLSessionTaskMetrics *)metrics forRequest:(NSMutableDictionary *)requestInfo {
    NSURLSessionTaskTransactionMetrics *transactionMetrics = metrics.transactionMetrics.firstObject;
    if (!transactionMetrics) {
        return;
    }
    
    // DNS解析时间
    if (transactionMetrics.domainLookupStartDate && transactionMetrics.domainLookupEndDate) {
        NSTimeInterval dnsTime = [transactionMetrics.domainLookupEndDate timeIntervalSinceDate:transactionMetrics.domainLookupStartDate];
        requestInfo[@"dns_time"] = @(dnsTime);
    }
    
    // 连接建立时间
    if (transactionMetrics.connectStartDate && transactionMetrics.connectEndDate) {
        NSTimeInterval connectTime = [transactionMetrics.connectEndDate timeIntervalSinceDate:transactionMetrics.connectStartDate];
        requestInfo[@"connect_time"] = @(connectTime);
    }
    
    // SSL握手时间
    if (transactionMetrics.secureConnectionStartDate && transactionMetrics.secureConnectionEndDate) {
        NSTimeInterval sslTime = [transactionMetrics.secureConnectionEndDate timeIntervalSinceDate:transactionMetrics.secureConnectionStartDate];
        requestInfo[@"ssl_time"] = @(sslTime);
    }
    
    // 请求发送时间
    if (transactionMetrics.requestStartDate && transactionMetrics.requestEndDate) {
        NSTimeInterval requestTime = [transactionMetrics.requestEndDate timeIntervalSinceDate:transactionMetrics.requestStartDate];
        requestInfo[@"request_time"] = @(requestTime);
    }
    
    // 响应接收时间
    if (transactionMetrics.responseStartDate && transactionMetrics.responseEndDate) {
        NSTimeInterval responseTime = [transactionMetrics.responseEndDate timeIntervalSinceDate:transactionMetrics.responseStartDate];
        requestInfo[@"response_time"] = @(responseTime);
    }
    
    // 网络协议信息
    requestInfo[@"network_protocol"] = transactionMetrics.networkProtocolName ?: @"unknown";
    requestInfo[@"proxy_connection"] = @(transactionMetrics.isProxyConnection);
    requestInfo[@"reused_connection"] = @(transactionMetrics.isReusedConnection);
}

- (NSDictionary *)getPerformanceReport {
    if (self.performanceHistory.count == 0) {
        return @{};
    }
    
    NSUInteger totalRequests = self.performanceHistory.count;
    NSUInteger successfulRequests = 0;
    NSTimeInterval totalTime = 0;
    NSTimeInterval totalDNSTime = 0;
    NSTimeInterval totalConnectTime = 0;
    NSTimeInterval totalSSLTime = 0;
    NSUInteger reusedConnections = 0;
    
    for (NSDictionary *request in self.performanceHistory) {
        if ([request[@"success"] boolValue]) {
            successfulRequests++;
        }
        
        totalTime += [request[@"total_time"] doubleValue];
        totalDNSTime += [request[@"dns_time"] doubleValue];
        totalConnectTime += [request[@"connect_time"] doubleValue];
        totalSSLTime += [request[@"ssl_time"] doubleValue];
        
        if ([request[@"reused_connection"] boolValue]) {
            reusedConnections++;
        }
    }
    
    return @{
        @"total_requests": @(totalRequests),
        @"successful_requests": @(successfulRequests),
        @"success_rate": @((double)successfulRequests / totalRequests),
        @"average_response_time": @(totalTime / totalRequests),
        @"average_dns_time": @(totalDNSTime / totalRequests),
        @"average_connect_time": @(totalConnectTime / totalRequests),
        @"average_ssl_time": @(totalSSLTime / totalRequests),
        @"connection_reuse_rate": @((double)reusedConnections / totalRequests)
    };
}

- (NSArray *)getSlowRequests {
    // 计算平均响应时间
    NSTimeInterval totalTime = 0;
    for (NSDictionary *request in self.performanceHistory) {
        totalTime += [request[@"total_time"] doubleValue];
    }
    NSTimeInterval averageTime = totalTime / self.performanceHistory.count;
    
    // 找出响应时间超过平均值2倍的请求
    NSMutableArray *slowRequests = [NSMutableArray array];
    for (NSDictionary *request in self.performanceHistory) {
        NSTimeInterval requestTime = [request[@"total_time"] doubleValue];
        if (requestTime > averageTime * 2) {
            [slowRequests addObject:request];
        }
    }
    
    // 按响应时间排序
    [slowRequests sortUsingComparator:^NSComparisonResult(NSDictionary *obj1, NSDictionary *obj2) {
        NSTimeInterval time1 = [obj1[@"total_time"] doubleValue];
        NSTimeInterval time2 = [obj2[@"total_time"] doubleValue];
        return time1 > time2 ? NSOrderedAscending : NSOrderedDescending;
    }];
    
    return [slowRequests copy];
}

- (NSDictionary *)getNetworkQualityMetrics {
    // 基于最近的请求分析网络质量
    NSArray *recentRequests = [self.performanceHistory subarrayWithRange:NSMakeRange(MAX(0, (NSInteger)self.performanceHistory.count - 50), MIN(50, self.performanceHistory.count))];
    
    NSTimeInterval totalTime = 0;
    NSUInteger successCount = 0;
    NSUInteger timeoutCount = 0;
    
    for (NSDictionary *request in recentRequests) {
        totalTime += [request[@"total_time"] doubleValue];
        
        if ([request[@"success"] boolValue]) {
            successCount++;
        }
        
        if ([request[@"error_domain"] isEqualToString:NSURLErrorDomain] &&
            [request[@"error_code"] integerValue] == NSURLErrorTimedOut) {
            timeoutCount++;
        }
    }
    
    NSTimeInterval averageTime = totalTime / recentRequests.count;
    double successRate = (double)successCount / recentRequests.count;
    double timeoutRate = (double)timeoutCount / recentRequests.count;
    
    // 评估网络质量
    NSString *quality = @"Good";
    if (averageTime > 3.0 || successRate < 0.9 || timeoutRate > 0.1) {
        quality = @"Poor";
    } else if (averageTime > 1.5 || successRate < 0.95 || timeoutRate > 0.05) {
        quality = @"Fair";
    }
    
    return @{
        @"quality": quality,
        @"average_response_time": @(averageTime),
        @"success_rate": @(successRate),
        @"timeout_rate": @(timeoutRate)
    };
}

- (NSArray *)getOptimizationSuggestions {
    NSMutableArray *suggestions = [NSMutableArray array];
    NSDictionary *report = [self getPerformanceReport];
    
    // 检查成功率
    double successRate = [report[@"success_rate"] doubleValue];
    if (successRate < 0.95) {
        [suggestions addObject:@"Success rate is low. Consider implementing retry logic and better error handling."];
    }
    
    // 检查平均响应时间
    NSTimeInterval averageTime = [report[@"average_response_time"] doubleValue];
    if (averageTime > 2.0) {
        [suggestions addObject:@"Average response time is high. Consider optimizing server performance or using CDN."];
    }
    
    // 检查DNS时间
    NSTimeInterval averageDNSTime = [report[@"average_dns_time"] doubleValue];
    if (averageDNSTime > 0.5) {
        [suggestions addObject:@"DNS lookup time is high. Consider using DNS prefetching or caching."];
    }
    
    // 检查连接复用率
    double reuseRate = [report[@"connection_reuse_rate"] doubleValue];
    if (reuseRate < 0.7) {
        [suggestions addObject:@"Connection reuse rate is low. Consider implementing HTTP/2 or connection pooling."];
    }
    
    // 检查慢请求
    NSArray *slowRequests = [self getSlowRequests];
    if (slowRequests.count > self.performanceHistory.count * 0.1) {
        [suggestions addObject:@"Too many slow requests detected. Consider implementing request optimization and caching."];
    }
    
    return [suggestions copy];
}

- (NSURLRequest *)optimizeRequest:(NSURLRequest *)request {
    NSMutableURLRequest *optimizedRequest = [request mutableCopy];
    
    // 设置合理的超时时间
    optimizedRequest.timeoutInterval = 30.0;
    
    // 启用压缩
    [optimizedRequest setValue:@"gzip, deflate" forHTTPHeaderField:@"Accept-Encoding"];
    
    // 设置缓存策略
    optimizedRequest.cachePolicy = NSURLRequestReturnCacheDataElseLoad;
    
    // 添加Keep-Alive头
    [optimizedRequest setValue:@"keep-alive" forHTTPHeaderField:@"Connection"];
    
    return [optimizedRequest copy];
}

- (void)optimizeConnectionPoolForHost:(NSString *)host {
    // 这里可以实现针对特定主机的连接池优化
    NSLog(@"Optimizing connection pool for host: %@", host);
}

- (void)preconnectToHost:(NSString *)host {
    // 预连接到主机以减少后续请求的延迟
    NSURL *URL = [NSURL URLWithString:[NSString stringWithFormat:@"https://%@", host]];
    NSURLRequest *request = [NSURLRequest requestWithURL:URL];
    
    NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:request
                                                                 completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        // 预连接完成，不需要处理响应
        NSLog(@"Preconnection to %@ completed", host);
    }];
    
    [task resume];
}

- (void)batchRequests:(NSArray *)requests
    completionHandler:(void(^)(NSArray *responses, NSArray *errors))completionHandler {
    
    NSMutableArray *responses = [NSMutableArray arrayWithCapacity:requests.count];
    NSMutableArray *errors = [NSMutableArray arrayWithCapacity:requests.count];
    
    dispatch_group_t group = dispatch_group_create();
    
    for (NSUInteger i = 0; i < requests.count; i++) {
        NSURLRequest *request = requests[i];
        
        dispatch_group_enter(group);
        
        NSURLSessionDataTask *task = [[NSURLSession sharedSession] dataTaskWithRequest:request
                                                                     completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            @synchronized(responses) {
                responses[i] = response ?: [NSNull null];
                errors[i] = error ?: [NSNull null];
            }
            
            dispatch_group_leave(group);
        }];
        
        [task resume];
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        if (completionHandler) {
            completionHandler([responses copy], [errors copy]);
        }
    });
}

@end
```

## 网络编程最佳实践

### 综合网络管理器

```objc
@interface ComprehensiveNetworkManager : NSObject

@property (nonatomic, strong) HTTPRequestManager *requestManager;
@property (nonatomic, strong) WebSocketManager *webSocketManager;
@property (nonatomic, strong) NetworkCacheManager *cacheManager;
@property (nonatomic, strong) NetworkSecurityManager *securityManager;
@property (nonatomic, strong) NetworkPerformanceOptimizer *performanceOptimizer;

+ (instancetype)sharedManager;

// 统一接口
- (void)setupWithConfiguration:(NSDictionary *)configuration;
- (void)performRequest:(NSURLRequest *)request
    completionHandler:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;

// 监控和报告
- (NSDictionary *)getComprehensiveReport;
- (void)enableDebugMode:(BOOL)enabled;

@end

@implementation ComprehensiveNetworkManager

+ (instancetype)sharedManager {
    static ComprehensiveNetworkManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.requestManager = [HTTPRequestManager sharedManager];
        self.webSocketManager = [WebSocketManager sharedManager];
        self.cacheManager = [NetworkCacheManager sharedManager];
        self.securityManager = [NetworkSecurityManager sharedManager];
        self.performanceOptimizer = [NetworkPerformanceOptimizer sharedOptimizer];
        
        [self setupDefaultConfiguration];
    }
    return self;
}

- (void)setupDefaultConfiguration {
    // 设置默认的请求拦截器
    [self.requestManager addRequestInterceptor:@"security" block:^NSURLRequest *(NSURLRequest *request) {
        // 应用安全配置
        return [self.securityManager encryptedRequestFromRequest:request
                                                      withAPIKey:@"your-api-key"
                                                          secret:@"your-secret"];
    }];
    
    // 设置默认的响应拦截器
    [self.requestManager addResponseInterceptor:@"cache" block:^id(NSURLResponse *response, id responseObject, NSError *error) {
        // 缓存成功的响应
        if (!error && responseObject) {
            NSData *data = nil;
            if ([responseObject isKindOfClass:[NSData class]]) {
                data = responseObject;
            } else if ([responseObject isKindOfClass:[NSString class]]) {
                data = [responseObject dataUsingEncoding:NSUTF8StringEncoding];
            } else {
                NSError *jsonError;
                data = [NSJSONSerialization dataWithJSONObject:responseObject options:0 error:&jsonError];
            }
            
            if (data) {
                NSURLRequest *request = nil; // 需要从上下文获取
                [self.cacheManager cacheResponse:response data:data forRequest:request];
            }
        }
        
        return responseObject;
    }];
}

- (void)setupWithConfiguration:(NSDictionary *)configuration {
    // 缓存配置
    NSDictionary *cacheConfig = configuration[@"cache"];
    if (cacheConfig) {
        NSUInteger memoryCapacity = [cacheConfig[@"memory_capacity"] unsignedIntegerValue] ?: 50 * 1024 * 1024;
        NSUInteger diskCapacity = [cacheConfig[@"disk_capacity"] unsignedIntegerValue] ?: 200 * 1024 * 1024;
        NSString *diskPath = cacheConfig[@"disk_path"];
        
        [self.cacheManager setupCacheWithMemoryCapacity:memoryCapacity
                                            diskCapacity:diskCapacity
                                                diskPath:diskPath];
    }
    
    // 安全配置
    NSDictionary *securityConfig = configuration[@"security"];
    if (securityConfig) {
        self.securityManager.allowInvalidCertificates = [securityConfig[@"allow_invalid_certificates"] boolValue];
        self.securityManager.validatesDomainName = [securityConfig[@"validate_domain_name"] boolValue];
        
        NSArray *pinnedCertificates = securityConfig[@"pinned_certificates"];
        for (NSString *certPath in pinnedCertificates) {
            NSData *certData = [NSData dataWithContentsOfFile:certPath];
            if (certData) {
                [self.securityManager pinCertificate:certData];
            }
        }
    }
}

- (void)performRequest:(NSURLRequest *)request
    completionHandler:(void(^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler {
    
    NSString *identifier = [[NSUUID UUID] UUIDString];
    
    // 开始性能监控
    [self.performanceOptimizer startMonitoringRequest:request withIdentifier:identifier];
    
    // 检查缓存
    NSData *cachedData = [self.cacheManager cachedDataForRequest:request];
    if (cachedData) {
        NSLog(@"Returning cached response for: %@", request.URL.absoluteString);
        if (completionHandler) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionHandler(cachedData, nil, nil);
            });
        }
        return;
    }
    
    // 优化请求
    NSURLRequest *optimizedRequest = [self.performanceOptimizer optimizeRequest:request];
    
    // 执行请求
    [self.requestManager performDataTaskWithRequest:optimizedRequest
                                             success:^(NSURLSessionDataTask *task, id responseObject) {
        
        // 结束性能监控
        [self.performanceOptimizer finishMonitoringRequest:identifier
                                               withResponse:task.response
                                                      error:nil
                                                    metrics:task.taskMetrics];
        
        NSData *data = nil;
        if ([responseObject isKindOfClass:[NSData class]]) {
            data = responseObject;
        } else if ([responseObject isKindOfClass:[NSString class]]) {
            data = [responseObject dataUsingEncoding:NSUTF8StringEncoding];
        } else {
            NSError *jsonError;
            data = [NSJSONSerialization dataWithJSONObject:responseObject options:0 error:&jsonError];
        }
        
        if (completionHandler) {
            completionHandler(data, task.response, nil);
        }
        
    } failure:^(NSURLSessionDataTask *task, NSError *error) {
        
        // 结束性能监控
        [self.performanceOptimizer finishMonitoringRequest:identifier
                                               withResponse:task.response
                                                      error:error
                                                    metrics:task.taskMetrics];
        
        if (completionHandler) {
            completionHandler(nil, task.response, error);
        }
    }];
}

- (NSDictionary *)getComprehensiveReport {
    return @{
        @"performance": [self.performanceOptimizer getPerformanceReport],
        @"cache": [self.cacheManager getCacheStatistics],
        @"network_quality": [self.performanceOptimizer getNetworkQualityMetrics],
        @"optimization_suggestions": [self.performanceOptimizer getOptimizationSuggestions]
    };
}

- (void)enableDebugMode:(BOOL)enabled {
    if (enabled) {
        NSLog(@"Network debug mode enabled");
        // 启用详细日志记录
    } else {
        NSLog(@"Network debug mode disabled");
    }
}

@end
```

## 总结

本文深入探讨了iOS网络编程的各个方面，从基础的URLSession使用到高级的安全和性能优化。主要内容包括：

### 核心技术要点

1. **网络架构设计**：构建可扩展的网络层架构，支持多种会话配置和请求管理
2. **URLSession深度应用**：实现完整的HTTP请求管理器，支持各种HTTP方法和文件传输
3. **WebSocket通信**：实现可靠的WebSocket连接管理，包括心跳、重连和消息队列
4. **智能缓存系统**：多层缓存策略，提高应用性能和用户体验
5. **网络安全实现**：SSL/TLS证书验证、证书固定和请求签名
6. **性能监控优化**：全面的性能指标收集和优化建议

### 最佳实践建议

1. **统一网络层**：使用综合网络管理器统一处理所有网络请求
2. **安全第一**：始终验证服务器证书，使用HTTPS和请求签名
3. **性能优化**：实施缓存策略、连接复用和请求优化
4. **错误处理**：完善的错误处理和重试机制
5. **监控分析**：持续监控网络性能，及时发现和解决问题

通过系统性地实施这些网络编程技术和最佳实践，可以构建出高效、安全、可靠的iOS应用网络层，为用户提供优质的网络体验。

## WebSocket通信实现

### WebSocket管理器

```objc
@interface WebSocketManager : NSObject <NSURLSessionWebSocketDelegate>

@property (nonatomic, strong) NSURLSessionWebSocketTask *webSocketTask;
@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, assign) BOOL isConnected;
@property (nonatomic, strong) NSMutableArray *messageQueue;
@property (nonatomic, strong) NSTimer *heartbeatTimer;
@property (nonatomic, strong) NSTimer *reconnectTimer;
@property (nonatomic, assign) NSInteger reconnectAttempts;
@property (nonatomic, assign) NSTimeInterval reconnectInterval;

+ (instancetype)sharedManager;

// 连接管理
- (void)connectToURL:(NSURL *)URL;
- (void)disconnect;
- (void)reconnect;

// 消息发送
- (void)sendMessage:(NSString *)message;
- (void)sendData:(NSData *)data;
- (void)sendPing;

// 事件回调
@property (nonatomic, copy) void(^onConnected)(void);
@property (nonatomic, copy) void(^onDisconnected)(NSError *error);
@property (nonatomic, copy) void(^onMessageReceived)(NSString *message);
@property (nonatomic, copy) void(^onDataReceived)(NSData *data);
@property (nonatomic, copy) void(^onError)(NSError *error);

// 心跳和重连
- (void)startHeartbeat;
- (void)stopHeartbeat;
- (void)startReconnectTimer;
- (void)stopReconnectTimer;

@end

@implementation WebSocketManager

+ (instancetype)sharedManager {
    static WebSocketManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
        self.messageQueue = [NSMutableArray array];
        self.reconnectInterval = 5.0;
        self.reconnectAttempts = 0;
    }
    return self;
}

- (void)connectToURL:(NSURL *)URL {
    if (self.isConnected) {
        [self disconnect];
    }
    
    NSURLRequest *request = [NSURLRequest requestWithURL:URL];
    self.webSocketTask = [self.session webSocketTaskWithRequest:request];
    [self.webSocketTask resume];
    
    [self receiveMessage];
}

- (void)disconnect {
    [self stopHeartbeat];
    [self stopReconnectTimer];
    
    if (self.webSocketTask) {
        [self.webSocketTask cancelWithCloseCode:NSURLSessionWebSocketCloseCodeNormalClosure reason:nil];
        self.webSocketTask = nil;
    }
    
    self.isConnected = NO;
}

- (void)reconnect {
    if (self.reconnectAttempts < 5) {
        self.reconnectAttempts++;
        NSLog(@"Attempting to reconnect... (attempt %ld)", (long)self.reconnectAttempts);
        
        // 指数退避
        NSTimeInterval delay = self.reconnectInterval * pow(2, self.reconnectAttempts - 1);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            if (self.webSocketTask.state != NSURLSessionTaskStateRunning) {
                [self.webSocketTask resume];
            }
        });
    } else {
        NSLog(@"Max reconnection attempts reached");
        if (self.onError) {
            NSError *error = [NSError errorWithDomain:@"WebSocketError"
                                               code:1001
                                           userInfo:@{NSLocalizedDescriptionKey: @"Max reconnection attempts reached"}];
            self.onError(error);
        }
    }
}

- (void)sendMessage:(NSString *)message {
    if (!self.isConnected) {
        [self.messageQueue addObject:message];
        return;
    }
    
    NSURLSessionWebSocketMessage *webSocketMessage = [[NSURLSessionWebSocketMessage alloc] initWithString:message];
    [self.webSocketTask sendMessage:webSocketMessage completionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to send message: %@", error);
            if (self.onError) {
                self.onError(error);
            }
        }
    }];
}

- (void)sendData:(NSData *)data {
    if (!self.isConnected) {
        return;
    }
    
    NSURLSessionWebSocketMessage *webSocketMessage = [[NSURLSessionWebSocketMessage alloc] initWithData:data];
    [self.webSocketTask sendMessage:webSocketMessage completionHandler:^(NSError *error) {
        if (error) {
            NSLog(@"Failed to send data: %@", error);
            if (self.onError) {
                self.onError(error);
            }
        }
    }];
}

- (void)sendPing {
    if (self.isConnected) {
        [self.webSocketTask sendPingWithPongReceiveHandler:^(NSError *error) {
            if (error) {
                NSLog(@"Ping failed: %@", error);
            } else {
                NSLog(@"Ping successful");
            }
        }];
    }
}

- (void)receiveMessage {
    [self.webSocketTask receiveMessageWithCompletionHandler:^(NSURLSessionWebSocketMessage *message, NSError *error) {
        if (error) {
            NSLog(@"Failed to receive message: %@", error);
            if (self.onError) {
                self.onError(error);
            }
            return;
        }
        
        if (message.type == NSURLSessionWebSocketMessageTypeString) {
            if (self.onMessageReceived) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.onMessageReceived(message.string);
                });
            }
        } else if (message.type == NSURLSessionWebSocketMessageTypeData) {
            if (self.onDataReceived) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.onDataReceived(message.data);
                });
            }
        }
        
        // 继续接收下一条消息
        [self receiveMessage];
    }];
}

- (void)startHeartbeat {
    [self stopHeartbeat];
    
    self.heartbeatTimer = [NSTimer scheduledTimerWithTimeInterval:30.0
                                                           target:self
                                                         selector:@selector(sendPing)
                                                         userInfo:nil
                                                          repeats:YES];
}

- (void)stopHeartbeat {
    if (self.heartbeatTimer) {
        [self.heartbeatTimer invalidate];
        self.heartbeatTimer = nil;
    }
}

- (void)startReconnectTimer {
    [self stopReconnectTimer];
    
    self.reconnectTimer = [NSTimer scheduledTimerWithTimeInterval:self.reconnectInterval
                                                           target:self
                                                         selector:@selector(reconnect)
                                                         userInfo:nil
                                                          repeats:NO];
}

- (void)stopReconnectTimer {
    if (self.reconnectTimer) {
        [self.reconnectTimer invalidate];
        self.reconnectTimer = nil;
    }
}

#pragma mark - NSURLSessionWebSocketDelegate

- (void)URLSession:(NSURLSession *)session webSocketTask:(NSURLSessionWebSocketTask *)webSocketTask didOpenWithProtocol:(NSString *)protocol {
    NSLog(@"WebSocket connected");
    self.isConnected = YES;
    self.reconnectAttempts = 0;
    
    // 发送队列中的消息
    for (NSString *message in self.messageQueue) {
        [self sendMessage:message];
    }
    [self.messageQueue removeAllObjects];
    
    [self startHeartbeat];
    
    if (self.onConnected) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.onConnected();
        });
    }
}

- (void)URLSession:(NSURLSession *)session webSocketTask:(NSURLSessionWebSocketTask *)webSocketTask didCloseWithCode:(NSURLSessionWebSocketCloseCode)closeCode reason:(NSData *)reason {
    NSLog(@"WebSocket disconnected with code: %ld", (long)closeCode);
    self.isConnected = NO;
    
    [self stopHeartbeat];
    
    NSError *error = nil;
    if (closeCode != NSURLSessionWebSocketCloseCodeNormalClosure) {
        error = [NSError errorWithDomain:@"WebSocketError"
                                    code:closeCode
                                userInfo:@{NSLocalizedDescriptionKey: @"WebSocket closed unexpectedly"}];
        [self startReconnectTimer];
    }
    
    if (self.onDisconnected) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.onDisconnected(error);
        });
    }
}

@end
```

## 网络缓存管理

### 智能缓存系统

```objc
@interface NetworkCacheManager : NSObject

@property (nonatomic, strong) NSURLCache *URLCache;
@property (nonatomic, strong) NSCache *memoryCache;
@property (nonatomic, strong) NSString *diskCachePath;
@property (nonatomic, assign) NSUInteger maxMemoryCacheSize;
@property (nonatomic, assign) NSUInteger maxDiskCacheSize;
@property (nonatomic, assign) NSTimeInterval defaultCacheAge;

+ (instancetype)sharedManager;

// 缓存配置
- (void)setupCacheWithMemoryCapacity:(NSUInteger)memoryCapacity
                        diskCapacity:(NSUInteger)diskCapacity
                            diskPath:(NSString *)path;

// 缓存操作
- (void)cacheResponse:(NSURLResponse *)response
                 data:(NSData *)data
           forRequest:(NSURLRequest *)request;

- (NSData *)cachedDataForRequest:(NSURLRequest *)request;
- (BOOL)hasCachedDataForRequest:(NSURLRequest *)request;
- (void)removeCachedDataForRequest:(NSURLRequest *)request;
- (void)clearAllCache;

// 缓存策略
- (NSURLRequest *)requestWithCachePolicy:(NSURLRequestCachePolicy)cachePolicy
                              forRequest:(NSURLRequest *)request;

- (BOOL)shouldCacheResponse:(NSURLResponse *)response
                       data:(NSData *)data
                 forRequest:(NSURLRequest *)request;

// 缓存统计
- (NSDictionary *)getCacheStatistics;
- (void)cleanExpiredCache;
- (NSUInteger)getCurrentCacheSize;

@end

@implementation NetworkCacheManager

+ (instancetype)sharedManager {
    static NetworkCacheManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.maxMemoryCacheSize = 50 * 1024 * 1024; // 50MB
        self.maxDiskCacheSize = 200 * 1024 * 1024;  // 200MB
        self.defaultCacheAge = 7 * 24 * 60 * 60;    // 7天
        
        [self setupDefaultCache];
    }
    return self;
}

- (void)setupDefaultCache {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);
    NSString *cachesDirectory = [paths firstObject];
    self.diskCachePath = [cachesDirectory stringByAppendingPathComponent:@"NetworkCache"];
    
    [self setupCacheWithMemoryCapacity:self.maxMemoryCacheSize
                          diskCapacity:self.maxDiskCacheSize
                              diskPath:self.diskCachePath];
}

- (void)setupCacheWithMemoryCapacity:(NSUInteger)memoryCapacity
                        diskCapacity:(NSUInteger)diskCapacity
                            diskPath:(NSString *)path {
    
    // 设置URLCache
    self.URLCache = [[NSURLCache alloc] initWithMemoryCapacity:memoryCapacity
                                                  diskCapacity:diskCapacity
                                                      diskPath:path];
    [NSURLCache setSharedURLCache:self.URLCache];
    
    // 设置内存缓存
    self.memoryCache = [[NSCache alloc] init];
    self.memoryCache.totalCostLimit = memoryCapacity;
    self.memoryCache.countLimit = 1000;
    
    // 创建磁盘缓存目录
    self.diskCachePath = path;
    [[NSFileManager defaultManager] createDirectoryAtPath:path
                               withIntermediateDirectories:YES
                                                attributes:nil
                                                     error:nil];
}

- (void)cacheResponse:(NSURLResponse *)response
                 data:(NSData *)data
           forRequest:(NSURLRequest *)request {
    
    if (![self shouldCacheResponse:response data:data forRequest:request]) {
        return;
    }
    
    // 缓存到URLCache
    NSCachedURLResponse *cachedResponse = [[NSCachedURLResponse alloc] initWithResponse:response data:data];
    [self.URLCache storeCachedResponse:cachedResponse forRequest:request];
    
    // 缓存到内存
    NSString *cacheKey = [self cacheKeyForRequest:request];
    [self.memoryCache setObject:data forKey:cacheKey cost:data.length];
    
    // 缓存到磁盘
    [self storeToDisk:data forKey:cacheKey];
}

- (NSData *)cachedDataForRequest:(NSURLRequest *)request {
    NSString *cacheKey = [self cacheKeyForRequest:request];
    
    // 首先检查内存缓存
    NSData *memoryData = [self.memoryCache objectForKey:cacheKey];
    if (memoryData) {
        return memoryData;
    }
    
    // 检查URLCache
    NSCachedURLResponse *cachedResponse = [self.URLCache cachedResponseForRequest:request];
    if (cachedResponse) {
        // 将数据加载到内存缓存
        [self.memoryCache setObject:cachedResponse.data forKey:cacheKey cost:cachedResponse.data.length];
        return cachedResponse.data;
    }
    
    // 检查磁盘缓存
    NSData *diskData = [self loadFromDisk:cacheKey];
    if (diskData) {
        // 将数据加载到内存缓存
        [self.memoryCache setObject:diskData forKey:cacheKey cost:diskData.length];
        return diskData;
    }
    
    return nil;
}

- (BOOL)hasCachedDataForRequest:(NSURLRequest *)request {
    return [self cachedDataForRequest:request] != nil;
}

- (void)removeCachedDataForRequest:(NSURLRequest *)request {
    NSString *cacheKey = [self cacheKeyForRequest:request];
    
    // 从内存缓存移除
    [self.memoryCache removeObjectForKey:cacheKey];
    
    // 从URLCache移除
    [self.URLCache removeCachedResponseForRequest:request];
    
    // 从磁盘缓存移除
    [self removeFromDisk:cacheKey];
}

- (void)clearAllCache {
    [self.memoryCache removeAllObjects];
    [self.URLCache removeAllCachedResponses];
    
    NSError *error;
    [[NSFileManager defaultManager] removeItemAtPath:self.diskCachePath error:&error];
    [[NSFileManager defaultManager] createDirectoryAtPath:self.diskCachePath
                               withIntermediateDirectories:YES
                                                attributes:nil
                                                     error:nil];
}

- (NSString *)cacheKeyForRequest:(NSURLRequest *)request {
    NSString *urlString = request.URL.absoluteString;
    NSString *method = request.HTTPMethod ?: @"GET";
    NSString *headers = [request.allHTTPHeaderFields description];
    NSString *body = [[NSString alloc] initWithData:request.HTTPBody encoding:NSUTF8StringEncoding] ?: @"";
    
    NSString *combinedString = [NSString stringWithFormat:@"%@-%@-%@-%@", method, urlString, headers, body];
    return [self MD5:combinedString];
}

- (NSString *)MD5:(NSString *)string {
    const char *cStr = [string UTF8String];
    unsigned char digest[CC_MD5_DIGEST_LENGTH];
    CC_MD5(cStr, (CC_LONG)strlen(cStr), digest);
    
    NSMutableString *result = [NSMutableString stringWithCapacity:CC_MD5_DIGEST_LENGTH * 2];
    for (int i = 0; i < CC_MD5_DIGEST_LENGTH; i++) {
        [result appendFormat:@"%02x", digest[i]];
    }
    
    return result;
}

- (void)storeToDisk:(NSData *)data forKey:(NSString *)key {
    NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:key];
    
    // 创建缓存元数据
    NSDictionary *metadata = @{
        @"timestamp": @([[NSDate date] timeIntervalSince1970]),
        @"size": @(data.length)
    };
    
    NSString *metadataPath = [filePath stringByAppendingString:@".meta"];
    [metadata writeToFile:metadataPath atomically:YES];
    
    // 存储数据
    [data writeToFile:filePath atomically:YES];
}

- (NSData *)loadFromDisk:(NSString *)key {
    NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:key];
    NSString *metadataPath = [filePath stringByAppendingString:@".meta"];
    
    // 检查文件是否存在
    if (![[NSFileManager defaultManager] fileExistsAtPath:filePath]) {
        return nil;
    }
    
    // 检查缓存是否过期
    NSDictionary *metadata = [NSDictionary dictionaryWithContentsOfFile:metadataPath];
    if (metadata) {
        NSTimeInterval timestamp = [metadata[@"timestamp"] doubleValue];
        NSTimeInterval age = [[NSDate date] timeIntervalSince1970] - timestamp;
        
        if (age > self.defaultCacheAge) {
            // 缓存过期，删除文件
            [self removeFromDisk:key];
            return nil;
        }
    }
    
    return [NSData dataWithContentsOfFile:filePath];
}

- (void)removeFromDisk:(NSString *)key {
    NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:key];
    NSString *metadataPath = [filePath stringByAppendingString:@".meta"];
    
    [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];
    [[NSFileManager defaultManager] removeItemAtPath:metadataPath error:nil];
}

- (NSURLRequest *)requestWithCachePolicy:(NSURLRequestCachePolicy)cachePolicy
                              forRequest:(NSURLRequest *)request {
    NSMutableURLRequest *mutableRequest = [request mutableCopy];
    mutableRequest.cachePolicy = cachePolicy;
    return [mutableRequest copy];
}

- (BOOL)shouldCacheResponse:(NSURLResponse *)response
                       data:(NSData *)data
                 forRequest:(NSURLRequest *)request {
    
    // 检查响应类型
    if (![response isKindOfClass:[NSHTTPURLResponse class]]) {
        return NO;
    }
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    
    // 检查状态码
    if (httpResponse.statusCode < 200 || httpResponse.statusCode >= 300) {
        return NO;
    }
    
    // 检查HTTP方法
    NSString *method = request.HTTPMethod ?: @"GET";
    if (![method isEqualToString:@"GET"] && ![method isEqualToString:@"HEAD"]) {
        return NO;
    }
    
    // 检查Cache-Control头
    NSString *cacheControl = httpResponse.allHeaderFields[@"Cache-Control"];
    if ([cacheControl containsString:@"no-cache"] || [cacheControl containsString:@"no-store"]) {
        return NO;
    }
    
    // 检查数据大小
    if (data.length > 10 * 1024 * 1024) { // 10MB
        return NO;
    }
    
    return YES;
}

- (NSDictionary *)getCacheStatistics {
    NSUInteger memoryCacheSize = 0;
    NSUInteger diskCacheSize = 0;
    NSUInteger fileCount = 0;
    
    // 计算磁盘缓存大小
    NSDirectoryEnumerator *enumerator = [[NSFileManager defaultManager] enumeratorAtPath:self.diskCachePath];
    NSString *fileName;
    
    while ((fileName = [enumerator nextObject])) {
        if (![fileName hasSuffix:@".meta"]) {
            NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:fileName];
            NSDictionary *attributes = [[NSFileManager defaultManager] attributesOfItemAtPath:filePath error:nil];
            diskCacheSize += [attributes[NSFileSize] unsignedIntegerValue];
            fileCount++;
        }
    }
    
    return @{
        @"memory_cache_size": @(memoryCacheSize),
        @"disk_cache_size": @(diskCacheSize),
        @"file_count": @(fileCount),
        @"url_cache_memory_usage": @(self.URLCache.currentMemoryUsage),
        @"url_cache_disk_usage": @(self.URLCache.currentDiskUsage)
    };
}

- (void)cleanExpiredCache {
    NSDirectoryEnumerator *enumerator = [[NSFileManager defaultManager] enumeratorAtPath:self.diskCachePath];
    NSString *fileName;
    NSMutableArray *expiredFiles = [NSMutableArray array];
    
    while ((fileName = [enumerator nextObject])) {
        if ([fileName hasSuffix:@".meta"]) {
            NSString *metadataPath = [self.diskCachePath stringByAppendingPathComponent:fileName];
            NSDictionary *metadata = [NSDictionary dictionaryWithContentsOfFile:metadataPath];
            
            if (metadata) {
                NSTimeInterval timestamp = [metadata[@"timestamp"] doubleValue];
                NSTimeInterval age = [[NSDate date] timeIntervalSince1970] - timestamp;
                
                if (age > self.defaultCacheAge) {
                    NSString *dataFileName = [fileName stringByReplacingOccurrencesOfString:@".meta" withString:@""];
                    [expiredFiles addObject:dataFileName];
                }
            }
        }
    }
    
    // 删除过期文件
    for (NSString *fileName in expiredFiles) {
        [self removeFromDisk:fileName];
    }
    
    NSLog(@"Cleaned %lu expired cache files", (unsigned long)expiredFiles.count);
}

- (NSUInteger)getCurrentCacheSize {
    NSDictionary *statistics = [self getCacheStatistics];
    return [statistics[@"disk_cache_size"] unsignedIntegerValue] + [statistics[@"url_cache_disk_usage"] unsignedIntegerValue];
}

@end
```

## URLSession深度解析

### HTTP请求管理器

```objc
@interface HTTPRequestManager : NSObject

@property (nonatomic, strong) NSURLSession *session;
@property (nonatomic, strong) NSMutableDictionary *activeTasks;
@property (nonatomic, strong) NSMutableDictionary *requestInterceptors;
@property (nonatomic, strong) NSMutableDictionary *responseInterceptors;

+ (instancetype)sharedManager;

// 基础请求方法
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(NSDictionary *)parameters
                      headers:(NSDictionary *)headers
                      success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                      failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure;

- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(NSDictionary *)parameters
                       headers:(NSDictionary *)headers
                       success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure;

- (NSURLSessionDataTask *)PUT:(NSString *)URLString
                   parameters:(NSDictionary *)parameters
                      headers:(NSDictionary *)headers
                      success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                      failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure;

- (NSURLSessionDataTask *)DELETE:(NSString *)URLString
                      parameters:(NSDictionary *)parameters
                         headers:(NSDictionary *)headers
                         success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                         failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure;

// 文件上传下载
- (NSURLSessionUploadTask *)uploadFile:(NSURL *)fileURL
                                 toURL:(NSString *)URLString
                            parameters:(NSDictionary *)parameters
                               headers:(NSDictionary *)headers
                              progress:(void(^)(NSProgress *progress))progress
                               success:(void(^)(NSURLSessionUploadTask *task, id responseObject))success
                               failure:(void(^)(NSURLSessionUploadTask *task, NSError *error))failure;

- (NSURLSessionDownloadTask *)downloadFile:(NSString *)URLString
                               destination:(NSURL *)destinationURL
                                  progress:(void(^)(NSProgress *progress))progress
                                   success:(void(^)(NSURLSessionDownloadTask *task, NSURL *filePath))success
                                   failure:(void(^)(NSURLSessionDownloadTask *task, NSError *error))failure;

// 请求拦截器
- (void)addRequestInterceptor:(NSString *)name
                        block:(NSURLRequest *(^)(NSURLRequest *request))block;
- (void)removeRequestInterceptor:(NSString *)name;

// 响应拦截器
- (void)addResponseInterceptor:(NSString *)name
                         block:(id(^)(NSURLResponse *response, id responseObject, NSError *error))block;
- (void)removeResponseInterceptor:(NSString *)name;

// 任务管理
- (void)cancelAllTasks;
- (void)cancelTasksWithTag:(NSString *)tag;
- (NSArray *)getActiveTasks;

@end

@implementation HTTPRequestManager

+ (instancetype)sharedManager {
    static HTTPRequestManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
        self.activeTasks = [NSMutableDictionary dictionary];
        self.requestInterceptors = [NSMutableDictionary dictionary];
        self.responseInterceptors = [NSMutableDictionary dictionary];
    }
    return self;
}

- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(NSDictionary *)parameters
                      headers:(NSDictionary *)headers
                      success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                      failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure {
    
    NSURLRequest *request = [self buildRequestWithMethod:@"GET"
                                                URLString:URLString
                                               parameters:parameters
                                                  headers:headers
                                                     body:nil];
    
    return [self performDataTaskWithRequest:request
                                     success:success
                                     failure:failure];
}

- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(NSDictionary *)parameters
                       headers:(NSDictionary *)headers
                       success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure {
    
    NSData *bodyData = [self serializeParameters:parameters];
    NSURLRequest *request = [self buildRequestWithMethod:@"POST"
                                                URLString:URLString
                                               parameters:nil
                                                  headers:headers
                                                     body:bodyData];
    
    return [self performDataTaskWithRequest:request
                                     success:success
                                     failure:failure];
}

- (NSURLSessionDataTask *)PUT:(NSString *)URLString
                   parameters:(NSDictionary *)parameters
                      headers:(NSDictionary *)headers
                      success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                      failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure {
    
    NSData *bodyData = [self serializeParameters:parameters];
    NSURLRequest *request = [self buildRequestWithMethod:@"PUT"
                                                URLString:URLString
                                               parameters:nil
                                                  headers:headers
                                                     body:bodyData];
    
    return [self performDataTaskWithRequest:request
                                     success:success
                                     failure:failure];
}

- (NSURLSessionDataTask *)DELETE:(NSString *)URLString
                      parameters:(NSDictionary *)parameters
                         headers:(NSDictionary *)headers
                         success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                         failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure {
    
    NSURLRequest *request = [self buildRequestWithMethod:@"DELETE"
                                                URLString:URLString
                                               parameters:parameters
                                                  headers:headers
                                                     body:nil];
    
    return [self performDataTaskWithRequest:request
                                     success:success
                                     failure:failure];
}

- (NSURLRequest *)buildRequestWithMethod:(NSString *)method
                               URLString:(NSString *)URLString
                              parameters:(NSDictionary *)parameters
                                 headers:(NSDictionary *)headers
                                    body:(NSData *)body {
    
    // 构建URL
    NSURL *URL = [NSURL URLWithString:URLString];
    if (parameters && ([method isEqualToString:@"GET"] || [method isEqualToString:@"DELETE"])) {
        NSString *queryString = [self buildQueryStringFromParameters:parameters];
        NSString *fullURLString = [NSString stringWithFormat:@"%@?%@", URLString, queryString];
        URL = [NSURL URLWithString:fullURLString];
    }
    
    // 创建请求
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:URL];
    request.HTTPMethod = method;
    
    // 设置头部
    if (headers) {
        for (NSString *key in headers) {
            [request setValue:headers[key] forHTTPHeaderField:key];
        }
    }
    
    // 设置默认头部
    if (![request valueForHTTPHeaderField:@"Content-Type"]) {
        [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    }
    
    // 设置请求体
    if (body) {
        request.HTTPBody = body;
    }
    
    // 应用请求拦截器
    NSURLRequest *interceptedRequest = [self applyRequestInterceptors:request];
    
    return interceptedRequest;
}

- (NSString *)buildQueryStringFromParameters:(NSDictionary *)parameters {
    NSMutableArray *queryItems = [NSMutableArray array];
    
    for (NSString *key in parameters) {
        id value = parameters[key];
        NSString *encodedKey = [self URLEncode:key];
        NSString *encodedValue = [self URLEncode:[value description]];
        [queryItems addObject:[NSString stringWithFormat:@"%@=%@", encodedKey, encodedValue]];
    }
    
    return [queryItems componentsJoinedByString:@"&"];
}

- (NSString *)URLEncode:(NSString *)string {
    NSCharacterSet *allowedCharacters = [NSCharacterSet characterSetWithCharactersInString:@"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~"];
    return [string stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacters];
}

- (NSData *)serializeParameters:(NSDictionary *)parameters {
    if (!parameters) {
        return nil;
    }
    
    NSError *error;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:parameters
                                                       options:0
                                                         error:&error];
    
    if (error) {
        NSLog(@"JSON serialization error: %@", error);
        return nil;
    }
    
    return jsonData;
}

- (NSURLSessionDataTask *)performDataTaskWithRequest:(NSURLRequest *)request
                                             success:(void(^)(NSURLSessionDataTask *task, id responseObject))success
                                             failure:(void(^)(NSURLSessionDataTask *task, NSError *error))failure {
    
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:request
                                                  completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        // 从活动任务中移除
        [self.activeTasks removeObjectForKey:@(task.taskIdentifier)];
        
        if (error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (failure) {
                    failure(task, error);
                }
            });
            return;
        }
        
        // 解析响应
        id responseObject = [self parseResponseData:data response:response error:&error];
        
        // 应用响应拦截器
        responseObject = [self applyResponseInterceptors:response responseObject:responseObject error:error];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                if (failure) {
                    failure(task, error);
                }
            } else {
                if (success) {
                    success(task, responseObject);
                }
            }
        });
    }];
    
    // 添加到活动任务
    self.activeTasks[@(task.taskIdentifier)] = task;
    
    [task resume];
    return task;
}

- (id)parseResponseData:(NSData *)data response:(NSURLResponse *)response error:(NSError **)error {
    if (!data) {
        return nil;
    }
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSString *contentType = httpResponse.allHeaderFields[@"Content-Type"];
    
    if ([contentType containsString:@"application/json"]) {
        return [NSJSONSerialization JSONObjectWithData:data options:0 error:error];
    } else if ([contentType containsString:@"text/"]) {
        return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    } else {
        return data;
    }
}

- (NSURLRequest *)applyRequestInterceptors:(NSURLRequest *)request {
    NSURLRequest *currentRequest = request;
    
    for (NSString *name in self.requestInterceptors) {
        NSURLRequest *(^interceptor)(NSURLRequest *) = self.requestInterceptors[name];
        currentRequest = interceptor(currentRequest);
    }
    
    return currentRequest;
}

- (id)applyResponseInterceptors:(NSURLResponse *)response responseObject:(id)responseObject error:(NSError *)error {
    id currentResponseObject = responseObject;
    
    for (NSString *name in self.responseInterceptors) {
        id(^interceptor)(NSURLResponse *, id, NSError *) = self.responseInterceptors[name];
        currentResponseObject = interceptor(response, currentResponseObject, error);
    }
    
    return currentResponseObject;
}

- (void)addRequestInterceptor:(NSString *)name
                        block:(NSURLRequest *(^)(NSURLRequest *request))block {
    self.requestInterceptors[name] = [block copy];
}

- (void)removeRequestInterceptor:(NSString *)name {
    [self.requestInterceptors removeObjectForKey:name];
}

- (void)addResponseInterceptor:(NSString *)name
                         block:(id(^)(NSURLResponse *response, id responseObject, NSError *error))block {
    self.responseInterceptors[name] = [block copy];
}

- (void)removeResponseInterceptor:(NSString *)name {
    [self.responseInterceptors removeObjectForKey:name];
}

- (void)cancelAllTasks {
    for (NSURLSessionTask *task in self.activeTasks.allValues) {
        [task cancel];
    }
    [self.activeTasks removeAllObjects];
}

- (void)cancelTasksWithTag:(NSString *)tag {
    // 实现基于标签的任务取消
    // 这里需要在任务创建时添加标签信息
}

- (NSArray *)getActiveTasks {
    return [self.activeTasks allValues];
}

@end
```