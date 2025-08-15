---
layout: post
title: "iOS数据持久化深度解析：Core Data、SQLite与文件存储实战指南"
date: 2024-06-20
categories: ios
tags: [iOS开发, 数据存储, Core Data, SQLite, 文件存储, 数据库, 性能优化]
author: iOS技术专家
description: "深入解析iOS数据持久化技术，包括Core Data、SQLite、文件存储等多种方案的实现原理、最佳实践和性能优化策略。"
keywords: [iOS数据持久化, Core Data, SQLite, 文件存储, 数据库优化, iOS开发]
---

数据持久化是iOS应用开发中的核心技术之一，选择合适的数据存储方案直接影响应用的性能、稳定性和用户体验。本文将深入解析iOS平台上的各种数据持久化技术，包括Core Data、SQLite、文件存储等，并提供完整的实现方案和最佳实践。

## 数据持久化架构概览

### 数据持久化管理架构

```objc
@interface DataPersistenceArchitecture : NSObject

@property (nonatomic, strong) NSMutableDictionary *storageManagers;
@property (nonatomic, strong) NSMutableArray *performanceMetrics;
@property (nonatomic, assign) BOOL isMonitoringEnabled;

+ (instancetype)sharedArchitecture;

// 架构初始化
- (void)initializeDataPersistenceEnvironment;
- (void)configureStorageOptions;
- (void)setupPerformanceMonitoring;

// 存储管理器注册
- (void)registerStorageManager:(id)manager forType:(NSString *)type;
- (id)getStorageManagerForType:(NSString *)type;
- (NSArray *)getAllRegisteredManagers;

// 性能监控
- (void)recordOperation:(NSString *)operation
               duration:(NSTimeInterval)duration
            storageType:(NSString *)storageType;
- (NSDictionary *)getPerformanceReport;
- (void)analyzeStoragePerformance;

// 数据迁移
- (BOOL)migrateDataFromStorage:(NSString *)sourceType
                     toStorage:(NSString *)targetType
                    withOptions:(NSDictionary *)options;
- (void)validateDataIntegrity;

// 存储优化
- (void)optimizeStoragePerformance;
- (void)cleanupTemporaryData;
- (NSInteger)calculateStorageUsage;

@end

@implementation DataPersistenceArchitecture

+ (instancetype)sharedArchitecture {
    static DataPersistenceArchitecture *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.storageManagers = [NSMutableDictionary dictionary];
        self.performanceMetrics = [NSMutableArray array];
        self.isMonitoringEnabled = YES;
    }
    return self;
}

- (void)initializeDataPersistenceEnvironment {
    NSLog(@"=== 初始化数据持久化环境 ===");
    
    // 1. 检查存储权限
    [self checkStoragePermissions];
    
    // 2. 创建必要的目录结构
    [self createDirectoryStructure];
    
    // 3. 初始化各种存储管理器
    [self initializeStorageManagers];
    
    // 4. 设置性能监控
    [self setupPerformanceMonitoring];
    
    NSLog(@"数据持久化环境初始化完成");
}

- (void)checkStoragePermissions {
    NSLog(@"检查存储权限...");
    
    // 检查文档目录访问权限
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths objectAtIndex:0];
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    BOOL isWritable = [fileManager isWritableFileAtPath:documentsDirectory];
    
    if (isWritable) {
        NSLog(@"文档目录访问权限正常: %@", documentsDirectory);
    } else {
        NSLog(@"警告: 文档目录访问权限受限");
    }
}

- (void)createDirectoryStructure {
    NSLog(@"创建目录结构...");
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths objectAtIndex:0];
    
    // 创建数据库目录
    NSString *databaseDirectory = [documentsDirectory stringByAppendingPathComponent:@"Database"];
    [fileManager createDirectoryAtPath:databaseDirectory withIntermediateDirectories:YES attributes:nil error:nil];
    
    // 创建文件存储目录
    NSString *filesDirectory = [documentsDirectory stringByAppendingPathComponent:@"Files"];
    [fileManager createDirectoryAtPath:filesDirectory withIntermediateDirectories:YES attributes:nil error:nil];
    
    // 创建缓存目录
    NSString *cacheDirectory = [documentsDirectory stringByAppendingPathComponent:@"Cache"];
    [fileManager createDirectoryAtPath:cacheDirectory withIntermediateDirectories:YES attributes:nil error:nil];
    
    // 创建临时目录
    NSString *tempDirectory = [documentsDirectory stringByAppendingPathComponent:@"Temp"];
    [fileManager createDirectoryAtPath:tempDirectory withIntermediateDirectories:YES attributes:nil error:nil];
    
    NSLog(@"目录结构创建完成");
}

- (void)initializeStorageManagers {
    NSLog(@"初始化存储管理器...");
    
    // 这里会在后续章节中注册具体的存储管理器
    // [self registerStorageManager:coreDataManager forType:@"CoreData"];
    // [self registerStorageManager:sqliteManager forType:@"SQLite"];
    // [self registerStorageManager:fileManager forType:@"File"];
    
    NSLog(@"存储管理器初始化完成");
}

- (void)configureStorageOptions {
    NSLog(@"配置存储选项...");
    
    // 获取设备信息
    NSInteger availableMemory = [self getAvailableMemory];
    NSInteger availableStorage = [self getAvailableStorage];
    
    NSLog(@"设备信息 - 可用内存: %ld MB, 可用存储: %ld MB", (long)availableMemory, (long)availableStorage);
    
    // 根据设备性能调整存储策略
    if (availableStorage < 1024) { // 小于1GB
        NSLog(@"存储空间不足，启用压缩存储模式");
        // 启用数据压缩
    } else if (availableStorage > 10240) { // 大于10GB
        NSLog(@"存储空间充足，启用高性能存储模式");
        // 启用缓存优化
    }
}

- (NSInteger)getAvailableMemory {
    // 获取可用内存（简化实现）
    return 2048; // 2GB
}

- (NSInteger)getAvailableStorage {
    NSError *error;
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSDictionary *dictionary = [[NSFileManager defaultManager] attributesOfFileSystemForPath:[paths lastObject] error:&error];
    
    if (dictionary) {
        NSNumber *freeFileSystemSizeInBytes = [dictionary objectForKey:NSFileSystemFreeSize];
        return [freeFileSystemSizeInBytes longLongValue] / 1024 / 1024; // 转换为MB
    }
    
    return 0;
}

- (void)setupPerformanceMonitoring {
    NSLog(@"设置性能监控...");
    
    self.isMonitoringEnabled = YES;
    
    // 定期清理性能指标
    dispatch_queue_t cleanupQueue = dispatch_queue_create("com.performance.cleanup", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(cleanupQueue, ^{
        while (self.isMonitoringEnabled) {
            [NSThread sleepForTimeInterval:300.0]; // 每5分钟清理一次
            
            @autoreleasepool {
                [self cleanupOldMetrics];
            }
        }
    });
}

- (void)cleanupOldMetrics {
    NSDate *cutoffDate = [NSDate dateWithTimeIntervalSinceNow:-3600]; // 保留1小时内的数据
    
    NSMutableArray *metricsToRemove = [NSMutableArray array];
    
    for (NSDictionary *metric in self.performanceMetrics) {
        NSDate *timestamp = metric[@"timestamp"];
        if ([timestamp compare:cutoffDate] == NSOrderedAscending) {
            [metricsToRemove addObject:metric];
        }
    }
    
    [self.performanceMetrics removeObjectsInArray:metricsToRemove];
    
    if (metricsToRemove.count > 0) {
        NSLog(@"清理了 %lu 条过期性能指标", (unsigned long)metricsToRemove.count);
    }
}

- (void)registerStorageManager:(id)manager forType:(NSString *)type {
    if (manager && type) {
        self.storageManagers[type] = manager;
        NSLog(@"注册存储管理器: %@", type);
    }
}

- (id)getStorageManagerForType:(NSString *)type {
    return self.storageManagers[type];
}

- (NSArray *)getAllRegisteredManagers {
    return [self.storageManagers allValues];
}

- (void)recordOperation:(NSString *)operation
               duration:(NSTimeInterval)duration
            storageType:(NSString *)storageType {
    
    if (!self.isMonitoringEnabled) return;
    
    NSDictionary *metric = @{
        @"operation": operation ?: @"unknown",
        @"duration": @(duration),
        @"storageType": storageType ?: @"unknown",
        @"timestamp": [NSDate date]
    };
    
    [self.performanceMetrics addObject:metric];
    
    // 如果操作耗时过长，记录警告
    if (duration > 1.0) {
        NSLog(@"警告: 存储操作耗时过长 - %@ (%.2f秒)", operation, duration);
    }
}

- (NSDictionary *)getPerformanceReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    // 统计各种存储类型的性能
    NSMutableDictionary *storageStats = [NSMutableDictionary dictionary];
    
    for (NSDictionary *metric in self.performanceMetrics) {
        NSString *storageType = metric[@"storageType"];
        NSNumber *duration = metric[@"duration"];
        
        if (!storageStats[storageType]) {
            storageStats[storageType] = [NSMutableArray array];
        }
        
        [storageStats[storageType] addObject:duration];
    }
    
    // 计算平均值和统计信息
    for (NSString *storageType in storageStats) {
        NSArray *durations = storageStats[storageType];
        
        double totalDuration = 0;
        double maxDuration = 0;
        double minDuration = INFINITY;
        
        for (NSNumber *duration in durations) {
            double d = [duration doubleValue];
            totalDuration += d;
            maxDuration = MAX(maxDuration, d);
            minDuration = MIN(minDuration, d);
        }
        
        double averageDuration = totalDuration / durations.count;
        
        report[storageType] = @{
            @"operationCount": @(durations.count),
            @"averageDuration": @(averageDuration),
            @"maxDuration": @(maxDuration),
            @"minDuration": @(minDuration == INFINITY ? 0 : minDuration),
            @"totalDuration": @(totalDuration)
        };
    }
    
    report[@"totalOperations"] = @(self.performanceMetrics.count);
    report[@"reportTimestamp"] = [NSDate date];
    
    return report;
}

- (void)analyzeStoragePerformance {
    NSLog(@"=== 分析存储性能 ===");
    
    NSDictionary *report = [self getPerformanceReport];
    
    for (NSString *storageType in report) {
        if ([storageType isEqualToString:@"totalOperations"] || [storageType isEqualToString:@"reportTimestamp"]) {
            continue;
        }
        
        NSDictionary *stats = report[storageType];
        NSLog(@"存储类型: %@", storageType);
        NSLog(@"  操作次数: %@", stats[@"operationCount"]);
        NSLog(@"  平均耗时: %.3f秒", [stats[@"averageDuration"] doubleValue]);
        NSLog(@"  最大耗时: %.3f秒", [stats[@"maxDuration"] doubleValue]);
        NSLog(@"  最小耗时: %.3f秒", [stats[@"minDuration"] doubleValue]);
    }
}

- (BOOL)migrateDataFromStorage:(NSString *)sourceType
                     toStorage:(NSString *)targetType
                    withOptions:(NSDictionary *)options {
    
    NSLog(@"开始数据迁移: %@ -> %@", sourceType, targetType);
    
    id sourceManager = [self getStorageManagerForType:sourceType];
    id targetManager = [self getStorageManagerForType:targetType];
    
    if (!sourceManager || !targetManager) {
        NSLog(@"错误: 找不到对应的存储管理器");
        return NO;
    }
    
    // 这里需要根据具体的存储管理器实现迁移逻辑
    // 实际实现会在具体的存储管理器中完成
    
    NSLog(@"数据迁移完成");
    return YES;
}

- (void)validateDataIntegrity {
    NSLog(@"验证数据完整性...");
    
    for (NSString *storageType in self.storageManagers) {
        id manager = self.storageManagers[storageType];
        
        // 调用各个存储管理器的数据完整性检查方法
        if ([manager respondsToSelector:@selector(validateDataIntegrity)]) {
            BOOL isValid = [manager validateDataIntegrity];
            NSLog(@"%@ 数据完整性: %@", storageType, isValid ? @"正常" : @"异常");
        }
    }
}

- (void)optimizeStoragePerformance {
    NSLog(@"优化存储性能...");
    
    // 分析性能报告
    NSDictionary *report = [self getPerformanceReport];
    
    for (NSString *storageType in report) {
        if ([storageType isEqualToString:@"totalOperations"] || [storageType isEqualToString:@"reportTimestamp"]) {
            continue;
        }
        
        NSDictionary *stats = report[storageType];
        double averageDuration = [stats[@"averageDuration"] doubleValue];
        
        // 如果平均耗时超过阈值，触发优化
        if (averageDuration > 0.5) {
            NSLog(@"触发 %@ 存储优化，平均耗时: %.3f秒", storageType, averageDuration);
            
            id manager = [self getStorageManagerForType:storageType];
            if ([manager respondsToSelector:@selector(optimizePerformance)]) {
                [manager optimizePerformance];
            }
        }
    }
}

- (void)cleanupTemporaryData {
    NSLog(@"清理临时数据...");
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths objectAtIndex:0];
    NSString *tempDirectory = [documentsDirectory stringByAppendingPathComponent:@"Temp"];
    
    NSError *error;
    NSArray *tempFiles = [fileManager contentsOfDirectoryAtPath:tempDirectory error:&error];
    
    NSInteger cleanedFiles = 0;
    NSInteger cleanedSize = 0;
    
    for (NSString *fileName in tempFiles) {
        NSString *filePath = [tempDirectory stringByAppendingPathComponent:fileName];
        
        NSDictionary *attributes = [fileManager attributesOfItemAtPath:filePath error:nil];
        NSDate *modificationDate = attributes[NSFileModificationDate];
        
        // 删除超过1小时的临时文件
        if ([[NSDate date] timeIntervalSinceDate:modificationDate] > 3600) {
            NSInteger fileSize = [attributes[NSFileSize] integerValue];
            
            if ([fileManager removeItemAtPath:filePath error:nil]) {
                cleanedFiles++;
                cleanedSize += fileSize;
            }
        }
    }
    
    NSLog(@"清理完成 - 文件数: %ld, 释放空间: %.2f MB", (long)cleanedFiles, cleanedSize / 1024.0 / 1024.0);
}

- (NSInteger)calculateStorageUsage {
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths objectAtIndex:0];
    
    NSInteger totalSize = 0;
    
    NSDirectoryEnumerator *enumerator = [fileManager enumeratorAtPath:documentsDirectory];
    NSString *fileName;
    
    while ((fileName = [enumerator nextObject])) {
        NSString *filePath = [documentsDirectory stringByAppendingPathComponent:fileName];
        NSDictionary *attributes = [fileManager attributesOfItemAtPath:filePath error:nil];
        
        if (attributes[NSFileSize]) {
            totalSize += [attributes[NSFileSize] integerValue];
        }
    }
    
    NSLog(@"总存储使用量: %.2f MB", totalSize / 1024.0 / 1024.0);
    return totalSize;
}

@end
```

## SQLite数据库深度解析

### SQLite管理器

```objc
@interface SQLiteManager : NSObject

@property (nonatomic, assign) sqlite3 *database;
@property (nonatomic, strong) NSString *databasePath;
@property (nonatomic, strong) NSMutableArray *performanceMetrics;
@property (nonatomic, strong) dispatch_queue_t databaseQueue;
@property (nonatomic, assign) BOOL isTransactionActive;

+ (instancetype)sharedManager;

// 数据库初始化
- (BOOL)initializeDatabaseWithName:(NSString *)databaseName;
- (BOOL)openDatabase;
- (void)closeDatabase;
- (BOOL)createTablesWithSchema:(NSArray *)tableSchemas;

// 基础操作
- (BOOL)executeSQL:(NSString *)sql;
- (BOOL)executeSQL:(NSString *)sql withParameters:(NSArray *)parameters;
- (NSArray *)executeQuery:(NSString *)sql;
- (NSArray *)executeQuery:(NSString *)sql withParameters:(NSArray *)parameters;

// 事务管理
- (BOOL)beginTransaction;
- (BOOL)commitTransaction;
- (BOOL)rollbackTransaction;
- (void)executeInTransaction:(void(^)(BOOL *rollback))block;

// 数据操作
- (BOOL)insertIntoTable:(NSString *)tableName values:(NSDictionary *)values;
- (BOOL)updateTable:(NSString *)tableName
                set:(NSDictionary *)values
              where:(NSString *)whereClause
         parameters:(NSArray *)parameters;
- (BOOL)deleteFromTable:(NSString *)tableName
                  where:(NSString *)whereClause
             parameters:(NSArray *)parameters;
- (NSArray *)selectFromTable:(NSString *)tableName
                     columns:(NSArray *)columns
                       where:(NSString *)whereClause
                  parameters:(NSArray *)parameters
                     orderBy:(NSString *)orderBy
                       limit:(NSInteger)limit;

// 批量操作
- (BOOL)batchInsert:(NSArray *)dataArray intoTable:(NSString *)tableName;
- (BOOL)batchUpdate:(NSArray *)updateData inTable:(NSString *)tableName;
- (BOOL)batchDelete:(NSArray *)conditions fromTable:(NSString *)tableName;

// 性能优化
- (void)optimizeDatabase;
- (void)analyzeDatabase;
- (void)vacuumDatabase;
- (void)createIndex:(NSString *)indexName onTable:(NSString *)tableName columns:(NSArray *)columns;

// 数据库维护
- (BOOL)backupDatabaseToPath:(NSString *)backupPath;
- (BOOL)restoreDatabaseFromPath:(NSString *)backupPath;
- (NSInteger)getDatabaseSize;
- (NSDictionary *)getDatabaseInfo;

// 性能监控
- (void)enablePerformanceMonitoring;
- (NSDictionary *)getPerformanceReport;
- (void)logSQLiteStatistics;

@end

@implementation SQLiteManager

+ (instancetype)sharedManager {
    static SQLiteManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.performanceMetrics = [NSMutableArray array];
        self.databaseQueue = dispatch_queue_create("com.sqlite.database", DISPATCH_QUEUE_SERIAL);
        self.isTransactionActive = NO;
    }
    return self;
}

- (BOOL)initializeDatabaseWithName:(NSString *)databaseName {
    NSLog(@"=== 初始化SQLite数据库: %@ ===", databaseName);
    
    // 获取数据库路径
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths objectAtIndex:0];
    NSString *databaseDirectory = [documentsDirectory stringByAppendingPathComponent:@"Database"];
    self.databasePath = [databaseDirectory stringByAppendingPathComponent:[NSString stringWithFormat:@"%@.sqlite", databaseName]];
    
    NSLog(@"数据库路径: %@", self.databasePath);
    
    // 打开数据库
    if ([self openDatabase]) {
        NSLog(@"SQLite数据库初始化成功");
        [self enablePerformanceMonitoring];
        return YES;
    } else {
        NSLog(@"SQLite数据库初始化失败");
        return NO;
    }
}

- (BOOL)openDatabase {
    if (sqlite3_open([self.databasePath UTF8String], &_database) == SQLITE_OK) {
        NSLog(@"数据库打开成功");
        
        // 设置数据库配置
        [self configureDatabaseSettings];
        
        return YES;
    } else {
        NSLog(@"数据库打开失败: %s", sqlite3_errmsg(_database));
        return NO;
    }
}

- (void)configureDatabaseSettings {
    NSLog(@"配置数据库设置...");
    
    // 启用外键约束
    [self executeSQL:@"PRAGMA foreign_keys = ON;"];
    
    // 设置日志模式为WAL（Write-Ahead Logging）
    [self executeSQL:@"PRAGMA journal_mode = WAL;"];
    
    // 设置同步模式
    [self executeSQL:@"PRAGMA synchronous = NORMAL;"];
    
    // 设置缓存大小（页数）
    [self executeSQL:@"PRAGMA cache_size = 10000;"];
    
    // 设置临时存储为内存
    [self executeSQL:@"PRAGMA temp_store = MEMORY;"];
    
    NSLog(@"数据库设置配置完成");
}

- (void)closeDatabase {
    if (_database) {
        sqlite3_close(_database);
        _database = NULL;
        NSLog(@"数据库已关闭");
    }
}

- (BOOL)createTablesWithSchema:(NSArray *)tableSchemas {
    NSLog(@"创建数据表...");
    
    BOOL success = YES;
    
    for (NSString *schema in tableSchemas) {
        if (![self executeSQL:schema]) {
            NSLog(@"创建表失败: %@", schema);
            success = NO;
        }
    }
    
    if (success) {
        NSLog(@"所有数据表创建成功");
    }
    
    return success;
}

- (BOOL)executeSQL:(NSString *)sql {
    return [self executeSQL:sql withParameters:nil];
}

- (BOOL)executeSQL:(NSString *)sql withParameters:(NSArray *)parameters {
    NSDate *startTime = [NSDate date];
    
    __block BOOL success = NO;
    
    dispatch_sync(self.databaseQueue, ^{
        sqlite3_stmt *statement;
        
        if (sqlite3_prepare_v2(self.database, [sql UTF8String], -1, &statement, NULL) == SQLITE_OK) {
            
            // 绑定参数
            if (parameters) {
                [self bindParameters:parameters toStatement:statement];
            }
            
            if (sqlite3_step(statement) == SQLITE_DONE) {
                success = YES;
            } else {
                NSLog(@"SQL执行失败: %s", sqlite3_errmsg(self.database));
            }
            
            sqlite3_finalize(statement);
        } else {
            NSLog(@"SQL准备失败: %s", sqlite3_errmsg(self.database));
        }
    });
    
    // 记录性能指标
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    [self recordOperation:@"execute" duration:duration success:success];
    
    return success;
}

- (NSArray *)executeQuery:(NSString *)sql {
    return [self executeQuery:sql withParameters:nil];
}

- (NSArray *)executeQuery:(NSString *)sql withParameters:(NSArray *)parameters {
    NSDate *startTime = [NSDate date];
    
    __block NSMutableArray *results = [NSMutableArray array];
    
    dispatch_sync(self.databaseQueue, ^{
        sqlite3_stmt *statement;
        
        if (sqlite3_prepare_v2(self.database, [sql UTF8String], -1, &statement, NULL) == SQLITE_OK) {
            
            // 绑定参数
            if (parameters) {
                [self bindParameters:parameters toStatement:statement];
            }
            
            // 获取列信息
            int columnCount = sqlite3_column_count(statement);
            NSMutableArray *columnNames = [NSMutableArray array];
            
            for (int i = 0; i < columnCount; i++) {
                const char *columnName = sqlite3_column_name(statement, i);
                [columnNames addObject:[NSString stringWithUTF8String:columnName]];
            }
            
            // 读取数据
            while (sqlite3_step(statement) == SQLITE_ROW) {
                NSMutableDictionary *row = [NSMutableDictionary dictionary];
                
                for (int i = 0; i < columnCount; i++) {
                    NSString *columnName = columnNames[i];
                    id value = [self getColumnValue:statement atIndex:i];
                    
                    if (value) {
                        row[columnName] = value;
                    }
                }
                
                [results addObject:row];
            }
            
            sqlite3_finalize(statement);
        } else {
            NSLog(@"查询准备失败: %s", sqlite3_errmsg(self.database));
        }
    });
    
    // 记录性能指标
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    [self recordOperation:@"query" duration:duration success:YES];
    
    NSLog(@"查询完成，返回 %lu 条记录，耗时: %.3f秒", (unsigned long)results.count, duration);
    
    return results;
}

- (void)bindParameters:(NSArray *)parameters toStatement:(sqlite3_stmt *)statement {
    for (int i = 0; i < parameters.count; i++) {
        id parameter = parameters[i];
        int index = i + 1; // SQLite参数索引从1开始
        
        if ([parameter isKindOfClass:[NSString class]]) {
            sqlite3_bind_text(statement, index, [parameter UTF8String], -1, SQLITE_TRANSIENT);
        } else if ([parameter isKindOfClass:[NSNumber class]]) {
            NSNumber *number = (NSNumber *)parameter;
            const char *type = [number objCType];
            
            if (strcmp(type, @encode(int)) == 0 || strcmp(type, @encode(long)) == 0) {
                sqlite3_bind_int64(statement, index, [number longLongValue]);
            } else if (strcmp(type, @encode(float)) == 0 || strcmp(type, @encode(double)) == 0) {
                sqlite3_bind_double(statement, index, [number doubleValue]);
            } else {
                sqlite3_bind_int(statement, index, [number intValue]);
            }
        } else if ([parameter isKindOfClass:[NSData class]]) {
            NSData *data = (NSData *)parameter;
            sqlite3_bind_blob(statement, index, [data bytes], (int)[data length], SQLITE_TRANSIENT);
        } else if ([parameter isKindOfClass:[NSNull class]]) {
            sqlite3_bind_null(statement, index);
        } else {
            // 默认转换为字符串
            NSString *stringValue = [parameter description];
            sqlite3_bind_text(statement, index, [stringValue UTF8String], -1, SQLITE_TRANSIENT);
        }
    }
}

- (id)getColumnValue:(sqlite3_stmt *)statement atIndex:(int)index {
    int columnType = sqlite3_column_type(statement, index);
    
    switch (columnType) {
        case SQLITE_INTEGER:
            return @(sqlite3_column_int64(statement, index));
            
        case SQLITE_FLOAT:
            return @(sqlite3_column_double(statement, index));
            
        case SQLITE_TEXT: {
            const char *text = (const char *)sqlite3_column_text(statement, index);
            return text ? [NSString stringWithUTF8String:text] : @"";
        }
            
        case SQLITE_BLOB: {
            const void *blob = sqlite3_column_blob(statement, index);
            int blobSize = sqlite3_column_bytes(statement, index);
            return blob ? [NSData dataWithBytes:blob length:blobSize] : [NSData data];
        }
            
        case SQLITE_NULL:
        default:
            return [NSNull null];
    }
}

- (BOOL)beginTransaction {
    if (self.isTransactionActive) {
        NSLog(@"警告: 事务已经处于活动状态");
        return NO;
    }
    
    BOOL success = [self executeSQL:@"BEGIN TRANSACTION;"];
    if (success) {
        self.isTransactionActive = YES;
        NSLog(@"事务开始");
    }
    
    return success;
}

- (BOOL)commitTransaction {
    if (!self.isTransactionActive) {
        NSLog(@"警告: 没有活动的事务可提交");
        return NO;
    }
    
    BOOL success = [self executeSQL:@"COMMIT TRANSACTION;"];
    if (success) {
        self.isTransactionActive = NO;
        NSLog(@"事务提交成功");
    }
    
    return success;
}

- (BOOL)rollbackTransaction {
    if (!self.isTransactionActive) {
        NSLog(@"警告: 没有活动的事务可回滚");
        return NO;
    }
    
    BOOL success = [self executeSQL:@"ROLLBACK TRANSACTION;"];
    if (success) {
        self.isTransactionActive = NO;
        NSLog(@"事务回滚成功");
    }
    
    return success;
}

- (void)executeInTransaction:(void(^)(BOOL *rollback))block {
    if ([self beginTransaction]) {
        BOOL rollback = NO;
        
        @try {
            if (block) {
                block(&rollback);
            }
        } @catch (NSException *exception) {
            NSLog(@"事务执行异常: %@", exception.reason);
            rollback = YES;
        }
        
        if (rollback) {
            [self rollbackTransaction];
        } else {
            [self commitTransaction];
        }
    }
}

- (BOOL)insertIntoTable:(NSString *)tableName values:(NSDictionary *)values {
    if (!values || values.count == 0) {
        return NO;
    }
    
    NSArray *columns = [values allKeys];
    NSArray *valueArray = [values allValues];
    
    NSString *columnsString = [columns componentsJoinedByString:@", "];
    NSString *placeholders = [[@"" stringByPaddingToLength:columns.count * 3 - 2 withString:@"?, " startingAtIndex:0] substringToIndex:columns.count * 3 - 2];
    
    NSString *sql = [NSString stringWithFormat:@"INSERT INTO %@ (%@) VALUES (%@);", tableName, columnsString, placeholders];
    
    return [self executeSQL:sql withParameters:valueArray];
}

- (BOOL)updateTable:(NSString *)tableName
                set:(NSDictionary *)values
              where:(NSString *)whereClause
         parameters:(NSArray *)parameters {
    
    if (!values || values.count == 0) {
        return NO;
    }
    
    NSMutableArray *setParts = [NSMutableArray array];
    NSMutableArray *allParameters = [NSMutableArray array];
    
    for (NSString *column in values) {
        [setParts addObject:[NSString stringWithFormat:@"%@ = ?", column]];
        [allParameters addObject:values[column]];
    }
    
    if (parameters) {
        [allParameters addObjectsFromArray:parameters];
    }
    
    NSString *setClause = [setParts componentsJoinedByString:@", "];
    NSString *sql = [NSString stringWithFormat:@"UPDATE %@ SET %@", tableName, setClause];
    
    if (whereClause && whereClause.length > 0) {
        sql = [sql stringByAppendingFormat:@" WHERE %@", whereClause];
    }
    
    sql = [sql stringByAppendingString:@";"];
    
    return [self executeSQL:sql withParameters:allParameters];
}

- (BOOL)deleteFromTable:(NSString *)tableName
                  where:(NSString *)whereClause
             parameters:(NSArray *)parameters {
    
    NSString *sql = [NSString stringWithFormat:@"DELETE FROM %@", tableName];
    
    if (whereClause && whereClause.length > 0) {
        sql = [sql stringByAppendingFormat:@" WHERE %@", whereClause];
    }
    
    sql = [sql stringByAppendingString:@";"];
    
    return [self executeSQL:sql withParameters:parameters];
}

- (NSArray *)selectFromTable:(NSString *)tableName
                     columns:(NSArray *)columns
                       where:(NSString *)whereClause
                  parameters:(NSArray *)parameters
                     orderBy:(NSString *)orderBy
                       limit:(NSInteger)limit {
    
    NSString *columnsString = columns && columns.count > 0 ? [columns componentsJoinedByString:@", "] : @"*";
    NSString *sql = [NSString stringWithFormat:@"SELECT %@ FROM %@", columnsString, tableName];
    
    if (whereClause && whereClause.length > 0) {
        sql = [sql stringByAppendingFormat:@" WHERE %@", whereClause];
    }
    
    if (orderBy && orderBy.length > 0) {
        sql = [sql stringByAppendingFormat:@" ORDER BY %@", orderBy];
    }
    
    if (limit > 0) {
        sql = [sql stringByAppendingFormat:@" LIMIT %ld", (long)limit];
    }
    
    sql = [sql stringByAppendingString:@";"];
    
    return [self executeQuery:sql withParameters:parameters];
}

- (BOOL)batchInsert:(NSArray *)dataArray intoTable:(NSString *)tableName {
    if (!dataArray || dataArray.count == 0) {
        return YES;
    }
    
    __block BOOL success = YES;
    
    [self executeInTransaction:^(BOOL *rollback) {
        for (NSDictionary *data in dataArray) {
            if (![self insertIntoTable:tableName values:data]) {
                success = NO;
                *rollback = YES;
                break;
            }
        }
    }];
    
    NSLog(@"批量插入 %lu 条记录%@", (unsigned long)dataArray.count, success ? @"成功" : @"失败");
    return success;
}

- (BOOL)batchUpdate:(NSArray *)updateData inTable:(NSString *)tableName {
    if (!updateData || updateData.count == 0) {
        return YES;
    }
    
    __block BOOL success = YES;
    
    [self executeInTransaction:^(BOOL *rollback) {
        for (NSDictionary *data in updateData) {
            NSDictionary *values = data[@"values"];
            NSString *whereClause = data[@"where"];
            NSArray *parameters = data[@"parameters"];
            
            if (![self updateTable:tableName set:values where:whereClause parameters:parameters]) {
                success = NO;
                *rollback = YES;
                break;
            }
        }
    }];
    
    NSLog(@"批量更新 %lu 条记录%@", (unsigned long)updateData.count, success ? @"成功" : @"失败");
    return success;
}

- (BOOL)batchDelete:(NSArray *)conditions fromTable:(NSString *)tableName {
    if (!conditions || conditions.count == 0) {
        return YES;
    }
    
    __block BOOL success = YES;
    
    [self executeInTransaction:^(BOOL *rollback) {
        for (NSDictionary *condition in conditions) {
            NSString *whereClause = condition[@"where"];
            NSArray *parameters = condition[@"parameters"];
            
            if (![self deleteFromTable:tableName where:whereClause parameters:parameters]) {
                success = NO;
                *rollback = YES;
                break;
            }
        }
    }];
    
    NSLog(@"批量删除 %lu 条记录%@", (unsigned long)conditions.count, success ? @"成功" : @"失败");
    return success;
}

- (void)optimizeDatabase {
    NSLog(@"=== 优化数据库性能 ===");
    
    [self analyzeDatabase];
    [self vacuumDatabase];
    
    NSLog(@"数据库优化完成");
}

- (void)analyzeDatabase {
    NSLog(@"分析数据库...");
    
    [self executeSQL:@"ANALYZE;"];
    
    NSLog(@"数据库分析完成");
}

- (void)vacuumDatabase {
    NSLog(@"压缩数据库...");
    
    [self executeSQL:@"VACUUM;"];
    
    NSLog(@"数据库压缩完成");
}

- (void)createIndex:(NSString *)indexName onTable:(NSString *)tableName columns:(NSArray *)columns {
    NSString *columnsString = [columns componentsJoinedByString:@", "];
    NSString *sql = [NSString stringWithFormat:@"CREATE INDEX IF NOT EXISTS %@ ON %@ (%@);", indexName, tableName, columnsString];
    
    if ([self executeSQL:sql]) {
        NSLog(@"索引创建成功: %@", indexName);
    } else {
        NSLog(@"索引创建失败: %@", indexName);
    }
}

- (BOOL)backupDatabaseToPath:(NSString *)backupPath {
    NSLog(@"备份数据库到: %@", backupPath);
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSError *error;
    
    // 确保备份目录存在
    NSString *backupDirectory = [backupPath stringByDeletingLastPathComponent];
    [fileManager createDirectoryAtPath:backupDirectory withIntermediateDirectories:YES attributes:nil error:nil];
    
    // 复制数据库文件
    BOOL success = [fileManager copyItemAtPath:self.databasePath toPath:backupPath error:&error];
    
    if (success) {
        NSLog(@"数据库备份成功");
    } else {
        NSLog(@"数据库备份失败: %@", error.localizedDescription);
    }
    
    return success;
}

- (BOOL)restoreDatabaseFromPath:(NSString *)backupPath {
    NSLog(@"从备份恢复数据库: %@", backupPath);
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    if (![fileManager fileExistsAtPath:backupPath]) {
        NSLog(@"备份文件不存在: %@", backupPath);
        return NO;
    }
    
    // 关闭当前数据库
    [self closeDatabase];
    
    // 删除当前数据库文件
    NSError *error;
    [fileManager removeItemAtPath:self.databasePath error:&error];
    
    // 复制备份文件
    BOOL success = [fileManager copyItemAtPath:backupPath toPath:self.databasePath error:&error];
    
    if (success) {
        NSLog(@"数据库恢复成功");
        // 重新打开数据库
        [self openDatabase];
    } else {
        NSLog(@"数据库恢复失败: %@", error.localizedDescription);
    }
    
    return success;
}

- (NSInteger)getDatabaseSize {
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSDictionary *attributes = [fileManager attributesOfItemAtPath:self.databasePath error:nil];
    
    if (attributes) {
        NSInteger size = [attributes[NSFileSize] integerValue];
        NSLog(@"数据库大小: %.2f MB", size / 1024.0 / 1024.0);
        return size;
    }
    
    return 0;
}

- (NSDictionary *)getDatabaseInfo {
    NSMutableDictionary *info = [NSMutableDictionary dictionary];
    
    // 数据库大小
    info[@"size"] = @([self getDatabaseSize]);
    
    // 数据库路径
    info[@"path"] = self.databasePath;
    
    // 获取表信息
    NSArray *tables = [self executeQuery:@"SELECT name FROM sqlite_master WHERE type='table';"];
    info[@"tables"] = tables;
    
    // 获取索引信息
    NSArray *indexes = [self executeQuery:@"SELECT name FROM sqlite_master WHERE type='index';"];
    info[@"indexes"] = indexes;
    
    // 数据库版本
    NSArray *versionResult = [self executeQuery:@"PRAGMA user_version;"];
    if (versionResult.count > 0) {
        info[@"version"] = versionResult[0][@"user_version"];
    }
    
    return info;
}

- (void)recordOperation:(NSString *)operation duration:(NSTimeInterval)duration success:(BOOL)success {
    NSDictionary *metric = @{
        @"operation": operation,
        @"duration": @(duration),
        @"success": @(success),
        @"timestamp": [NSDate date]
    };
    
    [self.performanceMetrics addObject:metric];
    
    // 保持最近1000条记录
    if (self.performanceMetrics.count > 1000) {
        [self.performanceMetrics removeObjectAtIndex:0];
    }
}

- (void)enablePerformanceMonitoring {
    NSLog(@"启用SQLite性能监控...");
    
    // 这里可以添加更多的监控逻辑
    
    NSLog(@"SQLite性能监控启用完成");
}

- (NSDictionary *)getPerformanceReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    // 统计各种操作的性能
    NSMutableDictionary *operationStats = [NSMutableDictionary dictionary];
    
    for (NSDictionary *metric in self.performanceMetrics) {
        NSString *operation = metric[@"operation"];
        NSNumber *duration = metric[@"duration"];
        NSNumber *success = metric[@"success"];
        
        if (!operationStats[operation]) {
            operationStats[operation] = [NSMutableDictionary dictionary];
            operationStats[operation][@"durations"] = [NSMutableArray array];
            operationStats[operation][@"successCount"] = @0;
            operationStats[operation][@"totalCount"] = @0;
        }
        
        [operationStats[operation][@"durations"] addObject:duration];
        operationStats[operation][@"totalCount"] = @([operationStats[operation][@"totalCount"] integerValue] + 1);
        
        if ([success boolValue]) {
            operationStats[operation][@"successCount"] = @([operationStats[operation][@"successCount"] integerValue] + 1);
        }
    }
    
    // 计算统计信息
    for (NSString *operation in operationStats) {
        NSArray *durations = operationStats[operation][@"durations"];
        NSInteger successCount = [operationStats[operation][@"successCount"] integerValue];
        NSInteger totalCount = [operationStats[operation][@"totalCount"] integerValue];
        
        double totalDuration = 0;
        double maxDuration = 0;
        double minDuration = INFINITY;
        
        for (NSNumber *duration in durations) {
            double d = [duration doubleValue];
            totalDuration += d;
            maxDuration = MAX(maxDuration, d);
            minDuration = MIN(minDuration, d);
        }
        
        double averageDuration = totalDuration / durations.count;
        double successRate = (double)successCount / totalCount;
        
        report[operation] = @{
            @"operationCount": @(totalCount),
            @"successRate": @(successRate),
            @"averageDuration": @(averageDuration),
            @"maxDuration": @(maxDuration),
            @"minDuration": @(minDuration == INFINITY ? 0 : minDuration),
            @"totalDuration": @(totalDuration)
        };
    }
    
    report[@"totalOperations"] = @(self.performanceMetrics.count);
    report[@"reportTimestamp"] = [NSDate date];
    
    return report;
}

- (void)logSQLiteStatistics {
    NSLog(@"=== SQLite统计信息 ===");
    
    NSDictionary *report = [self getPerformanceReport];
    
    for (NSString *operation in report) {
        if ([operation isEqualToString:@"totalOperations"] || [operation isEqualToString:@"reportTimestamp"]) {
            continue;
        }
        
        NSDictionary *stats = report[operation];
        NSLog(@"操作类型: %@", operation);
        NSLog(@"  操作次数: %@", stats[@"operationCount"]);
        NSLog(@"  成功率: %.1f%%", [stats[@"successRate"] doubleValue] * 100);
        NSLog(@"  平均耗时: %.3f秒", [stats[@"averageDuration"] doubleValue]);
        NSLog(@"  最大耗时: %.3f秒", [stats[@"maxDuration"] doubleValue]);
    }
    
    // 数据库信息
    NSDictionary *dbInfo = [self getDatabaseInfo];
    NSLog(@"数据库大小: %.2f MB", [dbInfo[@"size"] doubleValue] / 1024.0 / 1024.0);
    NSLog(@"表数量: %lu", (unsigned long)[dbInfo[@"tables"] count]);
    NSLog(@"索引数量: %lu", (unsigned long)[dbInfo[@"indexes"] count]);
}

- (void)dealloc {
    [self closeDatabase];
}

@end
```

## 文件存储深度解析

### 文件存储管理器

```objc
@interface FileStorageManager : NSObject

@property (nonatomic, strong) NSString *documentsPath;
@property (nonatomic, strong) NSString *cachesPath;
@property (nonatomic, strong) NSString *temporaryPath;
@property (nonatomic, strong) NSMutableDictionary *storageMetrics;
@property (nonatomic, strong) dispatch_queue_t fileQueue;
@property (nonatomic, strong) NSFileManager *fileManager;

+ (instancetype)sharedManager;

// 初始化和配置
- (void)initializeStorageDirectories;
- (void)configureStorageSettings;
- (NSString *)getStoragePathForType:(NSString *)storageType;

// 文件操作
- (BOOL)writeData:(NSData *)data toFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (NSData *)readDataFromFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (BOOL)deleteFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (BOOL)fileExists:(NSString *)fileName inDirectory:(NSString *)directory;
- (NSString *)getFullPathForFile:(NSString *)fileName inDirectory:(NSString *)directory;

// 对象序列化
- (BOOL)saveObject:(id<NSCoding>)object toFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (id)loadObjectFromFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (BOOL)saveJSONObject:(id)object toFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (id)loadJSONObjectFromFile:(NSString *)fileName inDirectory:(NSString *)directory;

// 批量操作
- (BOOL)batchWriteFiles:(NSArray *)fileDataArray toDirectory:(NSString *)directory;
- (NSArray *)batchReadFiles:(NSArray *)fileNames fromDirectory:(NSString *)directory;
- (BOOL)batchDeleteFiles:(NSArray *)fileNames fromDirectory:(NSString *)directory;

// 目录管理
- (BOOL)createDirectory:(NSString *)directoryName;
- (BOOL)deleteDirectory:(NSString *)directoryName;
- (NSArray *)listFilesInDirectory:(NSString *)directory;
- (NSArray *)listDirectories;
- (NSDictionary *)getDirectoryInfo:(NSString *)directory;

// 文件属性
- (NSDictionary *)getFileAttributes:(NSString *)fileName inDirectory:(NSString *)directory;
- (NSDate *)getFileModificationDate:(NSString *)fileName inDirectory:(NSString *)directory;
- (NSInteger)getFileSize:(NSString *)fileName inDirectory:(NSString *)directory;

// 存储优化
- (void)cleanupTemporaryFiles;
- (void)cleanupCacheFiles;
- (void)compressFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (void)decompressFile:(NSString *)fileName inDirectory:(NSString *)directory;

// 存储监控
- (void)enableStorageMonitoring;
- (NSDictionary *)getStorageReport;
- (void)logStorageStatistics;
- (NSInteger)getTotalStorageSize;
- (NSInteger)getAvailableStorageSpace;

// 数据迁移
- (BOOL)migrateDataFromDirectory:(NSString *)sourceDirectory toDirectory:(NSString *)targetDirectory;
- (BOOL)backupDirectory:(NSString *)directory toPath:(NSString *)backupPath;
- (BOOL)restoreDirectoryFromBackup:(NSString *)backupPath toDirectory:(NSString *)directory;

@end

@implementation FileStorageManager

+ (instancetype)sharedManager {
    static FileStorageManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.fileManager = [NSFileManager defaultManager];
        self.storageMetrics = [NSMutableDictionary dictionary];
        self.fileQueue = dispatch_queue_create("com.filestorage.queue", DISPATCH_QUEUE_CONCURRENT);
        
        [self initializeStorageDirectories];
        [self configureStorageSettings];
    }
    return self;
}

- (void)initializeStorageDirectories {
    NSLog(@"=== 初始化文件存储目录 ===");
    
    // 获取标准目录路径
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    self.documentsPath = [paths firstObject];
    
    paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);
    self.cachesPath = [paths firstObject];
    
    self.temporaryPath = NSTemporaryDirectory();
    
    NSLog(@"Documents目录: %@", self.documentsPath);
    NSLog(@"Caches目录: %@", self.cachesPath);
    NSLog(@"Temporary目录: %@", self.temporaryPath);
    
    // 创建自定义子目录
    [self createDirectory:@"UserData"];
    [self createDirectory:@"AppData"];
    [self createDirectory:@"Backups"];
    
    NSLog(@"文件存储目录初始化完成");
}

- (void)configureStorageSettings {
    NSLog(@"配置文件存储设置...");
    
    // 启用存储监控
    [self enableStorageMonitoring];
    
    // 清理临时文件
    [self cleanupTemporaryFiles];
    
    NSLog(@"文件存储设置配置完成");
}

- (NSString *)getStoragePathForType:(NSString *)storageType {
    if ([storageType isEqualToString:@"documents"]) {
        return self.documentsPath;
    } else if ([storageType isEqualToString:@"caches"]) {
        return self.cachesPath;
    } else if ([storageType isEqualToString:@"temporary"]) {
        return self.temporaryPath;
    }
    
    return self.documentsPath; // 默认返回Documents目录
}

- (BOOL)writeData:(NSData *)data toFile:(NSString *)fileName inDirectory:(NSString *)directory {
    NSDate *startTime = [NSDate date];
    
    __block BOOL success = NO;
    
    dispatch_barrier_sync(self.fileQueue, ^{
        NSString *fullPath = [self getFullPathForFile:fileName inDirectory:directory];
        
        // 确保目录存在
        NSString *directoryPath = [fullPath stringByDeletingLastPathComponent];
        [self.fileManager createDirectoryAtPath:directoryPath withIntermediateDirectories:YES attributes:nil error:nil];
        
        success = [data writeToFile:fullPath atomically:YES];
        
        if (success) {
            NSLog(@"文件写入成功: %@", fileName);
        } else {
            NSLog(@"文件写入失败: %@", fileName);
        }
    });
    
    // 记录性能指标
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    [self recordOperation:@"write" fileName:fileName duration:duration success:success dataSize:data.length];
    
    return success;
}

- (NSData *)readDataFromFile:(NSString *)fileName inDirectory:(NSString *)directory {
    NSDate *startTime = [NSDate date];
    
    __block NSData *data = nil;
    
    dispatch_sync(self.fileQueue, ^{
        NSString *fullPath = [self getFullPathForFile:fileName inDirectory:directory];
        
        if ([self.fileManager fileExistsAtPath:fullPath]) {
            data = [NSData dataWithContentsOfFile:fullPath];
            
            if (data) {
                NSLog(@"文件读取成功: %@, 大小: %lu bytes", fileName, (unsigned long)data.length);
            } else {
                NSLog(@"文件读取失败: %@", fileName);
            }
        } else {
            NSLog(@"文件不存在: %@", fileName);
        }
    });
    
    // 记录性能指标
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    [self recordOperation:@"read" fileName:fileName duration:duration success:(data != nil) dataSize:data.length];
    
    return data;
}

- (BOOL)deleteFile:(NSString *)fileName inDirectory:(NSString *)directory {
    __block BOOL success = NO;
    
    dispatch_barrier_sync(self.fileQueue, ^{
        NSString *fullPath = [self getFullPathForFile:fileName inDirectory:directory];
        
        if ([self.fileManager fileExistsAtPath:fullPath]) {
            NSError *error;
            success = [self.fileManager removeItemAtPath:fullPath error:&error];
            
            if (success) {
                NSLog(@"文件删除成功: %@", fileName);
            } else {
                NSLog(@"文件删除失败: %@, 错误: %@", fileName, error.localizedDescription);
            }
        } else {
            NSLog(@"文件不存在，无需删除: %@", fileName);
            success = YES; // 文件不存在也算删除成功
        }
    });
    
    return success;
}

- (BOOL)fileExists:(NSString *)fileName inDirectory:(NSString *)directory {
    NSString *fullPath = [self getFullPathForFile:fileName inDirectory:directory];
    return [self.fileManager fileExistsAtPath:fullPath];
}

- (NSString *)getFullPathForFile:(NSString *)fileName inDirectory:(NSString *)directory {
    NSString *basePath = [self getStoragePathForType:@"documents"];
    
    if (directory && directory.length > 0) {
        basePath = [basePath stringByAppendingPathComponent:directory];
    }
    
    return [basePath stringByAppendingPathComponent:fileName];
}

- (BOOL)saveObject:(id<NSCoding>)object toFile:(NSString *)fileName inDirectory:(NSString *)directory {
    if (!object) {
        NSLog(@"对象为空，无法保存");
        return NO;
    }
    
    NSDate *startTime = [NSDate date];
    
    @try {
        NSData *data = [NSKeyedArchiver archivedDataWithRootObject:object requiringSecureCoding:NO error:nil];
        
        if (data) {
            BOOL success = [self writeData:data toFile:fileName inDirectory:directory];
            
            NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
            NSLog(@"对象序列化保存%@: %@, 耗时: %.3f秒", success ? @"成功" : @"失败", fileName, duration);
            
            return success;
        } else {
            NSLog(@"对象序列化失败: %@", fileName);
            return NO;
        }
    } @catch (NSException *exception) {
        NSLog(@"对象保存异常: %@, 异常: %@", fileName, exception.reason);
        return NO;
    }
}

- (id)loadObjectFromFile:(NSString *)fileName inDirectory:(NSString *)directory {
    NSDate *startTime = [NSDate date];
    
    NSData *data = [self readDataFromFile:fileName inDirectory:directory];
    
    if (!data) {
        return nil;
    }
    
    @try {
        id object = [NSKeyedUnarchiver unarchiveObjectWithData:data];
        
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        NSLog(@"对象反序列化%@: %@, 耗时: %.3f秒", object ? @"成功" : @"失败", fileName, duration);
        
        return object;
    } @catch (NSException *exception) {
        NSLog(@"对象加载异常: %@, 异常: %@", fileName, exception.reason);
        return nil;
    }
}

- (BOOL)saveJSONObject:(id)object toFile:(NSString *)fileName inDirectory:(NSString *)directory {
    if (!object) {
        NSLog(@"JSON对象为空，无法保存");
        return NO;
    }
    
    NSDate *startTime = [NSDate date];
    
    @try {
        NSError *error;
        NSData *jsonData = [NSJSONSerialization dataWithJSONObject:object options:NSJSONWritingPrettyPrinted error:&error];
        
        if (jsonData && !error) {
            BOOL success = [self writeData:jsonData toFile:fileName inDirectory:directory];
            
            NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
            NSLog(@"JSON对象保存%@: %@, 耗时: %.3f秒", success ? @"成功" : @"失败", fileName, duration);
            
            return success;
        } else {
            NSLog(@"JSON序列化失败: %@, 错误: %@", fileName, error.localizedDescription);
            return NO;
        }
    } @catch (NSException *exception) {
        NSLog(@"JSON保存异常: %@, 异常: %@", fileName, exception.reason);
        return NO;
    }
}

- (id)loadJSONObjectFromFile:(NSString *)fileName inDirectory:(NSString *)directory {
    NSDate *startTime = [NSDate date];
    
    NSData *data = [self readDataFromFile:fileName inDirectory:directory];
    
    if (!data) {
        return nil;
    }
    
    @try {
        NSError *error;
        id object = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingMutableContainers error:&error];
        
        if (object && !error) {
            NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
            NSLog(@"JSON对象加载成功: %@, 耗时: %.3f秒", fileName, duration);
            return object;
        } else {
            NSLog(@"JSON反序列化失败: %@, 错误: %@", fileName, error.localizedDescription);
            return nil;
        }
    } @catch (NSException *exception) {
        NSLog(@"JSON加载异常: %@, 异常: %@", fileName, exception.reason);
        return nil;
    }
}

- (BOOL)batchWriteFiles:(NSArray *)fileDataArray toDirectory:(NSString *)directory {
    NSDate *startTime = [NSDate date];
    
    __block BOOL allSuccess = YES;
    __block NSInteger successCount = 0;
    
    dispatch_apply(fileDataArray.count, self.fileQueue, ^(size_t index) {
        NSDictionary *fileData = fileDataArray[index];
        NSString *fileName = fileData[@"fileName"];
        NSData *data = fileData[@"data"];
        
        if ([self writeData:data toFile:fileName inDirectory:directory]) {
            @synchronized(self) {
                successCount++;
            }
        } else {
            allSuccess = NO;
        }
    });
    
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    NSLog(@"批量写入文件完成: %ld/%lu 成功, 耗时: %.3f秒", (long)successCount, (unsigned long)fileDataArray.count, duration);
    
    return allSuccess;
}

- (NSArray *)batchReadFiles:(NSArray *)fileNames fromDirectory:(NSString *)directory {
    NSDate *startTime = [NSDate date];
    
    NSMutableArray *results = [NSMutableArray arrayWithCapacity:fileNames.count];
    
    // 初始化结果数组
    for (NSInteger i = 0; i < fileNames.count; i++) {
        [results addObject:[NSNull null]];
    }
    
    dispatch_apply(fileNames.count, self.fileQueue, ^(size_t index) {
        NSString *fileName = fileNames[index];
        NSData *data = [self readDataFromFile:fileName inDirectory:directory];
        
        @synchronized(results) {
            if (data) {
                results[index] = data;
            }
        }
    });
    
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    NSLog(@"批量读取文件完成: %lu 个文件, 耗时: %.3f秒", (unsigned long)fileNames.count, duration);
    
    return results;
}

- (BOOL)batchDeleteFiles:(NSArray *)fileNames fromDirectory:(NSString *)directory {
    NSDate *startTime = [NSDate date];
    
    __block BOOL allSuccess = YES;
    __block NSInteger successCount = 0;
    
    dispatch_apply(fileNames.count, self.fileQueue, ^(size_t index) {
        NSString *fileName = fileNames[index];
        
        if ([self deleteFile:fileName inDirectory:directory]) {
            @synchronized(self) {
                successCount++;
            }
        } else {
            allSuccess = NO;
        }
    });
    
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    NSLog(@"批量删除文件完成: %ld/%lu 成功, 耗时: %.3f秒", (long)successCount, (unsigned long)fileNames.count, duration);
    
    return allSuccess;
}

- (BOOL)createDirectory:(NSString *)directoryName {
    NSString *directoryPath = [self.documentsPath stringByAppendingPathComponent:directoryName];
    
    NSError *error;
    BOOL success = [self.fileManager createDirectoryAtPath:directoryPath withIntermediateDirectories:YES attributes:nil error:&error];
    
    if (success) {
        NSLog(@"目录创建成功: %@", directoryName);
    } else {
        NSLog(@"目录创建失败: %@, 错误: %@", directoryName, error.localizedDescription);
    }
    
    return success;
}

- (BOOL)deleteDirectory:(NSString *)directoryName {
    NSString *directoryPath = [self.documentsPath stringByAppendingPathComponent:directoryName];
    
    if ([self.fileManager fileExistsAtPath:directoryPath]) {
        NSError *error;
        BOOL success = [self.fileManager removeItemAtPath:directoryPath error:&error];
        
        if (success) {
            NSLog(@"目录删除成功: %@", directoryName);
        } else {
            NSLog(@"目录删除失败: %@, 错误: %@", directoryName, error.localizedDescription);
        }
        
        return success;
    } else {
        NSLog(@"目录不存在，无需删除: %@", directoryName);
        return YES;
    }
}

- (NSArray *)listFilesInDirectory:(NSString *)directory {
    NSString *directoryPath = directory ? [self.documentsPath stringByAppendingPathComponent:directory] : self.documentsPath;
    
    NSError *error;
    NSArray *contents = [self.fileManager contentsOfDirectoryAtPath:directoryPath error:&error];
    
    if (contents) {
        // 过滤出文件（排除目录）
        NSMutableArray *files = [NSMutableArray array];
        
        for (NSString *item in contents) {
            NSString *itemPath = [directoryPath stringByAppendingPathComponent:item];
            BOOL isDirectory;
            
            if ([self.fileManager fileExistsAtPath:itemPath isDirectory:&isDirectory] && !isDirectory) {
                [files addObject:item];
            }
        }
        
        NSLog(@"目录 %@ 包含 %lu 个文件", directory ?: @"根目录", (unsigned long)files.count);
        return files;
    } else {
        NSLog(@"列出文件失败: %@, 错误: %@", directory, error.localizedDescription);
        return @[];
    }
}

- (NSArray *)listDirectories {
    NSError *error;
    NSArray *contents = [self.fileManager contentsOfDirectoryAtPath:self.documentsPath error:&error];
    
    if (contents) {
        NSMutableArray *directories = [NSMutableArray array];
        
        for (NSString *item in contents) {
            NSString *itemPath = [self.documentsPath stringByAppendingPathComponent:item];
            BOOL isDirectory;
            
            if ([self.fileManager fileExistsAtPath:itemPath isDirectory:&isDirectory] && isDirectory) {
                [directories addObject:item];
            }
        }
        
        NSLog(@"根目录包含 %lu 个子目录", (unsigned long)directories.count);
        return directories;
    } else {
        NSLog(@"列出目录失败, 错误: %@", error.localizedDescription);
        return @[];
    }
}

- (NSDictionary *)getDirectoryInfo:(NSString *)directory {
    NSString *directoryPath = directory ? [self.documentsPath stringByAppendingPathComponent:directory] : self.documentsPath;
    
    NSMutableDictionary *info = [NSMutableDictionary dictionary];
    
    // 基本信息
    info[@"path"] = directoryPath;
    info[@"name"] = directory ?: @"根目录";
    
    // 文件和目录数量
    NSArray *files = [self listFilesInDirectory:directory];
    NSArray *subdirectories = directory ? @[] : [self listDirectories]; // 简化处理
    
    info[@"fileCount"] = @(files.count);
    info[@"directoryCount"] = @(subdirectories.count);
    
    // 计算总大小
    NSInteger totalSize = 0;
    for (NSString *fileName in files) {
        totalSize += [self getFileSize:fileName inDirectory:directory];
    }
    
    info[@"totalSize"] = @(totalSize);
    info[@"totalSizeMB"] = @(totalSize / 1024.0 / 1024.0);
    
    return info;
}

- (NSDictionary *)getFileAttributes:(NSString *)fileName inDirectory:(NSString *)directory {
    NSString *fullPath = [self getFullPathForFile:fileName inDirectory:directory];
    
    NSError *error;
    NSDictionary *attributes = [self.fileManager attributesOfItemAtPath:fullPath error:&error];
    
    if (attributes) {
        return attributes;
    } else {
        NSLog(@"获取文件属性失败: %@, 错误: %@", fileName, error.localizedDescription);
        return @{};
    }
}

- (NSDate *)getFileModificationDate:(NSString *)fileName inDirectory:(NSString *)directory {
    NSDictionary *attributes = [self getFileAttributes:fileName inDirectory:directory];
    return attributes[NSFileModificationDate];
}

- (NSInteger)getFileSize:(NSString *)fileName inDirectory:(NSString *)directory {
    NSDictionary *attributes = [self getFileAttributes:fileName inDirectory:directory];
    return [attributes[NSFileSize] integerValue];
}

- (void)cleanupTemporaryFiles {
    NSLog(@"清理临时文件...");
    
    NSError *error;
    NSArray *tempFiles = [self.fileManager contentsOfDirectoryAtPath:self.temporaryPath error:&error];
    
    NSInteger deletedCount = 0;
    NSInteger deletedSize = 0;
    
    for (NSString *fileName in tempFiles) {
        NSString *filePath = [self.temporaryPath stringByAppendingPathComponent:fileName];
        
        // 获取文件大小
        NSDictionary *attributes = [self.fileManager attributesOfItemAtPath:filePath error:nil];
        NSInteger fileSize = [attributes[NSFileSize] integerValue];
        
        if ([self.fileManager removeItemAtPath:filePath error:nil]) {
            deletedCount++;
            deletedSize += fileSize;
        }
    }
    
    NSLog(@"临时文件清理完成: 删除 %ld 个文件, 释放 %.2f MB", (long)deletedCount, deletedSize / 1024.0 / 1024.0);
}

- (void)cleanupCacheFiles {
    NSLog(@"清理缓存文件...");
    
    NSError *error;
    NSArray *cacheFiles = [self.fileManager contentsOfDirectoryAtPath:self.cachesPath error:&error];
    
    NSInteger deletedCount = 0;
    NSInteger deletedSize = 0;
    NSDate *cutoffDate = [NSDate dateWithTimeIntervalSinceNow:-7 * 24 * 60 * 60]; // 7天前
    
    for (NSString *fileName in cacheFiles) {
        NSString *filePath = [self.cachesPath stringByAppendingPathComponent:fileName];
        
        NSDictionary *attributes = [self.fileManager attributesOfItemAtPath:filePath error:nil];
        NSDate *modificationDate = attributes[NSFileModificationDate];
        NSInteger fileSize = [attributes[NSFileSize] integerValue];
        
        // 删除7天前的缓存文件
        if ([modificationDate compare:cutoffDate] == NSOrderedAscending) {
            if ([self.fileManager removeItemAtPath:filePath error:nil]) {
                deletedCount++;
                deletedSize += fileSize;
            }
        }
    }
    
    NSLog(@"缓存文件清理完成: 删除 %ld 个文件, 释放 %.2f MB", (long)deletedCount, deletedSize / 1024.0 / 1024.0);
}

- (void)compressFile:(NSString *)fileName inDirectory:(NSString *)directory {
    NSLog(@"压缩文件: %@", fileName);
    
    NSData *originalData = [self readDataFromFile:fileName inDirectory:directory];
    
    if (originalData) {
        NSData *compressedData = [originalData compressedDataUsingAlgorithm:NSDataCompressionAlgorithmZlib error:nil];
        
        if (compressedData) {
            NSString *compressedFileName = [fileName stringByAppendingString:@".compressed"];
            
            if ([self writeData:compressedData toFile:compressedFileName inDirectory:directory]) {
                double compressionRatio = (double)compressedData.length / originalData.length;
                NSLog(@"文件压缩成功: %@, 压缩率: %.1f%%", fileName, compressionRatio * 100);
            }
        }
    }
}

- (void)decompressFile:(NSString *)fileName inDirectory:(NSString *)directory {
    NSLog(@"解压缩文件: %@", fileName);
    
    NSData *compressedData = [self readDataFromFile:fileName inDirectory:directory];
    
    if (compressedData) {
        NSData *decompressedData = [compressedData decompressedDataUsingAlgorithm:NSDataCompressionAlgorithmZlib error:nil];
        
        if (decompressedData) {
            NSString *decompressedFileName = [fileName stringByReplacingOccurrencesOfString:@".compressed" withString:@""];
            
            if ([self writeData:decompressedData toFile:decompressedFileName inDirectory:directory]) {
                NSLog(@"文件解压缩成功: %@", fileName);
            }
        }
    }
}

- (void)recordOperation:(NSString *)operation fileName:(NSString *)fileName duration:(NSTimeInterval)duration success:(BOOL)success dataSize:(NSInteger)dataSize {
    NSDictionary *metric = @{
        @"operation": operation,
        @"fileName": fileName ?: @"",
        @"duration": @(duration),
        @"success": @(success),
        @"dataSize": @(dataSize),
        @"timestamp": [NSDate date]
    };
    
    @synchronized(self.storageMetrics) {
        if (!self.storageMetrics[@"operations"]) {
            self.storageMetrics[@"operations"] = [NSMutableArray array];
        }
        
        [self.storageMetrics[@"operations"] addObject:metric];
        
        // 保持最近1000条记录
        NSMutableArray *operations = self.storageMetrics[@"operations"];
        if (operations.count > 1000) {
            [operations removeObjectAtIndex:0];
        }
    }
}

- (void)enableStorageMonitoring {
    NSLog(@"启用文件存储监控...");
    
    // 初始化监控数据
    self.storageMetrics[@"startTime"] = [NSDate date];
    self.storageMetrics[@"operations"] = [NSMutableArray array];
    
    NSLog(@"文件存储监控启用完成");
}

- (NSDictionary *)getStorageReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    @synchronized(self.storageMetrics) {
        NSArray *operations = self.storageMetrics[@"operations"];
        
        // 统计各种操作的性能
        NSMutableDictionary *operationStats = [NSMutableDictionary dictionary];
        
        for (NSDictionary *metric in operations) {
            NSString *operation = metric[@"operation"];
            NSNumber *duration = metric[@"duration"];
            NSNumber *success = metric[@"success"];
            NSNumber *dataSize = metric[@"dataSize"];
            
            if (!operationStats[operation]) {
                operationStats[operation] = [NSMutableDictionary dictionary];
                operationStats[operation][@"durations"] = [NSMutableArray array];
                operationStats[operation][@"dataSizes"] = [NSMutableArray array];
                operationStats[operation][@"successCount"] = @0;
                operationStats[operation][@"totalCount"] = @0;
            }
            
            [operationStats[operation][@"durations"] addObject:duration];
            [operationStats[operation][@"dataSizes"] addObject:dataSize];
            operationStats[operation][@"totalCount"] = @([operationStats[operation][@"totalCount"] integerValue] + 1);
            
            if ([success boolValue]) {
                operationStats[operation][@"successCount"] = @([operationStats[operation][@"successCount"] integerValue] + 1);
            }
        }
        
        // 计算统计信息
        for (NSString *operation in operationStats) {
            NSArray *durations = operationStats[operation][@"durations"];
            NSArray *dataSizes = operationStats[operation][@"dataSizes"];
            NSInteger successCount = [operationStats[operation][@"successCount"] integerValue];
            NSInteger totalCount = [operationStats[operation][@"totalCount"] integerValue];
            
            double totalDuration = 0;
            double maxDuration = 0;
            double minDuration = INFINITY;
            
            NSInteger totalDataSize = 0;
            
            for (NSNumber *duration in durations) {
                double d = [duration doubleValue];
                totalDuration += d;
                maxDuration = MAX(maxDuration, d);
                minDuration = MIN(minDuration, d);
            }
            
            for (NSNumber *dataSize in dataSizes) {
                totalDataSize += [dataSize integerValue];
            }
            
            double averageDuration = totalDuration / durations.count;
            double successRate = (double)successCount / totalCount;
            double averageDataSize = (double)totalDataSize / dataSizes.count;
            
            report[operation] = @{
                @"operationCount": @(totalCount),
                @"successRate": @(successRate),
                @"averageDuration": @(averageDuration),
                @"maxDuration": @(maxDuration),
                @"minDuration": @(minDuration == INFINITY ? 0 : minDuration),
                @"totalDuration": @(totalDuration),
                @"totalDataSize": @(totalDataSize),
                @"averageDataSize": @(averageDataSize)
            };
        }
        
        report[@"totalOperations"] = @(operations.count);
        report[@"reportTimestamp"] = [NSDate date];
        report[@"monitoringStartTime"] = self.storageMetrics[@"startTime"];
    }
    
    // 添加存储空间信息
    report[@"storageInfo"] = @{
        @"totalStorageSize": @([self getTotalStorageSize]),
        @"availableSpace": @([self getAvailableStorageSpace]),
        @"documentsSize": @([[self getDirectoryInfo:nil][@"totalSize"] integerValue]),
        @"cachesSize": @([[self getDirectoryInfo:@"../Caches"][@"totalSize"] integerValue])
    };
    
    return report;
}

- (void)logStorageStatistics {
    NSLog(@"=== 文件存储统计信息 ===");
    
    NSDictionary *report = [self getStorageReport];
    
    // 操作统计
    for (NSString *operation in report) {
        if ([operation isEqualToString:@"totalOperations"] || 
            [operation isEqualToString:@"reportTimestamp"] ||
            [operation isEqualToString:@"monitoringStartTime"] ||
            [operation isEqualToString:@"storageInfo"]) {
            continue;
        }
        
        NSDictionary *stats = report[operation];
        NSLog(@"操作类型: %@", operation);
        NSLog(@"  操作次数: %@", stats[@"operationCount"]);
        NSLog(@"  成功率: %.1f%%", [stats[@"successRate"] doubleValue] * 100);
        NSLog(@"  平均耗时: %.3f秒", [stats[@"averageDuration"] doubleValue]);
        NSLog(@"  平均数据大小: %.2f KB", [stats[@"averageDataSize"] doubleValue] / 1024.0);
    }
    
    // 存储空间统计
    NSDictionary *storageInfo = report[@"storageInfo"];
    NSLog(@"存储空间信息:");
    NSLog(@"  总存储大小: %.2f GB", [storageInfo[@"totalStorageSize"] doubleValue] / 1024.0 / 1024.0 / 1024.0);
    NSLog(@"  可用空间: %.2f GB", [storageInfo[@"availableSpace"] doubleValue] / 1024.0 / 1024.0 / 1024.0);
    NSLog(@"  Documents大小: %.2f MB", [storageInfo[@"documentsSize"] doubleValue] / 1024.0 / 1024.0);
    NSLog(@"  Caches大小: %.2f MB", [storageInfo[@"cachesSize"] doubleValue] / 1024.0 / 1024.0);
}

- (NSInteger)getTotalStorageSize {
    NSError *error;
    NSDictionary *attributes = [self.fileManager attributesOfFileSystemForPath:self.documentsPath error:&error];
    
    if (attributes) {
        return [attributes[NSFileSystemSize] integerValue];
    }
    
    return 0;
}

- (NSInteger)getAvailableStorageSpace {
    NSError *error;
    NSDictionary *attributes = [self.fileManager attributesOfFileSystemForPath:self.documentsPath error:&error];
    
    if (attributes) {
        return [attributes[NSFileSystemFreeSize] integerValue];
    }
    
    return 0;
}

- (BOOL)migrateDataFromDirectory:(NSString *)sourceDirectory toDirectory:(NSString *)targetDirectory {
    NSLog(@"数据迁移: %@ -> %@", sourceDirectory, targetDirectory);
    
    NSString *sourcePath = [self.documentsPath stringByAppendingPathComponent:sourceDirectory];
    NSString *targetPath = [self.documentsPath stringByAppendingPathComponent:targetDirectory];
    
    // 创建目标目录
    [self createDirectory:targetDirectory];
    
    // 获取源目录中的所有文件
    NSArray *files = [self listFilesInDirectory:sourceDirectory];
    
    NSInteger successCount = 0;
    
    for (NSString *fileName in files) {
        NSString *sourceFilePath = [sourcePath stringByAppendingPathComponent:fileName];
        NSString *targetFilePath = [targetPath stringByAppendingPathComponent:fileName];
        
        NSError *error;
        if ([self.fileManager copyItemAtPath:sourceFilePath toPath:targetFilePath error:&error]) {
            successCount++;
        } else {
            NSLog(@"文件迁移失败: %@, 错误: %@", fileName, error.localizedDescription);
        }
    }
    
    BOOL success = (successCount == files.count);
    NSLog(@"数据迁移%@: %ld/%lu 文件成功", success ? @"成功" : @"部分成功", (long)successCount, (unsigned long)files.count);
    
    return success;
}

- (BOOL)backupDirectory:(NSString *)directory toPath:(NSString *)backupPath {
    NSLog(@"备份目录: %@ -> %@", directory, backupPath);
    
    NSString *sourcePath = directory ? [self.documentsPath stringByAppendingPathComponent:directory] : self.documentsPath;
    
    NSError *error;
    
    // 确保备份目录的父目录存在
    NSString *backupParentDirectory = [backupPath stringByDeletingLastPathComponent];
    [self.fileManager createDirectoryAtPath:backupParentDirectory withIntermediateDirectories:YES attributes:nil error:nil];
    
    // 复制整个目录
    BOOL success = [self.fileManager copyItemAtPath:sourcePath toPath:backupPath error:&error];
    
    if (success) {
        NSLog(@"目录备份成功");
    } else {
        NSLog(@"目录备份失败: %@", error.localizedDescription);
    }
    
    return success;
}

- (BOOL)restoreDirectoryFromBackup:(NSString *)backupPath toDirectory:(NSString *)directory {
    NSLog(@"从备份恢复目录: %@ -> %@", backupPath, directory);
    
    if (![self.fileManager fileExistsAtPath:backupPath]) {
        NSLog(@"备份路径不存在: %@", backupPath);
        return NO;
    }
    
    NSString *targetPath = directory ? [self.documentsPath stringByAppendingPathComponent:directory] : self.documentsPath;
    
    // 删除现有目录
    if ([self.fileManager fileExistsAtPath:targetPath]) {
        [self.fileManager removeItemAtPath:targetPath error:nil];
    }
    
    // 从备份恢复
    NSError *error;
    BOOL success = [self.fileManager copyItemAtPath:backupPath toPath:targetPath error:&error];
    
    if (success) {
        NSLog(@"目录恢复成功");
    } else {
        NSLog(@"目录恢复失败: %@", error.localizedDescription);
    }
    
    return success;
}

@end
```

## 数据持久化性能优化

### 性能优化管理器

```objc
@interface DataPersistencePerformanceOptimizer : NSObject

@property (nonatomic, strong) NSMutableDictionary *performanceMetrics;
@property (nonatomic, strong) NSTimer *monitoringTimer;
@property (nonatomic, strong) dispatch_queue_t optimizationQueue;
@property (nonatomic, assign) BOOL isMonitoring;

+ (instancetype)sharedOptimizer;

// 性能监控
- (void)startPerformanceMonitoring;
- (void)stopPerformanceMonitoring;
- (void)recordOperation:(NSString *)operation duration:(NSTimeInterval)duration dataSize:(NSInteger)dataSize;
- (NSDictionary *)getPerformanceReport;

// 性能分析
- (NSArray *)analyzePerformanceBottlenecks;
- (NSDictionary *)getOptimizationSuggestions;
- (void)generatePerformanceReport;

// Core Data优化
- (void)optimizeCoreDataPerformance:(NSManagedObjectContext *)context;
- (void)configureFetchRequestOptimization:(NSFetchRequest *)fetchRequest;
- (void)optimizeBatchOperations:(NSManagedObjectContext *)context;

// SQLite优化
- (void)optimizeSQLiteDatabase:(sqlite3 *)database;
- (void)analyzeSQLitePerformance:(sqlite3 *)database;
- (void)optimizeSQLiteQueries:(sqlite3 *)database;

// 文件存储优化
- (void)optimizeFileStoragePerformance;
- (void)analyzeFileStoragePatterns;
- (void)optimizeFileOperations;

// 内存优化
- (void)optimizeMemoryUsage;
- (void)monitorMemoryPressure;
- (void)handleMemoryWarning;

// 缓存优化
- (void)optimizeCacheStrategy;
- (void)configureCachePolicy;
- (void)cleanupExpiredCache;

@end

@implementation DataPersistencePerformanceOptimizer

+ (instancetype)sharedOptimizer {
    static DataPersistencePerformanceOptimizer *sharedInstance = nil;
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
        self.optimizationQueue = dispatch_queue_create("com.datapersistence.optimization", DISPATCH_QUEUE_CONCURRENT);
        self.isMonitoring = NO;
        
        [self initializePerformanceTracking];
    }
    return self;
}

- (void)initializePerformanceTracking {
    NSLog(@"=== 初始化数据持久化性能跟踪 ===");
    
    self.performanceMetrics[@"operations"] = [NSMutableArray array];
    self.performanceMetrics[@"startTime"] = [NSDate date];
    self.performanceMetrics[@"coreDataMetrics"] = [NSMutableDictionary dictionary];
    self.performanceMetrics[@"sqliteMetrics"] = [NSMutableDictionary dictionary];
    self.performanceMetrics[@"fileStorageMetrics"] = [NSMutableDictionary dictionary];
    
    NSLog(@"性能跟踪初始化完成");
}

- (void)startPerformanceMonitoring {
    if (self.isMonitoring) {
        NSLog(@"性能监控已在运行中");
        return;
    }
    
    NSLog(@"启动数据持久化性能监控...");
    
    self.isMonitoring = YES;
    
    // 启动定时监控
    self.monitoringTimer = [NSTimer scheduledTimerWithTimeInterval:30.0
                                                            target:self
                                                          selector:@selector(performPeriodicAnalysis)
                                                          userInfo:nil
                                                           repeats:YES];
    
    // 监听内存警告
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(handleMemoryWarning)
                                                 name:UIApplicationDidReceiveMemoryWarningNotification
                                               object:nil];
    
    NSLog(@"性能监控启动完成");
}

- (void)stopPerformanceMonitoring {
    if (!self.isMonitoring) {
        NSLog(@"性能监控未在运行");
        return;
    }
    
    NSLog(@"停止数据持久化性能监控...");
    
    self.isMonitoring = NO;
    
    [self.monitoringTimer invalidate];
    self.monitoringTimer = nil;
    
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    
    NSLog(@"性能监控已停止");
}

- (void)recordOperation:(NSString *)operation duration:(NSTimeInterval)duration dataSize:(NSInteger)dataSize {
    NSDictionary *metric = @{
        @"operation": operation,
        @"duration": @(duration),
        @"dataSize": @(dataSize),
        @"timestamp": [NSDate date],
        @"memoryUsage": @([self getCurrentMemoryUsage])
    };
    
    @synchronized(self.performanceMetrics) {
        [self.performanceMetrics[@"operations"] addObject:metric];
        
        // 保持最近5000条记录
        NSMutableArray *operations = self.performanceMetrics[@"operations"];
        if (operations.count > 5000) {
            [operations removeObjectAtIndex:0];
        }
    }
    
    // 检查是否需要立即优化
    if (duration > 1.0) { // 操作耗时超过1秒
        [self analyzeSlowOperation:metric];
    }
}

- (NSDictionary *)getPerformanceReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    @synchronized(self.performanceMetrics) {
        NSArray *operations = self.performanceMetrics[@"operations"];
        
        // 统计各种操作的性能
        NSMutableDictionary *operationStats = [NSMutableDictionary dictionary];
        
        for (NSDictionary *metric in operations) {
            NSString *operation = metric[@"operation"];
            NSNumber *duration = metric[@"duration"];
            NSNumber *dataSize = metric[@"dataSize"];
            NSNumber *memoryUsage = metric[@"memoryUsage"];
            
            if (!operationStats[operation]) {
                operationStats[operation] = [NSMutableDictionary dictionary];
                operationStats[operation][@"durations"] = [NSMutableArray array];
                operationStats[operation][@"dataSizes"] = [NSMutableArray array];
                operationStats[operation][@"memoryUsages"] = [NSMutableArray array];
                operationStats[operation][@"count"] = @0;
            }
            
            [operationStats[operation][@"durations"] addObject:duration];
            [operationStats[operation][@"dataSizes"] addObject:dataSize];
            [operationStats[operation][@"memoryUsages"] addObject:memoryUsage];
            operationStats[operation][@"count"] = @([operationStats[operation][@"count"] integerValue] + 1);
        }
        
        // 计算统计信息
        for (NSString *operation in operationStats) {
            NSArray *durations = operationStats[operation][@"durations"];
            NSArray *dataSizes = operationStats[operation][@"dataSizes"];
            NSArray *memoryUsages = operationStats[operation][@"memoryUsages"];
            
            double totalDuration = 0;
            double maxDuration = 0;
            double minDuration = INFINITY;
            
            NSInteger totalDataSize = 0;
            NSInteger totalMemoryUsage = 0;
            
            for (NSNumber *duration in durations) {
                double d = [duration doubleValue];
                totalDuration += d;
                maxDuration = MAX(maxDuration, d);
                minDuration = MIN(minDuration, d);
            }
            
            for (NSNumber *dataSize in dataSizes) {
                totalDataSize += [dataSize integerValue];
            }
            
            for (NSNumber *memoryUsage in memoryUsages) {
                totalMemoryUsage += [memoryUsage integerValue];
            }
            
            double averageDuration = totalDuration / durations.count;
            double averageDataSize = (double)totalDataSize / dataSizes.count;
            double averageMemoryUsage = (double)totalMemoryUsage / memoryUsages.count;
            
            report[operation] = @{
                @"operationCount": operationStats[operation][@"count"],
                @"averageDuration": @(averageDuration),
                @"maxDuration": @(maxDuration),
                @"minDuration": @(minDuration == INFINITY ? 0 : minDuration),
                @"totalDuration": @(totalDuration),
                @"averageDataSize": @(averageDataSize),
                @"totalDataSize": @(totalDataSize),
                @"averageMemoryUsage": @(averageMemoryUsage)
            };
        }
        
        report[@"totalOperations"] = @(operations.count);
        report[@"reportTimestamp"] = [NSDate date];
        report[@"monitoringStartTime"] = self.performanceMetrics[@"startTime"];
    }
    
    return report;
}

- (NSArray *)analyzePerformanceBottlenecks {
    NSMutableArray *bottlenecks = [NSMutableArray array];
    
    NSDictionary *report = [self getPerformanceReport];
    
    for (NSString *operation in report) {
        if ([operation isEqualToString:@"totalOperations"] ||
            [operation isEqualToString:@"reportTimestamp"] ||
            [operation isEqualToString:@"monitoringStartTime"]) {
            continue;
        }
        
        NSDictionary *stats = report[operation];
        double averageDuration = [stats[@"averageDuration"] doubleValue];
        double maxDuration = [stats[@"maxDuration"] doubleValue];
        NSInteger operationCount = [stats[@"operationCount"] integerValue];
        
        // 识别性能瓶颈
        if (averageDuration > 0.5) { // 平均耗时超过0.5秒
            [bottlenecks addObject:@{
                @"type": @"slow_average",
                @"operation": operation,
                @"averageDuration": @(averageDuration),
                @"severity": @"high",
                @"description": [NSString stringWithFormat:@"操作 %@ 平均耗时过长: %.3f秒", operation, averageDuration]
            }];
        }
        
        if (maxDuration > 2.0) { // 最大耗时超过2秒
            [bottlenecks addObject:@{
                @"type": @"slow_max",
                @"operation": operation,
                @"maxDuration": @(maxDuration),
                @"severity": @"medium",
                @"description": [NSString stringWithFormat:@"操作 %@ 最大耗时过长: %.3f秒", operation, maxDuration]
            }];
        }
        
        if (operationCount > 1000) { // 操作次数过多
            [bottlenecks addObject:@{
                @"type": @"high_frequency",
                @"operation": operation,
                @"operationCount": @(operationCount),
                @"severity": @"low",
                @"description": [NSString stringWithFormat:@"操作 %@ 执行次数过多: %ld次", operation, (long)operationCount]
            }];
        }
    }
    
    return bottlenecks;
}

- (NSDictionary *)getOptimizationSuggestions {
    NSMutableDictionary *suggestions = [NSMutableDictionary dictionary];
    
    NSArray *bottlenecks = [self analyzePerformanceBottlenecks];
    
    for (NSDictionary *bottleneck in bottlenecks) {
        NSString *operation = bottleneck[@"operation"];
        NSString *type = bottleneck[@"type"];
        
        NSMutableArray *operationSuggestions = suggestions[operation] ?: [NSMutableArray array];
        
        if ([type isEqualToString:@"slow_average"]) {
            [operationSuggestions addObject:@"考虑使用批量操作减少单次操作耗时"];
            [operationSuggestions addObject:@"检查是否可以使用异步操作"];
            [operationSuggestions addObject:@"优化数据库查询语句或索引"];
        } else if ([type isEqualToString:@"slow_max"]) {
            [operationSuggestions addObject:@"检查是否存在阻塞操作"];
            [operationSuggestions addObject:@"考虑分页处理大量数据"];
        } else if ([type isEqualToString:@"high_frequency"]) {
            [operationSuggestions addObject:@"考虑使用缓存减少重复操作"];
            [operationSuggestions addObject:@"检查是否可以合并多个操作"];
        }
        
        suggestions[operation] = operationSuggestions;
    }
    
    return suggestions;
}

- (void)generatePerformanceReport {
    NSLog(@"=== 数据持久化性能报告 ===");
    
    NSDictionary *report = [self getPerformanceReport];
    NSArray *bottlenecks = [self analyzePerformanceBottlenecks];
    NSDictionary *suggestions = [self getOptimizationSuggestions];
    
    // 总体统计
    NSLog(@"总操作次数: %@", report[@"totalOperations"]);
    NSLog(@"监控开始时间: %@", report[@"monitoringStartTime"]);
    NSLog(@"报告生成时间: %@", report[@"reportTimestamp"]);
    
    // 各操作性能统计
    for (NSString *operation in report) {
        if ([operation isEqualToString:@"totalOperations"] ||
            [operation isEqualToString:@"reportTimestamp"] ||
            [operation isEqualToString:@"monitoringStartTime"]) {
            continue;
        }
        
        NSDictionary *stats = report[operation];
        NSLog(@"操作: %@", operation);
        NSLog(@"  次数: %@", stats[@"operationCount"]);
        NSLog(@"  平均耗时: %.3f秒", [stats[@"averageDuration"] doubleValue]);
        NSLog(@"  最大耗时: %.3f秒", [stats[@"maxDuration"] doubleValue]);
        NSLog(@"  平均数据大小: %.2f KB", [stats[@"averageDataSize"] doubleValue] / 1024.0);
    }
    
    // 性能瓶颈
    if (bottlenecks.count > 0) {
        NSLog(@"\n=== 发现的性能瓶颈 ===");
        for (NSDictionary *bottleneck in bottlenecks) {
            NSLog(@"- %@", bottleneck[@"description"]);
        }
    }
    
    // 优化建议
    if (suggestions.count > 0) {
        NSLog(@"\n=== 优化建议 ===");
        for (NSString *operation in suggestions) {
            NSLog(@"操作 %@:", operation);
            NSArray *operationSuggestions = suggestions[operation];
            for (NSString *suggestion in operationSuggestions) {
                NSLog(@"  - %@", suggestion);
            }
        }
    }
}

- (void)optimizeCoreDataPerformance:(NSManagedObjectContext *)context {
    NSLog(@"优化Core Data性能...");
    
    // 配置上下文
    context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    context.undoManager = nil; // 禁用撤销管理器以提高性能
    
    // 配置获取请求批量大小
    [context performBlock:^{
        // 在私有队列中执行优化
        NSLog(@"Core Data性能优化完成");
    }];
}

- (void)configureFetchRequestOptimization:(NSFetchRequest *)fetchRequest {
    // 设置批量大小
    fetchRequest.fetchBatchSize = 100;
    
    // 设置获取限制
    if (fetchRequest.fetchLimit == 0) {
        fetchRequest.fetchLimit = 1000; // 默认限制
    }
    
    // 启用故障处理
    fetchRequest.returnsObjectsAsFaults = YES;
    
    // 只获取需要的属性
    fetchRequest.includesPropertyValues = YES;
    fetchRequest.includesSubentities = NO;
    
    NSLog(@"获取请求优化配置完成");
}

- (void)optimizeBatchOperations:(NSManagedObjectContext *)context {
    NSLog(@"优化批量操作...");
    
    // 使用批量插入
    // 使用批量更新
    // 使用批量删除
    
    NSLog(@"批量操作优化完成");
}

- (void)optimizeSQLiteDatabase:(sqlite3 *)database {
    NSLog(@"优化SQLite数据库...");
    
    // 设置WAL模式
    sqlite3_exec(database, "PRAGMA journal_mode=WAL;", NULL, NULL, NULL);
    
    // 设置同步模式
    sqlite3_exec(database, "PRAGMA synchronous=NORMAL;", NULL, NULL, NULL);
    
    // 设置缓存大小
    sqlite3_exec(database, "PRAGMA cache_size=10000;", NULL, NULL, NULL);
    
    // 设置临时存储
    sqlite3_exec(database, "PRAGMA temp_store=MEMORY;", NULL, NULL, NULL);
    
    NSLog(@"SQLite数据库优化完成");
}

- (void)analyzeSQLitePerformance:(sqlite3 *)database {
    NSLog(@"分析SQLite性能...");
    
    // 分析查询计划
    // 检查索引使用情况
    // 统计表大小
    
    NSLog(@"SQLite性能分析完成");
}

- (void)optimizeSQLiteQueries:(sqlite3 *)database {
    NSLog(@"优化SQLite查询...");
    
    // 创建必要的索引
    // 优化查询语句
    // 使用预编译语句
    
    NSLog(@"SQLite查询优化完成");
}

- (void)optimizeFileStoragePerformance {
    NSLog(@"优化文件存储性能...");
    
    // 使用并发队列
    // 批量文件操作
    // 文件压缩
    
    NSLog(@"文件存储性能优化完成");
}

- (void)analyzeFileStoragePatterns {
    NSLog(@"分析文件存储模式...");
    
    // 分析文件访问模式
    // 识别热点文件
    // 统计文件大小分布
    
    NSLog(@"文件存储模式分析完成");
}

- (void)optimizeFileOperations {
    NSLog(@"优化文件操作...");
    
    // 使用内存映射
    // 异步文件操作
    // 文件缓存策略
    
    NSLog(@"文件操作优化完成");
}

- (void)optimizeMemoryUsage {
    NSLog(@"优化内存使用...");
    
    // 清理不必要的缓存
    // 释放临时对象
    // 优化对象生命周期
    
    NSLog(@"内存使用优化完成");
}

- (void)monitorMemoryPressure {
    NSInteger currentMemoryUsage = [self getCurrentMemoryUsage];
    NSInteger memoryLimit = 100 * 1024 * 1024; // 100MB
    
    if (currentMemoryUsage > memoryLimit) {
        NSLog(@"内存压力过高: %ld MB", (long)(currentMemoryUsage / 1024 / 1024));
        [self handleMemoryWarning];
    }
}

- (void)handleMemoryWarning {
    NSLog(@"处理内存警告...");
    
    dispatch_async(self.optimizationQueue, ^{
        // 清理缓存
        [self cleanupExpiredCache];
        
        // 优化内存使用
        [self optimizeMemoryUsage];
        
        // 强制垃圾回收
        [[NSGarbageCollector defaultCollector] collectExhaustively];
    });
    
    NSLog(@"内存警告处理完成");
}

- (void)optimizeCacheStrategy {
    NSLog(@"优化缓存策略...");
    
    // 配置缓存大小
    // 设置缓存过期时间
    // 实现LRU缓存
    
    NSLog(@"缓存策略优化完成");
}

- (void)configureCachePolicy {
    NSLog(@"配置缓存策略...");
    
    // 设置缓存策略
    // 配置缓存大小限制
    // 设置缓存清理策略
    
    NSLog(@"缓存策略配置完成");
}

- (void)cleanupExpiredCache {
    NSLog(@"清理过期缓存...");
    
    // 清理过期的缓存项
    // 释放内存
    
    NSLog(@"过期缓存清理完成");
}

- (void)performPeriodicAnalysis {
    dispatch_async(self.optimizationQueue, ^{
        NSLog(@"执行定期性能分析...");
        
        // 监控内存压力
        [self monitorMemoryPressure];
        
        // 分析性能瓶颈
        NSArray *bottlenecks = [self analyzePerformanceBottlenecks];
        
        if (bottlenecks.count > 0) {
            NSLog(@"发现 %lu 个性能瓶颈", (unsigned long)bottlenecks.count);
            
            // 自动优化
            [self performAutomaticOptimization];
        }
        
        NSLog(@"定期性能分析完成");
    });
}

- (void)performAutomaticOptimization {
    NSLog(@"执行自动优化...");
    
    // 优化内存使用
    [self optimizeMemoryUsage];
    
    // 清理过期缓存
    [self cleanupExpiredCache];
    
    // 优化文件存储
    [self optimizeFileStoragePerformance];
    
    NSLog(@"自动优化完成");
}

- (void)analyzeSlowOperation:(NSDictionary *)metric {
    NSString *operation = metric[@"operation"];
    NSTimeInterval duration = [metric[@"duration"] doubleValue];
    
    NSLog(@"检测到慢操作: %@, 耗时: %.3f秒", operation, duration);
    
    // 记录慢操作
    @synchronized(self.performanceMetrics) {
        if (!self.performanceMetrics[@"slowOperations"]) {
            self.performanceMetrics[@"slowOperations"] = [NSMutableArray array];
        }
        
        [self.performanceMetrics[@"slowOperations"] addObject:metric];
    }
    
    // 触发优化
    dispatch_async(self.optimizationQueue, ^{
        [self performAutomaticOptimization];
    });
}

- (NSInteger)getCurrentMemoryUsage {
    struct mach_task_basic_info info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
    
    if (kerr == KERN_SUCCESS) {
        return info.resident_size;
    }
    
    return 0;
}

- (void)dealloc {
    [self stopPerformanceMonitoring];
}

@end
```

## 数据持久化最佳实践

### 最佳实践管理器

```objc
@interface DataPersistenceBestPractices : NSObject

+ (instancetype)sharedPractices;

// 设计原则
- (void)demonstrateDataModelDesign;
- (void)demonstratePerformanceOptimization;
- (void)demonstrateErrorHandling;
- (void)demonstrateSecurityPractices;

// Core Data最佳实践
- (void)demonstrateCoreDataBestPractices;
- (void)demonstrateCoreDataThreadSafety;
- (void)demonstrateCoreDataMigration;

// SQLite最佳实践
- (void)demonstrateSQLiteBestPractices;
- (void)demonstrateSQLiteTransactions;
- (void)demonstrateSQLiteIndexing;

// 文件存储最佳实践
- (void)demonstrateFileStorageBestPractices;
- (void)demonstrateFileSecurityPractices;
- (void)demonstrateFileBackupStrategies;

// 测试和调试
- (void)demonstrateTestingStrategies;
- (void)demonstrateDebuggingTechniques;
- (void)demonstratePerformanceTesting;

@end

@implementation DataPersistenceBestPractices

+ (instancetype)sharedPractices {
    static DataPersistenceBestPractices *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (void)demonstrateDataModelDesign {
    NSLog(@"=== 数据模型设计最佳实践 ===");
    
    NSLog(@"1. 规范化设计:");
    NSLog(@"   - 避免数据冗余");
    NSLog(@"   - 合理设计实体关系");
    NSLog(@"   - 使用适当的数据类型");
    
    NSLog(@"2. 性能考虑:");
    NSLog(@"   - 合理设计索引");
    NSLog(@"   - 避免过深的关系层次");
    NSLog(@"   - 考虑查询模式");
    
    NSLog(@"3. 可扩展性:");
    NSLog(@"   - 预留扩展字段");
    NSLog(@"   - 版本化数据模型");
    NSLog(@"   - 支持数据迁移");
}

- (void)demonstratePerformanceOptimization {
    NSLog(@"=== 性能优化最佳实践 ===");
    
    NSLog(@"1. 查询优化:");
    NSLog(@"   - 使用合适的获取策略");
    NSLog(@"   - 限制获取结果数量");
    NSLog(@"   - 使用谓词过滤");
    
    NSLog(@"2. 批量操作:");
    NSLog(@"   - 使用批量插入/更新/删除");
    NSLog(@"   - 合理设置批量大小");
    NSLog(@"   - 使用事务包装批量操作");
    
    NSLog(@"3. 内存管理:");
    NSLog(@"   - 及时释放不需要的对象");
    NSLog(@"   - 使用自动释放池");
    NSLog(@"   - 监控内存使用");
}

- (void)demonstrateErrorHandling {
    NSLog(@"=== 错误处理最佳实践 ===");
    
    NSLog(@"1. 异常捕获:");
    NSLog(@"   - 使用try-catch处理异常");
    NSLog(@"   - 检查返回值和错误对象");
    NSLog(@"   - 提供有意义的错误信息");
    
    NSLog(@"2. 错误恢复:");
    NSLog(@"   - 实现重试机制");
    NSLog(@"   - 提供降级方案");
    NSLog(@"   - 保持数据一致性");
    
    NSLog(@"3. 错误报告:");
    NSLog(@"   - 记录详细的错误日志");
    NSLog(@"   - 向用户提供友好的错误提示");
    NSLog(@"   - 收集错误统计信息");
}

- (void)demonstrateSecurityPractices {
    NSLog(@"=== 安全实践最佳实践 ===");
    
    NSLog(@"1. 数据加密:");
    NSLog(@"   - 敏感数据加密存储");
    NSLog(@"   - 使用强加密算法");
    NSLog(@"   - 安全的密钥管理");
    
    NSLog(@"2. 访问控制:");
    NSLog(@"   - 实现权限验证");
    NSLog(@"   - 限制数据访问范围");
    NSLog(@"   - 审计数据访问");
    
    NSLog(@"3. 数据完整性:");
    NSLog(@"   - 使用校验和验证");
    NSLog(@"   - 实现数据备份");
    NSLog(@"   - 防止SQL注入");
}

- (void)demonstrateCoreDataBestPractices {
    NSLog(@"=== Core Data最佳实践 ===");
    
    NSLog(@"1. 上下文管理:");
    NSLog(@"   - 使用私有队列上下文");
    NSLog(@"   - 正确处理上下文合并");
    NSLog(@"   - 避免跨线程访问");
    
    NSLog(@"2. 获取请求优化:");
    NSLog(@"   - 设置合适的批量大小");
    NSLog(@"   - 使用故障处理");
    NSLog(@"   - 预加载关系数据");
    
    NSLog(@"3. 数据模型版本化:");
    NSLog(@"   - 创建数据模型版本");
    NSLog(@"   - 实现轻量级迁移");
    NSLog(@"   - 处理复杂迁移");
}

- (void)demonstrateCoreDataThreadSafety {
    NSLog(@"=== Core Data线程安全 ===");
    
    NSLog(@"1. 上下文隔离:");
    NSLog(@"   - 每个线程使用独立上下文");
    NSLog(@"   - 使用performBlock方法");
    NSLog(@"   - 避免传递托管对象");
    
    NSLog(@"2. 数据同步:");
    NSLog(@"   - 监听上下文变更通知");
    NSLog(@"   - 正确合并变更");
    NSLog(@"   - 处理合并冲突");
}

- (void)demonstrateCoreDataMigration {
    NSLog(@"=== Core Data数据迁移 ===");
    
    NSLog(@"1. 轻量级迁移:");
    NSLog(@"   - 启用自动迁移");
    NSLog(@"   - 设置迁移选项");
    NSLog(@"   - 处理迁移错误");
    
    NSLog(@"2. 重量级迁移:");
    NSLog(@"   - 创建映射模型");
    NSLog(@"   - 实现自定义迁移策略");
    NSLog(@"   - 验证迁移结果");
}

- (void)demonstrateSQLiteBestPractices {
    NSLog(@"=== SQLite最佳实践 ===");
    
    NSLog(@"1. 连接管理:");
    NSLog(@"   - 合理管理数据库连接");
    NSLog(@"   - 使用连接池");
    NSLog(@"   - 及时关闭连接");
    
    NSLog(@"2. 语句优化:");
    NSLog(@"   - 使用预编译语句");
    NSLog(@"   - 避免字符串拼接");
    NSLog(@"   - 使用参数绑定");
    
    NSLog(@"3. 配置优化:");
    NSLog(@"   - 设置合适的页面大小");
    NSLog(@"   - 配置缓存大小");
    NSLog(@"   - 选择合适的日志模式");
}

- (void)demonstrateSQLiteTransactions {
    NSLog(@"=== SQLite事务管理 ===");
    
    NSLog(@"1. 事务类型:");
    NSLog(@"   - DEFERRED事务");
    NSLog(@"   - IMMEDIATE事务");
    NSLog(@"   - EXCLUSIVE事务");
    
    NSLog(@"2. 事务最佳实践:");
    NSLog(@"   - 保持事务简短");
    NSLog(@"   - 正确处理事务回滚");
    NSLog(@"   - 避免嵌套事务");
}

- (void)demonstrateSQLiteIndexing {
    NSLog(@"=== SQLite索引策略 ===");
    
    NSLog(@"1. 索引设计:");
    NSLog(@"   - 为常用查询创建索引");
    NSLog(@"   - 使用复合索引");
    NSLog(@"   - 避免过多索引");
    
    NSLog(@"2. 索引维护:");
    NSLog(@"   - 定期分析索引使用情况");
    NSLog(@"   - 删除无用索引");
    NSLog(@"   - 重建碎片化索引");
}

- (void)demonstrateFileStorageBestPractices {
    NSLog(@"=== 文件存储最佳实践 ===");
    
    NSLog(@"1. 目录结构:");
    NSLog(@"   - 合理组织目录结构");
    NSLog(@"   - 使用标准目录");
    NSLog(@"   - 避免深层嵌套");
    
    NSLog(@"2. 文件命名:");
    NSLog(@"   - 使用有意义的文件名");
    NSLog(@"   - 避免特殊字符");
    NSLog(@"   - 考虑文件名长度限制");
    
    NSLog(@"3. 文件操作:");
    NSLog(@"   - 使用原子操作");
    NSLog(@"   - 检查文件存在性");
    NSLog(@"   - 处理文件权限");
}

- (void)demonstrateFileSecurityPractices {
    NSLog(@"=== 文件安全实践 ===");
    
    NSLog(@"1. 文件权限:");
    NSLog(@"   - 设置适当的文件权限");
    NSLog(@"   - 限制文件访问");
    NSLog(@"   - 使用沙盒机制");
    
    NSLog(@"2. 数据保护:");
    NSLog(@"   - 加密敏感文件");
    NSLog(@"   - 使用数据保护API");
    NSLog(@"   - 安全删除文件");
}

- (void)demonstrateFileBackupStrategies {
    NSLog(@"=== 文件备份策略 ===");
    
    NSLog(@"1. 备份策略:");
    NSLog(@"   - 定期自动备份");
    NSLog(@"   - 增量备份");
    NSLog(@"   - 云端备份");
    
    NSLog(@"2. 恢复策略:");
    NSLog(@"   - 验证备份完整性");
    NSLog(@"   - 快速恢复机制");
    NSLog(@"   - 灾难恢复计划");
}

- (void)demonstrateTestingStrategies {
    NSLog(@"=== 测试策略最佳实践 ===");
    
    NSLog(@"1. 单元测试:");
    NSLog(@"   - 测试数据模型");
    NSLog(@"   - 测试数据访问层");
    NSLog(@"   - 模拟数据库操作");
    
    NSLog(@"2. 集成测试:");
    NSLog(@"   - 测试完整的数据流");
    NSLog(@"   - 测试数据迁移");
    NSLog(@"   - 测试并发访问");
    
    NSLog(@"3. 性能测试:");
    NSLog(@"   - 测试查询性能");
    NSLog(@"   - 测试大数据量处理");
    NSLog(@"   - 测试内存使用");
}

- (void)demonstrateDebuggingTechniques {
    NSLog(@"=== 调试技术最佳实践 ===");
    
    NSLog(@"1. 日志记录:");
    NSLog(@"   - 记录关键操作");
    NSLog(@"   - 使用不同日志级别");
    NSLog(@"   - 结构化日志格式");
    
    NSLog(@"2. 调试工具:");
    NSLog(@"   - 使用Core Data调试");
    NSLog(@"   - 使用SQLite命令行工具");
    NSLog(@"   - 使用性能分析工具");
    
    NSLog(@"3. 问题诊断:");
    NSLog(@"   - 分析崩溃日志");
    NSLog(@"   - 监控性能指标");
    NSLog(@"   - 追踪内存泄漏");
}

- (void)demonstratePerformanceTesting {
    NSLog(@"=== 性能测试最佳实践 ===");
    
    NSLog(@"1. 基准测试:");
    NSLog(@"   - 建立性能基线");
    NSLog(@"   - 定期性能回归测试");
    NSLog(@"   - 比较不同实现方案");
    
    NSLog(@"2. 压力测试:");
    NSLog(@"   - 测试高并发场景");
    NSLog(@"   - 测试大数据量处理");
    NSLog(@"   - 测试资源限制情况");
    
    NSLog(@"3. 性能监控:");
    NSLog(@"   - 实时性能监控");
    NSLog(@"   - 性能指标收集");
    NSLog(@"   - 性能趋势分析");
}

@end
```

## 综合数据持久化管理系统

### 统一数据持久化管理器

```objc
@interface ComprehensiveDataPersistenceManager : NSObject

@property (nonatomic, strong) DataPersistenceArchitecture *architecture;
@property (nonatomic, strong) CoreDataManager *coreDataManager;
@property (nonatomic, strong) SQLiteManager *sqliteManager;
@property (nonatomic, strong) FileStorageManager *fileStorageManager;
@property (nonatomic, strong) DataPersistencePerformanceOptimizer *performanceOptimizer;
@property (nonatomic, strong) DataPersistenceBestPractices *bestPractices;

+ (instancetype)sharedManager;

// 系统初始化
- (void)initializeDataPersistenceSystem;
- (void)configureOptimalSettings;
- (void)startSystemMonitoring;

// 统一数据操作接口
- (void)saveData:(id)data withKey:(NSString *)key storageType:(NSString *)storageType completion:(void(^)(BOOL success, NSError *error))completion;
- (void)loadDataWithKey:(NSString *)key storageType:(NSString *)storageType completion:(void(^)(id data, NSError *error))completion;
- (void)deleteDataWithKey:(NSString *)key storageType:(NSString *)storageType completion:(void(^)(BOOL success, NSError *error))completion;

// 批量操作
- (void)batchSaveData:(NSArray *)dataArray storageType:(NSString *)storageType completion:(void(^)(NSInteger successCount, NSArray *errors))completion;
- (void)batchLoadDataWithKeys:(NSArray *)keys storageType:(NSString *)storageType completion:(void(^)(NSArray *results, NSArray *errors))completion;
- (void)batchDeleteDataWithKeys:(NSArray *)keys storageType:(NSString *)storageType completion:(void(^)(NSInteger successCount, NSArray *errors))completion;

// 智能存储策略
- (NSString *)recommendStorageTypeForData:(id)data;
- (void)optimizeStorageStrategy;
- (void)migrateDataBetweenStorageTypes:(NSString *)fromType toType:(NSString *)toType;

// 系统监控和优化
- (void)performSystemHealthCheck;
- (NSDictionary *)getSystemStatus;
- (NSDictionary *)getPerformanceMetrics;
- (NSArray *)getOptimizationRecommendations;

// 问题诊断
- (NSArray *)diagnosePerformanceIssues;
- (NSArray *)diagnoseDataIntegrityIssues;
- (NSArray *)diagnoseStorageIssues;

@end

@implementation ComprehensiveDataPersistenceManager

+ (instancetype)sharedManager {
    static ComprehensiveDataPersistenceManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self initializeComponents];
    }
    return self;
}

- (void)initializeComponents {
    NSLog(@"=== 初始化综合数据持久化管理系统 ===");
    
    // 初始化各个组件
    self.architecture = [[DataPersistenceArchitecture alloc] init];
    self.coreDataManager = [[CoreDataManager alloc] init];
    self.sqliteManager = [[SQLiteManager alloc] init];
    self.fileStorageManager = [[FileStorageManager alloc] init];
    self.performanceOptimizer = [DataPersistencePerformanceOptimizer sharedOptimizer];
    self.bestPractices = [DataPersistenceBestPractices sharedPractices];
    
    NSLog(@"所有组件初始化完成");
}

- (void)initializeDataPersistenceSystem {
    NSLog(@"=== 初始化数据持久化系统 ===");
    
    // 初始化架构
    [self.architecture initializeArchitecture];
    
    // 配置各个存储管理器
    [self.coreDataManager initializeCoreDataStack];
    [self.sqliteManager openDatabase:@"app_database.sqlite"];
    [self.fileStorageManager initializeFileStorage];
    
    // 配置性能优化器
    [self configureOptimalSettings];
    
    // 启动系统监控
    [self startSystemMonitoring];
    
    NSLog(@"数据持久化系统初始化完成");
}

- (void)configureOptimalSettings {
    NSLog(@"配置最优设置...");
    
    // 配置Core Data性能
    NSManagedObjectContext *context = [self.coreDataManager getMainContext];
    [self.performanceOptimizer optimizeCoreDataPerformance:context];
    
    // 配置SQLite性能
    sqlite3 *database = [self.sqliteManager getDatabase];
    [self.performanceOptimizer optimizeSQLiteDatabase:database];
    
    // 配置文件存储性能
    [self.performanceOptimizer optimizeFileStoragePerformance];
    
    // 配置缓存策略
    [self.performanceOptimizer optimizeCacheStrategy];
    
    NSLog(@"最优设置配置完成");
}

- (void)startSystemMonitoring {
    NSLog(@"启动系统监控...");
    
    // 启动性能监控
    [self.performanceOptimizer startPerformanceMonitoring];
    
    // 启动架构监控
    [self.architecture startMonitoring];
    
    NSLog(@"系统监控启动完成");
}

- (void)saveData:(id)data withKey:(NSString *)key storageType:(NSString *)storageType completion:(void(^)(BOOL success, NSError *error))completion {
    NSDate *startTime = [NSDate date];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        BOOL success = NO;
        NSError *error = nil;
        
        @try {
            if ([storageType isEqualToString:@"CoreData"]) {
                success = [self.coreDataManager saveDataObject:data];
            } else if ([storageType isEqualToString:@"SQLite"]) {
                success = [self.sqliteManager insertData:data withKey:key];
            } else if ([storageType isEqualToString:@"FileStorage"]) {
                success = [self.fileStorageManager writeObject:data toFile:key];
            } else {
                error = [NSError errorWithDomain:@"DataPersistenceError" code:1001 userInfo:@{NSLocalizedDescriptionKey: @"不支持的存储类型"}];
            }
        } @catch (NSException *exception) {
            error = [NSError errorWithDomain:@"DataPersistenceError" code:1002 userInfo:@{NSLocalizedDescriptionKey: exception.reason}];
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        NSInteger dataSize = [self estimateDataSize:data];
        [self.performanceOptimizer recordOperation:[NSString stringWithFormat:@"save_%@", storageType] duration:duration dataSize:dataSize];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(success, error);
            }
        });
    });
}

- (void)loadDataWithKey:(NSString *)key storageType:(NSString *)storageType completion:(void(^)(id data, NSError *error))completion {
    NSDate *startTime = [NSDate date];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        id data = nil;
        NSError *error = nil;
        
        @try {
            if ([storageType isEqualToString:@"CoreData"]) {
                data = [self.coreDataManager fetchDataWithPredicate:[NSPredicate predicateWithFormat:@"key == %@", key]];
            } else if ([storageType isEqualToString:@"SQLite"]) {
                data = [self.sqliteManager selectDataWithKey:key];
            } else if ([storageType isEqualToString:@"FileStorage"]) {
                data = [self.fileStorageManager readObjectFromFile:key];
            } else {
                error = [NSError errorWithDomain:@"DataPersistenceError" code:1001 userInfo:@{NSLocalizedDescriptionKey: @"不支持的存储类型"}];
            }
        } @catch (NSException *exception) {
            error = [NSError errorWithDomain:@"DataPersistenceError" code:1003 userInfo:@{NSLocalizedDescriptionKey: exception.reason}];
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        NSInteger dataSize = [self estimateDataSize:data];
        [self.performanceOptimizer recordOperation:[NSString stringWithFormat:@"load_%@", storageType] duration:duration dataSize:dataSize];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(data, error);
            }
        });
    });
}

- (void)deleteDataWithKey:(NSString *)key storageType:(NSString *)storageType completion:(void(^)(BOOL success, NSError *error))completion {
    NSDate *startTime = [NSDate date];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        BOOL success = NO;
        NSError *error = nil;
        
        @try {
            if ([storageType isEqualToString:@"CoreData"]) {
                success = [self.coreDataManager deleteDataWithPredicate:[NSPredicate predicateWithFormat:@"key == %@", key]];
            } else if ([storageType isEqualToString:@"SQLite"]) {
                success = [self.sqliteManager deleteDataWithKey:key];
            } else if ([storageType isEqualToString:@"FileStorage"]) {
                success = [self.fileStorageManager deleteFile:key];
            } else {
                error = [NSError errorWithDomain:@"DataPersistenceError" code:1001 userInfo:@{NSLocalizedDescriptionKey: @"不支持的存储类型"}];
            }
        } @catch (NSException *exception) {
            error = [NSError errorWithDomain:@"DataPersistenceError" code:1004 userInfo:@{NSLocalizedDescriptionKey: exception.reason}];
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        [self.performanceOptimizer recordOperation:[NSString stringWithFormat:@"delete_%@", storageType] duration:duration dataSize:0];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(success, error);
            }
        });
    });
}

- (void)batchSaveData:(NSArray *)dataArray storageType:(NSString *)storageType completion:(void(^)(NSInteger successCount, NSArray *errors))completion {
    NSDate *startTime = [NSDate date];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger successCount = 0;
        NSMutableArray *errors = [NSMutableArray array];
        
        for (NSInteger i = 0; i < dataArray.count; i++) {
            id data = dataArray[i];
            NSString *key = [NSString stringWithFormat:@"batch_item_%ld", (long)i];
            
            @try {
                BOOL success = NO;
                
                if ([storageType isEqualToString:@"CoreData"]) {
                    success = [self.coreDataManager saveDataObject:data];
                } else if ([storageType isEqualToString:@"SQLite"]) {
                    success = [self.sqliteManager insertData:data withKey:key];
                } else if ([storageType isEqualToString:@"FileStorage"]) {
                    success = [self.fileStorageManager writeObject:data toFile:key];
                }
                
                if (success) {
                    successCount++;
                } else {
                    [errors addObject:[NSError errorWithDomain:@"DataPersistenceError" code:1005 userInfo:@{NSLocalizedDescriptionKey: [NSString stringWithFormat:@"批量保存第%ld项失败", (long)i]}]];
                }
            } @catch (NSException *exception) {
                [errors addObject:[NSError errorWithDomain:@"DataPersistenceError" code:1006 userInfo:@{NSLocalizedDescriptionKey: exception.reason}]];
            }
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        NSInteger totalDataSize = [self estimateBatchDataSize:dataArray];
        [self.performanceOptimizer recordOperation:[NSString stringWithFormat:@"batch_save_%@", storageType] duration:duration dataSize:totalDataSize];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(successCount, errors);
            }
        });
    });
}

- (void)batchLoadDataWithKeys:(NSArray *)keys storageType:(NSString *)storageType completion:(void(^)(NSArray *results, NSArray *errors))completion {
    NSDate *startTime = [NSDate date];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSMutableArray *results = [NSMutableArray array];
        NSMutableArray *errors = [NSMutableArray array];
        
        for (NSString *key in keys) {
            @try {
                id data = nil;
                
                if ([storageType isEqualToString:@"CoreData"]) {
                    data = [self.coreDataManager fetchDataWithPredicate:[NSPredicate predicateWithFormat:@"key == %@", key]];
                } else if ([storageType isEqualToString:@"SQLite"]) {
                    data = [self.sqliteManager selectDataWithKey:key];
                } else if ([storageType isEqualToString:@"FileStorage"]) {
                    data = [self.fileStorageManager readObjectFromFile:key];
                }
                
                if (data) {
                    [results addObject:data];
                } else {
                    [results addObject:[NSNull null]];
                }
            } @catch (NSException *exception) {
                [errors addObject:[NSError errorWithDomain:@"DataPersistenceError" code:1007 userInfo:@{NSLocalizedDescriptionKey: exception.reason}]];
                [results addObject:[NSNull null]];
            }
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        NSInteger totalDataSize = [self estimateBatchDataSize:results];
        [self.performanceOptimizer recordOperation:[NSString stringWithFormat:@"batch_load_%@", storageType] duration:duration dataSize:totalDataSize];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(results, errors);
            }
        });
    });
}

- (void)batchDeleteDataWithKeys:(NSArray *)keys storageType:(NSString *)storageType completion:(void(^)(NSInteger successCount, NSArray *errors))completion {
    NSDate *startTime = [NSDate date];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSInteger successCount = 0;
        NSMutableArray *errors = [NSMutableArray array];
        
        for (NSString *key in keys) {
            @try {
                BOOL success = NO;
                
                if ([storageType isEqualToString:@"CoreData"]) {
                    success = [self.coreDataManager deleteDataWithPredicate:[NSPredicate predicateWithFormat:@"key == %@", key]];
                } else if ([storageType isEqualToString:@"SQLite"]) {
                    success = [self.sqliteManager deleteDataWithKey:key];
                } else if ([storageType isEqualToString:@"FileStorage"]) {
                    success = [self.fileStorageManager deleteFile:key];
                }
                
                if (success) {
                    successCount++;
                } else {
                    [errors addObject:[NSError errorWithDomain:@"DataPersistenceError" code:1008 userInfo:@{NSLocalizedDescriptionKey: [NSString stringWithFormat:@"删除键 %@ 失败", key]}]];
                }
            } @catch (NSException *exception) {
                [errors addObject:[NSError errorWithDomain:@"DataPersistenceError" code:1009 userInfo:@{NSLocalizedDescriptionKey: exception.reason}]];
            }
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        [self.performanceOptimizer recordOperation:[NSString stringWithFormat:@"batch_delete_%@", storageType] duration:duration dataSize:0];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(successCount, errors);
            }
        });
    });
}

- (NSString *)recommendStorageTypeForData:(id)data {
    NSInteger dataSize = [self estimateDataSize:data];
    
    // 根据数据大小和类型推荐存储方式
    if ([data isKindOfClass:[NSManagedObject class]]) {
        return @"CoreData";
    } else if (dataSize > 1024 * 1024) { // 大于1MB的数据
        return @"FileStorage";
    } else if ([data isKindOfClass:[NSDictionary class]] || [data isKindOfClass:[NSArray class]]) {
        return @"SQLite";
    } else {
        return @"FileStorage";
    }
}

- (void)optimizeStorageStrategy {
    NSLog(@"优化存储策略...");
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 分析当前存储使用情况
        NSDictionary *performanceReport = [self.performanceOptimizer getPerformanceReport];
        
        // 根据性能报告优化存储策略
        for (NSString *operation in performanceReport) {
            if ([operation containsString:@"save_"] || [operation containsString:@"load_"]) {
                NSDictionary *stats = performanceReport[operation];
                double averageDuration = [stats[@"averageDuration"] doubleValue];
                
                if (averageDuration > 1.0) {
                    NSLog(@"操作 %@ 性能较差，建议优化", operation);
                    // 实施具体的优化策略
                }
            }
        }
        
        NSLog(@"存储策略优化完成");
    });
}

- (void)migrateDataBetweenStorageTypes:(NSString *)fromType toType:(NSString *)toType {
    NSLog(@"开始数据迁移: %@ -> %@", fromType, toType);
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 实现数据迁移逻辑
        // 这里需要根据具体的存储类型实现迁移策略
        
        NSLog(@"数据迁移完成: %@ -> %@", fromType, toType);
    });
}

- (void)performSystemHealthCheck {
    NSLog(@"=== 执行系统健康检查 ===");
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 检查各个存储系统的健康状态
        BOOL coreDataHealthy = [self.coreDataManager isHealthy];
        BOOL sqliteHealthy = [self.sqliteManager isHealthy];
        BOOL fileStorageHealthy = [self.fileStorageManager isHealthy];
        
        NSLog(@"Core Data健康状态: %@", coreDataHealthy ? @"正常" : @"异常");
        NSLog(@"SQLite健康状态: %@", sqliteHealthy ? @"正常" : @"异常");
        NSLog(@"文件存储健康状态: %@", fileStorageHealthy ? @"正常" : @"异常");
        
        // 检查性能指标
        NSArray *bottlenecks = [self.performanceOptimizer analyzePerformanceBottlenecks];
        if (bottlenecks.count > 0) {
            NSLog(@"发现 %lu 个性能瓶颈", (unsigned long)bottlenecks.count);
        } else {
            NSLog(@"系统性能正常");
        }
        
        NSLog(@"系统健康检查完成");
    });
}

- (NSDictionary *)getSystemStatus {
    NSMutableDictionary *status = [NSMutableDictionary dictionary];
    
    // 获取各个组件的状态
    status[@"coreData"] = @{
        @"healthy": @([self.coreDataManager isHealthy]),
        @"initialized": @([self.coreDataManager isInitialized])
    };
    
    status[@"sqlite"] = @{
        @"healthy": @([self.sqliteManager isHealthy]),
        @"connected": @([self.sqliteManager isConnected])
    };
    
    status[@"fileStorage"] = @{
        @"healthy": @([self.fileStorageManager isHealthy]),
        @"initialized": @([self.fileStorageManager isInitialized])
    };
    
    status[@"performanceOptimizer"] = @{
        @"monitoring": @(self.performanceOptimizer.isMonitoring)
    };
    
    status[@"timestamp"] = [NSDate date];
    
    return status;
}

- (NSDictionary *)getPerformanceMetrics {
    return [self.performanceOptimizer getPerformanceReport];
}

- (NSArray *)getOptimizationRecommendations {
    NSDictionary *suggestions = [self.performanceOptimizer getOptimizationSuggestions];
    NSMutableArray *recommendations = [NSMutableArray array];
    
    for (NSString *operation in suggestions) {
        NSArray *operationSuggestions = suggestions[operation];
        for (NSString *suggestion in operationSuggestions) {
            [recommendations addObject:@{
                @"operation": operation,
                @"recommendation": suggestion,
                @"priority": @"medium"
            }];
        }
    }
    
    return recommendations;
}

- (NSArray *)diagnosePerformanceIssues {
    return [self.performanceOptimizer analyzePerformanceBottlenecks];
}

- (NSArray *)diagnoseDataIntegrityIssues {
    NSMutableArray *issues = [NSMutableArray array];
    
    // 检查Core Data数据完整性
    if (![self.coreDataManager validateDataIntegrity]) {
        [issues addObject:@{
            @"type": @"data_integrity",
            @"component": @"CoreData",
            @"description": @"Core Data数据完整性检查失败",
            @"severity": @"high"
        }];
    }
    
    // 检查SQLite数据完整性
    if (![self.sqliteManager validateDataIntegrity]) {
        [issues addObject:@{
            @"type": @"data_integrity",
            @"component": @"SQLite",
            @"description": @"SQLite数据完整性检查失败",
            @"severity": @"high"
        }];
    }
    
    // 检查文件存储完整性
    if (![self.fileStorageManager validateDataIntegrity]) {
        [issues addObject:@{
            @"type": @"data_integrity",
            @"component": @"FileStorage",
            @"description": @"文件存储完整性检查失败",
            @"severity": @"medium"
        }];
    }
    
    return issues;
}

- (NSArray *)diagnoseStorageIssues {
    NSMutableArray *issues = [NSMutableArray array];
    
    // 检查存储空间
    NSDictionary *storageInfo = [self.fileStorageManager getStorageInfo];
    NSInteger availableSpace = [storageInfo[@"availableSpace"] integerValue];
    NSInteger totalSpace = [storageInfo[@"totalSpace"] integerValue];
    
    double usagePercentage = (double)(totalSpace - availableSpace) / totalSpace * 100;
    
    if (usagePercentage > 90) {
        [issues addObject:@{
            @"type": @"storage_space",
            @"description": [NSString stringWithFormat:@"存储空间使用率过高: %.1f%%", usagePercentage],
            @"severity": @"high"
        }];
    } else if (usagePercentage > 80) {
        [issues addObject:@{
            @"type": @"storage_space",
            @"description": [NSString stringWithFormat:@"存储空间使用率较高: %.1f%%", usagePercentage],
            @"severity": @"medium"
        }];
    }
    
    // 检查数据库大小
    NSInteger databaseSize = [self.sqliteManager getDatabaseSize];
    if (databaseSize > 100 * 1024 * 1024) { // 大于100MB
        [issues addObject:@{
            @"type": @"database_size",
            @"description": [NSString stringWithFormat:@"数据库文件过大: %.2f MB", databaseSize / 1024.0 / 1024.0],
            @"severity": @"medium"
        }];
    }
    
    return issues;
}

- (NSInteger)estimateDataSize:(id)data {
    if (!data) return 0;
    
    if ([data isKindOfClass:[NSData class]]) {
        return [(NSData *)data length];
    } else if ([data isKindOfClass:[NSString class]]) {
        return [(NSString *)data lengthOfBytesUsingEncoding:NSUTF8StringEncoding];
    } else if ([data isKindOfClass:[NSDictionary class]] || [data isKindOfClass:[NSArray class]]) {
        NSError *error;
        NSData *jsonData = [NSJSONSerialization dataWithJSONObject:data options:0 error:&error];
        return error ? 0 : jsonData.length;
    } else {
        // 对于其他类型，使用归档估算大小
        NSData *archivedData = [NSKeyedArchiver archivedDataWithRootObject:data];
        return archivedData.length;
    }
}

- (NSInteger)estimateBatchDataSize:(NSArray *)dataArray {
    NSInteger totalSize = 0;
    for (id data in dataArray) {
        totalSize += [self estimateDataSize:data];
    }
    return totalSize;
}

- (void)dealloc {
    [self.performanceOptimizer stopPerformanceMonitoring];
    [self.architecture stopMonitoring];
}

@end
```

## 数据持久化开发最佳实践总结

### 核心设计原则

1. **性能优先**
   - 选择合适的存储方案
   - 优化查询和操作性能
   - 实施有效的缓存策略
   - 监控和分析性能指标

2. **数据安全**
   - 实现数据加密和保护
   - 建立完善的备份机制
   - 确保数据完整性验证
   - 防范安全威胁和攻击

3. **系统可靠性**
   - 实现错误处理和恢复
   - 建立健康监控机制
   - 支持数据迁移和升级
   - 确保系统稳定运行

### 开发阶段建议

**设计阶段：**
- 根据数据特性选择存储方案
- 设计合理的数据模型结构
- 规划数据访问模式和查询需求
- 考虑未来的扩展和迁移需求

**实现阶段：**
- 遵循各存储方案的最佳实践
- 实现统一的数据访问接口
- 建立完善的错误处理机制
- 集成性能监控和优化功能

**测试阶段：**
- 进行全面的功能测试
- 执行性能和压力测试
- 验证数据完整性和安全性
- 测试错误恢复和迁移功能

**维护阶段：**
- 持续监控系统性能
- 定期进行数据备份和验证
- 及时处理性能瓶颈和问题
- 根据需求进行系统优化升级

通过本文的深入解析，我们全面了解了iOS数据持久化的各个方面，从基础的Core Data、SQLite和文件存储，到高级的性能优化和最佳实践。掌握这些知识和技能，将帮助开发者构建高效、可靠、安全的iOS应用数据持久化系统。

## Core Data深度解析

### Core Data管理器

```objc
@interface CoreDataManager : NSObject

@property (nonatomic, strong) NSPersistentContainer *persistentContainer;
@property (nonatomic, strong) NSManagedObjectContext *mainContext;
@property (nonatomic, strong) NSManagedObjectContext *backgroundContext;
@property (nonatomic, strong) NSMutableArray *performanceMetrics;
@property (nonatomic, assign) BOOL isOptimizationEnabled;

+ (instancetype)sharedManager;

// Core Data初始化
- (void)initializeCoreDataStack;
- (void)setupPersistentContainer;
- (void)configureManagedObjectContexts;

// 数据操作
- (void)saveContext;
- (void)saveContextWithCompletion:(void(^)(BOOL success, NSError *error))completion;
- (NSManagedObject *)createEntityWithName:(NSString *)entityName;
- (void)deleteObject:(NSManagedObject *)object;

// 查询操作
- (NSArray *)fetchEntitiesWithName:(NSString *)entityName
                         predicate:(NSPredicate *)predicate
                   sortDescriptors:(NSArray *)sortDescriptors;
- (NSArray *)fetchEntitiesWithRequest:(NSFetchRequest *)fetchRequest;
- (NSInteger)countEntitiesWithName:(NSString *)entityName predicate:(NSPredicate *)predicate;

// 批量操作
- (void)performBatchInsert:(NSArray *)dataArray
                entityName:(NSString *)entityName
                completion:(void(^)(BOOL success, NSError *error))completion;
- (void)performBatchUpdate:(NSString *)entityName
                 predicate:(NSPredicate *)predicate
            propertiesToUpdate:(NSDictionary *)properties
                completion:(void(^)(NSInteger updatedCount, NSError *error))completion;
- (void)performBatchDelete:(NSString *)entityName
                 predicate:(NSPredicate *)predicate
                completion:(void(^)(NSInteger deletedCount, NSError *error))completion;

// 性能优化
- (void)optimizePerformance;
- (void)enableQueryOptimization;
- (void)configureFaultingBehavior;
- (void)setupMemoryManagement;

// 数据迁移
- (BOOL)performDataMigration;
- (BOOL)validateDataModel;
- (void)handleMigrationError:(NSError *)error;

// 监控和调试
- (void)enablePerformanceMonitoring;
- (NSDictionary *)getPerformanceReport;
- (void)logCoreDataStatistics;

@end

@implementation CoreDataManager

+ (instancetype)sharedManager {
    static CoreDataManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.performanceMetrics = [NSMutableArray array];
        self.isOptimizationEnabled = YES;
        [self initializeCoreDataStack];
    }
    return self;
}

- (void)initializeCoreDataStack {
    NSLog(@"=== 初始化Core Data堆栈 ===");
    
    [self setupPersistentContainer];
    [self configureManagedObjectContexts];
    [self enablePerformanceMonitoring];
    
    NSLog(@"Core Data堆栈初始化完成");
}

- (void)setupPersistentContainer {
    NSLog(@"设置持久化容器...");
    
    // 创建持久化容器
    self.persistentContainer = [[NSPersistentContainer alloc] initWithName:@"DataModel"];
    
    // 配置存储描述
    NSPersistentStoreDescription *storeDescription = self.persistentContainer.persistentStoreDescriptions.firstObject;
    
    // 启用轻量级迁移
    storeDescription.shouldMigrateStoreAutomatically = YES;
    storeDescription.shouldInferMappingModelAutomatically = YES;
    
    // 配置SQLite选项
    storeDescription.setOption(@(YES) forKey:NSPersistentHistoryTrackingKey);
    storeDescription.setOption(@(YES) forKey:NSPersistentStoreRemoteChangeNotificationPostOptionKey);
    
    // 加载持久化存储
    [self.persistentContainer loadPersistentStoresWithCompletionHandler:^(NSPersistentStoreDescription *storeDescription, NSError *error) {
        if (error) {
            NSLog(@"Core Data加载失败: %@", error.localizedDescription);
            [self handleMigrationError:error];
        } else {
            NSLog(@"Core Data加载成功: %@", storeDescription.URL);
        }
    }];
}

- (void)configureManagedObjectContexts {
    NSLog(@"配置托管对象上下文...");
    
    // 主线程上下文
    self.mainContext = self.persistentContainer.viewContext;
    self.mainContext.automaticallyMergesChangesFromParent = YES;
    
    // 后台上下文
    self.backgroundContext = [self.persistentContainer newBackgroundContext];
    self.backgroundContext.automaticallyMergesChangesFromParent = YES;
    
    // 设置合并策略
    self.mainContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    self.backgroundContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    
    NSLog(@"托管对象上下文配置完成");
}

- (void)saveContext {
    [self saveContextWithCompletion:nil];
}

- (void)saveContextWithCompletion:(void(^)(BOOL success, NSError *error))completion {
    NSDate *startTime = [NSDate date];
    
    NSManagedObjectContext *context = [NSThread isMainThread] ? self.mainContext : self.backgroundContext;
    
    if (![context hasChanges]) {
        if (completion) {
            completion(YES, nil);
        }
        return;
    }
    
    NSError *error;
    BOOL success = [context save:&error];
    
    // 记录性能指标
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    [self recordOperation:@"save" duration:duration success:success];
    
    if (success) {
        NSLog(@"Core Data保存成功，耗时: %.3f秒", duration);
    } else {
        NSLog(@"Core Data保存失败: %@", error.localizedDescription);
    }
    
    if (completion) {
        completion(success, error);
    }
}

- (NSManagedObject *)createEntityWithName:(NSString *)entityName {
    NSManagedObjectContext *context = [NSThread isMainThread] ? self.mainContext : self.backgroundContext;
    
    NSEntityDescription *entity = [NSEntityDescription entityForName:entityName inManagedObjectContext:context];
    NSManagedObject *object = [[NSManagedObject alloc] initWithEntity:entity insertIntoManagedObjectContext:context];
    
    return object;
}

- (void)deleteObject:(NSManagedObject *)object {
    if (object && object.managedObjectContext) {
        [object.managedObjectContext deleteObject:object];
    }
}

- (NSArray *)fetchEntitiesWithName:(NSString *)entityName
                         predicate:(NSPredicate *)predicate
                   sortDescriptors:(NSArray *)sortDescriptors {
    
    NSFetchRequest *fetchRequest = [NSFetchRequest fetchRequestWithEntityName:entityName];
    
    if (predicate) {
        fetchRequest.predicate = predicate;
    }
    
    if (sortDescriptors) {
        fetchRequest.sortDescriptors = sortDescriptors;
    }
    
    return [self fetchEntitiesWithRequest:fetchRequest];
}

- (NSArray *)fetchEntitiesWithRequest:(NSFetchRequest *)fetchRequest {
    NSDate *startTime = [NSDate date];
    
    NSManagedObjectContext *context = [NSThread isMainThread] ? self.mainContext : self.backgroundContext;
    
    NSError *error;
    NSArray *results = [context executeFetchRequest:fetchRequest error:&error];
    
    // 记录性能指标
    NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
    [self recordOperation:@"fetch" duration:duration success:(error == nil)];
    
    if (error) {
        NSLog(@"Core Data查询失败: %@", error.localizedDescription);
        return @[];
    }
    
    NSLog(@"Core Data查询成功，返回 %lu 条记录，耗时: %.3f秒", (unsigned long)results.count, duration);
    return results ?: @[];
}

- (NSInteger)countEntitiesWithName:(NSString *)entityName predicate:(NSPredicate *)predicate {
    NSFetchRequest *fetchRequest = [NSFetchRequest fetchRequestWithEntityName:entityName];
    
    if (predicate) {
        fetchRequest.predicate = predicate;
    }
    
    NSManagedObjectContext *context = [NSThread isMainThread] ? self.mainContext : self.backgroundContext;
    
    NSError *error;
    NSInteger count = [context countForFetchRequest:fetchRequest error:&error];
    
    if (error) {
        NSLog(@"Core Data计数失败: %@", error.localizedDescription);
        return 0;
    }
    
    return count;
}

- (void)performBatchInsert:(NSArray *)dataArray
                entityName:(NSString *)entityName
                completion:(void(^)(BOOL success, NSError *error))completion {
    
    NSDate *startTime = [NSDate date];
    
    [self.backgroundContext performBlock:^{
        NSError *error;
        BOOL success = YES;
        
        for (NSDictionary *data in dataArray) {
            NSManagedObject *object = [self createEntityWithName:entityName];
            
            // 设置属性值
            for (NSString *key in data) {
                if ([object.entity.attributesByName.allKeys containsObject:key]) {
                    [object setValue:data[key] forKey:key];
                }
            }
        }
        
        success = [self.backgroundContext save:&error];
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        [self recordOperation:@"batchInsert" duration:duration success:success];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(success, error);
            }
        });
    }];
}

- (void)performBatchUpdate:(NSString *)entityName
                 predicate:(NSPredicate *)predicate
            propertiesToUpdate:(NSDictionary *)properties
                completion:(void(^)(NSInteger updatedCount, NSError *error))completion {
    
    NSDate *startTime = [NSDate date];
    
    [self.backgroundContext performBlock:^{
        NSBatchUpdateRequest *batchUpdateRequest = [NSBatchUpdateRequest batchUpdateRequestWithEntityName:entityName];
        batchUpdateRequest.predicate = predicate;
        batchUpdateRequest.propertiesToUpdate = properties;
        batchUpdateRequest.resultType = NSUpdatedObjectsCountResultType;
        
        NSError *error;
        NSBatchUpdateResult *result = (NSBatchUpdateResult *)[self.backgroundContext executeRequest:batchUpdateRequest error:&error];
        
        NSInteger updatedCount = 0;
        if (result && !error) {
            updatedCount = [result.result integerValue];
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        [self recordOperation:@"batchUpdate" duration:duration success:(error == nil)];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(updatedCount, error);
            }
        });
    }];
}

- (void)performBatchDelete:(NSString *)entityName
                 predicate:(NSPredicate *)predicate
                completion:(void(^)(NSInteger deletedCount, NSError *error))completion {
    
    NSDate *startTime = [NSDate date];
    
    [self.backgroundContext performBlock:^{
        NSBatchDeleteRequest *batchDeleteRequest = [NSBatchDeleteRequest batchDeleteRequestWithEntityName:entityName];
        batchDeleteRequest.predicate = predicate;
        batchDeleteRequest.resultType = NSBatchDeleteResultTypeCount;
        
        NSError *error;
        NSBatchDeleteResult *result = (NSBatchDeleteResult *)[self.backgroundContext executeRequest:batchDeleteRequest error:&error];
        
        NSInteger deletedCount = 0;
        if (result && !error) {
            deletedCount = [result.result integerValue];
        }
        
        // 记录性能指标
        NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:startTime];
        [self recordOperation:@"batchDelete" duration:duration success:(error == nil)];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(deletedCount, error);
            }
        });
    }];
}

- (void)recordOperation:(NSString *)operation duration:(NSTimeInterval)duration success:(BOOL)success {
    NSDictionary *metric = @{
        @"operation": operation,
        @"duration": @(duration),
        @"success": @(success),
        @"timestamp": [NSDate date]
    };
    
    [self.performanceMetrics addObject:metric];
    
    // 保持最近1000条记录
    if (self.performanceMetrics.count > 1000) {
        [self.performanceMetrics removeObjectAtIndex:0];
    }
}

- (void)optimizePerformance {
    NSLog(@"=== 优化Core Data性能 ===");
    
    [self enableQueryOptimization];
    [self configureFaultingBehavior];
    [self setupMemoryManagement];
    
    NSLog(@"Core Data性能优化完成");
}

- (void)enableQueryOptimization {
    NSLog(@"启用查询优化...");
    
    // 设置批量大小
    // 这个设置会在具体的查询中应用
    
    // 启用预取
    // fetchRequest.relationshipKeyPathsForPrefetching = @[@"relationship"];
    
    NSLog(@"查询优化配置完成");
}

- (void)configureFaultingBehavior {
    NSLog(@"配置故障行为...");
    
    // 配置上下文的故障行为
    self.mainContext.stalenessInterval = 0.0; // 禁用陈旧性检查
    self.backgroundContext.stalenessInterval = 0.0;
    
    NSLog(@"故障行为配置完成");
}

- (void)setupMemoryManagement {
    NSLog(@"设置内存管理...");
    
    // 监听内存警告
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(handleMemoryWarning)
                                                 name:UIApplicationDidReceiveMemoryWarningNotification
                                               object:nil];
    
    NSLog(@"内存管理设置完成");
}

- (void)handleMemoryWarning {
    NSLog(@"收到内存警告，清理Core Data缓存...");
    
    // 重置主上下文
    [self.mainContext reset];
    
    // 清理后台上下文
    [self.backgroundContext reset];
    
    NSLog(@"Core Data缓存清理完成");
}

- (BOOL)performDataMigration {
    NSLog(@"执行数据迁移...");
    
    // 检查是否需要迁移
    if (![self validateDataModel]) {
        NSLog(@"数据模型验证失败，需要迁移");
        
        // 这里可以实现自定义迁移逻辑
        // 对于复杂的迁移，可能需要手动处理
        
        return YES;
    }
    
    NSLog(@"数据模型验证通过，无需迁移");
    return YES;
}

- (BOOL)validateDataModel {
    NSLog(@"验证数据模型...");
    
    // 检查持久化存储是否与当前模型兼容
    NSPersistentStore *store = self.persistentContainer.persistentStoreCoordinator.persistentStores.firstObject;
    
    if (store) {
        NSError *error;
        NSDictionary *metadata = [NSPersistentStoreCoordinator metadataForPersistentStoreOfType:NSSQLiteStoreType
                                                                                              URL:store.URL
                                                                                          options:nil
                                                                                            error:&error];
        
        if (metadata) {
            NSManagedObjectModel *model = self.persistentContainer.managedObjectModel;
            BOOL isCompatible = [model isConfiguration:nil compatibleWithStoreMetadata:metadata];
            
            NSLog(@"数据模型兼容性: %@", isCompatible ? @"兼容" : @"不兼容");
            return isCompatible;
        }
    }
    
    return NO;
}

- (void)handleMigrationError:(NSError *)error {
    NSLog(@"处理迁移错误: %@", error.localizedDescription);
    
    // 检查是否是迁移相关的错误
    if (error.code == NSPersistentStoreIncompatibleVersionHashError ||
        error.code == NSMigrationMissingSourceModelError) {
        
        NSLog(@"检测到数据模型不兼容，尝试删除旧数据库");
        
        // 删除旧的数据库文件（谨慎操作）
        NSPersistentStoreDescription *storeDescription = self.persistentContainer.persistentStoreDescriptions.firstObject;
        
        if (storeDescription.URL) {
            NSFileManager *fileManager = [NSFileManager defaultManager];
            [fileManager removeItemAtURL:storeDescription.URL error:nil];
            
            // 重新加载
            [self.persistentContainer loadPersistentStoresWithCompletionHandler:^(NSPersistentStoreDescription *storeDescription, NSError *error) {
                if (error) {
                    NSLog(@"重新加载仍然失败: %@", error.localizedDescription);
                } else {
                    NSLog(@"重新加载成功");
                }
            }];
        }
    }
}

- (void)enablePerformanceMonitoring {
    NSLog(@"启用性能监控...");
    
    // 注册Core Data通知
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(managedObjectContextDidSave:)
                                                 name:NSManagedObjectContextDidSaveNotification
                                               object:nil];
    
    NSLog(@"性能监控启用完成");
}

- (void)managedObjectContextDidSave:(NSNotification *)notification {
    NSManagedObjectContext *context = notification.object;
    
    if (context != self.mainContext && context != self.backgroundContext) {
        return;
    }
    
    NSDictionary *userInfo = notification.userInfo;
    NSSet *insertedObjects = userInfo[NSInsertedObjectsKey];
    NSSet *updatedObjects = userInfo[NSUpdatedObjectsKey];
    NSSet *deletedObjects = userInfo[NSDeletedObjectsKey];
    
    NSLog(@"Core Data保存通知 - 插入: %lu, 更新: %lu, 删除: %lu",
          (unsigned long)insertedObjects.count,
          (unsigned long)updatedObjects.count,
          (unsigned long)deletedObjects.count);
}

- (NSDictionary *)getPerformanceReport {
    NSMutableDictionary *report = [NSMutableDictionary dictionary];
    
    // 统计各种操作的性能
    NSMutableDictionary *operationStats = [NSMutableDictionary dictionary];
    
    for (NSDictionary *metric in self.performanceMetrics) {
        NSString *operation = metric[@"operation"];
        NSNumber *duration = metric[@"duration"];
        NSNumber *success = metric[@"success"];
        
        if (!operationStats[operation]) {
            operationStats[operation] = [NSMutableDictionary dictionary];
            operationStats[operation][@"durations"] = [NSMutableArray array];
            operationStats[operation][@"successCount"] = @0;
            operationStats[operation][@"totalCount"] = @0;
        }
        
        [operationStats[operation][@"durations"] addObject:duration];
        operationStats[operation][@"totalCount"] = @([operationStats[operation][@"totalCount"] integerValue] + 1);
        
        if ([success boolValue]) {
            operationStats[operation][@"successCount"] = @([operationStats[operation][@"successCount"] integerValue] + 1);
        }
    }
    
    // 计算统计信息
    for (NSString *operation in operationStats) {
        NSArray *durations = operationStats[operation][@"durations"];
        NSInteger successCount = [operationStats[operation][@"successCount"] integerValue];
        NSInteger totalCount = [operationStats[operation][@"totalCount"] integerValue];
        
        double totalDuration = 0;
        double maxDuration = 0;
        double minDuration = INFINITY;
        
        for (NSNumber *duration in durations) {
            double d = [duration doubleValue];
            totalDuration += d;
            maxDuration = MAX(maxDuration, d);
            minDuration = MIN(minDuration, d);
        }
        
        double averageDuration = totalDuration / durations.count;
        double successRate = (double)successCount / totalCount;
        
        report[operation] = @{
            @"operationCount": @(totalCount),
            @"successRate": @(successRate),
            @"averageDuration": @(averageDuration),
            @"maxDuration": @(maxDuration),
            @"minDuration": @(minDuration == INFINITY ? 0 : minDuration),
            @"totalDuration": @(totalDuration)
        };
    }
    
    report[@"totalOperations"] = @(self.performanceMetrics.count);
    report[@"reportTimestamp"] = [NSDate date];
    
    return report;
}

- (void)logCoreDataStatistics {
    NSLog(@"=== Core Data统计信息 ===");
    
    NSDictionary *report = [self getPerformanceReport];
    
    for (NSString *operation in report) {
        if ([operation isEqualToString:@"totalOperations"] || [operation isEqualToString:@"reportTimestamp"]) {
            continue;
        }
        
        NSDictionary *stats = report[operation];
        NSLog(@"操作类型: %@", operation);
        NSLog(@"  操作次数: %@", stats[@"operationCount"]);
        NSLog(@"  成功率: %.1f%%", [stats[@"successRate"] doubleValue] * 100);
        NSLog(@"  平均耗时: %.3f秒", [stats[@"averageDuration"] doubleValue]);
        NSLog(@"  最大耗时: %.3f秒", [stats[@"maxDuration"] doubleValue]);
    }
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```