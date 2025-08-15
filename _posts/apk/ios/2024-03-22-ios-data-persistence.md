---
layout: post
title: "iOS数据持久化深度解析：Core Data、SQLite与文件存储实战指南"
date: 2024-03-22 10:30:00 +0800
categories: [iOS开发, 数据持久化]
tags: [iOS, Core Data, SQLite, 文件存储, 数据库, 持久化]
author: iOS技术专家
description: "深入探讨iOS平台的数据持久化技术，包括Core Data框架、SQLite数据库、文件存储系统的原理、实现和最佳实践，提供完整的代码示例和性能优化建议。"
keywords: ["iOS数据持久化", "Core Data", "SQLite", "文件存储", "数据库设计", "性能优化"]
image: "/assets/images/ios-data-persistence.jpg"
---

数据持久化是iOS应用开发中的核心技术之一，它决定了应用如何存储、管理和检索数据。本文将深入探讨iOS平台上的主要数据持久化技术，包括Core Data框架、SQLite数据库和文件存储系统，并提供实用的代码示例和最佳实践。

## 数据持久化概述

### 持久化技术选择

```objc
@interface DataPersistenceManager : NSObject

@property (nonatomic, strong) NSManagedObjectContext *managedObjectContext;
@property (nonatomic, strong) NSPersistentStoreCoordinator *persistentStoreCoordinator;
@property (nonatomic, strong) NSManagedObjectModel *managedObjectModel;

+ (instancetype)sharedManager;

// 数据持久化策略评估
- (NSString *)recommendPersistenceStrategyForDataType:(NSString *)dataType
                                             dataSize:(NSUInteger)dataSize
                                          complexity:(NSUInteger)complexity
                                         performance:(NSUInteger)performance;

// 数据迁移管理
- (BOOL)migrateDataFromVersion:(NSString *)fromVersion
                     toVersion:(NSString *)toVersion
                         error:(NSError **)error;

// 性能监控
- (NSDictionary *)getPerformanceMetrics;

@end

@implementation DataPersistenceManager

+ (instancetype)sharedManager {
    static DataPersistenceManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupCoreDataStack];
    }
    return self;
}

- (NSString *)recommendPersistenceStrategyForDataType:(NSString *)dataType
                                             dataSize:(NSUInteger)dataSize
                                          complexity:(NSUInteger)complexity
                                         performance:(NSUInteger)performance {
    
    // 评估矩阵
    NSMutableDictionary *scores = [NSMutableDictionary dictionary];
    
    // Core Data评分
    NSInteger coreDataScore = 0;
    if (complexity >= 7) coreDataScore += 30; // 复杂关系
    if (dataSize >= 1000) coreDataScore += 20; // 大数据量
    if ([dataType containsString:@"object"]) coreDataScore += 25; // 对象数据
    if (performance <= 5) coreDataScore += 15; // 性能要求不高
    scores[@"CoreData"] = @(coreDataScore);
    
    // SQLite评分
    NSInteger sqliteScore = 0;
    if (performance >= 8) sqliteScore += 30; // 高性能要求
    if (dataSize >= 10000) sqliteScore += 25; // 超大数据量
    if (complexity >= 5 && complexity <= 8) sqliteScore += 20; // 中等复杂度
    if ([dataType containsString:@"structured"]) sqliteScore += 15; // 结构化数据
    scores[@"SQLite"] = @(sqliteScore);
    
    // 文件存储评分
    NSInteger fileScore = 0;
    if (complexity <= 3) fileScore += 30; // 简单数据
    if (dataSize <= 100) fileScore += 25; // 小数据量
    if ([dataType containsString:@"simple"]) fileScore += 20; // 简单类型
    if (performance >= 9) fileScore += 15; // 极高性能
    scores[@"FileStorage"] = @(fileScore);
    
    // 返回最高分的策略
    NSString *bestStrategy = @"FileStorage";
    NSInteger maxScore = 0;
    
    for (NSString *strategy in scores.allKeys) {
        NSInteger score = [scores[strategy] integerValue];
        if (score > maxScore) {
            maxScore = score;
            bestStrategy = strategy;
        }
    }
    
    return bestStrategy;
}

- (void)setupCoreDataStack {
    // 设置Core Data堆栈
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"DataModel" withExtension:@"momd"];
    self.managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
    
    self.persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc]
                                      initWithManagedObjectModel:self.managedObjectModel];
    
    NSURL *storeURL = [[self applicationDocumentsDirectory]
                      URLByAppendingPathComponent:@"DataModel.sqlite"];
    
    NSError *error = nil;
    NSDictionary *options = @{
        NSMigratePersistentStoresAutomaticallyOption: @YES,
        NSInferMappingModelAutomaticallyOption: @YES,
        NSSQLitePragmasOption: @{@"journal_mode": @"WAL"}
    };
    
    NSPersistentStore *store = [self.persistentStoreCoordinator
                               addPersistentStoreWithType:NSSQLiteStoreType
                               configuration:nil
                               URL:storeURL
                               options:options
                               error:&error];
    
    if (!store) {
        NSLog(@"Failed to create persistent store: %@", error);
    }
    
    self.managedObjectContext = [[NSManagedObjectContext alloc]
                                initWithConcurrencyType:NSMainQueueConcurrencyType];
    self.managedObjectContext.persistentStoreCoordinator = self.persistentStoreCoordinator;
}

- (NSURL *)applicationDocumentsDirectory {
    return [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory
                                                   inDomains:NSUserDomainMask] lastObject];
}

- (BOOL)migrateDataFromVersion:(NSString *)fromVersion
                     toVersion:(NSString *)toVersion
                         error:(NSError **)error {
    
    NSLog(@"Migrating data from version %@ to %@", fromVersion, toVersion);
    
    // 实现数据迁移逻辑
    if ([fromVersion isEqualToString:@"1.0"] && [toVersion isEqualToString:@"2.0"]) {
        return [self migrateFromV1ToV2:error];
    }
    
    return YES;
}

- (BOOL)migrateFromV1ToV2:(NSError **)error {
    // 具体的迁移实现
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"OldEntity"];
    NSArray *oldEntities = [self.managedObjectContext executeFetchRequest:request error:error];
    
    if (*error) {
        return NO;
    }
    
    for (NSManagedObject *oldEntity in oldEntities) {
        // 创建新实体并迁移数据
        NSManagedObject *newEntity = [NSEntityDescription
                                     insertNewObjectForEntityForName:@"NewEntity"
                                     inManagedObjectContext:self.managedObjectContext];
        
        // 迁移属性
        [newEntity setValue:[oldEntity valueForKey:@"name"] forKey:@"name"];
        [newEntity setValue:[oldEntity valueForKey:@"date"] forKey:@"createdDate"];
        
        // 删除旧实体
        [self.managedObjectContext deleteObject:oldEntity];
    }
    
    return [self.managedObjectContext save:error];
}

- (NSDictionary *)getPerformanceMetrics {
    NSMutableDictionary *metrics = [NSMutableDictionary dictionary];
    
    // Core Data性能指标
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Entity"];
    request.includesSubentities = NO;
    request.includesPropertyValues = NO;
    
    NSDate *startTime = [NSDate date];
    NSError *error;
    NSUInteger count = [self.managedObjectContext countForFetchRequest:request error:&error];
    NSTimeInterval fetchTime = [[NSDate date] timeIntervalSinceDate:startTime];
    
    metrics[@"entityCount"] = @(count);
    metrics[@"fetchTime"] = @(fetchTime);
    metrics[@"memoryUsage"] = @([self getCurrentMemoryUsage]);
    
    return [metrics copy];
}

- (NSUInteger)getCurrentMemoryUsage {
    struct mach_task_basic_info info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
    
    if (kerr == KERN_SUCCESS) {
        return info.resident_size;
    }
    
    return 0;
}

@end
```

## SQLite数据库操作

### SQLite管理器

```objc
@interface SQLiteManager : NSObject

@property (nonatomic, assign) sqlite3 *database;
@property (nonatomic, strong) NSString *databasePath;
@property (nonatomic, strong) dispatch_queue_t databaseQueue;

+ (instancetype)sharedManager;

// 数据库操作
- (BOOL)openDatabase;
- (BOOL)closeDatabase;
- (BOOL)createTablesIfNeeded;

// 用户操作
- (BOOL)insertUser:(NSDictionary *)userData;
- (NSArray *)selectUsersWithCondition:(NSString *)condition parameters:(NSArray *)parameters;
- (BOOL)updateUserWithID:(NSString *)userID data:(NSDictionary *)userData;
- (BOOL)deleteUserWithID:(NSString *)userID;

// 事务操作
- (BOOL)executeTransaction:(BOOL (^)(void))transactionBlock;

// 性能优化
- (void)optimizeDatabase;
- (NSDictionary *)getDatabaseStatistics;

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
        [self setupDatabase];
    }
    return self;
}

- (void)setupDatabase {
    // 创建数据库队列
    self.databaseQueue = dispatch_queue_create("com.app.database", DISPATCH_QUEUE_SERIAL);
    
    // 设置数据库路径
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths objectAtIndex:0];
    self.databasePath = [documentsDirectory stringByAppendingPathComponent:@"app_database.sqlite"];
    
    // 打开数据库
    [self openDatabase];
    [self createTablesIfNeeded];
}

- (BOOL)openDatabase {
    int result = sqlite3_open([self.databasePath UTF8String], &_database);
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to open database: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    // 设置数据库配置
    [self configureDatabaseSettings];
    
    return YES;
}

- (void)configureDatabaseSettings {
    // 启用外键约束
    sqlite3_exec(_database, "PRAGMA foreign_keys = ON;", NULL, NULL, NULL);
    
    // 设置WAL模式
    sqlite3_exec(_database, "PRAGMA journal_mode = WAL;", NULL, NULL, NULL);
    
    // 设置同步模式
    sqlite3_exec(_database, "PRAGMA synchronous = NORMAL;", NULL, NULL, NULL);
    
    // 设置缓存大小
    sqlite3_exec(_database, "PRAGMA cache_size = 10000;", NULL, NULL, NULL);
    
    // 设置临时存储
    sqlite3_exec(_database, "PRAGMA temp_store = MEMORY;", NULL, NULL, NULL);
}

- (BOOL)closeDatabase {
    if (_database) {
        int result = sqlite3_close(_database);
        if (result == SQLITE_OK) {
            _database = NULL;
            return YES;
        } else {
            NSLog(@"Failed to close database: %s", sqlite3_errmsg(_database));
            return NO;
        }
    }
    return YES;
}

- (BOOL)createTablesIfNeeded {
    NSString *createUsersTable = @"
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT UNIQUE NOT NULL,
            username TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            created_date INTEGER NOT NULL,
            last_login_date INTEGER,
            is_active INTEGER DEFAULT 1,
            profile_data TEXT
        );
    ";
    
    NSString *createPostsTable = @"
        CREATE TABLE IF NOT EXISTS posts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            post_id TEXT UNIQUE NOT NULL,
            user_id TEXT NOT NULL,
            title TEXT NOT NULL,
            content TEXT,
            publish_date INTEGER NOT NULL,
            is_published INTEGER DEFAULT 0,
            view_count INTEGER DEFAULT 0,
            FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
        );
    ";
    
    NSString *createIndexes = @"
        CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
        CREATE INDEX IF NOT EXISTS idx_users_username ON users(username);
        CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id);
        CREATE INDEX IF NOT EXISTS idx_posts_publish_date ON posts(publish_date);
    ";
    
    char *errorMessage;
    
    // 创建用户表
    int result = sqlite3_exec(_database, [createUsersTable UTF8String], NULL, NULL, &errorMessage);
    if (result != SQLITE_OK) {
        NSLog(@"Failed to create users table: %s", errorMessage);
        sqlite3_free(errorMessage);
        return NO;
    }
    
    // 创建文章表
    result = sqlite3_exec(_database, [createPostsTable UTF8String], NULL, NULL, &errorMessage);
    if (result != SQLITE_OK) {
        NSLog(@"Failed to create posts table: %s", errorMessage);
        sqlite3_free(errorMessage);
        return NO;
    }
    
    // 创建索引
    result = sqlite3_exec(_database, [createIndexes UTF8String], NULL, NULL, &errorMessage);
    if (result != SQLITE_OK) {
        NSLog(@"Failed to create indexes: %s", errorMessage);
        sqlite3_free(errorMessage);
        return NO;
    }
    
    return YES;
}

- (BOOL)insertUser:(NSDictionary *)userData {
    NSString *sql = @"INSERT INTO users (user_id, username, email, password_hash, created_date, profile_data) VALUES (?, ?, ?, ?, ?, ?)";
    
    sqlite3_stmt *statement;
    int result = sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL);
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to prepare insert statement: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    // 绑定参数
    sqlite3_bind_text(statement, 1, [userData[@"user_id"] UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(statement, 2, [userData[@"username"] UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(statement, 3, [userData[@"email"] UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(statement, 4, [userData[@"password_hash"] UTF8String], -1, SQLITE_TRANSIENT);
    sqlite3_bind_int64(statement, 5, [[NSDate date] timeIntervalSince1970]);
    
    NSData *profileData = [NSJSONSerialization dataWithJSONObject:userData[@"profile"] options:0 error:nil];
    if (profileData) {
        sqlite3_bind_text(statement, 6, [[profileData base64EncodedStringWithOptions:0] UTF8String], -1, SQLITE_TRANSIENT);
    } else {
        sqlite3_bind_null(statement, 6);
    }
    
    result = sqlite3_step(statement);
    sqlite3_finalize(statement);
    
    if (result != SQLITE_DONE) {
        NSLog(@"Failed to insert user: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    return YES;
}

- (NSArray *)selectUsersWithCondition:(NSString *)condition parameters:(NSArray *)parameters {
    NSMutableArray *users = [NSMutableArray array];
    
    NSString *sql = [NSString stringWithFormat:@"SELECT * FROM users %@", condition ?: @""];
    
    sqlite3_stmt *statement;
    int result = sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL);
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to prepare select statement: %s", sqlite3_errmsg(_database));
        return users;
    }
    
    // 绑定参数
    for (int i = 0; i < parameters.count; i++) {
        id parameter = parameters[i];
        if ([parameter isKindOfClass:[NSString class]]) {
            sqlite3_bind_text(statement, i + 1, [parameter UTF8String], -1, SQLITE_TRANSIENT);
        } else if ([parameter isKindOfClass:[NSNumber class]]) {
            sqlite3_bind_int64(statement, i + 1, [parameter longLongValue]);
        }
    }
    
    while (sqlite3_step(statement) == SQLITE_ROW) {
        NSMutableDictionary *user = [NSMutableDictionary dictionary];
        
        // 读取列数据
        const char *userID = (const char *)sqlite3_column_text(statement, 1);
        const char *username = (const char *)sqlite3_column_text(statement, 2);
        const char *email = (const char *)sqlite3_column_text(statement, 3);
        sqlite3_int64 createdDate = sqlite3_column_int64(statement, 5);
        sqlite3_int64 lastLoginDate = sqlite3_column_int64(statement, 6);
        int isActive = sqlite3_column_int(statement, 7);
        const char *profileData = (const char *)sqlite3_column_text(statement, 8);
        
        if (userID) user[@"user_id"] = [NSString stringWithUTF8String:userID];
        if (username) user[@"username"] = [NSString stringWithUTF8String:username];
        if (email) user[@"email"] = [NSString stringWithUTF8String:email];
        user[@"created_date"] = [NSDate dateWithTimeIntervalSince1970:createdDate];
        if (lastLoginDate > 0) {
            user[@"last_login_date"] = [NSDate dateWithTimeIntervalSince1970:lastLoginDate];
        }
        user[@"is_active"] = @(isActive);
        
        if (profileData) {
            NSString *profileString = [NSString stringWithUTF8String:profileData];
            NSData *data = [[NSData alloc] initWithBase64EncodedString:profileString options:0];
            if (data) {
                NSError *error;
                id profile = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];
                if (!error && profile) {
                    user[@"profile"] = profile;
                }
            }
        }
        
        [users addObject:user];
    }
    
    sqlite3_finalize(statement);
    return users;
}

- (BOOL)updateUserWithID:(NSString *)userID data:(NSDictionary *)userData {
    NSMutableArray *setParts = [NSMutableArray array];
    NSMutableArray *values = [NSMutableArray array];
    
    for (NSString *key in userData.allKeys) {
        [setParts addObject:[NSString stringWithFormat:@"%@ = ?", key]];
        [values addObject:userData[key]];
    }
    
    [values addObject:userID]; // 添加WHERE条件的参数
    
    NSString *sql = [NSString stringWithFormat:@"UPDATE users SET %@ WHERE user_id = ?",
                    [setParts componentsJoinedByString:@", "]];
    
    sqlite3_stmt *statement;
    int result = sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL);
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to prepare update statement: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    // 绑定参数
    for (int i = 0; i < values.count; i++) {
        id value = values[i];
        if ([value isKindOfClass:[NSString class]]) {
            sqlite3_bind_text(statement, i + 1, [value UTF8String], -1, SQLITE_TRANSIENT);
        } else if ([value isKindOfClass:[NSNumber class]]) {
            sqlite3_bind_int64(statement, i + 1, [value longLongValue]);
        } else if ([value isKindOfClass:[NSDate class]]) {
            sqlite3_bind_int64(statement, i + 1, [value timeIntervalSince1970]);
        }
    }
    
    result = sqlite3_step(statement);
    sqlite3_finalize(statement);
    
    if (result != SQLITE_DONE) {
        NSLog(@"Failed to update user: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    return YES;
}

- (BOOL)deleteUserWithID:(NSString *)userID {
    NSString *sql = @"DELETE FROM users WHERE user_id = ?";
    
    sqlite3_stmt *statement;
    int result = sqlite3_prepare_v2(_database, [sql UTF8String], -1, &statement, NULL);
    
    if (result != SQLITE_OK) {
        NSLog(@"Failed to prepare delete statement: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    sqlite3_bind_text(statement, 1, [userID UTF8String], -1, SQLITE_TRANSIENT);
    
    result = sqlite3_step(statement);
    sqlite3_finalize(statement);
    
    if (result != SQLITE_DONE) {
        NSLog(@"Failed to delete user: %s", sqlite3_errmsg(_database));
        return NO;
    }
    
    return YES;
}

- (BOOL)executeTransaction:(BOOL (^)(void))transactionBlock {
    __block BOOL success = YES;
    
    dispatch_sync(self.databaseQueue, ^{
        // 开始事务
        char *errorMessage;
        int result = sqlite3_exec(_database, "BEGIN TRANSACTION;", NULL, NULL, &errorMessage);
        
        if (result != SQLITE_OK) {
            NSLog(@"Failed to begin transaction: %s", errorMessage);
            sqlite3_free(errorMessage);
            success = NO;
            return;
        }
        
        // 执行事务块
        BOOL blockSuccess = transactionBlock();
        
        if (blockSuccess) {
            // 提交事务
            result = sqlite3_exec(_database, "COMMIT;", NULL, NULL, &errorMessage);
            if (result != SQLITE_OK) {
                NSLog(@"Failed to commit transaction: %s", errorMessage);
                sqlite3_free(errorMessage);
                success = NO;
            }
        } else {
            // 回滚事务
            result = sqlite3_exec(_database, "ROLLBACK;", NULL, NULL, &errorMessage);
            if (result != SQLITE_OK) {
                NSLog(@"Failed to rollback transaction: %s", errorMessage);
                sqlite3_free(errorMessage);
            }
            success = NO;
        }
    });
    
    return success;
}

- (void)optimizeDatabase {
    dispatch_async(self.databaseQueue, ^{
        // 分析数据库
        sqlite3_exec(_database, "ANALYZE;", NULL, NULL, NULL);
        
        // 清理数据库
        sqlite3_exec(_database, "VACUUM;", NULL, NULL, NULL);
        
        // 重建索引
        sqlite3_exec(_database, "REINDEX;", NULL, NULL, NULL);
        
        NSLog(@"Database optimization completed");
    });
}

- (NSDictionary *)getDatabaseStatistics {
    NSMutableDictionary *stats = [NSMutableDictionary dictionary];
    
    // 获取数据库大小
    NSError *error;
    NSDictionary *attributes = [[NSFileManager defaultManager] attributesOfItemAtPath:self.databasePath error:&error];
    if (!error) {
        stats[@"database_size"] = attributes[NSFileSize];
    }
    
    // 获取表统计信息
    sqlite3_stmt *statement;
    const char *sql = "SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%';";
    
    if (sqlite3_prepare_v2(_database, sql, -1, &statement, NULL) == SQLITE_OK) {
        NSMutableDictionary *tableCounts = [NSMutableDictionary dictionary];
        
        while (sqlite3_step(statement) == SQLITE_ROW) {
            const char *tableName = (const char *)sqlite3_column_text(statement, 0);
            if (tableName) {
                NSString *tableNameStr = [NSString stringWithUTF8String:tableName];
                
                // 获取表行数
                NSString *countSQL = [NSString stringWithFormat:@"SELECT COUNT(*) FROM %@", tableNameStr];
                sqlite3_stmt *countStatement;
                
                if (sqlite3_prepare_v2(_database, [countSQL UTF8String], -1, &countStatement, NULL) == SQLITE_OK) {
                    if (sqlite3_step(countStatement) == SQLITE_ROW) {
                        int count = sqlite3_column_int(countStatement, 0);
                        tableCounts[tableNameStr] = @(count);
                    }
                    sqlite3_finalize(countStatement);
                }
            }
        }
        
        stats[@"table_counts"] = tableCounts;
        sqlite3_finalize(statement);
    }
    
    return [stats copy];
}

- (void)dealloc {
    [self closeDatabase];
}

@end
```

## 文件存储系统

### 文件管理器

```objc
@interface FileStorageManager : NSObject

@property (nonatomic, strong) NSString *documentsPath;
@property (nonatomic, strong) NSString *cachesPath;
@property (nonatomic, strong) NSString *temporaryPath;
@property (nonatomic, strong) dispatch_queue_t fileQueue;

+ (instancetype)sharedManager;

// 基本文件操作
- (BOOL)writeData:(NSData *)data toFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (NSData *)readDataFromFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (BOOL)deleteFile:(NSString *)fileName inDirectory:(NSString *)directory;
- (BOOL)fileExists:(NSString *)fileName inDirectory:(NSString *)directory;

// JSON文件操作
- (BOOL)writeJSONObject:(id)object toFile:(NSString *)fileName;
- (id)readJSONObjectFromFile:(NSString *)fileName;

// 图片文件操作
- (BOOL)saveImage:(UIImage *)image withName:(NSString *)imageName quality:(CGFloat)quality;
- (UIImage *)loadImageWithName:(NSString *)imageName;
- (BOOL)deleteImageWithName:(NSString *)imageName;

// 缓存管理
- (void)clearCacheDirectory;
- (void)clearTemporaryDirectory;
- (NSUInteger)getCacheSize;
- (void)cleanupOldFiles:(NSTimeInterval)maxAge;

// 文件加密
- (BOOL)writeEncryptedData:(NSData *)data toFile:(NSString *)fileName withKey:(NSString *)key;
- (NSData *)readEncryptedDataFromFile:(NSString *)fileName withKey:(NSString *)key;

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
        [self setupDirectories];
    }
    return self;
}

- (void)setupDirectories {
    // 创建文件操作队列
    self.fileQueue = dispatch_queue_create("com.app.filequeue", DISPATCH_QUEUE_CONCURRENT);
    
    // 获取标准目录路径
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    self.documentsPath = [paths firstObject];
    
    paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);
    self.cachesPath = [paths firstObject];
    
    self.temporaryPath = NSTemporaryDirectory();
    
    // 创建自定义子目录
    [self createDirectoryIfNeeded:[self.documentsPath stringByAppendingPathComponent:@"UserData"]];
    [self createDirectoryIfNeeded:[self.documentsPath stringByAppendingPathComponent:@"Images"]];
    [self createDirectoryIfNeeded:[self.cachesPath stringByAppendingPathComponent:@"TempImages"]];
}

- (void)createDirectoryIfNeeded:(NSString *)directoryPath {
    NSFileManager *fileManager = [NSFileManager defaultManager];
    BOOL isDirectory;
    
    if (![fileManager fileExistsAtPath:directoryPath isDirectory:&isDirectory] || !isDirectory) {
        NSError *error;
        BOOL success = [fileManager createDirectoryAtPath:directoryPath
                                withIntermediateDirectories:YES
                                                 attributes:nil
                                                      error:&error];
        if (!success) {
            NSLog(@"Failed to create directory %@: %@", directoryPath, error);
        }
    }
}

- (BOOL)writeData:(NSData *)data toFile:(NSString *)fileName inDirectory:(NSString *)directory {
    if (!data || !fileName) {
        return NO;
    }
    
    NSString *directoryPath = [self getDirectoryPath:directory];
    if (!directoryPath) {
        return NO;
    }
    
    NSString *filePath = [directoryPath stringByAppendingPathComponent:fileName];
    
    __block BOOL success = NO;
    dispatch_barrier_sync(self.fileQueue, ^{
        NSError *error;
        success = [data writeToFile:filePath options:NSDataWritingAtomic error:&error];
        
        if (!success) {
            NSLog(@"Failed to write file %@: %@", filePath, error);
        } else {
            // 设置文件保护属性
            NSDictionary *attributes = @{
                NSFileProtectionKey: NSFileProtectionCompleteUntilFirstUserAuthentication
            };
            [[NSFileManager defaultManager] setAttributes:attributes ofItemAtPath:filePath error:nil];
        }
    });
    
    return success;
}

- (NSData *)readDataFromFile:(NSString *)fileName inDirectory:(NSString *)directory {
    if (!fileName) {
        return nil;
    }
    
    NSString *directoryPath = [self getDirectoryPath:directory];
    if (!directoryPath) {
        return nil;
    }
    
    NSString *filePath = [directoryPath stringByAppendingPathComponent:fileName];
    
    __block NSData *data = nil;
    dispatch_sync(self.fileQueue, ^{
        NSError *error;
        data = [NSData dataWithContentsOfFile:filePath options:0 error:&error];
        
        if (!data && error) {
            NSLog(@"Failed to read file %@: %@", filePath, error);
        }
    });
    
    return data;
}

- (BOOL)deleteFile:(NSString *)fileName inDirectory:(NSString *)directory {
    if (!fileName) {
        return NO;
    }
    
    NSString *directoryPath = [self getDirectoryPath:directory];
    if (!directoryPath) {
        return NO;
    }
    
    NSString *filePath = [directoryPath stringByAppendingPathComponent:fileName];
    
    __block BOOL success = NO;
    dispatch_barrier_sync(self.fileQueue, ^{
        NSError *error;
        success = [[NSFileManager defaultManager] removeItemAtPath:filePath error:&error];
        
        if (!success && error.code != NSFileNoSuchFileError) {
            NSLog(@"Failed to delete file %@: %@", filePath, error);
        }
    });
    
    return success;
}

- (BOOL)fileExists:(NSString *)fileName inDirectory:(NSString *)directory {
    if (!fileName) {
        return NO;
    }
    
    NSString *directoryPath = [self getDirectoryPath:directory];
    if (!directoryPath) {
        return NO;
    }
    
    NSString *filePath = [directoryPath stringByAppendingPathComponent:fileName];
    
    __block BOOL exists = NO;
    dispatch_sync(self.fileQueue, ^{
        exists = [[NSFileManager defaultManager] fileExistsAtPath:filePath];
    });
    
    return exists;
}

- (NSString *)getDirectoryPath:(NSString *)directory {
    if ([directory isEqualToString:@"Documents"]) {
        return self.documentsPath;
    } else if ([directory isEqualToString:@"Caches"]) {
        return self.cachesPath;
    } else if ([directory isEqualToString:@"Temporary"]) {
        return self.temporaryPath;
    } else if ([directory isEqualToString:@"UserData"]) {
        return [self.documentsPath stringByAppendingPathComponent:@"UserData"];
    } else if ([directory isEqualToString:@"Images"]) {
        return [self.documentsPath stringByAppendingPathComponent:@"Images"];
    }
    
    return nil;
}

- (BOOL)writeJSONObject:(id)object toFile:(NSString *)fileName {
    if (!object || !fileName) {
        return NO;
    }
    
    NSError *error;
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:object
                                                       options:NSJSONWritingPrettyPrinted
                                                         error:&error];
    
    if (error) {
        NSLog(@"Failed to serialize JSON: %@", error);
        return NO;
    }
    
    return [self writeData:jsonData toFile:fileName inDirectory:@"UserData"];
}

- (id)readJSONObjectFromFile:(NSString *)fileName {
    NSData *jsonData = [self readDataFromFile:fileName inDirectory:@"UserData"];
    
    if (!jsonData) {
        return nil;
    }
    
    NSError *error;
    id jsonObject = [NSJSONSerialization JSONObjectWithData:jsonData
                                                    options:0
                                                      error:&error];
    
    if (error) {
        NSLog(@"Failed to deserialize JSON: %@", error);
        return nil;
    }
    
    return jsonObject;
}

- (BOOL)saveImage:(UIImage *)image withName:(NSString *)imageName quality:(CGFloat)quality {
    if (!image || !imageName) {
        return NO;
    }
    
    NSData *imageData;
    
    // 根据文件扩展名选择压缩格式
    if ([imageName.lowercaseString hasSuffix:@".png"]) {
        imageData = UIImagePNGRepresentation(image);
    } else {
        imageData = UIImageJPEGRepresentation(image, quality);
    }
    
    if (!imageData) {
        NSLog(@"Failed to convert image to data");
        return NO;
    }
    
    return [self writeData:imageData toFile:imageName inDirectory:@"Images"];
}

- (UIImage *)loadImageWithName:(NSString *)imageName {
    NSData *imageData = [self readDataFromFile:imageName inDirectory:@"Images"];
    
    if (!imageData) {
        return nil;
    }
    
    return [UIImage imageWithData:imageData];
}

- (BOOL)deleteImageWithName:(NSString *)imageName {
    return [self deleteFile:imageName inDirectory:@"Images"];
}

- (void)clearCacheDirectory {
    dispatch_barrier_async(self.fileQueue, ^{
        NSFileManager *fileManager = [NSFileManager defaultManager];
        NSError *error;
        
        NSArray *cacheContents = [fileManager contentsOfDirectoryAtPath:self.cachesPath error:&error];
        
        if (error) {
            NSLog(@"Failed to get cache contents: %@", error);
            return;
        }
        
        for (NSString *fileName in cacheContents) {
            NSString *filePath = [self.cachesPath stringByAppendingPathComponent:fileName];
            [fileManager removeItemAtPath:filePath error:nil];
        }
        
        NSLog(@"Cache directory cleared");
    });
}

- (void)clearTemporaryDirectory {
    dispatch_barrier_async(self.fileQueue, ^{
        NSFileManager *fileManager = [NSFileManager defaultManager];
        NSError *error;
        
        NSArray *tempContents = [fileManager contentsOfDirectoryAtPath:self.temporaryPath error:&error];
        
        if (error) {
            NSLog(@"Failed to get temporary contents: %@", error);
            return;
        }
        
        for (NSString *fileName in tempContents) {
            NSString *filePath = [self.temporaryPath stringByAppendingPathComponent:fileName];
            [fileManager removeItemAtPath:filePath error:nil];
        }
        
        NSLog(@"Temporary directory cleared");
    });
}

- (NSUInteger)getCacheSize {
    __block NSUInteger totalSize = 0;
    
    dispatch_sync(self.fileQueue, ^{
        NSFileManager *fileManager = [NSFileManager defaultManager];
        NSDirectoryEnumerator *enumerator = [fileManager enumeratorAtPath:self.cachesPath];
        
        for (NSString *fileName in enumerator) {
            NSString *filePath = [self.cachesPath stringByAppendingPathComponent:fileName];
            NSDictionary *attributes = [fileManager attributesOfItemAtPath:filePath error:nil];
            
            if (attributes) {
                totalSize += [attributes[NSFileSize] unsignedIntegerValue];
            }
        }
    });
    
    return totalSize;
}

- (void)cleanupOldFiles:(NSTimeInterval)maxAge {
    dispatch_barrier_async(self.fileQueue, ^{
        NSFileManager *fileManager = [NSFileManager defaultManager];
        NSDate *cutoffDate = [NSDate dateWithTimeIntervalSinceNow:-maxAge];
        
        NSArray *directories = @[self.cachesPath, self.temporaryPath];
        
        for (NSString *directory in directories) {
            NSDirectoryEnumerator *enumerator = [fileManager enumeratorAtPath:directory];
            
            for (NSString *fileName in enumerator) {
                NSString *filePath = [directory stringByAppendingPathComponent:fileName];
                NSDictionary *attributes = [fileManager attributesOfItemAtPath:filePath error:nil];
                
                if (attributes) {
                    NSDate *modificationDate = attributes[NSFileModificationDate];
                    if ([modificationDate compare:cutoffDate] == NSOrderedAscending) {
                        [fileManager removeItemAtPath:filePath error:nil];
                        NSLog(@"Removed old file: %@", fileName);
                    }
                }
            }
        }
    });
}

- (BOOL)writeEncryptedData:(NSData *)data toFile:(NSString *)fileName withKey:(NSString *)key {
    if (!data || !fileName || !key) {
        return NO;
    }
    
    // 使用AES加密数据
    NSData *keyData = [key dataUsingEncoding:NSUTF8StringEncoding];
    NSMutableData *encryptedData = [NSMutableData dataWithLength:data.length + kCCBlockSizeAES128];
    
    size_t numBytesEncrypted = 0;
    CCCryptorStatus cryptStatus = CCCrypt(kCCEncrypt,
                                         kCCAlgorithmAES128,
                                         kCCOptionPKCS7Padding,
                                         keyData.bytes,
                                         kCCKeySizeAES256,
                                         NULL,
                                         data.bytes,
                                         data.length,
                                         encryptedData.mutableBytes,
                                         encryptedData.length,
                                         &numBytesEncrypted);
    
    if (cryptStatus != kCCSuccess) {
        NSLog(@"Encryption failed with status: %d", cryptStatus);
        return NO;
    }
    
    encryptedData.length = numBytesEncrypted;
    
    return [self writeData:encryptedData toFile:fileName inDirectory:@"UserData"];
}

- (NSData *)readEncryptedDataFromFile:(NSString *)fileName withKey:(NSString *)key {
    NSData *encryptedData = [self readDataFromFile:fileName inDirectory:@"UserData"];
    
    if (!encryptedData || !key) {
        return nil;
    }
    
    // 使用AES解密数据
    NSData *keyData = [key dataUsingEncoding:NSUTF8StringEncoding];
    NSMutableData *decryptedData = [NSMutableData dataWithLength:encryptedData.length];
    
    size_t numBytesDecrypted = 0;
    CCCryptorStatus cryptStatus = CCCrypt(kCCDecrypt,
                                         kCCAlgorithmAES128,
                                         kCCOptionPKCS7Padding,
                                         keyData.bytes,
                                         kCCKeySizeAES256,
                                         NULL,
                                         encryptedData.bytes,
                                         encryptedData.length,
                                         decryptedData.mutableBytes,
                                         decryptedData.length,
                                         &numBytesDecrypted);
    
    if (cryptStatus != kCCSuccess) {
        NSLog(@"Decryption failed with status: %d", cryptStatus);
        return nil;
    }
    
    decryptedData.length = numBytesDecrypted;
    
    return decryptedData;
}

@end
```

## 数据持久化性能优化

### 性能监控器

```objc
@interface DataPersistencePerformanceMonitor : NSObject

@property (nonatomic, strong) NSMutableDictionary *operationMetrics;
@property (nonatomic, strong) dispatch_queue_t metricsQueue;

+ (instancetype)sharedMonitor;

// 性能监控
- (void)startMonitoringOperation:(NSString *)operationName;
- (void)endMonitoringOperation:(NSString *)operationName;
- (NSDictionary *)getPerformanceReport;

// Core Data性能优化
- (void)optimizeCoreDataPerformance:(NSManagedObjectContext *)context;
- (void)configureFetchRequest:(NSFetchRequest *)fetchRequest forOptimalPerformance;

// SQLite性能优化
- (void)optimizeSQLiteDatabase:(sqlite3 *)database;
- (NSString *)generateOptimizedQuery:(NSString *)baseQuery withParameters:(NSDictionary *)parameters;

// 文件操作性能优化
- (void)optimizeFileOperations;
- (NSData *)compressData:(NSData *)data;
- (NSData *)decompressData:(NSData *)compressedData;

@end

@implementation DataPersistencePerformanceMonitor

+ (instancetype)sharedMonitor {
    static DataPersistencePerformanceMonitor *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.operationMetrics = [NSMutableDictionary dictionary];
        self.metricsQueue = dispatch_queue_create("com.app.metrics", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}

- (void)startMonitoringOperation:(NSString *)operationName {
    dispatch_barrier_async(self.metricsQueue, ^{
        NSMutableDictionary *operation = [NSMutableDictionary dictionary];
        operation[@"start_time"] = @([[NSDate date] timeIntervalSince1970]);
        operation[@"count"] = @([self.operationMetrics[operationName][@"count"] integerValue] + 1);
        
        self.operationMetrics[operationName] = operation;
    });
}

- (void)endMonitoringOperation:(NSString *)operationName {
    dispatch_barrier_async(self.metricsQueue, ^{
        NSMutableDictionary *operation = [self.operationMetrics[operationName] mutableCopy];
        if (operation && operation[@"start_time"]) {
            NSTimeInterval duration = [[NSDate date] timeIntervalSince1970] - [operation[@"start_time"] doubleValue];
            
            // 计算平均执行时间
            NSNumber *totalDuration = operation[@"total_duration"] ?: @(0);
            operation[@"total_duration"] = @([totalDuration doubleValue] + duration);
            operation[@"average_duration"] = @([operation[@"total_duration"] doubleValue] / [operation[@"count"] integerValue]);
            operation[@"last_duration"] = @(duration);
            
            // 记录最大和最小执行时间
            NSNumber *maxDuration = operation[@"max_duration"];
            NSNumber *minDuration = operation[@"min_duration"];
            
            if (!maxDuration || duration > [maxDuration doubleValue]) {
                operation[@"max_duration"] = @(duration);
            }
            
            if (!minDuration || duration < [minDuration doubleValue]) {
                operation[@"min_duration"] = @(duration);
            }
            
            [operation removeObjectForKey:@"start_time"];
            self.operationMetrics[operationName] = operation;
        }
    });
}

- (NSDictionary *)getPerformanceReport {
    __block NSDictionary *report;
    dispatch_sync(self.metricsQueue, ^{
        report = [self.operationMetrics copy];
    });
    return report;
}

- (void)optimizeCoreDataPerformance:(NSManagedObjectContext *)context {
    // 设置合并策略
    context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    
    // 设置撤销管理器
    context.undoManager = nil;
    
    // 配置自动保存
    context.automaticallyMergesChangesFromParent = YES;
    
    // 设置删除规则
    context.shouldDeleteInaccessibleFaults = YES;
    
    NSLog(@"Core Data performance optimization applied");
}

- (void)configureFetchRequest:(NSFetchRequest *)fetchRequest forOptimalPerformance {
    // 设置批量大小
    fetchRequest.fetchBatchSize = 20;
    
    // 设置结果类型
    fetchRequest.resultType = NSManagedObjectResultType;
    
    // 设置关系键路径
    fetchRequest.relationshipKeyPathsForPrefetching = @[@"posts", @"profile"];
    
    // 设置属性键路径
    fetchRequest.propertiesToFetch = @[@"username", @"email", @"createdDate"];
    
    // 启用返回不同值
    fetchRequest.returnsDistinctResults = YES;
    
    // 设置包含待处理更改
    fetchRequest.includesPendingChanges = NO;
    
    NSLog(@"Fetch request optimized for performance");
}

- (void)optimizeSQLiteDatabase:(sqlite3 *)database {
    // 设置页面大小
    sqlite3_exec(database, "PRAGMA page_size = 4096;", NULL, NULL, NULL);
    
    // 设置缓存大小
    sqlite3_exec(database, "PRAGMA cache_size = 10000;", NULL, NULL, NULL);
    
    // 设置同步模式
    sqlite3_exec(database, "PRAGMA synchronous = NORMAL;", NULL, NULL, NULL);
    
    // 设置日志模式
    sqlite3_exec(database, "PRAGMA journal_mode = WAL;", NULL, NULL, NULL);
    
    // 设置临时存储
    sqlite3_exec(database, "PRAGMA temp_store = MEMORY;", NULL, NULL, NULL);
    
    // 设置锁定模式
    sqlite3_exec(database, "PRAGMA locking_mode = EXCLUSIVE;", NULL, NULL, NULL);
    
    NSLog(@"SQLite database optimization completed");
}

- (NSString *)generateOptimizedQuery:(NSString *)baseQuery withParameters:(NSDictionary *)parameters {
    NSMutableString *optimizedQuery = [baseQuery mutableCopy];
    
    // 添加索引提示
    if ([parameters[@"use_index"] boolValue]) {
        NSString *indexHint = [NSString stringWithFormat:@" INDEXED BY %@", parameters[@"index_name"]];
        [optimizedQuery appendString:indexHint];
    }
    
    // 添加LIMIT子句
    if (parameters[@"limit"]) {
        NSString *limitClause = [NSString stringWithFormat:@" LIMIT %@", parameters[@"limit"]];
        [optimizedQuery appendString:limitClause];
    }
    
    // 添加ORDER BY优化
    if (parameters[@"order_by"]) {
        NSString *orderClause = [NSString stringWithFormat:@" ORDER BY %@", parameters[@"order_by"]];
        [optimizedQuery appendString:orderClause];
    }
    
    return [optimizedQuery copy];
}

- (void)optimizeFileOperations {
    // 设置文件系统缓存
    NSURLResourceKey keys[] = {
        NSURLIsDirectoryKey,
        NSURLFileSizeKey,
        NSURLContentModificationDateKey
    };
    
    // 预加载文件属性
    NSArray *resourceKeys = @[
        NSURLIsDirectoryKey,
        NSURLFileSizeKey,
        NSURLContentModificationDateKey
    ];
    
    NSLog(@"File operations optimized with resource keys: %@", resourceKeys);
}

- (NSData *)compressData:(NSData *)data {
    if (!data || data.length == 0) {
        return data;
    }
    
    NSMutableData *compressedData = [NSMutableData data];
    
    // 使用zlib压缩
    z_stream stream;
    stream.zalloc = Z_NULL;
    stream.zfree = Z_NULL;
    stream.opaque = Z_NULL;
    stream.total_out = 0;
    stream.next_in = (Bytef *)[data bytes];
    stream.avail_in = (uInt)[data length];
    
    if (deflateInit(&stream, Z_DEFAULT_COMPRESSION) != Z_OK) {
        return data;
    }
    
    Byte buffer[1024];
    
    do {
        stream.next_out = buffer;
        stream.avail_out = sizeof(buffer);
        
        deflate(&stream, Z_FINISH);
        
        NSUInteger bytesProcessed = sizeof(buffer) - stream.avail_out;
        [compressedData appendBytes:buffer length:bytesProcessed];
        
    } while (stream.avail_out == 0);
    
    deflateEnd(&stream);
    
    return [compressedData copy];
}

- (NSData *)decompressData:(NSData *)compressedData {
    if (!compressedData || compressedData.length == 0) {
        return compressedData;
    }
    
    NSMutableData *decompressedData = [NSMutableData data];
    
    // 使用zlib解压缩
    z_stream stream;
    stream.zalloc = Z_NULL;
    stream.zfree = Z_NULL;
    stream.opaque = Z_NULL;
    stream.total_out = 0;
    stream.next_in = (Bytef *)[compressedData bytes];
    stream.avail_in = (uInt)[compressedData length];
    
    if (inflateInit(&stream) != Z_OK) {
        return compressedData;
    }
    
    Byte buffer[1024];
    
    do {
        stream.next_out = buffer;
        stream.avail_out = sizeof(buffer);
        
        int status = inflate(&stream, Z_NO_FLUSH);
        
        if (status == Z_STREAM_ERROR) {
            inflateEnd(&stream);
            return compressedData;
        }
        
        NSUInteger bytesProcessed = sizeof(buffer) - stream.avail_out;
        [decompressedData appendBytes:buffer length:bytesProcessed];
        
    } while (stream.avail_out == 0);
    
    inflateEnd(&stream);
    
    return [decompressedData copy];
}

@end
```

## 数据迁移管理

### 数据迁移器

```objc
@interface DataMigrationManager : NSObject

@property (nonatomic, strong) NSString *currentVersion;
@property (nonatomic, strong) NSArray *migrationSteps;

+ (instancetype)sharedManager;

// 版本管理
- (NSString *)getCurrentDataVersion;
- (BOOL)needsMigration;
- (NSArray *)getAvailableMigrations;

// Core Data迁移
- (BOOL)migrateCoreDataFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion;
- (NSMappingModel *)createMappingModelFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion;

// SQLite迁移
- (BOOL)migrateSQLiteFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion;
- (NSArray *)getSQLiteMigrationStepsFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion;

// 文件迁移
- (BOOL)migrateFilesFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion;
- (void)backupDataBeforeMigration;
- (void)restoreDataFromBackup;

@end

@implementation DataMigrationManager

+ (instancetype)sharedManager {
    static DataMigrationManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupMigrationSteps];
        self.currentVersion = [self getCurrentDataVersion];
    }
    return self;
}

- (void)setupMigrationSteps {
    self.migrationSteps = @[
        @{
            @"from_version": @"1.0",
            @"to_version": @"1.1",
            @"description": @"Add user profile table",
            @"core_data_model": @"UserModel_1_1",
            @"sqlite_steps": @[
                @"ALTER TABLE users ADD COLUMN profile_data TEXT;",
                @"CREATE INDEX idx_users_profile ON users(profile_data);"
            ]
        },
        @{
            @"from_version": @"1.1",
            @"to_version": @"1.2",
            @"description": @"Add post categories",
            @"core_data_model": @"UserModel_1_2",
            @"sqlite_steps": @[
                @"CREATE TABLE categories (id INTEGER PRIMARY KEY, name TEXT UNIQUE);",
                @"ALTER TABLE posts ADD COLUMN category_id INTEGER REFERENCES categories(id);"
            ]
        },
        @{
            @"from_version": @"1.2",
            @"to_version": @"2.0",
            @"description": @"Major schema restructure",
            @"core_data_model": @"UserModel_2_0",
            @"sqlite_steps": @[
                @"CREATE TABLE new_users AS SELECT user_id, username, email, created_date FROM users;",
                @"DROP TABLE users;",
                @"ALTER TABLE new_users RENAME TO users;"
            ]
        }
    ];
}

- (NSString *)getCurrentDataVersion {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    NSString *version = [defaults stringForKey:@"DataVersion"];
    
    if (!version) {
        version = @"1.0";
        [defaults setObject:version forKey:@"DataVersion"];
        [defaults synchronize];
    }
    
    return version;
}

- (BOOL)needsMigration {
    NSString *latestVersion = [self.migrationSteps lastObject][@"to_version"];
    return ![self.currentVersion isEqualToString:latestVersion];
}

- (NSArray *)getAvailableMigrations {
    NSMutableArray *availableMigrations = [NSMutableArray array];
    
    for (NSDictionary *step in self.migrationSteps) {
        if ([step[@"from_version"] isEqualToString:self.currentVersion] ||
            [availableMigrations count] > 0) {
            [availableMigrations addObject:step];
        }
    }
    
    return [availableMigrations copy];
}

- (BOOL)migrateCoreDataFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion {
    NSLog(@"Starting Core Data migration from %@ to %@", fromVersion, toVersion);
    
    // 获取迁移步骤
    NSDictionary *migrationStep = [self getMigrationStepFromVersion:fromVersion toVersion:toVersion];
    if (!migrationStep) {
        NSLog(@"No migration step found for version %@ to %@", fromVersion, toVersion);
        return NO;
    }
    
    // 创建映射模型
    NSMappingModel *mappingModel = [self createMappingModelFromVersion:fromVersion toVersion:toVersion];
    if (!mappingModel) {
        NSLog(@"Failed to create mapping model");
        return NO;
    }
    
    // 执行迁移
    NSError *error;
    NSMigrationManager *migrationManager = [[NSMigrationManager alloc] initWithSourceModel:[self getModelForVersion:fromVersion]
                                                                           destinationModel:[self getModelForVersion:toVersion]];
    
    NSURL *sourceURL = [self getStoreURLForVersion:fromVersion];
    NSURL *destinationURL = [self getStoreURLForVersion:toVersion];
    
    BOOL success = [migrationManager migrateStoreFromURL:sourceURL
                                                    type:NSSQLiteStoreType
                                                 options:nil
                                        withMappingModel:mappingModel
                                        toDestinationURL:destinationURL
                                         destinationType:NSSQLiteStoreType
                                      destinationOptions:nil
                                                   error:&error];
    
    if (!success) {
        NSLog(@"Core Data migration failed: %@", error);
        return NO;
    }
    
    NSLog(@"Core Data migration completed successfully");
    return YES;
}

- (NSMappingModel *)createMappingModelFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion {
    NSManagedObjectModel *sourceModel = [self getModelForVersion:fromVersion];
    NSManagedObjectModel *destinationModel = [self getModelForVersion:toVersion];
    
    NSMappingModel *mappingModel = [NSMappingModel inferredMappingModelForSourceModel:sourceModel
                                                                     destinationModel:destinationModel
                                                                                error:nil];
    
    return mappingModel;
}

- (BOOL)migrateSQLiteFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion {
    NSLog(@"Starting SQLite migration from %@ to %@", fromVersion, toVersion);
    
    NSArray *migrationSteps = [self getSQLiteMigrationStepsFromVersion:fromVersion toVersion:toVersion];
    
    SQLiteManager *sqliteManager = [SQLiteManager sharedManager];
    
    BOOL success = [sqliteManager executeTransaction:^BOOL{
        for (NSString *sql in migrationSteps) {
            char *errorMessage;
            int result = sqlite3_exec(sqliteManager.database, [sql UTF8String], NULL, NULL, &errorMessage);
            
            if (result != SQLITE_OK) {
                NSLog(@"SQLite migration step failed: %s", errorMessage);
                sqlite3_free(errorMessage);
                return NO;
            }
        }
        return YES;
    }];
    
    if (success) {
        NSLog(@"SQLite migration completed successfully");
    }
    
    return success;
}

- (NSArray *)getSQLiteMigrationStepsFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion {
    NSDictionary *migrationStep = [self getMigrationStepFromVersion:fromVersion toVersion:toVersion];
    return migrationStep[@"sqlite_steps"] ?: @[];
}

- (BOOL)migrateFilesFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion {
    NSLog(@"Starting file migration from %@ to %@", fromVersion, toVersion);
    
    FileStorageManager *fileManager = [FileStorageManager sharedManager];
    
    // 备份当前文件
    [self backupDataBeforeMigration];
    
    // 执行文件迁移逻辑
    if ([fromVersion isEqualToString:@"1.0"] && [toVersion isEqualToString:@"1.1"]) {
        // 迁移用户配置文件格式
        NSArray *userFiles = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:[fileManager getDirectoryPath:@"UserData"] error:nil];
        
        for (NSString *fileName in userFiles) {
            if ([fileName hasSuffix:@".plist"]) {
                // 将plist文件转换为JSON格式
                NSDictionary *plistData = [NSDictionary dictionaryWithContentsOfFile:[[fileManager getDirectoryPath:@"UserData"] stringByAppendingPathComponent:fileName]];
                
                if (plistData) {
                    NSString *jsonFileName = [[fileName stringByDeletingPathExtension] stringByAppendingPathExtension:@"json"];
                    [fileManager writeJSONObject:plistData toFile:jsonFileName];
                    [fileManager deleteFile:fileName inDirectory:@"UserData"];
                }
            }
        }
    }
    
    NSLog(@"File migration completed successfully");
    return YES;
}

- (void)backupDataBeforeMigration {
    NSString *backupPath = [NSTemporaryDirectory() stringByAppendingPathComponent:@"DataBackup"];
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    // 创建备份目录
    [fileManager createDirectoryAtPath:backupPath withIntermediateDirectories:YES attributes:nil error:nil];
    
    // 备份Documents目录
    FileStorageManager *storageManager = [FileStorageManager sharedManager];
    NSString *documentsPath = storageManager.documentsPath;
    NSString *backupDocumentsPath = [backupPath stringByAppendingPathComponent:@"Documents"];
    
    NSError *error;
    [fileManager copyItemAtPath:documentsPath toPath:backupDocumentsPath error:&error];
    
    if (error) {
        NSLog(@"Failed to backup documents: %@", error);
    } else {
        NSLog(@"Data backup completed at: %@", backupPath);
    }
}

- (void)restoreDataFromBackup {
    NSString *backupPath = [NSTemporaryDirectory() stringByAppendingPathComponent:@"DataBackup"];
    NSString *backupDocumentsPath = [backupPath stringByAppendingPathComponent:@"Documents"];
    
    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    if ([fileManager fileExistsAtPath:backupDocumentsPath]) {
        FileStorageManager *storageManager = [FileStorageManager sharedManager];
        NSString *documentsPath = storageManager.documentsPath;
        
        // 删除当前Documents目录
        [fileManager removeItemAtPath:documentsPath error:nil];
        
        // 恢复备份
        NSError *error;
        [fileManager copyItemAtPath:backupDocumentsPath toPath:documentsPath error:&error];
        
        if (error) {
            NSLog(@"Failed to restore data from backup: %@", error);
        } else {
            NSLog(@"Data restored from backup successfully");
        }
    }
}

// 辅助方法
- (NSDictionary *)getMigrationStepFromVersion:(NSString *)fromVersion toVersion:(NSString *)toVersion {
    for (NSDictionary *step in self.migrationSteps) {
        if ([step[@"from_version"] isEqualToString:fromVersion] &&
            [step[@"to_version"] isEqualToString:toVersion]) {
            return step;
        }
    }
    return nil;
}

- (NSManagedObjectModel *)getModelForVersion:(NSString *)version {
    // 实现获取指定版本的Core Data模型
    NSBundle *bundle = [NSBundle mainBundle];
    NSString *modelPath = [bundle pathForResource:[NSString stringWithFormat:@"UserModel_%@", [version stringByReplacingOccurrencesOfString:@"." withString:@"_"]] ofType:@"momd"];
    
    if (modelPath) {
        NSURL *modelURL = [NSURL fileURLWithPath:modelPath];
        return [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
    }
    
    return nil;
}

- (NSURL *)getStoreURLForVersion:(NSString *)version {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentsDirectory = [paths firstObject];
    NSString *storePath = [documentsDirectory stringByAppendingPathComponent:[NSString stringWithFormat:@"UserData_%@.sqlite", version]];
    
    return [NSURL fileURLWithPath:storePath];
}

@end
```

## 数据持久化最佳实践

### 选择合适的持久化方案

```objc
@interface DataPersistenceBestPractices : NSObject

+ (instancetype)sharedPractices;

// 方案选择指导
- (NSString *)recommendPersistenceStrategyForDataType:(NSString *)dataType
                                           complexity:(NSString *)complexity
                                         performanceRequirement:(NSString *)performance;

// 开发最佳实践
- (NSArray *)getCoreDataBestPractices;
- (NSArray *)getSQLiteBestPractices;
- (NSArray *)getFileStorageBestPractices;

// 性能优化建议
- (NSDictionary *)getPerformanceOptimizationTips;

// 安全性建议
- (NSArray *)getSecurityBestPractices;

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

- (NSString *)recommendPersistenceStrategyForDataType:(NSString *)dataType
                                           complexity:(NSString *)complexity
                                         performanceRequirement:(NSString *)performance {
    
    // 简单数据类型推荐
    if ([complexity isEqualToString:@"simple"]) {
        if ([dataType isEqualToString:@"user_preferences"]) {
            return @"NSUserDefaults - 适合存储用户偏好设置";
        } else if ([dataType isEqualToString:@"cache_data"]) {
            return @"File Storage - 适合临时缓存数据";
        }
    }
    
    // 复杂数据类型推荐
    else if ([complexity isEqualToString:@"complex"]) {
        if ([performance isEqualToString:@"high"]) {
            return @"SQLite - 高性能复杂查询场景";
        } else {
            return @"Core Data - 复杂对象关系管理";
        }
    }
    
    // 大数据量推荐
    else if ([complexity isEqualToString:@"large_dataset"]) {
        return @"SQLite + 分页 - 大数据量高效处理";
    }
    
    return @"根据具体需求选择合适方案";
}

- (NSArray *)getCoreDataBestPractices {
    return @[
        @"1. 模型设计原则",
        @"   • 合理设计实体关系，避免过度复杂化",
        @"   • 使用适当的删除规则（Cascade、Nullify、Deny）",
        @"   • 为经常查询的属性添加索引",
        @"   • 使用轻量级迁移处理模型变更",
        @"",
        @"2. 性能优化",
        @"   • 使用NSFetchedResultsController管理表视图数据",
        @"   • 设置合适的fetchBatchSize避免内存问题",
        @"   • 使用relationshipKeyPathsForPrefetching预加载关系",
        @"   • 在后台队列执行耗时操作",
        @"",
        @"3. 内存管理",
        @"   • 及时释放不需要的NSManagedObject",
        @"   • 使用refreshObject:mergeChanges:刷新对象",
        @"   • 避免在主线程执行大量数据操作",
        @"",
        @"4. 错误处理",
        @"   • 始终检查保存操作的错误",
        @"   • 实现适当的数据验证逻辑",
        @"   • 提供用户友好的错误信息"
    ];
}

- (NSArray *)getSQLiteBestPractices {
    return @[
        @"1. 数据库设计",
        @"   • 规范化数据库结构，避免数据冗余",
        @"   • 为经常查询的列创建索引",
        @"   • 使用适当的数据类型优化存储空间",
        @"   • 设计合理的主键和外键约束",
        @"",
        @"2. 查询优化",
        @"   • 使用参数化查询防止SQL注入",
        @"   • 避免SELECT *，只查询需要的列",
        @"   • 使用LIMIT限制结果集大小",
        @"   • 合理使用JOIN操作",
        @"",
        @"3. 事务管理",
        @"   • 将相关操作包装在事务中",
        @"   • 保持事务尽可能短小",
        @"   • 正确处理事务回滚",
        @"",
        @"4. 连接管理",
        @"   • 使用连接池管理数据库连接",
        @"   • 及时关闭不需要的连接",
        @"   • 处理数据库锁定情况"
    ];
}

- (NSArray *)getFileStorageBestPractices {
    return @[
        @"1. 文件组织",
        @"   • 使用合理的目录结构组织文件",
        @"   • 为不同类型数据创建专门目录",
        @"   • 使用有意义的文件命名规则",
        @"   • 定期清理临时文件和缓存",
        @"",
        @"2. 数据格式",
        @"   • 选择合适的序列化格式（JSON、Plist、Binary）",
        @"   • 考虑数据压缩以节省空间",
        @"   • 使用版本控制处理格式变更",
        @"",
        @"3. 安全性",
        @"   • 对敏感数据进行加密存储",
        @"   • 使用文件属性控制访问权限",
        @"   • 避免在可访问位置存储敏感信息",
        @"",
        @"4. 性能考虑",
        @"   • 使用异步I/O操作避免阻塞主线程",
        @"   • 实现智能缓存策略",
        @"   • 监控磁盘空间使用情况"
    ];
}

- (NSDictionary *)getPerformanceOptimizationTips {
    return @{
        @"Core Data": @[
            @"使用NSPrivateQueueConcurrencyType进行后台操作",
            @"实现增量数据同步",
            @"使用NSFetchRequest的includesSubentities属性",
            @"避免在循环中频繁保存上下文"
        ],
        @"SQLite": @[
            @"使用WAL模式提高并发性能",
            @"调整页面大小和缓存大小",
            @"使用ANALYZE命令优化查询计划",
            @"实现读写分离架构"
        ],
        @"File Storage": @[
            @"使用内存映射文件处理大文件",
            @"实现分块读写大数据",
            @"使用NSFileCoordinator处理文件协调",
            @"利用系统缓存机制"
        ],
        @"通用优化": @[
            @"监控内存使用情况",
            @"实现数据预加载策略",
            @"使用性能分析工具定位瓶颈",
            @"定期进行性能测试"
        ]
    };
}

- (NSArray *)getSecurityBestPractices {
    return @[
        @"1. 数据加密",
        @"   • 使用AES加密敏感数据",
        @"   • 将加密密钥存储在Keychain中",
        @"   • 实现密钥轮换机制",
        @"   • 对数据库文件进行整体加密",
        @"",
        @"2. 访问控制",
        @"   • 使用文件保护属性",
        @"   • 实现用户身份验证",
        @"   • 限制数据访问权限",
        @"   • 记录数据访问日志",
        @"",
        @"3. 数据完整性",
        @"   • 使用校验和验证数据完整性",
        @"   • 实现数据备份和恢复机制",
        @"   • 定期验证数据一致性",
        @"   • 处理数据损坏情况",
        @"",
        @"4. 隐私保护",
        @"   • 遵循最小权限原则",
        @"   • 实现数据匿名化",
        @"   • 提供数据删除功能",
        @"   • 符合隐私法规要求"
    ];
}

@end
```

### 综合示例：数据持久化管理器

```objc
@interface ComprehensiveDataManager : NSObject

@property (nonatomic, strong) CoreDataManager *coreDataManager;
@property (nonatomic, strong) SQLiteManager *sqliteManager;
@property (nonatomic, strong) FileStorageManager *fileStorageManager;
@property (nonatomic, strong) DataPersistencePerformanceMonitor *performanceMonitor;
@property (nonatomic, strong) DataMigrationManager *migrationManager;

+ (instancetype)sharedManager;

// 统一数据接口
- (void)saveUser:(User *)user completion:(void(^)(BOOL success, NSError *error))completion;
- (void)fetchUsersWithPredicate:(NSPredicate *)predicate completion:(void(^)(NSArray *users, NSError *error))completion;
- (void)deleteUser:(User *)user completion:(void(^)(BOOL success, NSError *error))completion;

// 数据同步
- (void)syncDataWithRemoteServer:(void(^)(BOOL success, NSError *error))completion;

// 性能监控
- (NSDictionary *)getPerformanceMetrics;

// 数据迁移
- (void)performDataMigrationIfNeeded:(void(^)(BOOL success, NSError *error))completion;

@end

@implementation ComprehensiveDataManager

+ (instancetype)sharedManager {
    static ComprehensiveDataManager *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        self.coreDataManager = [CoreDataManager sharedManager];
        self.sqliteManager = [SQLiteManager sharedManager];
        self.fileStorageManager = [FileStorageManager sharedManager];
        self.performanceMonitor = [DataPersistencePerformanceMonitor sharedMonitor];
        self.migrationManager = [DataMigrationManager sharedManager];
        
        [self setupDataManagers];
    }
    return self;
}

- (void)setupDataManagers {
    // 初始化Core Data
    [self.coreDataManager setupCoreDataStack];
    
    // 初始化SQLite
    [self.sqliteManager openDatabase];
    
    // 检查数据迁移
    if ([self.migrationManager needsMigration]) {
        [self performDataMigrationIfNeeded:^(BOOL success, NSError *error) {
            if (success) {
                NSLog(@"Data migration completed successfully");
            } else {
                NSLog(@"Data migration failed: %@", error);
            }
        }];
    }
}

- (void)saveUser:(User *)user completion:(void(^)(BOOL success, NSError *error))completion {
    [self.performanceMonitor startMonitoringOperation:@"save_user"];
    
    // 使用Core Data保存用户基本信息
    [self.coreDataManager saveUser:user completion:^(BOOL success, NSError *error) {
        if (success) {
            // 使用文件存储保存用户头像
            if (user.avatarImage) {
                [self.fileStorageManager saveImage:user.avatarImage withName:[NSString stringWithFormat:@"avatar_%@", user.username]];
            }
            
            // 使用SQLite保存用户活动日志
            [self.sqliteManager insertUserActivityLog:user.username activity:@"user_created" timestamp:[NSDate date]];
        }
        
        [self.performanceMonitor endMonitoringOperation:@"save_user"];
        
        if (completion) {
            completion(success, error);
        }
    }];
}

- (void)fetchUsersWithPredicate:(NSPredicate *)predicate completion:(void(^)(NSArray *users, NSError *error))completion {
    [self.performanceMonitor startMonitoringOperation:@"fetch_users"];
    
    [self.coreDataManager fetchUsersWithPredicate:predicate completion:^(NSArray *users, NSError *error) {
        if (users) {
            // 为每个用户加载头像
            for (User *user in users) {
                NSString *avatarName = [NSString stringWithFormat:@"avatar_%@", user.username];
                UIImage *avatar = [self.fileStorageManager loadImageWithName:avatarName];
                user.avatarImage = avatar;
            }
        }
        
        [self.performanceMonitor endMonitoringOperation:@"fetch_users"];
        
        if (completion) {
            completion(users, error);
        }
    }];
}

- (void)deleteUser:(User *)user completion:(void(^)(BOOL success, NSError *error))completion {
    [self.performanceMonitor startMonitoringOperation:@"delete_user"];
    
    [self.coreDataManager deleteUser:user completion:^(BOOL success, NSError *error) {
        if (success) {
            // 删除用户头像文件
            NSString *avatarName = [NSString stringWithFormat:@"avatar_%@", user.username];
            [self.fileStorageManager deleteFile:avatarName inDirectory:@"Images"];
            
            // 记录删除日志
            [self.sqliteManager insertUserActivityLog:user.username activity:@"user_deleted" timestamp:[NSDate date]];
        }
        
        [self.performanceMonitor endMonitoringOperation:@"delete_user"];
        
        if (completion) {
            completion(success, error);
        }
    }];
}

- (void)syncDataWithRemoteServer:(void(^)(BOOL success, NSError *error))completion {
    [self.performanceMonitor startMonitoringOperation:@"data_sync"];
    
    // 实现数据同步逻辑
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 模拟网络同步
        sleep(2);
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.performanceMonitor endMonitoringOperation:@"data_sync"];
            
            if (completion) {
                completion(YES, nil);
            }
        });
    });
}

- (NSDictionary *)getPerformanceMetrics {
    return [self.performanceMonitor getPerformanceReport];
}

- (void)performDataMigrationIfNeeded:(void(^)(BOOL success, NSError *error))completion {
    if (![self.migrationManager needsMigration]) {
        if (completion) {
            completion(YES, nil);
        }
        return;
    }
    
    NSArray *migrations = [self.migrationManager getAvailableMigrations];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        BOOL allSuccess = YES;
        NSError *migrationError = nil;
        
        for (NSDictionary *migration in migrations) {
            NSString *fromVersion = migration[@"from_version"];
            NSString *toVersion = migration[@"to_version"];
            
            // 执行Core Data迁移
            BOOL coreDataSuccess = [self.migrationManager migrateCoreDataFromVersion:fromVersion toVersion:toVersion];
            
            // 执行SQLite迁移
            BOOL sqliteSuccess = [self.migrationManager migrateSQLiteFromVersion:fromVersion toVersion:toVersion];
            
            // 执行文件迁移
            BOOL fileSuccess = [self.migrationManager migrateFilesFromVersion:fromVersion toVersion:toVersion];
            
            if (!coreDataSuccess || !sqliteSuccess || !fileSuccess) {
                allSuccess = NO;
                migrationError = [NSError errorWithDomain:@"DataMigrationError" code:1001 userInfo:@{NSLocalizedDescriptionKey: @"Migration failed"}];
                break;
            }
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(allSuccess, migrationError);
            }
        });
    });
}

@end
```

## 总结

本文深入探讨了iOS数据持久化的三种主要方案：Core Data、SQLite和文件存储。每种方案都有其适用场景和优缺点：

### 核心要点

1. **Core Data**：适合复杂对象关系管理，提供了强大的ORM功能和自动内存管理
2. **SQLite**：适合高性能查询和大数据量处理，提供了灵活的SQL操作能力
3. **文件存储**：适合简单数据和媒体文件存储，实现简单且高效

### 选择建议

- **简单配置数据**：使用NSUserDefaults或Plist文件
- **复杂业务对象**：使用Core Data进行对象关系管理
- **高性能查询**：使用SQLite实现复杂查询逻辑
- **媒体文件**：使用文件系统直接存储
- **混合场景**：结合多种方案，发挥各自优势

### 最佳实践

1. **性能优化**：监控操作性能，优化查询和存储策略
2. **数据迁移**：制定完善的版本迁移方案
3. **安全性**：对敏感数据进行加密保护
4. **错误处理**：实现完善的错误处理和恢复机制
5. **测试验证**：进行充分的单元测试和集成测试

通过合理选择和组合使用这些数据持久化技术，可以构建出高效、稳定、安全的iOS应用数据层架构。在实际开发中，应根据具体的业务需求、性能要求和团队技术栈来选择最适合的方案。

### Core Data模型设计

```objc
// User实体
@interface User : NSManagedObject

@property (nonatomic, strong) NSString *userID;
@property (nonatomic, strong) NSString *username;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSDate *createdDate;
@property (nonatomic, strong) NSDate *lastLoginDate;
@property (nonatomic, strong) NSSet<Post *> *posts;
@property (nonatomic, strong) Profile *profile;

@end

@implementation User

@dynamic userID, username, email, createdDate, lastLoginDate, posts, profile;

+ (NSString *)entityName {
    return @"User";
}

+ (NSFetchRequest<User *> *)fetchRequest {
    return [NSFetchRequest fetchRequestWithEntityName:[self entityName]];
}

- (void)awakeFromInsert {
    [super awakeFromInsert];
    self.createdDate = [NSDate date];
    self.userID = [[NSUUID UUID] UUIDString];
}

@end

// Post实体
@interface Post : NSManagedObject

@property (nonatomic, strong) NSString *postID;
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSString *content;
@property (nonatomic, strong) NSDate *publishDate;
@property (nonatomic, assign) BOOL isPublished;
@property (nonatomic, strong) User *author;
@property (nonatomic, strong) NSSet<Comment *> *comments;
@property (nonatomic, strong) NSSet<Tag *> *tags;

@end

@implementation Post

@dynamic postID, title, content, publishDate, isPublished, author, comments, tags;

+ (NSString *)entityName {
    return @"Post";
}

+ (NSFetchRequest<Post *> *)fetchRequest {
    return [NSFetchRequest fetchRequestWithEntityName:[self entityName]];
}

- (void)awakeFromInsert {
    [super awakeFromInsert];
    self.postID = [[NSUUID UUID] UUIDString];
    self.publishDate = [NSDate date];
    self.isPublished = NO;
}

@end
```

### Core Data操作管理器

```objc
@interface CoreDataManager : NSObject

@property (nonatomic, strong, readonly) NSManagedObjectContext *mainContext;
@property (nonatomic, strong, readonly) NSManagedObjectContext *backgroundContext;

+ (instancetype)sharedManager;

// CRUD操作
- (User *)createUserWithUsername:(NSString *)username email:(NSString *)email;
- (NSArray<User *> *)fetchUsersWithPredicate:(NSPredicate *)predicate
                                   sortBy:(NSArray<NSSortDescriptor *> *)sortDescriptors
                                    limit:(NSUInteger)limit;
- (BOOL)updateUser:(User *)user withInfo:(NSDictionary *)info;
- (BOOL)deleteUser:(User *)user;

// 批量操作
- (BOOL)batchInsertUsers:(NSArray<NSDictionary *> *)usersData;
- (BOOL)batchUpdateUsersWithPredicate:(NSPredicate *)predicate
                              updates:(NSDictionary *)updates;
- (BOOL)batchDeleteUsersWithPredicate:(NSPredicate *)predicate;

// 关系管理
- (BOOL)addPost:(Post *)post toUser:(User *)user;
- (BOOL)removePost:(Post *)post fromUser:(User *)user;

// 数据同步
- (void)saveContext;
- (void)saveContextAndWait;
- (void)performBackgroundTask:(void (^)(NSManagedObjectContext *context))block;

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
        [self setupCoreDataStack];
    }
    return self;
}

- (void)setupCoreDataStack {
    // 获取共享的数据持久化管理器
    DataPersistenceManager *persistenceManager = [DataPersistenceManager sharedManager];
    _mainContext = persistenceManager.managedObjectContext;
    
    // 创建后台上下文
    _backgroundContext = [[NSManagedObjectContext alloc]
                         initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    _backgroundContext.parentContext = _mainContext;
}

- (User *)createUserWithUsername:(NSString *)username email:(NSString *)email {
    User *user = [NSEntityDescription insertNewObjectForEntityForName:@"User"
                                               inManagedObjectContext:self.mainContext];
    user.username = username;
    user.email = email;
    
    NSError *error;
    if ([self.mainContext save:&error]) {
        return user;
    } else {
        NSLog(@"Failed to create user: %@", error);
        return nil;
    }
}

- (NSArray<User *> *)fetchUsersWithPredicate:(NSPredicate *)predicate
                                      sortBy:(NSArray<NSSortDescriptor *> *)sortDescriptors
                                       limit:(NSUInteger)limit {
    
    NSFetchRequest<User *> *request = [User fetchRequest];
    
    if (predicate) {
        request.predicate = predicate;
    }
    
    if (sortDescriptors) {
        request.sortDescriptors = sortDescriptors;
    }
    
    if (limit > 0) {
        request.fetchLimit = limit;
    }
    
    // 性能优化
    request.returnsObjectsAsFaults = NO;
    request.relationshipKeyPathsForPrefetching = @[@"profile", @"posts"];
    
    NSError *error;
    NSArray<User *> *users = [self.mainContext executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Failed to fetch users: %@", error);
        return @[];
    }
    
    return users;
}

- (BOOL)updateUser:(User *)user withInfo:(NSDictionary *)info {
    for (NSString *key in info.allKeys) {
        if ([user respondsToSelector:NSSelectorFromString(key)]) {
            [user setValue:info[key] forKey:key];
        }
    }
    
    NSError *error;
    BOOL success = [self.mainContext save:&error];
    
    if (!success) {
        NSLog(@"Failed to update user: %@", error);
    }
    
    return success;
}

- (BOOL)deleteUser:(User *)user {
    [self.mainContext deleteObject:user];
    
    NSError *error;
    BOOL success = [self.mainContext save:&error];
    
    if (!success) {
        NSLog(@"Failed to delete user: %@", error);
    }
    
    return success;
}

- (BOOL)batchInsertUsers:(NSArray<NSDictionary *> *)usersData {
    __block BOOL success = YES;
    
    [self performBackgroundTask:^(NSManagedObjectContext *context) {
        for (NSDictionary *userData in usersData) {
            User *user = [NSEntityDescription insertNewObjectForEntityForName:@"User"
                                                       inManagedObjectContext:context];
            user.username = userData[@"username"];
            user.email = userData[@"email"];
        }
        
        NSError *error;
        if (![context save:&error]) {
            NSLog(@"Batch insert failed: %@", error);
            success = NO;
        }
    }];
    
    return success;
}

- (BOOL)batchUpdateUsersWithPredicate:(NSPredicate *)predicate
                              updates:(NSDictionary *)updates {
    
    NSBatchUpdateRequest *batchUpdate = [[NSBatchUpdateRequest alloc] initWithEntityName:@"User"];
    batchUpdate.predicate = predicate;
    batchUpdate.propertiesToUpdate = updates;
    batchUpdate.resultType = NSUpdatedObjectsCountResultType;
    
    NSError *error;
    NSBatchUpdateResult *result = [self.mainContext executeRequest:batchUpdate error:&error];
    
    if (error) {
        NSLog(@"Batch update failed: %@", error);
        return NO;
    }
    
    NSLog(@"Updated %@ users", result.result);
    return YES;
}

- (BOOL)batchDeleteUsersWithPredicate:(NSPredicate *)predicate {
    NSBatchDeleteRequest *batchDelete = [[NSBatchDeleteRequest alloc] initWithFetchRequest:[User fetchRequest]];
    batchDelete.predicate = predicate;
    batchDelete.resultType = NSBatchDeleteResultTypeCount;
    
    NSError *error;
    NSBatchDeleteResult *result = [self.mainContext executeRequest:batchDelete error:&error];
    
    if (error) {
        NSLog(@"Batch delete failed: %@", error);
        return NO;
    }
    
    NSLog(@"Deleted %@ users", result.result);
    return YES;
}

- (BOOL)addPost:(Post *)post toUser:(User *)user {
    NSMutableSet *posts = [user.posts mutableCopy];
    if (!posts) {
        posts = [NSMutableSet set];
    }
    [posts addObject:post];
    user.posts = [posts copy];
    post.author = user;
    
    NSError *error;
    BOOL success = [self.mainContext save:&error];
    
    if (!success) {
        NSLog(@"Failed to add post to user: %@", error);
    }
    
    return success;
}

- (BOOL)removePost:(Post *)post fromUser:(User *)user {
    NSMutableSet *posts = [user.posts mutableCopy];
    [posts removeObject:post];
    user.posts = [posts copy];
    post.author = nil;
    
    NSError *error;
    BOOL success = [self.mainContext save:&error];
    
    if (!success) {
        NSLog(@"Failed to remove post from user: %@", error);
    }
    
    return success;
}

- (void)saveContext {
    if (self.mainContext.hasChanges) {
        NSError *error;
        if (![self.mainContext save:&error]) {
            NSLog(@"Failed to save main context: %@", error);
        }
    }
}

- (void)saveContextAndWait {
    [self.mainContext performBlockAndWait:^{
        [self saveContext];
    }];
}

- (void)performBackgroundTask:(void (^)(NSManagedObjectContext *context))block {
    [self.backgroundContext performBlock:^{
        block(self.backgroundContext);
        
        if (self.backgroundContext.hasChanges) {
            NSError *error;
            if ([self.backgroundContext save:&error]) {
                // 保存到主上下文
                [self.mainContext performBlock:^{
                    NSError *mainError;
                    if (![self.mainContext save:&mainError]) {
                        NSLog(@"Failed to save main context: %@", mainError);
                    }
                }];
            } else {
                NSLog(@"Failed to save background context: %@", error);
            }
        }
    }];
}

@end
```