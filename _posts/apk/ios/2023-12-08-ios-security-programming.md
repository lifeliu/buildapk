---
layout: post
title: "iOS安全编程深度解析：数据保护、网络安全与应用防护实战指南"
date: 2023-12-08
categories: ios
tags: [iOS, Security, Encryption, Keychain, Network Security, App Protection]
author: iOS技术专家
description: "深入探讨iOS安全编程的核心技术，包括数据加密、Keychain服务、网络安全、应用防护等关键安全措施的实现与最佳实践。"
---

在移动应用开发中，安全性是至关重要的考虑因素。iOS平台提供了丰富的安全框架和机制，帮助开发者构建安全可靠的应用程序。本文将深入探讨iOS安全编程的核心技术，包括数据加密、身份验证、网络安全、应用防护等关键领域的实现方法和最佳实践。

## 数据加密与保护

### 对称加密实现

```objc
@interface SymmetricEncryption : NSObject

+ (NSData *)encryptData:(NSData *)data withKey:(NSData *)key algorithm:(CCAlgorithm)algorithm;
+ (NSData *)decryptData:(NSData *)encryptedData withKey:(NSData *)key algorithm:(CCAlgorithm)algorithm;
+ (NSData *)generateRandomKey:(size_t)keyLength;
+ (NSData *)encryptString:(NSString *)string withPassword:(NSString *)password;
+ (NSString *)decryptDataToString:(NSData *)encryptedData withPassword:(NSString *)password;

@end

@implementation SymmetricEncryption

+ (NSData *)encryptData:(NSData *)data withKey:(NSData *)key algorithm:(CCAlgorithm)algorithm {
    if (!data || !key) return nil;
    
    size_t keyLength;
    switch (algorithm) {
        case kCCAlgorithmAES128:
            keyLength = kCCKeySizeAES128;
            break;
        case kCCAlgorithmAES:
            keyLength = kCCKeySizeAES256;
            break;
        case kCCAlgorithmDES:
            keyLength = kCCKeySizeDES;
            break;
        default:
            return nil;
    }
    
    if (key.length != keyLength) {
        NSLog(@"Invalid key length for algorithm");
        return nil;
    }
    
    // 生成随机IV
    size_t blockSize = (algorithm == kCCAlgorithmAES128 || algorithm == kCCAlgorithmAES) ? kCCBlockSizeAES128 : kCCBlockSizeDES;
    NSMutableData *iv = [NSMutableData dataWithLength:blockSize];
    SecRandomCopyBytes(kSecRandomDefault, blockSize, iv.mutableBytes);
    
    // 计算输出缓冲区大小
    size_t bufferSize = data.length + blockSize;
    NSMutableData *encryptedData = [NSMutableData dataWithLength:bufferSize];
    
    size_t numBytesEncrypted = 0;
    CCCryptorStatus cryptStatus = CCCrypt(
        kCCEncrypt,
        algorithm,
        kCCOptionPKCS7Padding,
        key.bytes,
        keyLength,
        iv.bytes,
        data.bytes,
        data.length,
        encryptedData.mutableBytes,
        bufferSize,
        &numBytesEncrypted
    );
    
    if (cryptStatus != kCCSuccess) {
        NSLog(@"Encryption failed with status: %d", cryptStatus);
        return nil;
    }
    
    // 将IV和加密数据组合
    NSMutableData *result = [NSMutableData dataWithData:iv];
    [result appendBytes:encryptedData.bytes length:numBytesEncrypted];
    
    return [result copy];
}

+ (NSData *)decryptData:(NSData *)encryptedData withKey:(NSData *)key algorithm:(CCAlgorithm)algorithm {
    if (!encryptedData || !key) return nil;
    
    size_t keyLength;
    size_t blockSize;
    
    switch (algorithm) {
        case kCCAlgorithmAES128:
            keyLength = kCCKeySizeAES128;
            blockSize = kCCBlockSizeAES128;
            break;
        case kCCAlgorithmAES:
            keyLength = kCCKeySizeAES256;
            blockSize = kCCBlockSizeAES128;
            break;
        case kCCAlgorithmDES:
            keyLength = kCCKeySizeDES;
            blockSize = kCCBlockSizeDES;
            break;
        default:
            return nil;
    }
    
    if (key.length != keyLength || encryptedData.length <= blockSize) {
        NSLog(@"Invalid key length or encrypted data size");
        return nil;
    }
    
    // 提取IV
    NSData *iv = [encryptedData subdataWithRange:NSMakeRange(0, blockSize)];
    NSData *cipherData = [encryptedData subdataWithRange:NSMakeRange(blockSize, encryptedData.length - blockSize)];
    
    // 解密
    size_t bufferSize = cipherData.length + blockSize;
    NSMutableData *decryptedData = [NSMutableData dataWithLength:bufferSize];
    
    size_t numBytesDecrypted = 0;
    CCCryptorStatus cryptStatus = CCCrypt(
        kCCDecrypt,
        algorithm,
        kCCOptionPKCS7Padding,
        key.bytes,
        keyLength,
        iv.bytes,
        cipherData.bytes,
        cipherData.length,
        decryptedData.mutableBytes,
        bufferSize,
        &numBytesDecrypted
    );
    
    if (cryptStatus != kCCSuccess) {
        NSLog(@"Decryption failed with status: %d", cryptStatus);
        return nil;
    }
    
    return [NSData dataWithBytes:decryptedData.bytes length:numBytesDecrypted];
}

+ (NSData *)generateRandomKey:(size_t)keyLength {
    NSMutableData *key = [NSMutableData dataWithLength:keyLength];
    OSStatus status = SecRandomCopyBytes(kSecRandomDefault, keyLength, key.mutableBytes);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to generate random key");
        return nil;
    }
    
    return [key copy];
}

+ (NSData *)encryptString:(NSString *)string withPassword:(NSString *)password {
    if (!string || !password) return nil;
    
    // 从密码派生密钥
    NSData *salt = [self generateRandomKey:16]; // 16字节盐值
    NSData *key = [self deriveKeyFromPassword:password salt:salt keyLength:kCCKeySizeAES256];
    
    if (!key) return nil;
    
    NSData *stringData = [string dataUsingEncoding:NSUTF8StringEncoding];
    NSData *encryptedData = [self encryptData:stringData withKey:key algorithm:kCCAlgorithmAES];
    
    if (!encryptedData) return nil;
    
    // 将盐值和加密数据组合
    NSMutableData *result = [NSMutableData dataWithData:salt];
    [result appendData:encryptedData];
    
    return [result copy];
}

+ (NSString *)decryptDataToString:(NSData *)encryptedData withPassword:(NSString *)password {
    if (!encryptedData || !password || encryptedData.length <= 16) return nil;
    
    // 提取盐值
    NSData *salt = [encryptedData subdataWithRange:NSMakeRange(0, 16)];
    NSData *cipherData = [encryptedData subdataWithRange:NSMakeRange(16, encryptedData.length - 16)];
    
    // 从密码派生密钥
    NSData *key = [self deriveKeyFromPassword:password salt:salt keyLength:kCCKeySizeAES256];
    
    if (!key) return nil;
    
    NSData *decryptedData = [self decryptData:cipherData withKey:key algorithm:kCCAlgorithmAES];
    
    if (!decryptedData) return nil;
    
    return [[NSString alloc] initWithData:decryptedData encoding:NSUTF8StringEncoding];
}

+ (NSData *)deriveKeyFromPassword:(NSString *)password salt:(NSData *)salt keyLength:(size_t)keyLength {
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    NSMutableData *derivedKey = [NSMutableData dataWithLength:keyLength];
    
    int result = CCKeyDerivationPBKDF(
        kCCPBKDF2,
        passwordData.bytes,
        passwordData.length,
        salt.bytes,
        salt.length,
        kCCPRFHmacAlgSHA256,
        10000, // 迭代次数
        derivedKey.mutableBytes,
        keyLength
    );
    
    if (result != kCCSuccess) {
        NSLog(@"Key derivation failed");
        return nil;
    }
    
    return [derivedKey copy];
}

@end
```

## 生物识别认证

### Touch ID和Face ID集成

```objc
@interface BiometricAuthentication : NSObject

+ (BOOL)isBiometricAuthenticationAvailable;
+ (LABiometryType)availableBiometryType;
+ (void)authenticateWithReason:(NSString *)reason
                    completion:(void (^)(BOOL success, NSError *error))completion;
+ (void)authenticateWithReason:(NSString *)reason
                      fallback:(NSString *)fallbackTitle
                    completion:(void (^)(BOOL success, NSError *error))completion;
+ (BOOL)canEvaluatePolicy:(LAPolicy)policy error:(NSError **)error;
+ (void)evaluateAccessControl:(SecAccessControlRef)accessControl
                    operation:(NSString *)operation
                   completion:(void (^)(BOOL success, NSError *error))completion;

@end

@implementation BiometricAuthentication

+ (BOOL)isBiometricAuthenticationAvailable {
    LAContext *context = [[LAContext alloc] init];
    NSError *error = nil;
    
    BOOL canEvaluate = [context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
                                            error:&error];
    
    if (error) {
        NSLog(@"Biometric authentication error: %@", error.localizedDescription);
    }
    
    return canEvaluate;
}

+ (LABiometryType)availableBiometryType {
    if (@available(iOS 11.0, *)) {
        LAContext *context = [[LAContext alloc] init];
        NSError *error = nil;
        
        if ([context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error]) {
            return context.biometryType;
        }
    }
    
    return LABiometryTypeNone;
}

+ (void)authenticateWithReason:(NSString *)reason
                    completion:(void (^)(BOOL success, NSError *error))completion {
    
    LAContext *context = [[LAContext alloc] init];
    
    // 设置本地化字符串
    context.localizedFallbackTitle = @"使用密码";
    context.localizedCancelTitle = @"取消";
    
    NSError *error = nil;
    if (![context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error]) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(NO, error);
        });
        return;
    }
    
    [context evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
            localizedReason:reason
                      reply:^(BOOL success, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(success, error);
        });
    }];
}

+ (void)authenticateWithReason:(NSString *)reason
                      fallback:(NSString *)fallbackTitle
                    completion:(void (^)(BOOL success, NSError *error))completion {
    
    LAContext *context = [[LAContext alloc] init];
    
    if (fallbackTitle) {
        context.localizedFallbackTitle = fallbackTitle;
    }
    
    [context evaluatePolicy:LAPolicyDeviceOwnerAuthentication
            localizedReason:reason
                      reply:^(BOOL success, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(success, error);
        });
    }];
}

+ (BOOL)canEvaluatePolicy:(LAPolicy)policy error:(NSError **)error {
    LAContext *context = [[LAContext alloc] init];
    return [context canEvaluatePolicy:policy error:error];
}

+ (void)evaluateAccessControl:(SecAccessControlRef)accessControl
                    operation:(NSString *)operation
                   completion:(void (^)(BOOL success, NSError *error))completion {
    
    LAContext *context = [[LAContext alloc] init];
    
    [context evaluateAccessControl:accessControl
                         operation:LAAccessControlOperationUseItem
                   localizedReason:operation
                             reply:^(BOOL success, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            completion(success, error);
        });
    }];
}

@end
```

### 安全存储管理器

```objc
@interface SecureStorageManager : NSObject

+ (instancetype)sharedManager;
- (BOOL)storeSecureData:(NSData *)data forKey:(NSString *)key requireBiometric:(BOOL)requireBiometric;
- (void)retrieveSecureDataForKey:(NSString *)key
                      completion:(void (^)(NSData *data, NSError *error))completion;
- (BOOL)deleteSecureDataForKey:(NSString *)key;
- (void)storeCredentials:(NSString *)username
                password:(NSString *)password
                 service:(NSString *)service
         requireBiometric:(BOOL)requireBiometric
              completion:(void (^)(BOOL success, NSError *error))completion;
- (void)retrieveCredentialsForService:(NSString *)service
                           completion:(void (^)(NSString *username, NSString *password, NSError *error))completion;

@end

@implementation SecureStorageManager

+ (instancetype)sharedManager {
    static SecureStorageManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (BOOL)storeSecureData:(NSData *)data forKey:(NSString *)key requireBiometric:(BOOL)requireBiometric {
    if (!data || !key) return NO;
    
    CFErrorRef error = NULL;
    SecAccessControlRef accessControl = NULL;
    
    if (requireBiometric) {
        accessControl = SecAccessControlCreateWithFlags(
            kCFAllocatorDefault,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            kSecAccessControlBiometryAny,
            &error
        );
        
        if (error) {
            NSLog(@"Failed to create access control: %@", (__bridge NSError *)error);
            CFRelease(error);
            return NO;
        }
    }
    
    NSMutableDictionary *query = [@{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"SecureStorage",
        (__bridge id)kSecValueData: data
    } mutableCopy];
    
    if (accessControl) {
        query[(__bridge id)kSecAttrAccessControl] = (__bridge id)accessControl;
    } else {
        query[(__bridge id)kSecAttrAccessible] = (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly;
    }
    
    // 删除现有项目
    [self deleteSecureDataForKey:key];
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (accessControl) {
        CFRelease(accessControl);
    }
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to store secure data. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

- (void)retrieveSecureDataForKey:(NSString *)key
                      completion:(void (^)(NSData *data, NSError *error))completion {
    if (!key || !completion) return;
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"SecureStorage",
        (__bridge id)kSecReturnData: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    };
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        CFDataRef data = NULL;
        OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, (CFTypeRef *)&data);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (status == errSecSuccess) {
                completion((__bridge_transfer NSData *)data, nil);
            } else {
                NSError *error = [NSError errorWithDomain:@"SecureStorageError"
                                                     code:status
                                                 userInfo:@{NSLocalizedDescriptionKey: @"Failed to retrieve secure data"}];
                completion(nil, error);
            }
        });
    });
}

- (BOOL)deleteSecureDataForKey:(NSString *)key {
    if (!key) return NO;
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"SecureStorage"
    };
    
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    
    if (status != errSecSuccess && status != errSecItemNotFound) {
        NSLog(@"Failed to delete secure data. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

- (void)storeCredentials:(NSString *)username
                password:(NSString *)password
                 service:(NSString *)service
         requireBiometric:(BOOL)requireBiometric
              completion:(void (^)(BOOL success, NSError *error))completion {
    
    if (!username || !password || !service) {
        if (completion) {
            NSError *error = [NSError errorWithDomain:@"SecureStorageError"
                                                 code:-1
                                             userInfo:@{NSLocalizedDescriptionKey: @"Invalid parameters"}];
            completion(NO, error);
        }
        return;
    }
    
    // 创建凭据字典
    NSDictionary *credentials = @{
        @"username": username,
        @"password": password
    };
    
    NSError *jsonError = nil;
    NSData *credentialsData = [NSJSONSerialization dataWithJSONObject:credentials
                                                              options:0
                                                                error:&jsonError];
    
    if (jsonError) {
        if (completion) {
            completion(NO, jsonError);
        }
        return;
    }
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        BOOL success = [self storeSecureData:credentialsData
                                      forKey:service
                             requireBiometric:requireBiometric];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                NSError *error = success ? nil : [NSError errorWithDomain:@"SecureStorageError"
                                                                     code:-2
                                                                 userInfo:@{NSLocalizedDescriptionKey: @"Failed to store credentials"}];
                completion(success, error);
            }
        });
    });
}

- (void)retrieveCredentialsForService:(NSString *)service
                           completion:(void (^)(NSString *username, NSString *password, NSError *error))completion {
    
    if (!service || !completion) return;
    
    [self retrieveSecureDataForKey:service completion:^(NSData *data, NSError *error) {
        if (error) {
            completion(nil, nil, error);
            return;
        }
        
        NSError *jsonError = nil;
        NSDictionary *credentials = [NSJSONSerialization JSONObjectWithData:data
                                                                    options:0
                                                                      error:&jsonError];
        
        if (jsonError) {
            completion(nil, nil, jsonError);
            return;
        }
        
        NSString *username = credentials[@"username"];
        NSString *password = credentials[@"password"];
        
        completion(username, password, nil);
    }];
}

@end
```

## 网络安全

### SSL/TLS证书验证

```objc
@interface NetworkSecurityManager : NSObject <NSURLSessionDelegate>

+ (instancetype)sharedManager;
- (NSURLSession *)createSecureSessionWithPinning:(BOOL)enablePinning;
- (BOOL)validateCertificateChain:(SecTrustRef)serverTrust forHost:(NSString *)host;
- (BOOL)validateCertificatePinning:(SecTrustRef)serverTrust;
- (NSData *)sha256HashOfCertificate:(SecCertificateRef)certificate;

@property (nonatomic, strong) NSSet<NSString *> *pinnedCertificateHashes;
@property (nonatomic, assign) BOOL allowInvalidCertificates;
@property (nonatomic, assign) BOOL validatesDomainName;

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
        _allowInvalidCertificates = NO;
        _validatesDomainName = YES;
        
        // 示例：预设的证书哈希值（实际使用时应该是你的服务器证书的哈希）
        _pinnedCertificateHashes = [NSSet setWithArray:@[
            @"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=", // 替换为实际的证书哈希
            @"BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="  // 备用证书哈希
        ]];
    }
    return self;
}

- (NSURLSession *)createSecureSessionWithPinning:(BOOL)enablePinning {
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    
    // 设置超时
    config.timeoutIntervalForRequest = 30.0;
    config.timeoutIntervalForResource = 60.0;
    
    // 设置缓存策略
    config.requestCachePolicy = NSURLRequestReloadIgnoringLocalCacheData;
    
    // 创建会话
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config
                                                          delegate:enablePinning ? self : nil
                                                     delegateQueue:nil];
    
    return session;
}

- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *))completionHandler {
    
    NSString *authMethod = challenge.protectionSpace.authenticationMethod;
    
    if ([authMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        NSString *host = challenge.protectionSpace.host;
        SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
        
        // 验证证书链
        BOOL isValidChain = [self validateCertificateChain:serverTrust forHost:host];
        
        // 验证证书固定
        BOOL isValidPinning = [self validateCertificatePinning:serverTrust];
        
        if (isValidChain && isValidPinning) {
            NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        } else {
            NSLog(@"Certificate validation failed for host: %@", host);
            completionHandler(NSURLSessionAuthChallengeRejectProtectionSpace, nil);
        }
    } else {
        completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
    }
}

- (BOOL)validateCertificateChain:(SecTrustRef)serverTrust forHost:(NSString *)host {
    if (!serverTrust) return NO;
    
    // 如果允许无效证书，跳过验证
    if (self.allowInvalidCertificates) {
        return YES;
    }
    
    // 设置验证策略
    SecPolicyRef policy = SecPolicyCreateSSL(true, (__bridge CFStringRef)host);
    SecTrustSetPolicies(serverTrust, policy);
    CFRelease(policy);
    
    // 执行验证
    SecTrustResultType result;
    OSStatus status = SecTrustEvaluate(serverTrust, &result);
    
    if (status != errSecSuccess) {
        NSLog(@"Certificate evaluation failed with status: %d", (int)status);
        return NO;
    }
    
    // 检查验证结果
    BOOL isValid = (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);
    
    if (!isValid) {
        NSLog(@"Certificate validation failed with result: %d", result);
    }
    
    return isValid;
}

- (BOOL)validateCertificatePinning:(SecTrustRef)serverTrust {
    if (!serverTrust || self.pinnedCertificateHashes.count == 0) {
        return YES; // 如果没有设置固定证书，跳过验证
    }
    
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    
    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        NSData *certificateHash = [self sha256HashOfCertificate:certificate];
        
        if (certificateHash) {
            NSString *hashString = [certificateHash base64EncodedStringWithOptions:0];
            
            if ([self.pinnedCertificateHashes containsObject:hashString]) {
                return YES;
            }
        }
    }
    
    NSLog(@"Certificate pinning validation failed");
    return NO;
}

- (NSData *)sha256HashOfCertificate:(SecCertificateRef)certificate {
    if (!certificate) return nil;
    
    CFDataRef certificateData = SecCertificateCopyData(certificate);
    if (!certificateData) return nil;
    
    NSData *data = (__bridge NSData *)certificateData;
    NSMutableData *hash = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH];
    
    CC_SHA256(data.bytes, (CC_LONG)data.length, hash.mutableBytes);
    
    CFRelease(certificateData);
    
    return [hash copy];
}

@end
```

### 安全网络请求管理

```objc
@interface SecureNetworkManager : NSObject

+ (instancetype)sharedManager;
- (void)performSecureRequest:(NSURLRequest *)request
                  completion:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completion;
- (NSURLRequest *)createSecureRequestWithURL:(NSURL *)url
                                      method:(NSString *)method
                                        body:(NSData *)body
                                     headers:(NSDictionary *)headers;
- (void)uploadData:(NSData *)data
             toURL:(NSURL *)url
        completion:(void (^)(NSData *responseData, NSError *error))completion;
- (void)downloadFromURL:(NSURL *)url
             completion:(void (^)(NSURL *fileURL, NSError *error))completion;

@property (nonatomic, strong) NSString *apiKey;
@property (nonatomic, strong) NSString *userAgent;
@property (nonatomic, assign) NSTimeInterval requestTimeout;

@end

@implementation SecureNetworkManager

+ (instancetype)sharedManager {
    static SecureNetworkManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _requestTimeout = 30.0;
        _userAgent = [NSString stringWithFormat:@"SecureApp/1.0 (%@; iOS %@)",
                     [[UIDevice currentDevice] model],
                     [[UIDevice currentDevice] systemVersion]];
    }
    return self;
}

- (void)performSecureRequest:(NSURLRequest *)request
                  completion:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completion {
    
    if (!request || !completion) return;
    
    // 创建安全会话
    NSURLSession *session = [[NetworkSecurityManager sharedManager] createSecureSessionWithPinning:YES];
    
    NSURLSessionDataTask *task = [session dataTaskWithRequest:request
                                            completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                NSLog(@"Network request failed: %@", error.localizedDescription);
                completion(nil, response, error);
                return;
            }
            
            // 验证响应
            if ([response isKindOfClass:[NSHTTPURLResponse class]]) {
                NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
                
                if (httpResponse.statusCode >= 200 && httpResponse.statusCode < 300) {
                    completion(data, response, nil);
                } else {
                    NSError *httpError = [NSError errorWithDomain:@"HTTPError"
                                                             code:httpResponse.statusCode
                                                         userInfo:@{NSLocalizedDescriptionKey: [NSString stringWithFormat:@"HTTP Error %ld", (long)httpResponse.statusCode]}];
                    completion(nil, response, httpError);
                }
            } else {
                completion(data, response, nil);
            }
        });
    }];
    
    [task resume];
}

- (NSURLRequest *)createSecureRequestWithURL:(NSURL *)url
                                      method:(NSString *)method
                                        body:(NSData *)body
                                     headers:(NSDictionary *)headers {
    
    if (!url) return nil;
    
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = method ?: @"GET";
    request.timeoutInterval = self.requestTimeout;
    
    // 设置默认头部
    [request setValue:self.userAgent forHTTPHeaderField:@"User-Agent"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    
    // 添加API密钥
    if (self.apiKey) {
        [request setValue:self.apiKey forHTTPHeaderField:@"X-API-Key"];
    }
    
    // 添加自定义头部
    for (NSString *key in headers) {
        [request setValue:headers[key] forHTTPHeaderField:key];
    }
    
    // 设置请求体
    if (body && [method isEqualToString:@"POST"] || [method isEqualToString:@"PUT"]) {
        request.HTTPBody = body;
        [request setValue:[NSString stringWithFormat:@"%lu", (unsigned long)body.length]
       forHTTPHeaderField:@"Content-Length"];
    }
    
    // 添加时间戳和随机数（防重放攻击）
    NSTimeInterval timestamp = [[NSDate date] timeIntervalSince1970];
    NSString *nonce = [[NSUUID UUID] UUIDString];
    
    [request setValue:[NSString stringWithFormat:@"%.0f", timestamp] forHTTPHeaderField:@"X-Timestamp"];
    [request setValue:nonce forHTTPHeaderField:@"X-Nonce"];
    
    return [request copy];
}

- (void)uploadData:(NSData *)data
             toURL:(NSURL *)url
        completion:(void (^)(NSData *responseData, NSError *error))completion {
    
    if (!data || !url || !completion) return;
    
    NSURLRequest *request = [self createSecureRequestWithURL:url
                                                      method:@"POST"
                                                        body:data
                                                     headers:@{@"Content-Type": @"application/octet-stream"}];
    
    [self performSecureRequest:request completion:^(NSData *responseData, NSURLResponse *response, NSError *error) {
        completion(responseData, error);
    }];
}

- (void)downloadFromURL:(NSURL *)url
             completion:(void (^)(NSURL *fileURL, NSError *error))completion {
    
    if (!url || !completion) return;
    
    NSURLSession *session = [[NetworkSecurityManager sharedManager] createSecureSessionWithPinning:YES];
    
    NSURLSessionDownloadTask *downloadTask = [session downloadTaskWithURL:url
                                                        completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error) {
                completion(nil, error);
                return;
            }
            
            // 移动文件到安全位置
            NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
            NSString *documentsDirectory = [paths objectAtIndex:0];
            NSString *fileName = [NSString stringWithFormat:@"download_%@", [[NSUUID UUID] UUIDString]];
            NSString *filePath = [documentsDirectory stringByAppendingPathComponent:fileName];
            NSURL *destinationURL = [NSURL fileURLWithPath:filePath];
            
            NSError *moveError = nil;
            BOOL success = [[NSFileManager defaultManager] moveItemAtURL:location
                                                                   toURL:destinationURL
                                                                   error:&moveError];
            
            if (success) {
                completion(destinationURL, nil);
            } else {
                completion(nil, moveError);
            }
        });
    }];
    
    [downloadTask resume];
}

@end
```

## 应用防护与反调试

### 反调试检测

```objc
@interface AntiDebuggingManager : NSObject

+ (BOOL)isDebuggerAttached;
+ (BOOL)isJailbroken;
+ (BOOL)isRunningOnSimulator;
+ (void)enableAntiDebuggingProtection;
+ (BOOL)detectDynamicLibraryInjection;
+ (BOOL)detectCodeInjection;
+ (void)obfuscateString:(NSString *)string;

@end

@implementation AntiDebuggingManager

+ (BOOL)isDebuggerAttached {
    // 方法1：检查ptrace
    int mib[4];
    struct kinfo_proc info;
    size_t size = sizeof(info);
    
    mib[0] = CTL_KERN;
    mib[1] = KERN_PROC;
    mib[2] = KERN_PROC_PID;
    mib[3] = getpid();
    
    if (sysctl(mib, sizeof(mib) / sizeof(*mib), &info, &size, NULL, 0) != 0) {
        return NO;
    }
    
    // 检查P_TRACED标志
    BOOL isDebugged = (info.kp_proc.p_flag & P_TRACED) != 0;
    
    // 方法2：检查调试器端口
    mach_port_t port = MACH_PORT_NULL;
    if (task_get_exception_ports(mach_task_self(), EXC_MASK_ALL, NULL, NULL, NULL, NULL, NULL) == KERN_SUCCESS) {
        // 如果能获取异常端口，可能有调试器
    }
    
    // 方法3：检查父进程
    pid_t ppid = getppid();
    if (ppid != 1) { // 正常情况下父进程应该是launchd (PID 1)
        char pathbuf[PROC_PIDPATHINFO_MAXSIZE];
        if (proc_pidpath(ppid, pathbuf, sizeof(pathbuf)) > 0) {
            NSString *parentPath = [NSString stringWithUTF8String:pathbuf];
            if ([parentPath containsString:@"debugserver"] || [parentPath containsString:@"lldb"]) {
                isDebugged = YES;
            }
        }
    }
    
    return isDebugged;
}

+ (BOOL)isJailbroken {
    // 检查常见的越狱文件和目录
    NSArray *jailbreakPaths = @[
        @"/Applications/Cydia.app",
        @"/Library/MobileSubstrate/MobileSubstrate.dylib",
        @"/bin/bash",
        @"/usr/sbin/sshd",
        @"/etc/apt",
        @"/private/var/lib/apt/",
        @"/private/var/lib/cydia",
        @"/private/var/mobile/Library/SBSettings/Themes",
        @"/Library/MobileSubstrate/DynamicLibraries/LiveClock.plist",
        @"/System/Library/LaunchDaemons/com.ikey.bbot.plist",
        @"/System/Library/LaunchDaemons/com.saurik.Cydia.Startup.plist",
        @"/var/cache/apt",
        @"/var/lib/apt",
        @"/var/lib/cydia",
        @"/usr/libexec/sftp-server",
        @"/usr/bin/sshd"
    ];
    
    for (NSString *path in jailbreakPaths) {
        if ([[NSFileManager defaultManager] fileExistsAtPath:path]) {
            return YES;
        }
    }
    
    // 检查是否能写入系统目录
    NSString *testPath = @"/private/test_jailbreak.txt";
    NSError *error;
    [@"test" writeToFile:testPath atomically:YES encoding:NSUTF8StringEncoding error:&error];
    if (!error) {
        [[NSFileManager defaultManager] removeItemAtPath:testPath error:nil];
        return YES;
    }
    
    // 检查URL Scheme
    if ([[UIApplication sharedApplication] canOpenURL:[NSURL URLWithString:@"cydia://package/com.example.package"]]) {
        return YES;
    }
    
    // 检查动态库
    uint32_t count = _dyld_image_count();
    for (uint32_t i = 0; i < count; i++) {
        const char *name = _dyld_get_image_name(i);
        if (name) {
            NSString *imageName = [NSString stringWithUTF8String:name];
            if ([imageName containsString:@"MobileSubstrate"] ||
                [imageName containsString:@"Substrate"] ||
                [imageName containsString:@"Cycript"]) {
                return YES;
            }
        }
    }
    
    return NO;
}

+ (BOOL)isRunningOnSimulator {
#if TARGET_IPHONE_SIMULATOR
    return YES;
#else
    return NO;
#endif
}

+ (void)enableAntiDebuggingProtection {
    // 禁用ptrace
    ptrace(PT_DENY_ATTACH, 0, 0, 0);
    
    // 设置信号处理器
    signal(SIGINT, SIG_IGN);
    signal(SIGTERM, SIG_IGN);
    signal(SIGKILL, SIG_IGN);
    
    // 定期检查调试器
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    dispatch_async(queue, ^{
        while (YES) {
            if ([self isDebuggerAttached]) {
                // 检测到调试器，执行保护措施
                exit(0); // 或者其他保护措施
            }
            sleep(1);
        }
    });
}

+ (BOOL)detectDynamicLibraryInjection {
    uint32_t count = _dyld_image_count();
    
    // 记录预期的系统库数量（这个值需要根据实际情况调整）
    static uint32_t expectedLibraryCount = 0;
    if (expectedLibraryCount == 0) {
        expectedLibraryCount = count;
        return NO;
    }
    
    // 如果库数量显著增加，可能有注入
    if (count > expectedLibraryCount + 5) {
        return YES;
    }
    
    // 检查可疑的库名
    for (uint32_t i = 0; i < count; i++) {
        const char *name = _dyld_get_image_name(i);
        if (name) {
            NSString *imageName = [NSString stringWithUTF8String:name];
            NSArray *suspiciousLibraries = @[
                @"FridaGadget",
                @"frida",
                @"cycript",
                @"substrate",
                @"substitute"
            ];
            
            for (NSString *suspicious in suspiciousLibraries) {
                if ([imageName.lowercaseString containsString:suspicious.lowercaseString]) {
                    return YES;
                }
            }
        }
    }
    
    return NO;
}

+ (BOOL)detectCodeInjection {
    // 检查内存中的可疑模式
    vm_address_t address = 0;
    vm_size_t size = 0;
    vm_region_basic_info_data_64_t info;
    mach_msg_type_number_t count = VM_REGION_BASIC_INFO_COUNT_64;
    mach_port_t object_name;
    
    while (vm_region_64(mach_task_self(), &address, &size, VM_REGION_BASIC_INFO_64,
                       (vm_region_info_t)&info, &count, &object_name) == KERN_SUCCESS) {
        
        // 检查可执行内存区域
        if (info.protection & VM_PROT_EXECUTE) {
            // 这里可以添加更复杂的代码完整性检查
            // 例如计算关键函数的哈希值并与预期值比较
        }
        
        address += size;
    }
    
    return NO;
}

+ (void)obfuscateString:(NSString *)string {
    // 简单的字符串混淆示例
    // 实际应用中应该使用更复杂的混淆技术
    const char *cString = [string UTF8String];
    char *obfuscated = malloc(strlen(cString) + 1);
    
    for (int i = 0; i < strlen(cString); i++) {
        obfuscated[i] = cString[i] ^ 0xAA; // 简单的XOR混淆
    }
    obfuscated[strlen(cString)] = '\0';
    
    // 使用混淆后的字符串...
    
    free(obfuscated);
}

@end
```

### 代码完整性验证

```objc
@interface CodeIntegrityChecker : NSObject

+ (BOOL)verifyApplicationIntegrity;
+ (NSString *)calculateExecutableHash;
+ (BOOL)verifyCodeSignature;
+ (BOOL)checkApplicationBundle;
+ (void)performRuntimeIntegrityCheck;

@end

@implementation CodeIntegrityChecker

+ (BOOL)verifyApplicationIntegrity {
    // 检查代码签名
    if (![self verifyCodeSignature]) {
        NSLog(@"Code signature verification failed");
        return NO;
    }
    
    // 检查应用包完整性
    if (![self checkApplicationBundle]) {
        NSLog(@"Application bundle integrity check failed");
        return NO;
    }
    
    // 计算并验证可执行文件哈希
    NSString *currentHash = [self calculateExecutableHash];
    NSString *expectedHash = @"YOUR_EXPECTED_HASH_HERE"; // 预期的哈希值
    
    if (![currentHash isEqualToString:expectedHash]) {
        NSLog(@"Executable hash verification failed");
        return NO;
    }
    
    return YES;
}

+ (NSString *)calculateExecutableHash {
    NSString *executablePath = [[NSBundle mainBundle] executablePath];
    NSData *executableData = [NSData dataWithContentsOfFile:executablePath];
    
    if (!executableData) {
        return nil;
    }
    
    NSMutableData *hash = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH];
    CC_SHA256(executableData.bytes, (CC_LONG)executableData.length, hash.mutableBytes);
    
    // 转换为十六进制字符串
    NSMutableString *hashString = [NSMutableString string];
    const unsigned char *bytes = hash.bytes;
    for (int i = 0; i < CC_SHA256_DIGEST_LENGTH; i++) {
        [hashString appendFormat:@"%02x", bytes[i]];
    }
    
    return [hashString copy];
}

+ (BOOL)verifyCodeSignature {
    SecStaticCodeRef staticCode = NULL;
    OSStatus status;
    
    // 获取当前应用的静态代码引用
    status = SecStaticCodeCreateWithPath((__bridge CFURLRef)[[NSBundle mainBundle] bundleURL],
                                        kSecCSDefaultFlags, &staticCode);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to create static code reference: %d", (int)status);
        return NO;
    }
    
    // 验证代码签名
    status = SecStaticCodeCheckValidity(staticCode, kSecCSDefaultFlags, NULL);
    
    CFRelease(staticCode);
    
    if (status != errSecSuccess) {
        NSLog(@"Code signature validation failed: %d", (int)status);
        return NO;
    }
    
    return YES;
}

+ (BOOL)checkApplicationBundle {
    NSBundle *mainBundle = [NSBundle mainBundle];
    
    // 检查Info.plist
    NSDictionary *infoPlist = [mainBundle infoDictionary];
    if (!infoPlist) {
        return NO;
    }
    
    // 检查Bundle ID
    NSString *bundleID = [mainBundle bundleIdentifier];
    NSString *expectedBundleID = @"com.yourcompany.yourapp"; // 替换为实际的Bundle ID
    if (![bundleID isEqualToString:expectedBundleID]) {
        return NO;
    }
    
    // 检查关键文件是否存在
    NSArray *criticalFiles = @[
        @"Info.plist",
        @"PkgInfo"
    ];
    
    for (NSString *fileName in criticalFiles) {
        NSString *filePath = [mainBundle pathForResource:fileName.stringByDeletingPathExtension
                                                  ofType:fileName.pathExtension];
        if (!filePath || ![[NSFileManager defaultManager] fileExistsAtPath:filePath]) {
            return NO;
        }
    }
    
    return YES;
}

+ (void)performRuntimeIntegrityCheck {
    // 定期执行完整性检查
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
    dispatch_async(queue, ^{
        while (YES) {
            if (![self verifyApplicationIntegrity]) {
                // 检测到完整性问题，执行保护措施
                dispatch_async(dispatch_get_main_queue(), ^{
                    // 可以显示警告或退出应用
                    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"安全警告"
                                                                               message:@"检测到应用完整性问题"
                                                                        preferredStyle:UIAlertControllerStyleAlert];
                    
                    UIAlertAction *exitAction = [UIAlertAction actionWithTitle:@"退出"
                                                                         style:UIAlertActionStyleDefault
                                                                       handler:^(UIAlertAction *action) {
                        exit(0);
                    }];
                    
                    [alert addAction:exitAction];
                    
                    // 显示警告（需要获取当前视图控制器）
                    // [[UIApplication sharedApplication].keyWindow.rootViewController presentViewController:alert animated:YES completion:nil];
                });
                break;
            }
            
            sleep(60); // 每分钟检查一次
        }
    });
}

@end
```

## 安全编程最佳实践

### 安全开发指南

```objc
@interface SecurityBestPractices : NSObject

// 安全的数据处理
+ (void)secureDataHandlingExample;

// 安全的网络通信
+ (void)secureNetworkCommunicationExample;

// 安全的用户输入处理
+ (NSString *)sanitizeUserInput:(NSString *)input;

// 安全的文件操作
+ (BOOL)secureFileOperationWithData:(NSData *)data fileName:(NSString *)fileName;

// 安全的内存管理
+ (void)secureMemoryManagementExample;

@end

@implementation SecurityBestPractices

+ (void)secureDataHandlingExample {
    // 1. 敏感数据加密存储
    NSString *sensitiveData = @"用户的敏感信息";
    NSString *password = @"用户密码";
    
    // 使用对称加密
    NSData *encryptedData = [SymmetricEncryption encryptString:sensitiveData withPassword:password];
    
    // 存储到Keychain而不是UserDefaults
    [[SecureStorageManager sharedManager] storeSecureData:encryptedData
                                                   forKey:@"sensitive_data"
                                          requireBiometric:YES];
    
    // 2. 清除内存中的敏感数据
    // 使用安全的字符串清除方法
    [self secureStringClear:sensitiveData];
    [self secureStringClear:password];
}

+ (void)secureNetworkCommunicationExample {
    // 1. 使用HTTPS和证书固定
    NSURL *url = [NSURL URLWithString:@"https://api.example.com/data"];
    
    NSDictionary *requestData = @{
        @"user_id": @"12345",
        @"timestamp": @([[NSDate date] timeIntervalSince1970])
    };
    
    NSError *jsonError;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:requestData
                                                       options:0
                                                         error:&jsonError];
    
    if (!jsonError) {
        NSURLRequest *request = [[SecureNetworkManager sharedManager]
                                createSecureRequestWithURL:url
                                                    method:@"POST"
                                                      body:jsonData
                                                   headers:@{@"Content-Type": @"application/json"}];
        
        [[SecureNetworkManager sharedManager] performSecureRequest:request
                                                         completion:^(NSData *data, NSURLResponse *response, NSError *error) {
            if (!error) {
                // 处理响应数据
                NSLog(@"Request successful");
            } else {
                NSLog(@"Request failed: %@", error.localizedDescription);
            }
        }];
    }
}

+ (NSString *)sanitizeUserInput:(NSString *)input {
    if (!input) return nil;
    
    // 1. 移除危险字符
    NSCharacterSet *dangerousChars = [NSCharacterSet characterSetWithCharactersInString:@"<>\"'&"];
    NSString *sanitized = [[input componentsSeparatedByCharactersInSet:dangerousChars] componentsJoinedByString:@""];
    
    // 2. 限制长度
    NSUInteger maxLength = 1000;
    if (sanitized.length > maxLength) {
        sanitized = [sanitized substringToIndex:maxLength];
    }
    
    // 3. 验证格式（例如邮箱格式）
    if ([self isValidEmail:sanitized]) {
        return sanitized;
    }
    
    return nil;
}

+ (BOOL)isValidEmail:(NSString *)email {
    NSString *emailRegex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}";
    NSPredicate *emailPredicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", emailRegex];
    return [emailPredicate evaluateWithObject:email];
}

+ (BOOL)secureFileOperationWithData:(NSData *)data fileName:(NSString *)fileName {
    if (!data || !fileName) return NO;
    
    // 1. 验证文件名安全性
    if ([fileName containsString:@"../"] || [fileName containsString:@"~"]) {
        NSLog(@"Unsafe file name detected");
        return NO;
    }
    
    // 2. 使用应用沙盒目录
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths objectAtIndex:0];
    NSString *filePath = [documentsDirectory stringByAppendingPathComponent:fileName];
    
    // 3. 设置文件保护级别
    NSError *error;
    BOOL success = [data writeToFile:filePath options:NSDataWritingFileProtectionComplete error:&error];
    
    if (success) {
        // 4. 设置文件属性
        NSDictionary *attributes = @{
            NSFileProtectionKey: NSFileProtectionComplete
        };
        
        [[NSFileManager defaultManager] setAttributes:attributes
                                          ofItemAtPath:filePath
                                                 error:&error];
    }
    
    if (error) {
        NSLog(@"File operation failed: %@", error.localizedDescription);
        return NO;
    }
    
    return success;
}

+ (void)secureMemoryManagementExample {
    // 1. 使用安全的内存分配
    size_t size = 1024;
    void *secureBuffer = malloc(size);
    
    if (secureBuffer) {
        // 2. 初始化内存
        memset(secureBuffer, 0, size);
        
        // 使用内存...
        
        // 3. 安全清除内存
        memset_s(secureBuffer, size, 0, size);
        
        // 4. 释放内存
        free(secureBuffer);
        secureBuffer = NULL;
    }
    
    // 5. 避免在栈上存储敏感数据
    // 不好的做法：
    // char password[100] = "sensitive_password";
    
    // 好的做法：使用堆内存并及时清除
    char *password = malloc(100);
    strcpy(password, "sensitive_password");
    
    // 使用密码...
    
    // 清除并释放
    memset_s(password, 100, 0, 100);
    free(password);
    password = NULL;
}

+ (void)secureStringClear:(NSString *)string {
    // 注意：NSString是不可变的，这个方法主要用于演示
    // 实际应用中应该使用NSMutableString或C字符串
    if ([string isKindOfClass:[NSMutableString class]]) {
        NSMutableString *mutableString = (NSMutableString *)string;
        
        // 用随机字符覆盖
        for (NSUInteger i = 0; i < mutableString.length; i++) {
            [mutableString replaceCharactersInRange:NSMakeRange(i, 1)
                                         withString:[NSString stringWithFormat:@"%c", arc4random() % 256]];
        }
        
        // 清空字符串
        [mutableString setString:@""];
    }
}

@end
```

## 总结

iOS安全编程是一个复杂而重要的主题，涉及多个层面的安全考虑：

### 核心安全原则

1. **数据保护**：使用强加密算法保护敏感数据，合理使用Keychain服务
2. **身份验证**：集成生物识别认证，提供多层身份验证机制
3. **网络安全**：实施SSL/TLS证书验证和证书固定，防范中间人攻击
4. **应用防护**：实现反调试和代码完整性检查，防范逆向工程
5. **安全开发**：遵循安全编程最佳实践，建立安全的开发流程

### 实施建议

1. **分层防护**：不要依赖单一的安全措施，建立多层防护体系
2. **持续监控**：实施运行时安全检查和异常监控
3. **定期更新**：及时更新安全策略和防护措施
4. **安全测试**：进行渗透测试和安全审计
5. **用户教育**：提高用户的安全意识

通过系统性地实施这些安全措施，可以显著提高iOS应用的安全性，保护用户数据和应用完整性。安全是一个持续的过程，需要在整个应用生命周期中不断关注和改进。

### 非对称加密实现

```objc
@interface AsymmetricEncryption : NSObject

+ (BOOL)generateKeyPairWithKeySize:(NSUInteger)keySize
                        publicKey:(SecKeyRef *)publicKey
                       privateKey:(SecKeyRef *)privateKey;
+ (NSData *)encryptData:(NSData *)data withPublicKey:(SecKeyRef)publicKey;
+ (NSData *)decryptData:(NSData *)encryptedData withPrivateKey:(SecKeyRef)privateKey;
+ (NSData *)signData:(NSData *)data withPrivateKey:(SecKeyRef)privateKey;
+ (BOOL)verifySignature:(NSData *)signature forData:(NSData *)data withPublicKey:(SecKeyRef)publicKey;
+ (NSData *)exportPublicKey:(SecKeyRef)publicKey;
+ (SecKeyRef)importPublicKeyFromData:(NSData *)keyData;

@end

@implementation AsymmetricEncryption

+ (BOOL)generateKeyPairWithKeySize:(NSUInteger)keySize
                        publicKey:(SecKeyRef *)publicKey
                       privateKey:(SecKeyRef *)privateKey {
    
    NSDictionary *keyPairAttributes = @{
        (__bridge id)kSecAttrKeyType: (__bridge id)kSecAttrKeyTypeRSA,
        (__bridge id)kSecAttrKeySizeInBits: @(keySize),
        (__bridge id)kSecPrivateKeyAttrs: @{
            (__bridge id)kSecAttrIsPermanent: @NO,
            (__bridge id)kSecAttrApplicationTag: [@"com.app.privatekey" dataUsingEncoding:NSUTF8StringEncoding]
        },
        (__bridge id)kSecPublicKeyAttrs: @{
            (__bridge id)kSecAttrIsPermanent: @NO,
            (__bridge id)kSecAttrApplicationTag: [@"com.app.publickey" dataUsingEncoding:NSUTF8StringEncoding]
        }
    };
    
    OSStatus status = SecKeyGeneratePair((__bridge CFDictionaryRef)keyPairAttributes, publicKey, privateKey);
    
    if (status != errSecSuccess) {
        NSLog(@"Key pair generation failed with status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

+ (NSData *)encryptData:(NSData *)data withPublicKey:(SecKeyRef)publicKey {
    if (!data || !publicKey) return nil;
    
    size_t blockSize = SecKeyGetBlockSize(publicKey);
    size_t maxPlaintextLength = blockSize - 11; // PKCS#1 padding
    
    if (data.length > maxPlaintextLength) {
        NSLog(@"Data too large for RSA encryption. Max size: %zu bytes", maxPlaintextLength);
        return nil;
    }
    
    NSMutableData *encryptedData = [NSMutableData dataWithLength:blockSize];
    size_t encryptedLength = blockSize;
    
    OSStatus status = SecKeyEncrypt(
        publicKey,
        kSecPaddingPKCS1,
        data.bytes,
        data.length,
        encryptedData.mutableBytes,
        &encryptedLength
    );
    
    if (status != errSecSuccess) {
        NSLog(@"Encryption failed with status: %d", (int)status);
        return nil;
    }
    
    return [NSData dataWithBytes:encryptedData.bytes length:encryptedLength];
}

+ (NSData *)decryptData:(NSData *)encryptedData withPrivateKey:(SecKeyRef)privateKey {
    if (!encryptedData || !privateKey) return nil;
    
    size_t blockSize = SecKeyGetBlockSize(privateKey);
    NSMutableData *decryptedData = [NSMutableData dataWithLength:blockSize];
    size_t decryptedLength = blockSize;
    
    OSStatus status = SecKeyDecrypt(
        privateKey,
        kSecPaddingPKCS1,
        encryptedData.bytes,
        encryptedData.length,
        decryptedData.mutableBytes,
        &decryptedLength
    );
    
    if (status != errSecSuccess) {
        NSLog(@"Decryption failed with status: %d", (int)status);
        return nil;
    }
    
    return [NSData dataWithBytes:decryptedData.bytes length:decryptedLength];
}

+ (NSData *)signData:(NSData *)data withPrivateKey:(SecKeyRef)privateKey {
    if (!data || !privateKey) return nil;
    
    // 计算SHA-256哈希
    NSMutableData *hash = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH];
    CC_SHA256(data.bytes, (CC_LONG)data.length, hash.mutableBytes);
    
    size_t blockSize = SecKeyGetBlockSize(privateKey);
    NSMutableData *signature = [NSMutableData dataWithLength:blockSize];
    size_t signatureLength = blockSize;
    
    OSStatus status = SecKeyRawSign(
        privateKey,
        kSecPaddingPKCS1SHA256,
        hash.bytes,
        hash.length,
        signature.mutableBytes,
        &signatureLength
    );
    
    if (status != errSecSuccess) {
        NSLog(@"Signing failed with status: %d", (int)status);
        return nil;
    }
    
    return [NSData dataWithBytes:signature.bytes length:signatureLength];
}

+ (BOOL)verifySignature:(NSData *)signature forData:(NSData *)data withPublicKey:(SecKeyRef)publicKey {
    if (!signature || !data || !publicKey) return NO;
    
    // 计算SHA-256哈希
    NSMutableData *hash = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH];
    CC_SHA256(data.bytes, (CC_LONG)data.length, hash.mutableBytes);
    
    OSStatus status = SecKeyRawVerify(
        publicKey,
        kSecPaddingPKCS1SHA256,
        hash.bytes,
        hash.length,
        signature.bytes,
        signature.length
    );
    
    return status == errSecSuccess;
}

+ (NSData *)exportPublicKey:(SecKeyRef)publicKey {
    if (!publicKey) return nil;
    
    CFDataRef keyData = NULL;
    OSStatus status = SecItemExport(publicKey, kSecFormatOpenSSL, kSecItemPemArmour, NULL, &keyData);
    
    if (status != errSecSuccess) {
        NSLog(@"Public key export failed with status: %d", (int)status);
        return nil;
    }
    
    NSData *result = (__bridge_transfer NSData *)keyData;
    return result;
}

+ (SecKeyRef)importPublicKeyFromData:(NSData *)keyData {
    if (!keyData) return NULL;
    
    SecExternalFormat format = kSecFormatOpenSSL;
    SecExternalItemType itemType = kSecItemTypePublicKey;
    
    CFArrayRef items = NULL;
    OSStatus status = SecItemImport(
        (__bridge CFDataRef)keyData,
        NULL,
        &format,
        &itemType,
        kSecItemPemArmour,
        NULL,
        NULL,
        &items
    );
    
    if (status != errSecSuccess || !items || CFArrayGetCount(items) == 0) {
        NSLog(@"Public key import failed with status: %d", (int)status);
        if (items) CFRelease(items);
        return NULL;
    }
    
    SecKeyRef publicKey = (SecKeyRef)CFArrayGetValueAtIndex(items, 0);
    CFRetain(publicKey);
    CFRelease(items);
    
    return publicKey;
}

@end
```

## Keychain服务管理

### Keychain访问封装

```objc
@interface KeychainManager : NSObject

+ (BOOL)storePassword:(NSString *)password forAccount:(NSString *)account service:(NSString *)service;
+ (NSString *)retrievePasswordForAccount:(NSString *)account service:(NSString *)service;
+ (BOOL)updatePassword:(NSString *)newPassword forAccount:(NSString *)account service:(NSString *)service;
+ (BOOL)deletePasswordForAccount:(NSString *)account service:(NSString *)service;
+ (BOOL)storeData:(NSData *)data forKey:(NSString *)key accessGroup:(NSString *)accessGroup;
+ (NSData *)retrieveDataForKey:(NSString *)key accessGroup:(NSString *)accessGroup;
+ (NSArray<NSDictionary *> *)allKeychainItems;
+ (BOOL)clearAllKeychainItems;

@end

@implementation KeychainManager

+ (BOOL)storePassword:(NSString *)password forAccount:(NSString *)account service:(NSString *)service {
    if (!password || !account || !service) return NO;
    
    NSData *passwordData = [password dataUsingEncoding:NSUTF8StringEncoding];
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: account,
        (__bridge id)kSecAttrService: service,
        (__bridge id)kSecValueData: passwordData,
        (__bridge id)kSecAttrAccessible: (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    };
    
    // 首先尝试删除现有项目
    [self deletePasswordForAccount:account service:service];
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to store password. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

+ (NSString *)retrievePasswordForAccount:(NSString *)account service:(NSString *)service {
    if (!account || !service) return nil;
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: account,
        (__bridge id)kSecAttrService: service,
        (__bridge id)kSecReturnData: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    };
    
    CFDataRef passwordData = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, (CFTypeRef *)&passwordData);
    
    if (status != errSecSuccess) {
        if (status != errSecItemNotFound) {
            NSLog(@"Failed to retrieve password. Status: %d", (int)status);
        }
        return nil;
    }
    
    NSString *password = [[NSString alloc] initWithData:(__bridge_transfer NSData *)passwordData
                                               encoding:NSUTF8StringEncoding];
    return password;
}

+ (BOOL)updatePassword:(NSString *)newPassword forAccount:(NSString *)account service:(NSString *)service {
    if (!newPassword || !account || !service) return NO;
    
    NSData *passwordData = [newPassword dataUsingEncoding:NSUTF8StringEncoding];
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: account,
        (__bridge id)kSecAttrService: service
    };
    
    NSDictionary *attributesToUpdate = @{
        (__bridge id)kSecValueData: passwordData
    };
    
    OSStatus status = SecItemUpdate((__bridge CFDictionaryRef)query,
                                   (__bridge CFDictionaryRef)attributesToUpdate);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to update password. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

+ (BOOL)deletePasswordForAccount:(NSString *)account service:(NSString *)service {
    if (!account || !service) return NO;
    
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: account,
        (__bridge id)kSecAttrService: service
    };
    
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    
    if (status != errSecSuccess && status != errSecItemNotFound) {
        NSLog(@"Failed to delete password. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

+ (BOOL)storeData:(NSData *)data forKey:(NSString *)key accessGroup:(NSString *)accessGroup {
    if (!data || !key) return NO;
    
    NSMutableDictionary *query = [@{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"SecureDataStorage",
        (__bridge id)kSecValueData: data,
        (__bridge id)kSecAttrAccessible: (__bridge id)kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    } mutableCopy];
    
    if (accessGroup) {
        query[(__bridge id)kSecAttrAccessGroup] = accessGroup;
    }
    
    // 删除现有项目
    [self deleteDataForKey:key accessGroup:accessGroup];
    
    OSStatus status = SecItemAdd((__bridge CFDictionaryRef)query, NULL);
    
    if (status != errSecSuccess) {
        NSLog(@"Failed to store data. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

+ (NSData *)retrieveDataForKey:(NSString *)key accessGroup:(NSString *)accessGroup {
    if (!key) return nil;
    
    NSMutableDictionary *query = [@{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"SecureDataStorage",
        (__bridge id)kSecReturnData: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitOne
    } mutableCopy];
    
    if (accessGroup) {
        query[(__bridge id)kSecAttrAccessGroup] = accessGroup;
    }
    
    CFDataRef data = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, (CFTypeRef *)&data);
    
    if (status != errSecSuccess) {
        if (status != errSecItemNotFound) {
            NSLog(@"Failed to retrieve data. Status: %d", (int)status);
        }
        return nil;
    }
    
    return (__bridge_transfer NSData *)data;
}

+ (BOOL)deleteDataForKey:(NSString *)key accessGroup:(NSString *)accessGroup {
    if (!key) return NO;
    
    NSMutableDictionary *query = [@{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecAttrAccount: key,
        (__bridge id)kSecAttrService: @"SecureDataStorage"
    } mutableCopy];
    
    if (accessGroup) {
        query[(__bridge id)kSecAttrAccessGroup] = accessGroup;
    }
    
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    
    if (status != errSecSuccess && status != errSecItemNotFound) {
        NSLog(@"Failed to delete data. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

+ (NSArray<NSDictionary *> *)allKeychainItems {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
        (__bridge id)kSecReturnAttributes: @YES,
        (__bridge id)kSecMatchLimit: (__bridge id)kSecMatchLimitAll
    };
    
    CFArrayRef items = NULL;
    OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, (CFTypeRef *)&items);
    
    if (status != errSecSuccess) {
        if (status != errSecItemNotFound) {
            NSLog(@"Failed to retrieve keychain items. Status: %d", (int)status);
        }
        return @[];
    }
    
    return (__bridge_transfer NSArray *)items;
}

+ (BOOL)clearAllKeychainItems {
    NSDictionary *query = @{
        (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword
    };
    
    OSStatus status = SecItemDelete((__bridge CFDictionaryRef)query);
    
    if (status != errSecSuccess && status != errSecItemNotFound) {
        NSLog(@"Failed to clear keychain items. Status: %d", (int)status);
        return NO;
    }
    
    return YES;
}

@end
```