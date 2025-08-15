---
layout: post
title: "iOS Core Data高级应用：数据持久化与性能优化深度解析"
date: 2022-11-30 10:15:00 +0800
categories: [iOS, 数据库]
tags: [iOS, Core Data, 数据持久化, 性能优化, NSManagedObject]
author: iOS技术团队
description: "深入探讨iOS Core Data框架的高级特性，包括数据模型设计、性能优化、多线程处理和最佳实践"
---

## 引言

Core Data是苹果提供的强大数据持久化框架，它不仅仅是一个简单的数据库包装器，而是一个完整的对象图管理和持久化解决方案。Core Data提供了对象关系映射（ORM）、数据验证、撤销管理、数据迁移等高级功能，是iOS应用数据层的核心技术。本文将深入解析Core Data的高级特性、性能优化技巧和最佳实践，帮助开发者构建高效、稳定的数据层架构。

## Core Data架构深度解析

### Core Data技术栈

Core Data采用分层架构设计，主要组件包括：

**1. NSManagedObjectModel（数据模型）**
定义数据结构和关系的元数据：

```objc
// 程序化创建数据模型
- (NSManagedObjectModel *)createDataModel {
    NSManagedObjectModel *model = [[NSManagedObjectModel alloc] init];
    
    // 创建User实体
    NSEntityDescription *userEntity = [[NSEntityDescription alloc] init];
    userEntity.name = @"User";
    userEntity.managedObjectClassName = @"User";
    
    // 创建属性
    NSAttributeDescription *nameAttribute = [[NSAttributeDescription alloc] init];
    nameAttribute.name = @"name";
    nameAttribute.attributeType = NSStringAttributeType;
    nameAttribute.optional = NO;
    
    NSAttributeDescription *ageAttribute = [[NSAttributeDescription alloc] init];
    ageAttribute.name = @"age";
    ageAttribute.attributeType = NSInteger32AttributeType;
    ageAttribute.optional = YES;
    ageAttribute.defaultValue = @(0);
    
    NSAttributeDescription *emailAttribute = [[NSAttributeDescription alloc] init];
    emailAttribute.name = @"email";
    emailAttribute.attributeType = NSStringAttributeType;
    emailAttribute.optional = YES;
    
    NSAttributeDescription *createdAtAttribute = [[NSAttributeDescription alloc] init];
    createdAtAttribute.name = @"createdAt";
    createdAtAttribute.attributeType = NSDateAttributeType;
    createdAtAttribute.optional = NO;
    createdAtAttribute.defaultValue = [NSDate date];
    
    userEntity.properties = @[nameAttribute, ageAttribute, emailAttribute, createdAtAttribute];
    
    // 创建Post实体
    NSEntityDescription *postEntity = [[NSEntityDescription alloc] init];
    postEntity.name = @"Post";
    postEntity.managedObjectClassName = @"Post";
    
    NSAttributeDescription *titleAttribute = [[NSAttributeDescription alloc] init];
    titleAttribute.name = @"title";
    titleAttribute.attributeType = NSStringAttributeType;
    titleAttribute.optional = NO;
    
    NSAttributeDescription *contentAttribute = [[NSAttributeDescription alloc] init];
    contentAttribute.name = @"content";
    contentAttribute.attributeType = NSStringAttributeType;
    contentAttribute.optional = YES;
    
    NSAttributeDescription *publishedAtAttribute = [[NSAttributeDescription alloc] init];
    publishedAtAttribute.name = @"publishedAt";
    publishedAtAttribute.attributeType = NSDateAttributeType;
    publishedAtAttribute.optional = YES;
    
    postEntity.properties = @[titleAttribute, contentAttribute, publishedAtAttribute];
    
    // 创建关系
    NSRelationshipDescription *userPostsRelationship = [[NSRelationshipDescription alloc] init];
    userPostsRelationship.name = @"posts";
    userPostsRelationship.destinationEntity = postEntity;
    userPostsRelationship.minCount = 0;
    userPostsRelationship.maxCount = 0; // 0表示无限制
    userPostsRelationship.deleteRule = NSCascadeDeleteRule;
    
    NSRelationshipDescription *postAuthorRelationship = [[NSRelationshipDescription alloc] init];
    postAuthorRelationship.name = @"author";
    postAuthorRelationship.destinationEntity = userEntity;
    postAuthorRelationship.minCount = 1;
    postAuthorRelationship.maxCount = 1;
    postAuthorRelationship.deleteRule = NSNullifyDeleteRule;
    
    // 设置反向关系
    userPostsRelationship.inverseRelationship = postAuthorRelationship;
    postAuthorRelationship.inverseRelationship = userPostsRelationship;
    
    userEntity.properties = [userEntity.properties arrayByAddingObject:userPostsRelationship];
    postEntity.properties = [postEntity.properties arrayByAddingObject:postAuthorRelationship];
    
    model.entities = @[userEntity, postEntity];
    
    return model;
}
```

**2. NSPersistentStoreCoordinator（持久化存储协调器）**
管理数据模型和持久化存储之间的关系：

```objc
@interface CoreDataStack : NSObject
@property (nonatomic, strong, readonly) NSManagedObjectContext *mainContext;
@property (nonatomic, strong, readonly) NSManagedObjectContext *backgroundContext;
@property (nonatomic, strong, readonly) NSPersistentStoreCoordinator *persistentStoreCoordinator;
@property (nonatomic, strong, readonly) NSManagedObjectModel *managedObjectModel;
@end

@implementation CoreDataStack

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupCoreDataStack];
    }
    return self;
}

- (void)setupCoreDataStack {
    // 1. 创建数据模型
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"DataModel" withExtension:@"momd"];
    _managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
    
    // 2. 创建持久化存储协调器
    _persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:_managedObjectModel];
    
    // 3. 添加持久化存储
    NSURL *documentsURL = [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory
                                                                   inDomains:NSUserDomainMask] lastObject];
    NSURL *storeURL = [documentsURL URLByAppendingPathComponent:@"DataModel.sqlite"];
    
    NSDictionary *options = @{
        NSMigratePersistentStoresAutomaticallyOption: @YES,
        NSInferMappingModelAutomaticallyOption: @YES,
        NSSQLitePragmasOption: @{
            @"journal_mode": @"WAL",
            @"synchronous": @"NORMAL",
            @"cache_size": @"10000",
            @"temp_store": @"MEMORY"
        }
    };
    
    NSError *error;
    if (![_persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType
                                                   configuration:nil
                                                             URL:storeURL
                                                         options:options
                                                           error:&error]) {
        NSLog(@"Failed to add persistent store: %@", error.localizedDescription);
        abort();
    }
    
    // 4. 创建主上下文（主线程）
    _mainContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
    _mainContext.persistentStoreCoordinator = _persistentStoreCoordinator;
    _mainContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    
    // 5. 创建后台上下文（后台线程）
    _backgroundContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    _backgroundContext.persistentStoreCoordinator = _persistentStoreCoordinator;
    _backgroundContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    
    // 6. 设置上下文合并通知
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(contextDidSave:)
                                                 name:NSManagedObjectContextDidSaveNotification
                                               object:nil];
}

- (void)contextDidSave:(NSNotification *)notification {
    NSManagedObjectContext *savedContext = notification.object;
    
    // 将后台上下文的更改合并到主上下文
    if (savedContext == self.backgroundContext) {
        [self.mainContext performBlock:^{
            [self.mainContext mergeChangesFromContextDidSaveNotification:notification];
        }];
    }
    // 将主上下文的更改合并到后台上下文
    else if (savedContext == self.mainContext) {
        [self.backgroundContext performBlock:^{
            [self.backgroundContext mergeChangesFromContextDidSaveNotification:notification];
        }];
    }
}

- (void)saveMainContext {
    [self.mainContext performBlockAndWait:^{
        if ([self.mainContext hasChanges]) {
            NSError *error;
            if (![self.mainContext save:&error]) {
                NSLog(@"Failed to save main context: %@", error.localizedDescription);
            }
        }
    }];
}

- (void)saveBackgroundContext {
    [self.backgroundContext performBlock:^{
        if ([self.backgroundContext hasChanges]) {
            NSError *error;
            if (![self.backgroundContext save:&error]) {
                NSLog(@"Failed to save background context: %@", error.localizedDescription);
            }
        }
    }];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

## Core Data性能优化策略

### 查询性能优化

**1. 索引优化**

```objc
@interface CoreDataPerformanceOptimizer : NSObject
@property (nonatomic, strong) NSManagedObjectContext *context;

- (instancetype)initWithContext:(NSManagedObjectContext *)context;
- (void)optimizeDataModel;
- (void)createIndexes;
- (void)analyzeFetchPerformance;
@end

@implementation CoreDataPerformanceOptimizer

- (instancetype)initWithContext:(NSManagedObjectContext *)context {
    self = [super init];
    if (self) {
        _context = context;
    }
    return self;
}

- (void)optimizeDataModel {
    // 1. 为经常查询的属性添加索引
    NSManagedObjectModel *model = self.context.persistentStoreCoordinator.managedObjectModel;
    
    for (NSEntityDescription *entity in model.entities) {
        if ([entity.name isEqualToString:@"User"]) {
            // 为email属性添加索引
            NSAttributeDescription *emailAttr = entity.attributesByName[@"email"];
            if (emailAttr) {
                emailAttr.indexed = YES;
            }
            
            // 为name属性添加索引
            NSAttributeDescription *nameAttr = entity.attributesByName[@"name"];
            if (nameAttr) {
                nameAttr.indexed = YES;
            }
        }
        
        if ([entity.name isEqualToString:@"Post"]) {
            // 为publishedAt属性添加索引
            NSAttributeDescription *publishedAtAttr = entity.attributesByName[@"publishedAt"];
            if (publishedAtAttr) {
                publishedAtAttr.indexed = YES;
            }
        }
    }
}

- (void)createIndexes {
    // 使用SQL直接创建复合索引
    NSPersistentStore *store = self.context.persistentStoreCoordinator.persistentStores.firstObject;
    
    if ([store.type isEqualToString:NSSQLiteStoreType]) {
        NSURL *storeURL = store.URL;
        
        // 创建复合索引的SQL语句
        NSArray *indexQueries = @[
            @"CREATE INDEX IF NOT EXISTS idx_user_name_email ON ZUSER (ZNAME, ZEMAIL)",
            @"CREATE INDEX IF NOT EXISTS idx_post_author_published ON ZPOST (ZAUTHOR, ZPUBLISHEDAT)",
            @"CREATE INDEX IF NOT EXISTS idx_user_age_range ON ZUSER (ZAGE) WHERE ZAGE IS NOT NULL"
        ];
        
        // 注意：直接执行SQL需要谨慎，建议在开发环境测试
        for (NSString *query in indexQueries) {
            NSLog(@"Would execute: %@", query);
            // 实际执行需要使用SQLite C API或第三方库
        }
    }
}

- (void)analyzeFetchPerformance {
    // 分析查询性能
    NSDate *startTime = [NSDate date];
    
    // 测试查询1：获取所有用户
    NSFetchRequest *allUsersRequest = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    NSError *error;
    NSArray *allUsers = [self.context executeFetchRequest:allUsersRequest error:&error];
    
    NSTimeInterval allUsersTime = [[NSDate date] timeIntervalSinceDate:startTime];
    NSLog(@"Fetch all users (%lu): %.4f seconds", (unsigned long)allUsers.count, allUsersTime);
    
    // 测试查询2：按邮箱查找用户
    startTime = [NSDate date];
    NSFetchRequest *emailRequest = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    emailRequest.predicate = [NSPredicate predicateWithFormat:@"email == %@", @"test@example.com"];
    NSArray *emailUsers = [self.context executeFetchRequest:emailRequest error:&error];
    
    NSTimeInterval emailTime = [[NSDate date] timeIntervalSinceDate:startTime];
    NSLog(@"Fetch user by email (%lu): %.4f seconds", (unsigned long)emailUsers.count, emailTime);
    
    // 测试查询3：复杂关联查询
    startTime = [NSDate date];
    NSFetchRequest *complexRequest = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    complexRequest.predicate = [NSPredicate predicateWithFormat:@"posts.@count > 5 AND age > 25"];
    complexRequest.relationshipKeyPathsForPrefetching = @[@"posts"];
    NSArray *complexUsers = [self.context executeFetchRequest:complexRequest error:&error];
    
    NSTimeInterval complexTime = [[NSDate date] timeIntervalSinceDate:startTime];
    NSLog(@"Complex fetch (%lu): %.4f seconds", (unsigned long)complexUsers.count, complexTime);
}

@end
```

**2. 内存管理优化**

```objc
@interface CoreDataMemoryManager : NSObject
@property (nonatomic, strong) NSManagedObjectContext *context;

- (instancetype)initWithContext:(NSManagedObjectContext *)context;
- (void)optimizeMemoryUsage;
- (void)implementFaultingStrategy;
- (void)manageLargeDataSets;
@end

@implementation CoreDataMemoryManager

- (instancetype)initWithContext:(NSManagedObjectContext *)context {
    self = [super init];
    if (self) {
        _context = context;
    }
    return self;
}

- (void)optimizeMemoryUsage {
    // 1. 定期清理上下文
    [self.context refreshAllObjects];
    
    // 2. 重置上下文（清除所有对象）
    [self.context reset];
    
    // 3. 设置合理的撤销管理器限制
    if (self.context.undoManager) {
        self.context.undoManager.levelsOfUndo = 10; // 限制撤销级别
    }
    
    // 4. 监控内存使用
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(memoryWarningReceived:)
                                                 name:UIApplicationDidReceiveMemoryWarningNotification
                                               object:nil];
}

- (void)memoryWarningReceived:(NSNotification *)notification {
    NSLog(@"Memory warning received, cleaning up Core Data context");
    
    // 清理未使用的对象
    [self.context refreshAllObjects];
    
    // 强制垃圾回收
    [self.context processPendingChanges];
}

- (void)implementFaultingStrategy {
    // 实现智能故障处理策略
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    // 1. 设置批量大小
    request.fetchBatchSize = 20;
    
    // 2. 只获取需要的属性
    request.propertiesToFetch = @[@"name", @"email"];
    request.resultType = NSDictionaryResultType;
    
    // 3. 使用故障对象
    request.returnsObjectsAsFaults = YES;
    
    NSError *error;
    NSArray *results = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return;
    }
    
    // 处理结果时才触发故障
    for (NSDictionary *userDict in results) {
        NSLog(@"User: %@ - %@", userDict[@"name"], userDict[@"email"]);
    }
}

- (void)manageLargeDataSets {
    // 处理大数据集的策略
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Post"];
    
    // 1. 分页处理
    NSInteger pageSize = 100;
    NSInteger offset = 0;
    
    do {
        request.fetchLimit = pageSize;
        request.fetchOffset = offset;
        
        NSError *error;
        NSArray *posts = [self.context executeFetchRequest:request error:&error];
        
        if (error) {
            NSLog(@"Fetch error: %@", error.localizedDescription);
            break;
        }
        
        // 处理当前页的数据
        [self processPostsBatch:posts];
        
        // 清理当前批次的对象
        for (NSManagedObject *post in posts) {
            [self.context refreshObject:post mergeChanges:NO];
        }
        
        offset += pageSize;
        
        // 如果返回的数据少于页面大小，说明已经到最后一页
        if (posts.count < pageSize) {
            break;
        }
        
    } while (YES);
}

- (void)processPostsBatch:(NSArray<Post *> *)posts {
    // 处理帖子批次的业务逻辑
    for (Post *post in posts) {
        // 执行必要的业务逻辑
        NSLog(@"Processing post: %@", post.title);
    }
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

### 多线程处理最佳实践

**1. 多上下文架构**

```objc
@interface MultiContextCoreDataManager : NSObject
@property (nonatomic, strong, readonly) NSManagedObjectContext *mainContext;
@property (nonatomic, strong, readonly) NSManagedObjectContext *backgroundContext;
@property (nonatomic, strong, readonly) NSManagedObjectContext *writerContext;
@property (nonatomic, strong, readonly) NSPersistentStoreCoordinator *coordinator;

- (void)performBackgroundTask:(void(^)(NSManagedObjectContext *context))task;
- (void)performBackgroundTaskAndSave:(void(^)(NSManagedObjectContext *context))task
                          completion:(void(^)(BOOL success, NSError *error))completion;
@end

@implementation MultiContextCoreDataManager

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setupMultiContextArchitecture];
    }
    return self;
}

- (void)setupMultiContextArchitecture {
    // 1. 创建持久化存储协调器
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"DataModel" withExtension:@"momd"];
    NSManagedObjectModel *model = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
    _coordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:model];
    
    // 添加持久化存储
    NSURL *documentsURL = [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory
                                                                   inDomains:NSUserDomainMask] lastObject];
    NSURL *storeURL = [documentsURL URLByAppendingPathComponent:@"DataModel.sqlite"];
    
    NSDictionary *options = @{
        NSMigratePersistentStoresAutomaticallyOption: @YES,
        NSInferMappingModelAutomaticallyOption: @YES,
        NSSQLitePragmasOption: @{
            @"journal_mode": @"WAL",
            @"synchronous": @"NORMAL"
        }
    };
    
    NSError *error;
    if (![_coordinator addPersistentStoreWithType:NSSQLiteStoreType
                                    configuration:nil
                                              URL:storeURL
                                          options:options
                                            error:&error]) {
        NSLog(@"Failed to add persistent store: %@", error.localizedDescription);
        abort();
    }
    
    // 2. 创建写入上下文（私有队列）
    _writerContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    _writerContext.persistentStoreCoordinator = _coordinator;
    _writerContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    
    // 3. 创建主上下文（主队列）
    _mainContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
    _mainContext.parentContext = _writerContext;
    _mainContext.mergePolicy = NSMergeByPropertyStoreTrumpMergePolicy;
    
    // 4. 创建后台上下文（私有队列）
    _backgroundContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    _backgroundContext.parentContext = _mainContext;
    _backgroundContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
    
    // 5. 设置自动合并策略
    [self setupContextMerging];
}

- (void)setupContextMerging {
    // 监听上下文保存通知
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(contextDidSave:)
                                                 name:NSManagedObjectContextDidSaveNotification
                                               object:nil];
}

- (void)contextDidSave:(NSNotification *)notification {
    NSManagedObjectContext *savedContext = notification.object;
    
    // 确保在正确的队列中执行合并
    if (savedContext == self.backgroundContext) {
        // 后台上下文保存后，合并到主上下文
        [self.mainContext performBlock:^{
            [self.mainContext mergeChangesFromContextDidSaveNotification:notification];
        }];
    } else if (savedContext == self.mainContext) {
        // 主上下文保存后，合并到写入上下文
        [self.writerContext performBlock:^{
            [self.writerContext mergeChangesFromContextDidSaveNotification:notification];
        }];
    }
}

- (void)performBackgroundTask:(void(^)(NSManagedObjectContext *context))task {
    [self.backgroundContext performBlock:^{
        task(self.backgroundContext);
    }];
}

- (void)performBackgroundTaskAndSave:(void(^)(NSManagedObjectContext *context))task
                          completion:(void(^)(BOOL success, NSError *error))completion {
    
    [self.backgroundContext performBlock:^{
        // 执行任务
        task(self.backgroundContext);
        
        // 保存后台上下文
        NSError *backgroundError;
        BOOL backgroundSuccess = [self.backgroundContext save:&backgroundError];
        
        if (!backgroundSuccess) {
            dispatch_async(dispatch_get_main_queue(), ^{
                if (completion) {
                    completion(NO, backgroundError);
                }
            });
            return;
        }
        
        // 保存主上下文
        [self.mainContext performBlock:^{
            NSError *mainError;
            BOOL mainSuccess = [self.mainContext save:&mainError];
            
            if (!mainSuccess) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) {
                        completion(NO, mainError);
                    }
                });
                return;
            }
            
            // 保存写入上下文
            [self.writerContext performBlock:^{
                NSError *writerError;
                BOOL writerSuccess = [self.writerContext save:&writerError];
                
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (completion) {
                        completion(writerSuccess, writerError);
                    }
                });
            }];
        }];
    }];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

**2. 线程安全操作示例**

```objc
@interface ThreadSafeCoreDataOperations : NSObject
@property (nonatomic, strong) MultiContextCoreDataManager *coreDataManager;

- (instancetype)initWithCoreDataManager:(MultiContextCoreDataManager *)manager;
- (void)importUsersFromJSON:(NSArray *)jsonArray completion:(void(^)(BOOL success, NSError *error))completion;
- (void)bulkUpdateUserAges:(NSDictionary *)ageUpdates completion:(void(^)(BOOL success, NSError *error))completion;
- (void)exportUsersToJSON:(void(^)(NSArray *jsonArray, NSError *error))completion;
@end

@implementation ThreadSafeCoreDataOperations

- (instancetype)initWithCoreDataManager:(MultiContextCoreDataManager *)manager {
    self = [super init];
    if (self) {
        _coreDataManager = manager;
    }
    return self;
}

- (void)importUsersFromJSON:(NSArray *)jsonArray completion:(void(^)(BOOL success, NSError *error))completion {
    [self.coreDataManager performBackgroundTaskAndSave:^(NSManagedObjectContext *context) {
        
        for (NSDictionary *userDict in jsonArray) {
            // 检查用户是否已存在
            NSString *email = userDict[@"email"];
            if (!email) continue;
            
            NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
            request.predicate = [NSPredicate predicateWithFormat:@"email == %@", email];
            request.fetchLimit = 1;
            
            NSError *fetchError;
            NSArray *existingUsers = [context executeFetchRequest:request error:&fetchError];
            
            if (fetchError) {
                NSLog(@"Fetch error: %@", fetchError.localizedDescription);
                continue;
            }
            
            User *user;
            if (existingUsers.count > 0) {
                // 更新现有用户
                user = existingUsers.firstObject;
            } else {
                // 创建新用户
                user = [NSEntityDescription insertNewObjectForEntityForName:@"User" inManagedObjectContext:context];
                user.email = email;
            }
            
            // 更新用户信息
            user.name = userDict[@"name"];
            user.age = userDict[@"age"];
            
            // 验证数据
            NSError *validationError;
            if (![DataValidator validateUser:user error:&validationError]) {
                NSLog(@"Validation error for user %@: %@", user.email, validationError.localizedDescription);
                [context deleteObject:user];
            }
        }
        
    } completion:completion];
}

- (void)bulkUpdateUserAges:(NSDictionary *)ageUpdates completion:(void(^)(BOOL success, NSError *error))completion {
    [self.coreDataManager performBackgroundTaskAndSave:^(NSManagedObjectContext *context) {
        
        for (NSString *email in ageUpdates.allKeys) {
            NSNumber *newAge = ageUpdates[email];
            
            NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
            request.predicate = [NSPredicate predicateWithFormat:@"email == %@", email];
            request.fetchLimit = 1;
            
            NSError *fetchError;
            NSArray *users = [context executeFetchRequest:request error:&fetchError];
            
            if (fetchError) {
                NSLog(@"Fetch error: %@", fetchError.localizedDescription);
                continue;
            }
            
            if (users.count > 0) {
                User *user = users.firstObject;
                user.age = newAge;
            }
        }
        
    } completion:completion];
}

- (void)exportUsersToJSON:(void(^)(NSArray *jsonArray, NSError *error))completion {
    [self.coreDataManager performBackgroundTask:^(NSManagedObjectContext *context) {
        
        NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
        request.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES]];
        
        NSError *fetchError;
        NSArray *users = [context executeFetchRequest:request error:&fetchError];
        
        if (fetchError) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion(nil, fetchError);
            });
            return;
        }
        
        NSMutableArray *jsonArray = [[NSMutableArray alloc] init];
        
        for (User *user in users) {
            NSDictionary *userDict = @{
                @"name": user.name ?: @"",
                @"email": user.email ?: @"",
                @"age": user.age ?: @(0),
                @"postCount": @(user.posts.count)
            };
            [jsonArray addObject:userDict];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            completion([jsonArray copy], nil);
        });
    }];
}

@end
```

## 数据迁移与版本管理

### 轻量级迁移

```objc
@interface CoreDataMigrationManager : NSObject
@property (nonatomic, strong) NSPersistentStoreCoordinator *coordinator;

- (instancetype)initWithCoordinator:(NSPersistentStoreCoordinator *)coordinator;
- (BOOL)performLightweightMigration:(NSError **)error;
- (BOOL)requiresMigration:(NSURL *)storeURL error:(NSError **)error;
- (void)backupStoreBeforeMigration:(NSURL *)storeURL;
@end

@implementation CoreDataMigrationManager

- (instancetype)initWithCoordinator:(NSPersistentStoreCoordinator *)coordinator {
    self = [super init];
    if (self) {
        _coordinator = coordinator;
    }
    return self;
}

- (BOOL)performLightweightMigration:(NSError **)error {
    NSURL *documentsURL = [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory
                                                                   inDomains:NSUserDomainMask] lastObject];
    NSURL *storeURL = [documentsURL URLByAppendingPathComponent:@"DataModel.sqlite"];
    
    // 检查是否需要迁移
    if (![self requiresMigration:storeURL error:error]) {
        return YES; // 不需要迁移
    }
    
    NSLog(@"Starting lightweight migration...");
    
    // 备份数据库
    [self backupStoreBeforeMigration:storeURL];
    
    // 配置迁移选项
    NSDictionary *options = @{
        NSMigratePersistentStoresAutomaticallyOption: @YES,
        NSInferMappingModelAutomaticallyOption: @YES,
        NSSQLitePragmasOption: @{
            @"journal_mode": @"WAL",
            @"synchronous": @"NORMAL"
        }
    };
    
    // 执行迁移
    NSPersistentStore *store = [self.coordinator addPersistentStoreWithType:NSSQLiteStoreType
                                                              configuration:nil
                                                                        URL:storeURL
                                                                    options:options
                                                                      error:error];
    
    if (store) {
        NSLog(@"Lightweight migration completed successfully");
        return YES;
    } else {
        NSLog(@"Lightweight migration failed: %@", (*error).localizedDescription);
        return NO;
    }
}

- (BOOL)requiresMigration:(NSURL *)storeURL error:(NSError **)error {
    if (![[NSFileManager defaultManager] fileExistsAtPath:storeURL.path]) {
        return NO; // 新数据库，不需要迁移
    }
    
    // 获取存储的元数据
    NSDictionary *metadata = [NSPersistentStoreCoordinator metadataForPersistentStoreOfType:NSSQLiteStoreType
                                                                                        URL:storeURL
                                                                                    options:nil
                                                                                      error:error];
    
    if (!metadata) {
        return NO;
    }
    
    // 检查模型兼容性
    NSManagedObjectModel *currentModel = self.coordinator.managedObjectModel;
    BOOL compatible = [currentModel isConfiguration:nil compatibleWithStoreMetadata:metadata];
    
    return !compatible; // 不兼容则需要迁移
}

- (void)backupStoreBeforeMigration:(NSURL *)storeURL {
    NSString *backupPath = [storeURL.path stringByAppendingString:@".backup"];
    NSURL *backupURL = [NSURL fileURLWithPath:backupPath];
    
    NSError *error;
    [[NSFileManager defaultManager] copyItemAtURL:storeURL toURL:backupURL error:&error];
    
    if (error) {
        NSLog(@"Failed to backup store: %@", error.localizedDescription);
    } else {
        NSLog(@"Store backed up to: %@", backupPath);
    }
}

@end
```

### 重量级迁移

```objc
@interface HeavyweightMigrationManager : NSObject
- (BOOL)performHeavyweightMigrationFromVersion:(NSString *)sourceVersion
                                     toVersion:(NSString *)targetVersion
                                      storeURL:(NSURL *)storeURL
                                         error:(NSError **)error;
- (NSMappingModel *)createMappingModelFromVersion:(NSString *)sourceVersion
                                        toVersion:(NSString *)targetVersion;
@end

@implementation HeavyweightMigrationManager

- (BOOL)performHeavyweightMigrationFromVersion:(NSString *)sourceVersion
                                     toVersion:(NSString *)targetVersion
                                      storeURL:(NSURL *)storeURL
                                         error:(NSError **)error {
    
    NSLog(@"Starting heavyweight migration from %@ to %@", sourceVersion, targetVersion);
    
    // 1. 获取源模型和目标模型
    NSManagedObjectModel *sourceModel = [self modelForVersion:sourceVersion];
    NSManagedObjectModel *targetModel = [self modelForVersion:targetVersion];
    
    if (!sourceModel || !targetModel) {
        if (error) {
            *error = [NSError errorWithDomain:@"MigrationError"
                                         code:1001
                                     userInfo:@{NSLocalizedDescriptionKey: @"Could not load models"}];
        }
        return NO;
    }
    
    // 2. 创建映射模型
    NSMappingModel *mappingModel = [self createMappingModelFromVersion:sourceVersion toVersion:targetVersion];
    if (!mappingModel) {
        mappingModel = [NSMappingModel inferredMappingModelForSourceModel:sourceModel
                                                         destinationModel:targetModel
                                                                    error:error];
    }
    
    if (!mappingModel) {
        return NO;
    }
    
    // 3. 创建迁移管理器
    NSMigrationManager *migrationManager = [[NSMigrationManager alloc] initWithSourceModel:sourceModel
                                                                           destinationModel:targetModel];
    
    // 4. 设置临时存储URL
    NSString *tempPath = [storeURL.path stringByAppendingString:@".temp"];
    NSURL *tempURL = [NSURL fileURLWithPath:tempPath];
    
    // 5. 执行迁移
    BOOL success = [migrationManager migrateStoreFromURL:storeURL
                                                     type:NSSQLiteStoreType
                                                  options:nil
                                         withMappingModel:mappingModel
                                         toDestinationURL:tempURL
                                          destinationType:NSSQLiteStoreType
                                       destinationOptions:nil
                                                    error:error];
    
    if (success) {
        // 6. 替换原文件
        NSString *backupPath = [storeURL.path stringByAppendingString:@".backup"];
        
        // 备份原文件
        [[NSFileManager defaultManager] moveItemAtPath:storeURL.path toPath:backupPath error:nil];
        
        // 移动新文件
        [[NSFileManager defaultManager] moveItemAtPath:tempPath toPath:storeURL.path error:nil];
        
        NSLog(@"Heavyweight migration completed successfully");
    } else {
        // 清理临时文件
        [[NSFileManager defaultManager] removeItemAtURL:tempURL error:nil];
        NSLog(@"Heavyweight migration failed: %@", (*error).localizedDescription);
    }
    
    return success;
}

- (NSManagedObjectModel *)modelForVersion:(NSString *)version {
    NSBundle *bundle = [NSBundle mainBundle];
    NSString *modelPath = [bundle pathForResource:[NSString stringWithFormat:@"DataModel %@", version]
                                           ofType:@"mom"
                                      inDirectory:@"DataModel.momd"];
    
    if (!modelPath) {
        return nil;
    }
    
    NSURL *modelURL = [NSURL fileURLWithPath:modelPath];
    return [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
}

- (NSMappingModel *)createMappingModelFromVersion:(NSString *)sourceVersion
                                        toVersion:(NSString *)targetVersion {
    
    NSBundle *bundle = [NSBundle mainBundle];
    NSString *mappingPath = [bundle pathForResource:[NSString stringWithFormat:@"Migration_%@_to_%@", sourceVersion, targetVersion]
                                             ofType:@"xcmappingmodel"];
    
    if (!mappingPath) {
        return nil;
    }
    
    NSURL *mappingURL = [NSURL fileURLWithPath:mappingPath];
    return [[NSMappingModel alloc] initWithContentsOfURL:mappingURL];
}

@end
```

## Core Data最佳实践总结

### 架构设计原则

1. **单一职责原则**：每个NSManagedObject子类只负责一个实体的业务逻辑
2. **依赖注入**：通过依赖注入传递NSManagedObjectContext，便于测试和维护
3. **分层架构**：将数据访问层、业务逻辑层和表现层分离
4. **错误处理**：完善的错误处理机制，包括验证错误、网络错误和数据库错误

### 性能优化要点

1. **合理使用索引**：为经常查询的属性添加索引
2. **批量操作**：使用NSBatchUpdateRequest和NSBatchDeleteRequest处理大量数据
3. **分页查询**：使用fetchLimit和fetchOffset实现分页
4. **预加载关联**：使用relationshipKeyPathsForPrefetching预加载关联对象
5. **内存管理**：定期清理上下文，使用故障对象减少内存占用

### 多线程安全

1. **上下文隔离**：每个线程使用独立的NSManagedObjectContext
2. **父子上下文**：使用父子上下文架构实现数据同步
3. **队列约束**：严格在指定队列中操作对应的上下文
4. **对象传递**：在线程间传递NSManagedObjectID而不是NSManagedObject

### 数据迁移策略

1. **版本控制**：为每个数据模型版本创建独立的.xcdatamodel文件
2. **轻量级优先**：优先使用轻量级迁移，只在必要时使用重量级迁移
3. **备份机制**：迁移前备份数据库文件
4. **测试验证**：充分测试迁移过程，确保数据完整性

### 调试和监控

1. **SQL日志**：启用Core Data SQL调试日志
2. **性能监控**：监控查询执行时间和内存使用
3. **错误日志**：记录详细的错误信息和堆栈跟踪
4. **数据验证**：实现完善的数据验证机制

## 总结

Core Data作为iOS平台的核心数据持久化框架，提供了强大的对象关系映射、数据验证、撤销管理等功能。通过合理的架构设计、性能优化、多线程处理和数据迁移策略，可以构建高效、稳定的数据层。

掌握Core Data的高级特性和最佳实践，对于开发大型iOS应用至关重要。随着应用复杂度的增加，良好的数据层设计将成为应用性能和用户体验的关键因素。开发者应该根据具体需求选择合适的技术方案，并持续优化数据层的性能和稳定性。

### NSManagedObject高级特性

**1. 自定义NSManagedObject子类**

```objc
// User.h
@interface User : NSManagedObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSNumber *age;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, strong) NSSet<Post *> *posts;

// 计算属性
@property (nonatomic, readonly) NSString *displayName;
@property (nonatomic, readonly) NSInteger postCount;
@property (nonatomic, readonly) BOOL isAdult;

// 业务方法
- (void)addPost:(Post *)post;
- (void)removePost:(Post *)post;
- (NSArray<Post *> *)recentPosts:(NSInteger)count;
- (void)updateLastActiveDate;
@end

// User.m
@implementation User

@dynamic name, age, email, createdAt, posts;

#pragma mark - 计算属性

- (NSString *)displayName {
    if (self.name && self.name.length > 0) {
        return self.name;
    }
    return self.email ?: @"Unknown User";
}

- (NSInteger)postCount {
    return self.posts.count;
}

- (BOOL)isAdult {
    return self.age.integerValue >= 18;
}

#pragma mark - 生命周期方法

- (void)awakeFromInsert {
    [super awakeFromInsert];
    
    // 设置默认值
    if (!self.createdAt) {
        self.createdAt = [NSDate date];
    }
    
    if (!self.age) {
        self.age = @(0);
    }
}

- (void)prepareForDeletion {
    [super prepareForDeletion];
    
    // 清理相关资源
    NSLog(@"User %@ is being deleted", self.displayName);
}

- (BOOL)validateName:(id *)value error:(NSError **)error {
    NSString *name = *value;
    
    if (!name || name.length == 0) {
        if (error) {
            *error = [NSError errorWithDomain:@"UserValidationError"
                                         code:1001
                                     userInfo:@{NSLocalizedDescriptionKey: @"Name cannot be empty"}];
        }
        return NO;
    }
    
    if (name.length > 50) {
        if (error) {
            *error = [NSError errorWithDomain:@"UserValidationError"
                                         code:1002
                                     userInfo:@{NSLocalizedDescriptionKey: @"Name cannot exceed 50 characters"}];
        }
        return NO;
    }
    
    return YES;
}

- (BOOL)validateEmail:(id *)value error:(NSError **)error {
    NSString *email = *value;
    
    if (email && email.length > 0) {
        NSString *emailRegex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}";
        NSPredicate *emailPredicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", emailRegex];
        
        if (![emailPredicate evaluateWithObject:email]) {
            if (error) {
                *error = [NSError errorWithDomain:@"UserValidationError"
                                             code:1003
                                         userInfo:@{NSLocalizedDescriptionKey: @"Invalid email format"}];
            }
            return NO;
        }
    }
    
    return YES;
}

#pragma mark - 业务方法

- (void)addPost:(Post *)post {
    NSMutableSet *mutablePosts = [self mutableSetValueForKey:@"posts"];
    [mutablePosts addObject:post];
    post.author = self;
}

- (void)removePost:(Post *)post {
    NSMutableSet *mutablePosts = [self mutableSetValueForKey:@"posts"];
    [mutablePosts removeObject:post];
    post.author = nil;
}

- (NSArray<Post *> *)recentPosts:(NSInteger)count {
    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"publishedAt" ascending:NO];
    NSArray *sortedPosts = [self.posts sortedArrayUsingDescriptors:@[sortDescriptor]];
    
    if (sortedPosts.count <= count) {
        return sortedPosts;
    }
    
    return [sortedPosts subarrayWithRange:NSMakeRange(0, count)];
}

- (void)updateLastActiveDate {
    // 这里可以添加一个lastActiveDate属性
    // self.lastActiveDate = [NSDate date];
}

@end
```

**2. 数据验证和约束**

```objc
@interface DataValidator : NSObject
+ (BOOL)validateUser:(User *)user error:(NSError **)error;
+ (BOOL)validatePost:(Post *)post error:(NSError **)error;
+ (BOOL)validateUniqueEmail:(NSString *)email inContext:(NSManagedObjectContext *)context excludeUser:(User *)excludeUser error:(NSError **)error;
@end

@implementation DataValidator

+ (BOOL)validateUser:(User *)user error:(NSError **)error {
    // 验证必填字段
    if (!user.name || user.name.length == 0) {
        if (error) {
            *error = [NSError errorWithDomain:@"ValidationError"
                                         code:1001
                                     userInfo:@{NSLocalizedDescriptionKey: @"User name is required"}];
        }
        return NO;
    }
    
    // 验证邮箱唯一性
    if (user.email && user.email.length > 0) {
        if (![self validateUniqueEmail:user.email inContext:user.managedObjectContext excludeUser:user error:error]) {
            return NO;
        }
    }
    
    // 验证年龄范围
    if (user.age && (user.age.integerValue < 0 || user.age.integerValue > 150)) {
        if (error) {
            *error = [NSError errorWithDomain:@"ValidationError"
                                         code:1002
                                     userInfo:@{NSLocalizedDescriptionKey: @"Age must be between 0 and 150"}];
        }
        return NO;
    }
    
    return YES;
}

+ (BOOL)validatePost:(Post *)post error:(NSError **)error {
    // 验证标题
    if (!post.title || post.title.length == 0) {
        if (error) {
            *error = [NSError errorWithDomain:@"ValidationError"
                                         code:2001
                                     userInfo:@{NSLocalizedDescriptionKey: @"Post title is required"}];
        }
        return NO;
    }
    
    // 验证作者
    if (!post.author) {
        if (error) {
            *error = [NSError errorWithDomain:@"ValidationError"
                                         code:2002
                                     userInfo:@{NSLocalizedDescriptionKey: @"Post must have an author"}];
        }
        return NO;
    }
    
    return YES;
}

+ (BOOL)validateUniqueEmail:(NSString *)email
                  inContext:(NSManagedObjectContext *)context
                excludeUser:(User *)excludeUser
                      error:(NSError **)error {
    
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    request.predicate = [NSPredicate predicateWithFormat:@"email == %@", email];
    
    NSError *fetchError;
    NSArray *existingUsers = [context executeFetchRequest:request error:&fetchError];
    
    if (fetchError) {
        if (error) {
            *error = fetchError;
        }
        return NO;
    }
    
    // 排除当前用户（用于更新操作）
    if (excludeUser) {
        NSMutableArray *filteredUsers = [existingUsers mutableCopy];
        [filteredUsers removeObject:excludeUser];
        existingUsers = filteredUsers;
    }
    
    if (existingUsers.count > 0) {
        if (error) {
            *error = [NSError errorWithDomain:@"ValidationError"
                                         code:1003
                                     userInfo:@{NSLocalizedDescriptionKey: @"Email already exists"}];
        }
        return NO;
    }
    
    return YES;
}

@end
```

## 高级查询与性能优化

### NSFetchRequest高级用法

```objc
@interface CoreDataQueryManager : NSObject
@property (nonatomic, strong) NSManagedObjectContext *context;

- (instancetype)initWithContext:(NSManagedObjectContext *)context;

// 基础查询
- (NSArray<User *> *)fetchAllUsers;
- (User *)fetchUserByEmail:(NSString *)email;
- (NSArray<User *> *)fetchUsersWithAgeRange:(NSRange)ageRange;

// 高级查询
- (NSArray<User *> *)fetchActiveUsersWithPosts;
- (NSArray<User *> *)fetchUsersOrderedByPostCount;
- (NSArray<Post *> *)fetchRecentPosts:(NSInteger)count;
- (NSArray<User *> *)fetchUsersWithNameContaining:(NSString *)searchText;

// 聚合查询
- (NSInteger)countUsers;
- (NSInteger)countPostsByUser:(User *)user;
- (NSNumber *)averageUserAge;
- (NSArray *)fetchUserPostCounts;

// 批量操作
- (void)batchUpdateUserAges:(NSDictionary *)ageUpdates;
- (void)batchDeleteInactiveUsers;
@end

@implementation CoreDataQueryManager

- (instancetype)initWithContext:(NSManagedObjectContext *)context {
    self = [super init];
    if (self) {
        _context = context;
    }
    return self;
}

#pragma mark - 基础查询

- (NSArray<User *> *)fetchAllUsers {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    // 设置排序
    NSSortDescriptor *nameSort = [NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES];
    request.sortDescriptors = @[nameSort];
    
    NSError *error;
    NSArray *users = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return @[];
    }
    
    return users;
}

- (User *)fetchUserByEmail:(NSString *)email {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    request.predicate = [NSPredicate predicateWithFormat:@"email == %@", email];
    request.fetchLimit = 1;
    
    NSError *error;
    NSArray *users = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return nil;
    }
    
    return users.firstObject;
}

- (NSArray<User *> *)fetchUsersWithAgeRange:(NSRange)ageRange {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    request.predicate = [NSPredicate predicateWithFormat:@"age >= %ld AND age <= %ld",
                        ageRange.location, ageRange.location + ageRange.length];
    
    NSSortDescriptor *ageSort = [NSSortDescriptor sortDescriptorWithKey:@"age" ascending:YES];
    request.sortDescriptors = @[ageSort];
    
    NSError *error;
    NSArray *users = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return @[];
    }
    
    return users;
}

#pragma mark - 高级查询

- (NSArray<User *> *)fetchActiveUsersWithPosts {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    // 使用子查询查找有帖子的用户
    request.predicate = [NSPredicate predicateWithFormat:@"SUBQUERY(posts, $post, $post != nil).@count > 0"];
    
    // 预加载关联对象
    request.relationshipKeyPathsForPrefetching = @[@"posts"];
    
    NSSortDescriptor *nameSort = [NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES];
    request.sortDescriptors = @[nameSort];
    
    NSError *error;
    NSArray *users = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return @[];
    }
    
    return users;
}

- (NSArray<User *> *)fetchUsersOrderedByPostCount {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    // 使用表达式描述符按帖子数量排序
    NSExpression *countExpression = [NSExpression expressionWithFormat:@"posts.@count"];
    NSExpressionDescription *countDesc = [[NSExpressionDescription alloc] init];
    countDesc.name = @"postCount";
    countDesc.expression = countExpression;
    countDesc.expressionResultType = NSInteger32AttributeType;
    
    request.propertiesToFetch = @[@"name", @"email", countDesc];
    request.resultType = NSDictionaryResultType;
    
    NSSortDescriptor *countSort = [NSSortDescriptor sortDescriptorWithKey:@"postCount" ascending:NO];
    request.sortDescriptors = @[countSort];
    
    NSError *error;
    NSArray *results = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return @[];
    }
    
    // 将字典结果转换为User对象
    NSMutableArray *users = [[NSMutableArray alloc] init];
    for (NSDictionary *result in results) {
        NSFetchRequest *userRequest = [NSFetchRequest fetchRequestWithEntityName:@"User"];
        userRequest.predicate = [NSPredicate predicateWithFormat:@"name == %@ AND email == %@",
                                result[@"name"], result[@"email"]];
        userRequest.fetchLimit = 1;
        
        NSArray *userResults = [self.context executeFetchRequest:userRequest error:nil];
        if (userResults.count > 0) {
            [users addObject:userResults.firstObject];
        }
    }
    
    return users;
}

- (NSArray<Post *> *)fetchRecentPosts:(NSInteger)count {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Post"];
    
    // 只获取已发布的帖子
    request.predicate = [NSPredicate predicateWithFormat:@"publishedAt != nil"];
    
    // 按发布时间倒序排列
    NSSortDescriptor *publishedSort = [NSSortDescriptor sortDescriptorWithKey:@"publishedAt" ascending:NO];
    request.sortDescriptors = @[publishedSort];
    
    // 限制结果数量
    request.fetchLimit = count;
    
    // 预加载作者信息
    request.relationshipKeyPathsForPrefetching = @[@"author"];
    
    NSError *error;
    NSArray *posts = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return @[];
    }
    
    return posts;
}

- (NSArray<User *> *)fetchUsersWithNameContaining:(NSString *)searchText {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    // 使用CONTAINS进行模糊搜索，[cd]表示不区分大小写和变音符号
    request.predicate = [NSPredicate predicateWithFormat:@"name CONTAINS[cd] %@", searchText];
    
    NSSortDescriptor *nameSort = [NSSortDescriptor sortDescriptorWithKey:@"name" ascending:YES];
    request.sortDescriptors = @[nameSort];
    
    NSError *error;
    NSArray *users = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return @[];
    }
    
    return users;
}

#pragma mark - 聚合查询

- (NSInteger)countUsers {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    NSError *error;
    NSUInteger count = [self.context countForFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Count error: %@", error.localizedDescription);
        return 0;
    }
    
    return count;
}

- (NSInteger)countPostsByUser:(User *)user {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Post"];
    request.predicate = [NSPredicate predicateWithFormat:@"author == %@", user];
    
    NSError *error;
    NSUInteger count = [self.context countForFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Count error: %@", error.localizedDescription);
        return 0;
    }
    
    return count;
}

- (NSNumber *)averageUserAge {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    // 创建平均值表达式
    NSExpression *ageExpression = [NSExpression expressionForKeyPath:@"age"];
    NSExpression *averageExpression = [NSExpression expressionForFunction:@"average:"
                                                                 arguments:@[ageExpression]];
    
    NSExpressionDescription *averageDesc = [[NSExpressionDescription alloc] init];
    averageDesc.name = @"averageAge";
    averageDesc.expression = averageExpression;
    averageDesc.expressionResultType = NSDecimalAttributeType;
    
    request.propertiesToFetch = @[averageDesc];
    request.resultType = NSDictionaryResultType;
    
    NSError *error;
    NSArray *results = [self.context executeFetchRequest:request error:&error];
    
    if (error || results.count == 0) {
        NSLog(@"Average calculation error: %@", error.localizedDescription);
        return @(0);
    }
    
    return results.firstObject[@"averageAge"];
}

- (NSArray *)fetchUserPostCounts {
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
    
    // 创建计数表达式
    NSExpression *countExpression = [NSExpression expressionWithFormat:@"posts.@count"];
    NSExpressionDescription *countDesc = [[NSExpressionDescription alloc] init];
    countDesc.name = @"postCount";
    countDesc.expression = countExpression;
    countDesc.expressionResultType = NSInteger32AttributeType;
    
    request.propertiesToFetch = @[@"name", @"email", countDesc];
    request.resultType = NSDictionaryResultType;
    
    NSSortDescriptor *countSort = [NSSortDescriptor sortDescriptorWithKey:@"postCount" ascending:NO];
    request.sortDescriptors = @[countSort];
    
    NSError *error;
    NSArray *results = [self.context executeFetchRequest:request error:&error];
    
    if (error) {
        NSLog(@"Fetch error: %@", error.localizedDescription);
        return @[];
    }
    
    return results;
}

#pragma mark - 批量操作

- (void)batchUpdateUserAges:(NSDictionary *)ageUpdates {
    // 使用NSBatchUpdateRequest进行批量更新
    for (NSString *email in ageUpdates.allKeys) {
        NSNumber *newAge = ageUpdates[email];
        
        NSBatchUpdateRequest *batchUpdate = [NSBatchUpdateRequest batchUpdateRequestWithEntityName:@"User"];
        batchUpdate.predicate = [NSPredicate predicateWithFormat:@"email == %@", email];
        batchUpdate.propertiesToUpdate = @{@"age": newAge};
        batchUpdate.resultType = NSUpdatedObjectsCountResultType;
        
        NSError *error;
        NSBatchUpdateResult *result = [self.context executeRequest:batchUpdate error:&error];
        
        if (error) {
            NSLog(@"Batch update error: %@", error.localizedDescription);
        } else {
            NSLog(@"Updated %@ users", result.result);
        }
    }
}

- (void)batchDeleteInactiveUsers {
    // 删除30天内没有发帖的用户
    NSDate *thirtyDaysAgo = [NSDate dateWithTimeIntervalSinceNow:-30 * 24 * 60 * 60];
    
    NSBatchDeleteRequest *batchDelete = [NSBatchDeleteRequest batchDeleteRequestWithEntityName:@"User"];
    batchDelete.predicate = [NSPredicate predicateWithFormat:
                            @"SUBQUERY(posts, $post, $post.publishedAt > %@).@count == 0", thirtyDaysAgo];
    batchDelete.resultType = NSBatchDeleteResultTypeCount;
    
    NSError *error;
    NSBatchDeleteResult *result = [self.context executeRequest:batchDelete error:&error];
    
    if (error) {
        NSLog(@"Batch delete error: %@", error.localizedDescription);
    } else {
        NSLog(@"Deleted %@ inactive users", result.result);
        
        // 刷新上下文以反映删除的对象
        [self.context refreshAllObjects];
    }
}

@end
```