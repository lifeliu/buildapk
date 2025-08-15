---
layout: post
title: "iOS Core Data高级应用：数据持久化与性能优化深度解析"
date: 2022-11-30 10:15:00 +0800
categories: [iOS, 数据库]
tags: [iOS, Core Data, 数据持久化, 性能优化, NSManagedObject]
author: iOS技术团队
description: "深入探讨iOS Core Data框架的高级特性，包括数据模型设计、性能优化、多线程处理和最佳实践"
---

# iOS Core Data高级应用：数据持久化与性能优化深度解析

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