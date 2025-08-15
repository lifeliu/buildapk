---
layout: post
title: "iOS应用架构设计模式深度解析：MVC、MVP、MVVM与VIPER实战指南"
date: 2023-08-14
categories: ios
tags: [iOS, MVC, MVP, MVVM, VIPER, Architecture, Swift, Objective-C]
author: "iOS Developer"
description: "深入探讨iOS应用架构设计模式，包括MVC、MVP、MVVM和VIPER的原理、实现和最佳实践，帮助开发者构建可维护、可测试的iOS应用。"
---

在iOS应用开发中，选择合适的架构设计模式对于构建可维护、可测试和可扩展的应用至关重要。本文将深入探讨iOS开发中常用的架构模式，包括MVC、MVP、MVVM和VIPER，分析它们的优缺点，并提供实际的代码示例和最佳实践。

## 架构设计模式概述

### 为什么需要架构设计模式

**1. 代码组织**
- 将代码按照职责分离，提高可读性
- 降低模块间的耦合度
- 便于团队协作开发

**2. 可维护性**
- 易于修改和扩展功能
- 减少bug的产生
- 提高代码质量

**3. 可测试性**
- 支持单元测试
- 便于模拟和隔离测试
- 提高测试覆盖率

**4. 可扩展性**
- 支持功能的快速迭代
- 便于添加新特性
- 适应业务需求变化

### 架构模式选择原则

```objc
@interface ArchitectureEvaluator : NSObject

// 评估标准
@property (nonatomic, assign) NSInteger teamSize;
@property (nonatomic, assign) NSInteger projectComplexity;
@property (nonatomic, assign) NSInteger testingRequirement;
@property (nonatomic, assign) NSInteger maintenanceRequirement;
@property (nonatomic, assign) NSInteger developmentSpeed;

- (NSString *)recommendedArchitecture;
- (NSDictionary *)evaluateArchitectures;
@end

@implementation ArchitectureEvaluator

- (NSString *)recommendedArchitecture {
    NSDictionary *scores = [self evaluateArchitectures];
    
    NSString *bestArchitecture = nil;
    NSInteger highestScore = 0;
    
    for (NSString *architecture in scores.allKeys) {
        NSInteger score = [scores[architecture] integerValue];
        if (score > highestScore) {
            highestScore = score;
            bestArchitecture = architecture;
        }
    }
    
    return bestArchitecture;
}

- (NSDictionary *)evaluateArchitectures {
    NSMutableDictionary *scores = [NSMutableDictionary dictionary];
    
    // MVC评分
    NSInteger mvcScore = 0;
    if (_teamSize <= 3) mvcScore += 2;
    if (_projectComplexity <= 3) mvcScore += 2;
    if (_developmentSpeed >= 4) mvcScore += 2;
    scores[@"MVC"] = @(mvcScore);
    
    // MVP评分
    NSInteger mvpScore = 0;
    if (_teamSize <= 5) mvpScore += 1;
    if (_testingRequirement >= 3) mvpScore += 2;
    if (_projectComplexity >= 3) mvpScore += 1;
    scores[@"MVP"] = @(mvpScore);
    
    // MVVM评分
    NSInteger mvvmScore = 0;
    if (_testingRequirement >= 4) mvvmScore += 2;
    if (_maintenanceRequirement >= 4) mvvmScore += 2;
    if (_projectComplexity >= 4) mvvmScore += 1;
    scores[@"MVVM"] = @(mvvmScore);
    
    // VIPER评分
    NSInteger viperScore = 0;
    if (_teamSize >= 5) viperScore += 2;
    if (_projectComplexity >= 4) viperScore += 2;
    if (_maintenanceRequirement >= 4) viperScore += 1;
    if (_testingRequirement >= 4) viperScore += 1;
    scores[@"VIPER"] = @(viperScore);
    
    return scores;
}

@end
```

**5. Router实现**

```objc
// UserListRouter.h
@interface UserListRouter : NSObject <UserListRouterProtocol>
@property (nonatomic, weak) UIViewController *viewController;
@end

// UserListRouter.m
@implementation UserListRouter

- (void)navigateToUserDetail:(UserEntity *)user {
    // 创建用户详情模块
    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    UIViewController *userDetailVC = [storyboard instantiateViewControllerWithIdentifier:@"UserDetailViewController"];
    
    // 传递用户数据
    // [userDetailVC configureWithUser:user];
    
    [self.viewController.navigationController pushViewController:userDetailVC animated:YES];
}

- (void)navigateToAddUser {
    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    UIViewController *addUserVC = [storyboard instantiateViewControllerWithIdentifier:@"AddUserViewController"];
    
    UINavigationController *navController = [[UINavigationController alloc] initWithRootViewController:addUserVC];
    [self.viewController presentViewController:navController animated:YES completion:nil];
}

@end
```

**6. View实现**

```objc
// UserListVIPERViewController.h
@interface UserListVIPERViewController : UIViewController <UserListViewProtocol>
@property (nonatomic, weak) IBOutlet UITableView *tableView;
@property (nonatomic, weak) IBOutlet UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, strong) id<UserListPresenterProtocol> presenter;
@end

// UserListVIPERViewController.m
@implementation UserListVIPERViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self setupTableView];
    [self.presenter viewDidLoad];
}

- (void)setupUI {
    self.title = @"Users (VIPER)";
    
    UIBarButtonItem *refreshButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemRefresh 
                                                                                   target:self 
                                                                                   action:@selector(refreshUsers)];
    
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
                                                                               target:self 
                                                                               action:@selector(addUser)];
    
    self.navigationItem.rightBarButtonItems = @[addButton, refreshButton];
}

- (void)setupTableView {
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    
    UINib *cellNib = [UINib nibWithNibName:@"UserTableViewCell" bundle:nil];
    [self.tableView registerNib:cellNib forCellReuseIdentifier:@"UserTableViewCell"];
    
    self.tableView.rowHeight = 80;
    self.tableView.estimatedRowHeight = 80;
    
    UIRefreshControl *refreshControl = [[UIRefreshControl alloc] init];
    [refreshControl addTarget:self action:@selector(refreshUsers) forControlEvents:UIControlEventValueChanged];
    self.tableView.refreshControl = refreshControl;
}

- (void)refreshUsers {
    [self.presenter refreshUsers];
}

- (void)addUser {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Add User" 
                                                                   message:@"Enter user information" 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Username";
    }];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Email";
        textField.keyboardType = UIKeyboardTypeEmailAddress;
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                           style:UIAlertActionStyleCancel 
                                                         handler:nil];
    
    UIAlertAction *addAction = [UIAlertAction actionWithTitle:@"Add" 
                                                        style:UIAlertActionStyleDefault 
                                                      handler:^(UIAlertAction *action) {
        NSString *username = alert.textFields[0].text;
        NSString *email = alert.textFields[1].text;
        [self.presenter addUserWithUsername:username email:email];
    }];
    
    [alert addAction:cancelAction];
    [alert addAction:addAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - UserListViewProtocol

- (void)showUsers:(NSArray<UserEntity *> *)users {
    [self.tableView reloadData];
}

- (void)showLoading:(BOOL)loading {
    if (loading) {
        [self.loadingIndicator startAnimating];
    } else {
        [self.loadingIndicator stopAnimating];
        [self.tableView.refreshControl endRefreshing];
    }
    
    self.tableView.userInteractionEnabled = !loading;
}

- (void)showError:(NSString *)errorMessage {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error" 
                                                                   message:errorMessage 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" 
                                                       style:UIAlertActionStyleDefault 
                                                     handler:nil];
    
    [alert addAction:okAction];
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)showUserAdded {
    NSIndexPath *indexPath = [NSIndexPath indexPathForRow:[self.presenter numberOfUsers] - 1 inSection:0];
    [self.tableView insertRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
}

- (void)showUserDeleted {
    [self.tableView reloadData];
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return [self.presenter numberOfUsers];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserTableViewCell" forIndexPath:indexPath];
    
    UserEntity *userEntity = [self.presenter userAtIndex:indexPath.row];
    User *user = [userEntity toUser];
    [cell configureWithUser:user];
    
    return cell;
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    [self.presenter selectUserAtIndex:indexPath.row];
}

- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Delete User" 
                                                                       message:@"Are you sure you want to delete this user?" 
                                                                preferredStyle:UIAlertControllerStyleAlert];
        
        UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                               style:UIAlertActionStyleCancel 
                                                             handler:nil];
        
        UIAlertAction *deleteAction = [UIAlertAction actionWithTitle:@"Delete" 
                                                               style:UIAlertActionStyleDestructive 
                                                             handler:^(UIAlertAction *action) {
            [self.presenter deleteUserAtIndex:indexPath.row];
        }];
        
        [alert addAction:cancelAction];
        [alert addAction:deleteAction];
        
        [self presentViewController:alert animated:YES completion:nil];
    }
}

@end
```

**7. 模块装配器**

```objc
// UserListVIPERAssembler.h
@interface UserListVIPERAssembler : NSObject
+ (UIViewController *)assembleUserListModule;
@end

// UserListVIPERAssembler.m
@implementation UserListVIPERAssembler

+ (UIViewController *)assembleUserListModule {
    // 创建各个组件
    UserListVIPERViewController *view = [[UserListVIPERViewController alloc] init];
    UserListPresenter *presenter = [[UserListPresenter alloc] init];
    UserListInteractor *interactor = [[UserListInteractor alloc] init];
    UserListRouter *router = [[UserListRouter alloc] init];
    
    // 建立连接
    view.presenter = presenter;
    presenter.view = view;
    presenter.interactor = interactor;
    presenter.router = router;
    interactor.output = presenter;
    router.viewController = view;
    
    return view;
}

@end
```

### VIPER优缺点分析

**优点：**
1. 职责分离非常清晰
2. 高度可测试性
3. 易于团队协作开发
4. 支持复杂业务逻辑
5. 导航逻辑独立管理

**缺点：**
1. 代码量大，开发复杂度高
2. 学习成本高
3. 过度设计，适合大型项目
4. 需要大量的协议定义
5. 调试相对困难

## 架构模式对比分析

### 复杂度对比

```objc
@interface ArchitectureComplexityAnalyzer : NSObject

+ (NSDictionary *)analyzeComplexity;
+ (NSString *)recommendArchitectureForProjectSize:(NSInteger)projectSize 
                                         teamSize:(NSInteger)teamSize 
                                       complexity:(NSInteger)complexity;
@end

@implementation ArchitectureComplexityAnalyzer

+ (NSDictionary *)analyzeComplexity {
    return @{
        @"MVC": @{
            @"learning_curve": @1,
            @"code_volume": @1,
            @"testability": @2,
            @"maintainability": @2,
            @"scalability": @2
        },
        @"MVP": @{
            @"learning_curve": @2,
            @"code_volume": @2,
            @"testability": @4,
            @"maintainability": @3,
            @"scalability": @3
        },
        @"MVVM": @{
            @"learning_curve": @3,
            @"code_volume": @3,
            @"testability": @4,
            @"maintainability": @4,
            @"scalability": @4
        },
        @"VIPER": @{
            @"learning_curve": @4,
            @"code_volume": @4,
            @"testability": @5,
            @"maintainability": @5,
            @"scalability": @5
        }
    };
}

+ (NSString *)recommendArchitectureForProjectSize:(NSInteger)projectSize 
                                         teamSize:(NSInteger)teamSize 
                                       complexity:(NSInteger)complexity {
    NSInteger totalScore = projectSize + teamSize + complexity;
    
    if (totalScore <= 6) {
        return @"MVC";
    } else if (totalScore <= 9) {
        return @"MVP";
    } else if (totalScore <= 12) {
        return @"MVVM";
    } else {
        return @"VIPER";
    }
}

@end
```

### 性能对比

| 架构模式 | 内存占用 | CPU开销 | 启动时间 | 运行效率 |
|---------|---------|---------|----------|----------|
| MVC     | 低      | 低      | 快       | 高       |
| MVP     | 中      | 中      | 中       | 中       |
| MVVM    | 中高    | 中高    | 中       | 中       |
| VIPER   | 高      | 高      | 慢       | 中低     |

### 适用场景分析

**MVC适用场景：**
- 小型项目（< 20个页面）
- 快速原型开发
- 团队规模小（1-3人）
- 简单的业务逻辑

**MVP适用场景：**
- 中小型项目（20-50个页面）
- 需要单元测试
- 团队规模中等（3-5人）
- 中等复杂度业务逻辑

**MVVM适用场景：**
- 中大型项目（50-100个页面）
- 复杂的数据绑定需求
- 团队有响应式编程经验
- 需要高度可测试性

**VIPER适用场景：**
- 大型企业级项目（> 100个页面）
- 复杂的业务逻辑
- 大型团队（> 5人）
- 长期维护项目

## 架构设计最佳实践

### 1. 渐进式架构演进

```objc
@interface ArchitectureEvolutionManager : NSObject

@property (nonatomic, strong) NSString *currentArchitecture;
@property (nonatomic, strong) NSArray<NSString *> *evolutionPath;

- (void)planEvolution:(NSString *)targetArchitecture;
- (BOOL)canEvolveToArchitecture:(NSString *)architecture;
- (NSArray<NSString *> *)getEvolutionSteps:(NSString *)targetArchitecture;
@end

@implementation ArchitectureEvolutionManager

- (instancetype)init {
    self = [super init];
    if (self) {
        _currentArchitecture = @"MVC";
        _evolutionPath = @[@"MVC", @"MVP", @"MVVM", @"VIPER"];
    }
    return self;
}

- (void)planEvolution:(NSString *)targetArchitecture {
    NSArray<NSString *> *steps = [self getEvolutionSteps:targetArchitecture];
    
    NSLog(@"Architecture Evolution Plan:");
    for (NSInteger i = 0; i < steps.count; i++) {
        NSLog(@"Step %ld: %@", (long)(i + 1), steps[i]);
    }
}

- (BOOL)canEvolveToArchitecture:(NSString *)architecture {
    NSInteger currentIndex = [self.evolutionPath indexOfObject:self.currentArchitecture];
    NSInteger targetIndex = [self.evolutionPath indexOfObject:architecture];
    
    return targetIndex > currentIndex;
}

- (NSArray<NSString *> *)getEvolutionSteps:(NSString *)targetArchitecture {
    NSInteger currentIndex = [self.evolutionPath indexOfObject:self.currentArchitecture];
    NSInteger targetIndex = [self.evolutionPath indexOfObject:targetArchitecture];
    
    if (targetIndex <= currentIndex) {
        return @[];
    }
    
    NSRange range = NSMakeRange(currentIndex + 1, targetIndex - currentIndex);
    return [self.evolutionPath subarrayWithRange:range];
}

@end
```

### 2. 架构质量评估

```objc
@interface ArchitectureQualityAssessment : NSObject

- (NSDictionary *)assessCodebase:(NSString *)projectPath;
- (NSInteger)calculateMaintainabilityIndex:(NSDictionary *)metrics;
- (NSArray<NSString *> *)identifyArchitecturalSmells:(NSDictionary *)metrics;
- (NSDictionary *)generateImprovementSuggestions:(NSDictionary *)assessment;

@end

@implementation ArchitectureQualityAssessment

- (NSDictionary *)assessCodebase:(NSString *)projectPath {
    // 模拟代码质量评估
    return @{
        @"coupling": @{
            @"score": @75,
            @"threshold": @80,
            @"status": @"good"
        },
        @"cohesion": @{
            @"score": @85,
            @"threshold": @80,
            @"status": @"excellent"
        },
        @"complexity": @{
            @"score": @65,
            @"threshold": @70,
            @"status": @"warning"
        },
        @"testability": @{
            @"score": @70,
            @"threshold": @75,
            @"status": @"warning"
        },
        @"maintainability": @{
            @"score": @78,
            @"threshold": @75,
            @"status": @"good"
        }
    };
}

- (NSInteger)calculateMaintainabilityIndex:(NSDictionary *)metrics {
    NSInteger coupling = [metrics[@"coupling"][@"score"] integerValue];
    NSInteger cohesion = [metrics[@"cohesion"][@"score"] integerValue];
    NSInteger complexity = [metrics[@"complexity"][@"score"] integerValue];
    NSInteger testability = [metrics[@"testability"][@"score"] integerValue];
    
    return (coupling + cohesion + complexity + testability) / 4;
}

- (NSArray<NSString *> *)identifyArchitecturalSmells:(NSDictionary *)metrics {
    NSMutableArray<NSString *> *smells = [NSMutableArray array];
    
    if ([metrics[@"coupling"][@"score"] integerValue] < 70) {
        [smells addObject:@"High Coupling - Consider dependency injection"];
    }
    
    if ([metrics[@"cohesion"][@"score"] integerValue] < 70) {
        [smells addObject:@"Low Cohesion - Review single responsibility principle"];
    }
    
    if ([metrics[@"complexity"][@"score"] integerValue] < 60) {
        [smells addObject:@"High Complexity - Consider breaking down large classes"];
    }
    
    if ([metrics[@"testability"][@"score"] integerValue] < 60) {
        [smells addObject:@"Poor Testability - Increase dependency injection usage"];
    }
    
    return [smells copy];
}

- (NSDictionary *)generateImprovementSuggestions:(NSDictionary *)assessment {
    NSMutableDictionary *suggestions = [NSMutableDictionary dictionary];
    
    for (NSString *metric in assessment.allKeys) {
        NSDictionary *metricData = assessment[metric];
        NSString *status = metricData[@"status"];
        
        if ([status isEqualToString:@"warning"] || [status isEqualToString:@"critical"]) {
            if ([metric isEqualToString:@"coupling"]) {
                suggestions[metric] = @"Implement dependency injection and use protocols for abstraction";
            } else if ([metric isEqualToString:@"complexity"]) {
                suggestions[metric] = @"Break down large classes and methods, apply single responsibility principle";
            } else if ([metric isEqualToString:@"testability"]) {
                suggestions[metric] = @"Increase unit test coverage and use mock objects for dependencies";
            }
        }
    }
    
    return [suggestions copy];
}

@end
```

## 总结

在iOS应用开发中，选择合适的架构设计模式对于项目的成功至关重要。每种架构模式都有其适用的场景和优缺点：

**关键要点：**

1. **MVC**：适合小型项目和快速开发，但容易导致Controller臃肿
2. **MVP**：通过Presenter分离业务逻辑，提高可测试性
3. **MVVM**：利用数据绑定机制，适合复杂的UI交互场景
4. **VIPER**：最细粒度的职责分离，适合大型企业级项目

**最佳实践建议：**

1. **渐进式演进**：从简单架构开始，随着项目复杂度增长逐步演进
2. **团队能力匹配**：选择团队能够掌握和维护的架构模式
3. **项目特点考虑**：根据项目规模、复杂度和维护周期选择合适架构
4. **质量持续监控**：定期评估架构质量，及时重构和优化
5. **文档和培训**：确保团队成员理解架构设计原则和实现细节

通过合理选择和实施架构设计模式，可以显著提高iOS应用的代码质量、可维护性和开发效率，为项目的长期成功奠定坚实基础。

## MVC架构模式

### MVC基本原理

MVC（Model-View-Controller）是iOS开发中最传统的架构模式，将应用分为三个核心组件：

- **Model**：数据模型和业务逻辑
- **View**：用户界面展示
- **Controller**：协调Model和View的交互

### MVC实现示例

**1. Model层实现**

```objc
// User.h
@interface User : NSObject
@property (nonatomic, strong) NSString *userId;
@property (nonatomic, strong) NSString *username;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, assign) BOOL isActive;

- (instancetype)initWithDictionary:(NSDictionary *)dictionary;
- (NSDictionary *)toDictionary;
- (BOOL)isValidEmail;
@end

// User.m
@implementation User

- (instancetype)initWithDictionary:(NSDictionary *)dictionary {
    self = [super init];
    if (self) {
        _userId = dictionary[@"id"];
        _username = dictionary[@"username"];
        _email = dictionary[@"email"];
        _isActive = [dictionary[@"is_active"] boolValue];
        
        NSString *dateString = dictionary[@"created_at"];
        if (dateString) {
            NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
            formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ss'Z'";
            _createdAt = [formatter dateFromString:dateString];
        }
    }
    return self;
}

- (NSDictionary *)toDictionary {
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    
    if (_userId) dict[@"id"] = _userId;
    if (_username) dict[@"username"] = _username;
    if (_email) dict[@"email"] = _email;
    dict[@"is_active"] = @(_isActive);
    
    if (_createdAt) {
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        formatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ss'Z'";
        dict[@"created_at"] = [formatter stringFromDate:_createdAt];
    }
    
    return dict;
}

- (BOOL)isValidEmail {
    NSString *emailRegex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}";
    NSPredicate *emailPredicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", emailRegex];
    return [emailPredicate evaluateWithObject:_email];
}

@end

// UserService.h
@interface UserService : NSObject

+ (instancetype)sharedService;
- (void)fetchUsersWithCompletion:(void(^)(NSArray<User *> *users, NSError *error))completion;
- (void)createUser:(User *)user completion:(void(^)(User *user, NSError *error))completion;
- (void)updateUser:(User *)user completion:(void(^)(User *user, NSError *error))completion;
- (void)deleteUserWithId:(NSString *)userId completion:(void(^)(BOOL success, NSError *error))completion;
@end

// UserService.m
@implementation UserService

+ (instancetype)sharedService {
    static UserService *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[UserService alloc] init];
    });
    return sharedInstance;
}

- (void)fetchUsersWithCompletion:(void(^)(NSArray<User *> *users, NSError *error))completion {
    // 模拟网络请求
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
        // 模拟数据
        NSArray *mockData = @[
            @{
                @"id": @"1",
                @"username": @"john_doe",
                @"email": @"john@example.com",
                @"is_active": @YES,
                @"created_at": @"2023-01-15T10:30:00Z"
            },
            @{
                @"id": @"2",
                @"username": @"jane_smith",
                @"email": @"jane@example.com",
                @"is_active": @YES,
                @"created_at": @"2023-02-20T14:45:00Z"
            }
        ];
        
        NSMutableArray *users = [NSMutableArray array];
        for (NSDictionary *userData in mockData) {
            User *user = [[User alloc] initWithDictionary:userData];
            [users addObject:user];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion([users copy], nil);
            }
        });
    });
}

- (void)createUser:(User *)user completion:(void(^)(User *user, NSError *error))completion {
    // 验证用户数据
    if (!user.username || user.username.length == 0) {
        NSError *error = [NSError errorWithDomain:@"UserServiceError" 
                                             code:1001 
                                         userInfo:@{NSLocalizedDescriptionKey: @"Username is required"}];
        if (completion) completion(nil, error);
        return;
    }
    
    if (![user isValidEmail]) {
        NSError *error = [NSError errorWithDomain:@"UserServiceError" 
                                             code:1002 
                                         userInfo:@{NSLocalizedDescriptionKey: @"Invalid email format"}];
        if (completion) completion(nil, error);
        return;
    }
    
    // 模拟网络请求
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
        // 模拟服务器响应
        user.userId = [[NSUUID UUID] UUIDString];
        user.createdAt = [NSDate date];
        user.isActive = YES;
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(user, nil);
            }
        });
    });
}

- (void)updateUser:(User *)user completion:(void(^)(User *user, NSError *error))completion {
    if (!user.userId) {
        NSError *error = [NSError errorWithDomain:@"UserServiceError" 
                                             code:1003 
                                         userInfo:@{NSLocalizedDescriptionKey: @"User ID is required for update"}];
        if (completion) completion(nil, error);
        return;
    }
    
    // 模拟网络请求
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(user, nil);
            }
        });
    });
}

- (void)deleteUserWithId:(NSString *)userId completion:(void(^)(BOOL success, NSError *error))completion {
    if (!userId || userId.length == 0) {
        NSError *error = [NSError errorWithDomain:@"UserServiceError" 
                                             code:1004 
                                         userInfo:@{NSLocalizedDescriptionKey: @"User ID is required for deletion"}];
        if (completion) completion(NO, error);
        return;
    }
    
    // 模拟网络请求
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completion) {
                completion(YES, nil);
            }
        });
    });
}

@end
```

**2. View层实现**

```objc
// UserTableViewCell.h
@interface UserTableViewCell : UITableViewCell
@property (nonatomic, weak) IBOutlet UILabel *usernameLabel;
@property (nonatomic, weak) IBOutlet UILabel *emailLabel;
@property (nonatomic, weak) IBOutlet UILabel *statusLabel;
@property (nonatomic, weak) IBOutlet UIImageView *avatarImageView;

- (void)configureWithUser:(User *)user;
@end

// UserTableViewCell.m
@implementation UserTableViewCell

- (void)awakeFromNib {
    [super awakeFromNib];
    [self setupUI];
}

- (void)setupUI {
    // 设置头像圆角
    self.avatarImageView.layer.cornerRadius = 25;
    self.avatarImageView.layer.masksToBounds = YES;
    self.avatarImageView.backgroundColor = [UIColor lightGrayColor];
    
    // 设置标签样式
    self.usernameLabel.font = [UIFont boldSystemFontOfSize:16];
    self.emailLabel.font = [UIFont systemFontOfSize:14];
    self.emailLabel.textColor = [UIColor grayColor];
    self.statusLabel.font = [UIFont systemFontOfSize:12];
    self.statusLabel.layer.cornerRadius = 8;
    self.statusLabel.layer.masksToBounds = YES;
    self.statusLabel.textAlignment = NSTextAlignmentCenter;
}

- (void)configureWithUser:(User *)user {
    self.usernameLabel.text = user.username;
    self.emailLabel.text = user.email;
    
    if (user.isActive) {
        self.statusLabel.text = @"Active";
        self.statusLabel.backgroundColor = [UIColor greenColor];
        self.statusLabel.textColor = [UIColor whiteColor];
    } else {
        self.statusLabel.text = @"Inactive";
        self.statusLabel.backgroundColor = [UIColor redColor];
        self.statusLabel.textColor = [UIColor whiteColor];
    }
    
    // 设置默认头像
    self.avatarImageView.image = [UIImage imageNamed:@"default_avatar"];
}

- (void)prepareForReuse {
    [super prepareForReuse];
    self.usernameLabel.text = nil;
    self.emailLabel.text = nil;
    self.statusLabel.text = nil;
    self.avatarImageView.image = nil;
}

@end
```

**3. Controller层实现**

```objc
// UserListViewController.h
@interface UserListViewController : UIViewController
@property (nonatomic, weak) IBOutlet UITableView *tableView;
@property (nonatomic, weak) IBOutlet UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, strong) NSMutableArray<User *> *users;
@property (nonatomic, strong) UserService *userService;
@end

// UserListViewController.m
@implementation UserListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self setupTableView];
    [self loadUsers];
}

- (void)setupUI {
    self.title = @"Users";
    
    // 添加刷新按钮
    UIBarButtonItem *refreshButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemRefresh 
                                                                                   target:self 
                                                                                   action:@selector(refreshUsers)];
    
    // 添加新增按钮
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
                                                                               target:self 
                                                                               action:@selector(addUser)];
    
    self.navigationItem.rightBarButtonItems = @[addButton, refreshButton];
    
    // 初始化数据
    self.users = [NSMutableArray array];
    self.userService = [UserService sharedService];
}

- (void)setupTableView {
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    
    // 注册cell
    UINib *cellNib = [UINib nibWithNibName:@"UserTableViewCell" bundle:nil];
    [self.tableView registerNib:cellNib forCellReuseIdentifier:@"UserTableViewCell"];
    
    // 设置行高
    self.tableView.rowHeight = 80;
    self.tableView.estimatedRowHeight = 80;
    
    // 添加下拉刷新
    UIRefreshControl *refreshControl = [[UIRefreshControl alloc] init];
    [refreshControl addTarget:self action:@selector(refreshUsers) forControlEvents:UIControlEventValueChanged];
    self.tableView.refreshControl = refreshControl;
}

- (void)loadUsers {
    [self showLoading:YES];
    
    [self.userService fetchUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
        [self showLoading:NO];
        
        if (error) {
            [self showErrorAlert:error.localizedDescription];
        } else {
            [self.users removeAllObjects];
            [self.users addObjectsFromArray:users];
            [self.tableView reloadData];
        }
    }];
}

- (void)refreshUsers {
    [self.userService fetchUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
        [self.tableView.refreshControl endRefreshing];
        
        if (error) {
            [self showErrorAlert:error.localizedDescription];
        } else {
            [self.users removeAllObjects];
            [self.users addObjectsFromArray:users];
            [self.tableView reloadData];
        }
    }];
}

- (void)addUser {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Add User" 
                                                                   message:@"Enter user information" 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Username";
    }];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Email";
        textField.keyboardType = UIKeyboardTypeEmailAddress;
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                           style:UIAlertActionStyleCancel 
                                                         handler:nil];
    
    UIAlertAction *addAction = [UIAlertAction actionWithTitle:@"Add" 
                                                        style:UIAlertActionStyleDefault 
                                                      handler:^(UIAlertAction *action) {
        NSString *username = alert.textFields[0].text;
        NSString *email = alert.textFields[1].text;
        
        if (username.length > 0 && email.length > 0) {
            [self createUserWithUsername:username email:email];
        } else {
            [self showErrorAlert:@"Please fill in all fields"];
        }
    }];
    
    [alert addAction:cancelAction];
    [alert addAction:addAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)createUserWithUsername:(NSString *)username email:(NSString *)email {
    User *newUser = [[User alloc] init];
    newUser.username = username;
    newUser.email = email;
    
    [self showLoading:YES];
    
    [self.userService createUser:newUser completion:^(User *user, NSError *error) {
        [self showLoading:NO];
        
        if (error) {
            [self showErrorAlert:error.localizedDescription];
        } else {
            [self.users addObject:user];
            NSIndexPath *indexPath = [NSIndexPath indexPathForRow:self.users.count - 1 inSection:0];
            [self.tableView insertRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
        }
    }];
}

- (void)deleteUserAtIndexPath:(NSIndexPath *)indexPath {
    User *user = self.users[indexPath.row];
    
    [self showLoading:YES];
    
    [self.userService deleteUserWithId:user.userId completion:^(BOOL success, NSError *error) {
        [self showLoading:NO];
        
        if (error) {
            [self showErrorAlert:error.localizedDescription];
        } else if (success) {
            [self.users removeObjectAtIndex:indexPath.row];
            [self.tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
        }
    }];
}

- (void)showLoading:(BOOL)show {
    if (show) {
        [self.loadingIndicator startAnimating];
    } else {
        [self.loadingIndicator stopAnimating];
    }
    
    self.tableView.userInteractionEnabled = !show;
}

- (void)showErrorAlert:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error" 
                                                                   message:message 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" 
                                                       style:UIAlertActionStyleDefault 
                                                     handler:nil];
    
    [alert addAction:okAction];
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.users.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserTableViewCell" forIndexPath:indexPath];
    
    User *user = self.users[indexPath.row];
    [cell configureWithUser:user];
    
    return cell;
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    User *user = self.users[indexPath.row];
    // 可以在这里导航到用户详情页面
    NSLog(@"Selected user: %@", user.username);
}

- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Delete User" 
                                                                       message:@"Are you sure you want to delete this user?" 
                                                                preferredStyle:UIAlertControllerStyleAlert];
        
        UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                               style:UIAlertActionStyleCancel 
                                                             handler:nil];
        
        UIAlertAction *deleteAction = [UIAlertAction actionWithTitle:@"Delete" 
                                                               style:UIAlertActionStyleDestructive 
                                                             handler:^(UIAlertAction *action) {
            [self deleteUserAtIndexPath:indexPath];
        }];
        
        [alert addAction:cancelAction];
        [alert addAction:deleteAction];
        
        [self presentViewController:alert animated:YES completion:nil];
    }
}

@end
```

### MVC优缺点分析

**优点：**
1. 简单易懂，学习成本低
2. 开发速度快，适合小型项目
3. iOS原生支持，框架集成度高
4. 代码结构清晰，职责分离明确

**缺点：**
1. Controller容易变得臃肿（Massive View Controller）
2. 难以进行单元测试
3. View和Controller耦合度高
4. 复杂业务逻辑难以维护

## MVP架构模式

### MVP基本原理

MVP（Model-View-Presenter）模式通过引入Presenter层来解决MVC中Controller臃肿的问题：

- **Model**：数据模型和业务逻辑
- **View**：用户界面展示和用户交互
- **Presenter**：处理业务逻辑，协调Model和View

### MVP实现示例

**1. View协议定义**

```objc
// UserListViewProtocol.h
@protocol UserListViewProtocol <NSObject>

- (void)showUsers:(NSArray<User *> *)users;
- (void)showLoading:(BOOL)loading;
- (void)showError:(NSString *)errorMessage;
- (void)showUserAdded:(User *)user;
- (void)showUserDeleted:(NSIndexPath *)indexPath;
- (void)refreshUserList;

@end
```

**2. Presenter实现**

```objc
// UserListPresenter.h
@interface UserListPresenter : NSObject

@property (nonatomic, weak) id<UserListViewProtocol> view;
@property (nonatomic, strong) UserService *userService;
@property (nonatomic, strong) NSMutableArray<User *> *users;

- (instancetype)initWithView:(id<UserListViewProtocol>)view;
- (void)viewDidLoad;
- (void)refreshUsers;
- (void)addUserWithUsername:(NSString *)username email:(NSString *)email;
- (void)deleteUserAtIndex:(NSInteger)index;
- (User *)userAtIndex:(NSInteger)index;
- (NSInteger)numberOfUsers;

@end

// UserListPresenter.m
@implementation UserListPresenter

- (instancetype)initWithView:(id<UserListViewProtocol>)view {
    self = [super init];
    if (self) {
        _view = view;
        _userService = [UserService sharedService];
        _users = [NSMutableArray array];
    }
    return self;
}

- (void)viewDidLoad {
    [self loadUsers];
}

- (void)loadUsers {
    [self.view showLoading:YES];
    
    [self.userService fetchUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
        [self.view showLoading:NO];
        
        if (error) {
            [self.view showError:error.localizedDescription];
        } else {
            [self.users removeAllObjects];
            [self.users addObjectsFromArray:users];
            [self.view showUsers:[self.users copy]];
        }
    }];
}

- (void)refreshUsers {
    [self.userService fetchUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
        if (error) {
            [self.view showError:error.localizedDescription];
        } else {
            [self.users removeAllObjects];
            [self.users addObjectsFromArray:users];
            [self.view refreshUserList];
        }
    }];
}

- (void)addUserWithUsername:(NSString *)username email:(NSString *)email {
    // 输入验证
    if (!username || username.length == 0) {
        [self.view showError:@"Username is required"];
        return;
    }
    
    if (!email || email.length == 0) {
        [self.view showError:@"Email is required"];
        return;
    }
    
    User *newUser = [[User alloc] init];
    newUser.username = username;
    newUser.email = email;
    
    [self.view showLoading:YES];
    
    [self.userService createUser:newUser completion:^(User *user, NSError *error) {
        [self.view showLoading:NO];
        
        if (error) {
            [self.view showError:error.localizedDescription];
        } else {
            [self.users addObject:user];
            [self.view showUserAdded:user];
        }
    }];
}

- (void)deleteUserAtIndex:(NSInteger)index {
    if (index < 0 || index >= self.users.count) {
        [self.view showError:@"Invalid user index"];
        return;
    }
    
    User *user = self.users[index];
    
    [self.view showLoading:YES];
    
    [self.userService deleteUserWithId:user.userId completion:^(BOOL success, NSError *error) {
        [self.view showLoading:NO];
        
        if (error) {
            [self.view showError:error.localizedDescription];
        } else if (success) {
            [self.users removeObjectAtIndex:index];
            NSIndexPath *indexPath = [NSIndexPath indexPathForRow:index inSection:0];
            [self.view showUserDeleted:indexPath];
        }
    }];
}

- (User *)userAtIndex:(NSInteger)index {
    if (index >= 0 && index < self.users.count) {
        return self.users[index];
    }
    return nil;
}

- (NSInteger)numberOfUsers {
    return self.users.count;
}

@end
```

**3. View实现**

```objc
// UserListMVPViewController.h
@interface UserListMVPViewController : UIViewController <UserListViewProtocol>

@property (nonatomic, weak) IBOutlet UITableView *tableView;
@property (nonatomic, weak) IBOutlet UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, strong) UserListPresenter *presenter;

@end

// UserListMVPViewController.m
@implementation UserListMVPViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self setupTableView];
    [self setupPresenter];
}

- (void)setupUI {
    self.title = @"Users (MVP)";
    
    UIBarButtonItem *refreshButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemRefresh 
                                                                                   target:self 
                                                                                   action:@selector(refreshUsers)];
    
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
                                                                               target:self 
                                                                               action:@selector(addUser)];
    
    self.navigationItem.rightBarButtonItems = @[addButton, refreshButton];
}

- (void)setupTableView {
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    
    UINib *cellNib = [UINib nibWithNibName:@"UserTableViewCell" bundle:nil];
    [self.tableView registerNib:cellNib forCellReuseIdentifier:@"UserTableViewCell"];
    
    self.tableView.rowHeight = 80;
    self.tableView.estimatedRowHeight = 80;
    
    UIRefreshControl *refreshControl = [[UIRefreshControl alloc] init];
    [refreshControl addTarget:self action:@selector(refreshUsers) forControlEvents:UIControlEventValueChanged];
    self.tableView.refreshControl = refreshControl;
}

- (void)setupPresenter {
    self.presenter = [[UserListPresenter alloc] initWithView:self];
    [self.presenter viewDidLoad];
}

- (void)refreshUsers {
    [self.presenter refreshUsers];
}

- (void)addUser {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Add User" 
                                                                   message:@"Enter user information" 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Username";
    }];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Email";
        textField.keyboardType = UIKeyboardTypeEmailAddress;
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                           style:UIAlertActionStyleCancel 
                                                         handler:nil];
    
    UIAlertAction *addAction = [UIAlertAction actionWithTitle:@"Add" 
                                                        style:UIAlertActionStyleDefault 
                                                      handler:^(UIAlertAction *action) {
        NSString *username = alert.textFields[0].text;
        NSString *email = alert.textFields[1].text;
        [self.presenter addUserWithUsername:username email:email];
    }];
    
    [alert addAction:cancelAction];
    [alert addAction:addAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - UserListViewProtocol

- (void)showUsers:(NSArray<User *> *)users {
    [self.tableView reloadData];
}

- (void)showLoading:(BOOL)loading {
    if (loading) {
        [self.loadingIndicator startAnimating];
    } else {
        [self.loadingIndicator stopAnimating];
    }
    
    self.tableView.userInteractionEnabled = !loading;
}

- (void)showError:(NSString *)errorMessage {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error" 
                                                                   message:errorMessage 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" 
                                                       style:UIAlertActionStyleDefault 
                                                     handler:nil];
    
    [alert addAction:okAction];
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)showUserAdded:(User *)user {
    NSIndexPath *indexPath = [NSIndexPath indexPathForRow:[self.presenter numberOfUsers] - 1 inSection:0];
    [self.tableView insertRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
}

- (void)showUserDeleted:(NSIndexPath *)indexPath {
    [self.tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
}

- (void)refreshUserList {
    [self.tableView.refreshControl endRefreshing];
    [self.tableView reloadData];
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return [self.presenter numberOfUsers];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserTableViewCell" forIndexPath:indexPath];
    
    User *user = [self.presenter userAtIndex:indexPath.row];
    [cell configureWithUser:user];
    
    return cell;
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    User *user = [self.presenter userAtIndex:indexPath.row];
    NSLog(@"Selected user: %@", user.username);
}

- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Delete User" 
                                                                       message:@"Are you sure you want to delete this user?" 
                                                                preferredStyle:UIAlertControllerStyleAlert];
        
        UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                               style:UIAlertActionStyleCancel 
                                                             handler:nil];
        
        UIAlertAction *deleteAction = [UIAlertAction actionWithTitle:@"Delete" 
                                                               style:UIAlertActionStyleDestructive 
                                                             handler:^(UIAlertAction *action) {
            [self.presenter deleteUserAtIndex:indexPath.row];
        }];
        
        [alert addAction:cancelAction];
        [alert addAction:deleteAction];
        
        [self presentViewController:alert animated:YES completion:nil];
    }
}

@end
```

### MVP优缺点分析

**优点：**
1. 业务逻辑与UI分离，便于测试
2. Presenter可以独立测试
3. View变得更加简单
4. 代码复用性更好

**缺点：**
1. 代码量增加，复杂度提升
2. 需要定义更多的协议和接口
3. Presenter可能变得臃肿
4. 学习成本相对较高

## MVVM架构模式

### MVVM基本原理

MVVM（Model-View-ViewModel）模式通过数据绑定机制实现View和ViewModel的自动同步：

- **Model**：数据模型和业务逻辑
- **View**：用户界面展示
- **ViewModel**：视图模型，处理业务逻辑和数据绑定

### MVVM实现示例

**1. ViewModel实现**

```objc
// UserListViewModel.h
@interface UserListViewModel : NSObject

// 数据属性
@property (nonatomic, strong, readonly) NSArray<User *> *users;
@property (nonatomic, assign, readonly) BOOL isLoading;
@property (nonatomic, strong, readonly) NSString *errorMessage;

// 命令属性
@property (nonatomic, strong, readonly) RACCommand *loadUsersCommand;
@property (nonatomic, strong, readonly) RACCommand *refreshUsersCommand;
@property (nonatomic, strong, readonly) RACCommand *addUserCommand;
@property (nonatomic, strong, readonly) RACCommand *deleteUserCommand;

// 信号属性
@property (nonatomic, strong, readonly) RACSignal *usersSignal;
@property (nonatomic, strong, readonly) RACSignal *loadingSignal;
@property (nonatomic, strong, readonly) RACSignal *errorSignal;

- (instancetype)initWithUserService:(UserService *)userService;
- (User *)userAtIndex:(NSInteger)index;
- (NSInteger)numberOfUsers;

@end

// UserListViewModel.m
@implementation UserListViewModel {
    UserService *_userService;
    NSMutableArray<User *> *_mutableUsers;
}

- (instancetype)initWithUserService:(UserService *)userService {
    self = [super init];
    if (self) {
        _userService = userService;
        _mutableUsers = [NSMutableArray array];
        [self setupCommands];
        [self setupSignals];
    }
    return self;
}

- (void)setupCommands {
    // 加载用户命令
    _loadUsersCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
        return [self loadUsersSignal];
    }];
    
    // 刷新用户命令
    _refreshUsersCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
        return [self loadUsersSignal];
    }];
    
    // 添加用户命令
    _addUserCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(NSDictionary *userInfo) {
        return [self addUserSignal:userInfo];
    }];
    
    // 删除用户命令
    _deleteUserCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(NSNumber *index) {
        return [self deleteUserSignal:index.integerValue];
    }];
}

- (void)setupSignals {
    // 用户数据信号
    _usersSignal = RACObserve(self, users);
    
    // 加载状态信号
    _loadingSignal = [RACSignal merge:@[
        [self.loadUsersCommand.executing skip:1],
        [self.refreshUsersCommand.executing skip:1],
        [self.addUserCommand.executing skip:1],
        [self.deleteUserCommand.executing skip:1]
    ]];
    
    // 错误信号
    _errorSignal = [RACSignal merge:@[
        self.loadUsersCommand.errors,
        self.refreshUsersCommand.errors,
        self.addUserCommand.errors,
        self.deleteUserCommand.errors
    ]];
}

- (RACSignal *)loadUsersSignal {
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [self->_userService fetchUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
            if (error) {
                [subscriber sendError:error];
            } else {
                [self->_mutableUsers removeAllObjects];
                [self->_mutableUsers addObjectsFromArray:users];
                [subscriber sendNext:[self->_mutableUsers copy]];
                [subscriber sendCompleted];
            }
        }];
        
        return nil;
    }];
}

- (RACSignal *)addUserSignal:(NSDictionary *)userInfo {
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        NSString *username = userInfo[@"username"];
        NSString *email = userInfo[@"email"];
        
        // 输入验证
        if (!username || username.length == 0) {
            NSError *error = [NSError errorWithDomain:@"ValidationError" 
                                                 code:1001 
                                             userInfo:@{NSLocalizedDescriptionKey: @"Username is required"}];
            [subscriber sendError:error];
            return nil;
        }
        
        if (!email || email.length == 0) {
            NSError *error = [NSError errorWithDomain:@"ValidationError" 
                                                 code:1002 
                                             userInfo:@{NSLocalizedDescriptionKey: @"Email is required"}];
            [subscriber sendError:error];
            return nil;
        }
        
        User *newUser = [[User alloc] init];
        newUser.username = username;
        newUser.email = email;
        
        [self->_userService createUser:newUser completion:^(User *user, NSError *error) {
            if (error) {
                [subscriber sendError:error];
            } else {
                [self->_mutableUsers addObject:user];
                [subscriber sendNext:user];
                [subscriber sendCompleted];
            }
        }];
        
        return nil;
    }];
}

- (RACSignal *)deleteUserSignal:(NSInteger)index {
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        if (index < 0 || index >= self->_mutableUsers.count) {
            NSError *error = [NSError errorWithDomain:@"ValidationError" 
                                                 code:1003 
                                             userInfo:@{NSLocalizedDescriptionKey: @"Invalid user index"}];
            [subscriber sendError:error];
            return nil;
        }
        
        User *user = self->_mutableUsers[index];
        
        [self->_userService deleteUserWithId:user.userId completion:^(BOOL success, NSError *error) {
            if (error) {
                [subscriber sendError:error];
            } else if (success) {
                [self->_mutableUsers removeObjectAtIndex:index];
                [subscriber sendNext:@(index)];
                [subscriber sendCompleted];
            }
        }];
        
        return nil;
    }];
}

// 属性访问器
- (NSArray<User *> *)users {
    return [_mutableUsers copy];
}

- (BOOL)isLoading {
    return self.loadUsersCommand.executing.boolValue || 
           self.refreshUsersCommand.executing.boolValue ||
           self.addUserCommand.executing.boolValue ||
           self.deleteUserCommand.executing.boolValue;
}

- (User *)userAtIndex:(NSInteger)index {
    if (index >= 0 && index < _mutableUsers.count) {
        return _mutableUsers[index];
    }
    return nil;
}

- (NSInteger)numberOfUsers {
    return _mutableUsers.count;
}

@end
```

**2. View实现**

```objc
// UserListMVVMViewController.h
@interface UserListMVVMViewController : UIViewController

@property (nonatomic, weak) IBOutlet UITableView *tableView;
@property (nonatomic, weak) IBOutlet UIActivityIndicatorView *loadingIndicator;
@property (nonatomic, strong) UserListViewModel *viewModel;

@end

// UserListMVVMViewController.m
@implementation UserListMVVMViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupUI];
    [self setupTableView];
    [self setupViewModel];
    [self bindViewModel];
}

- (void)setupUI {
    self.title = @"Users (MVVM)";
    
    UIBarButtonItem *refreshButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemRefresh 
                                                                                   target:self 
                                                                                   action:@selector(refreshUsers)];
    
    UIBarButtonItem *addButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemAdd 
                                                                               target:self 
                                                                               action:@selector(addUser)];
    
    self.navigationItem.rightBarButtonItems = @[addButton, refreshButton];
}

- (void)setupTableView {
    self.tableView.delegate = self;
    self.tableView.dataSource = self;
    
    UINib *cellNib = [UINib nibWithNibName:@"UserTableViewCell" bundle:nil];
    [self.tableView registerNib:cellNib forCellReuseIdentifier:@"UserTableViewCell"];
    
    self.tableView.rowHeight = 80;
    self.tableView.estimatedRowHeight = 80;
    
    UIRefreshControl *refreshControl = [[UIRefreshControl alloc] init];
    self.tableView.refreshControl = refreshControl;
}

- (void)setupViewModel {
    UserService *userService = [UserService sharedService];
    self.viewModel = [[UserListViewModel alloc] initWithUserService:userService];
}

- (void)bindViewModel {
    // 绑定用户数据
    [self.viewModel.usersSignal subscribeNext:^(NSArray<User *> *users) {
        [self.tableView reloadData];
    }];
    
    // 绑定加载状态
    [self.viewModel.loadingSignal subscribeNext:^(NSNumber *loading) {
        if (loading.boolValue) {
            [self.loadingIndicator startAnimating];
        } else {
            [self.loadingIndicator stopAnimating];
            [self.tableView.refreshControl endRefreshing];
        }
        
        self.tableView.userInteractionEnabled = !loading.boolValue;
    }];
    
    // 绑定错误信息
    [self.viewModel.errorSignal subscribeNext:^(NSError *error) {
        [self showErrorAlert:error.localizedDescription];
    }];
    
    // 绑定刷新控件
    self.tableView.refreshControl.rac_command = self.viewModel.refreshUsersCommand;
    
    // 绑定添加用户结果
    [self.viewModel.addUserCommand.executionSignals.switchToLatest subscribeNext:^(User *user) {
        NSIndexPath *indexPath = [NSIndexPath indexPathForRow:[self.viewModel numberOfUsers] - 1 inSection:0];
        [self.tableView insertRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
    }];
    
    // 绑定删除用户结果
    [self.viewModel.deleteUserCommand.executionSignals.switchToLatest subscribeNext:^(NSNumber *index) {
        NSIndexPath *indexPath = [NSIndexPath indexPathForRow:index.integerValue inSection:0];
        [self.tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
    }];
    
    // 初始加载
    [self.viewModel.loadUsersCommand execute:nil];
}

- (void)refreshUsers {
    [self.viewModel.refreshUsersCommand execute:nil];
}

- (void)addUser {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Add User" 
                                                                   message:@"Enter user information" 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Username";
    }];
    
    [alert addTextFieldWithConfigurationHandler:^(UITextField *textField) {
        textField.placeholder = @"Email";
        textField.keyboardType = UIKeyboardTypeEmailAddress;
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                           style:UIAlertActionStyleCancel 
                                                         handler:nil];
    
    UIAlertAction *addAction = [UIAlertAction actionWithTitle:@"Add" 
                                                        style:UIAlertActionStyleDefault 
                                                      handler:^(UIAlertAction *action) {
        NSString *username = alert.textFields[0].text;
        NSString *email = alert.textFields[1].text;
        
        NSDictionary *userInfo = @{
            @"username": username ?: @"",
            @"email": email ?: @""
        };
        
        [self.viewModel.addUserCommand execute:userInfo];
    }];
    
    [alert addAction:cancelAction];
    [alert addAction:addAction];
    
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)showErrorAlert:(NSString *)message {
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Error" 
                                                                   message:message 
                                                            preferredStyle:UIAlertControllerStyleAlert];
    
    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" 
                                                       style:UIAlertActionStyleDefault 
                                                     handler:nil];
    
    [alert addAction:okAction];
    [self presentViewController:alert animated:YES completion:nil];
}

#pragma mark - UITableViewDataSource

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return [self.viewModel numberOfUsers];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserTableViewCell" forIndexPath:indexPath];
    
    User *user = [self.viewModel userAtIndex:indexPath.row];
    [cell configureWithUser:user];
    
    return cell;
}

#pragma mark - UITableViewDelegate

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    
    User *user = [self.viewModel userAtIndex:indexPath.row];
    NSLog(@"Selected user: %@", user.username);
}

- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Delete User" 
                                                                       message:@"Are you sure you want to delete this user?" 
                                                                preferredStyle:UIAlertControllerStyleAlert];
        
        UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" 
                                                               style:UIAlertActionStyleCancel 
                                                             handler:nil];
        
        UIAlertAction *deleteAction = [UIAlertAction actionWithTitle:@"Delete" 
                                                               style:UIAlertActionStyleDestructive 
                                                             handler:^(UIAlertAction *action) {
            [self.viewModel.deleteUserCommand execute:@(indexPath.row)];
        }];
        
        [alert addAction:cancelAction];
        [alert addAction:deleteAction];
        
        [self presentViewController:alert animated:YES completion:nil];
    }
}

@end
```

### MVVM优缺点分析

**优点：**
1. 数据绑定机制，减少样板代码
2. ViewModel易于测试
3. 支持双向数据绑定
4. 代码结构清晰，职责分离好

**缺点：**
1. 学习成本高，需要理解响应式编程
2. 调试相对困难
3. 过度使用可能导致性能问题
4. 需要额外的响应式编程框架支持

## VIPER架构模式

### VIPER基本原理

VIPER（View-Interactor-Presenter-Entity-Router）是一种更加细粒度的架构模式，将应用分为五个核心组件：

- **View**：用户界面展示和用户交互
- **Interactor**：业务逻辑处理
- **Presenter**：格式化数据，处理View和Interactor之间的通信
- **Entity**：数据模型
- **Router**：导航逻辑处理

### VIPER实现示例

**1. Entity层**

```objc
// UserEntity.h
@interface UserEntity : NSObject
@property (nonatomic, strong) NSString *userId;
@property (nonatomic, strong) NSString *username;
@property (nonatomic, strong) NSString *email;
@property (nonatomic, strong) NSDate *createdAt;
@property (nonatomic, assign) BOOL isActive;

- (instancetype)initWithUser:(User *)user;
- (User *)toUser;
@end

// UserEntity.m
@implementation UserEntity

- (instancetype)initWithUser:(User *)user {
    self = [super init];
    if (self) {
        _userId = user.userId;
        _username = user.username;
        _email = user.email;
        _createdAt = user.createdAt;
        _isActive = user.isActive;
    }
    return self;
}

- (User *)toUser {
    User *user = [[User alloc] init];
    user.userId = self.userId;
    user.username = self.username;
    user.email = self.email;
    user.createdAt = self.createdAt;
    user.isActive = self.isActive;
    return user;
}

@end
```

**2. 协议定义**

```objc
// UserListVIPERProtocols.h

// View协议
@protocol UserListViewProtocol <NSObject>
- (void)showUsers:(NSArray<UserEntity *> *)users;
- (void)showLoading:(BOOL)loading;
- (void)showError:(NSString *)errorMessage;
- (void)showUserAdded;
- (void)showUserDeleted;
@end

// Presenter协议
@protocol UserListPresenterProtocol <NSObject>
- (void)viewDidLoad;
- (void)refreshUsers;
- (void)addUserWithUsername:(NSString *)username email:(NSString *)email;
- (void)deleteUserAtIndex:(NSInteger)index;
- (void)selectUserAtIndex:(NSInteger)index;
- (UserEntity *)userAtIndex:(NSInteger)index;
- (NSInteger)numberOfUsers;
@end

// Interactor协议
@protocol UserListInteractorProtocol <NSObject>
- (void)fetchUsers;
- (void)createUserWithUsername:(NSString *)username email:(NSString *)email;
- (void)deleteUserAtIndex:(NSInteger)index;
@end

// Router协议
@protocol UserListRouterProtocol <NSObject>
- (void)navigateToUserDetail:(UserEntity *)user;
- (void)navigateToAddUser;
@end

// Interactor输出协议
@protocol UserListInteractorOutputProtocol <NSObject>
- (void)usersFetched:(NSArray<UserEntity *> *)users;
- (void)usersFetchFailed:(NSString *)errorMessage;
- (void)userCreated:(UserEntity *)user;
- (void)userCreationFailed:(NSString *)errorMessage;
- (void)userDeleted:(NSInteger)index;
- (void)userDeletionFailed:(NSString *)errorMessage;
@end
```

**3. Interactor实现**

```objc
// UserListInteractor.h
@interface UserListInteractor : NSObject <UserListInteractorProtocol>
@property (nonatomic, weak) id<UserListInteractorOutputProtocol> output;
@property (nonatomic, strong) UserService *userService;
@property (nonatomic, strong) NSMutableArray<UserEntity *> *users;
@end

// UserListInteractor.m
@implementation UserListInteractor

- (instancetype)init {
    self = [super init];
    if (self) {
        _userService = [UserService sharedService];
        _users = [NSMutableArray array];
    }
    return self;
}

- (void)fetchUsers {
    [self.userService fetchUsersWithCompletion:^(NSArray<User *> *users, NSError *error) {
        if (error) {
            [self.output usersFetchFailed:error.localizedDescription];
        } else {
            [self.users removeAllObjects];
            
            for (User *user in users) {
                UserEntity *entity = [[UserEntity alloc] initWithUser:user];
                [self.users addObject:entity];
            }
            
            [self.output usersFetched:[self.users copy]];
        }
    }];
}

- (void)createUserWithUsername:(NSString *)username email:(NSString *)email {
    // 输入验证
    if (!username || username.length == 0) {
        [self.output userCreationFailed:@"Username is required"];
        return;
    }
    
    if (!email || email.length == 0) {
        [self.output userCreationFailed:@"Email is required"];
        return;
    }
    
    User *newUser = [[User alloc] init];
    newUser.username = username;
    newUser.email = email;
    
    [self.userService createUser:newUser completion:^(User *user, NSError *error) {
        if (error) {
            [self.output userCreationFailed:error.localizedDescription];
        } else {
            UserEntity *entity = [[UserEntity alloc] initWithUser:user];
            [self.users addObject:entity];
            [self.output userCreated:entity];
        }
    }];
}

- (void)deleteUserAtIndex:(NSInteger)index {
    if (index < 0 || index >= self.users.count) {
        [self.output userDeletionFailed:@"Invalid user index"];
        return;
    }
    
    UserEntity *entity = self.users[index];
    
    [self.userService deleteUserWithId:entity.userId completion:^(BOOL success, NSError *error) {
        if (error) {
            [self.output userDeletionFailed:error.localizedDescription];
        } else if (success) {
            [self.users removeObjectAtIndex:index];
            [self.output userDeleted:index];
        }
    }];
}

@end
```

**4. Presenter实现**

```objc
// UserListPresenter.h
@interface UserListPresenter : NSObject <UserListPresenterProtocol, UserListInteractorOutputProtocol>
@property (nonatomic, weak) id<UserListViewProtocol> view;
@property (nonatomic, strong) id<UserListInteractorProtocol> interactor;
@property (nonatomic, strong) id<UserListRouterProtocol> router;
@property (nonatomic, strong) NSArray<UserEntity *> *users;
@end

// UserListPresenter.m
@implementation UserListPresenter

- (instancetype)init {
    self = [super init];
    if (self) {
        _users = @[];
    }
    return self;
}

#pragma mark - UserListPresenterProtocol

- (void)viewDidLoad {
    [self.view showLoading:YES];
    [self.interactor fetchUsers];
}

- (void)refreshUsers {
    [self.interactor fetchUsers];
}

- (void)addUserWithUsername:(NSString *)username email:(NSString *)email {
    [self.view showLoading:YES];
    [self.interactor createUserWithUsername:username email:email];
}

- (void)deleteUserAtIndex:(NSInteger)index {
    [self.view showLoading:YES];
    [self.interactor deleteUserAtIndex:index];
}

- (void)selectUserAtIndex:(NSInteger)index {
    if (index >= 0 && index < self.users.count) {
        UserEntity *user = self.users[index];
        [self.router navigateToUserDetail:user];
    }
}

- (UserEntity *)userAtIndex:(NSInteger)index {
    if (index >= 0 && index < self.users.count) {
        return self.users[index];
    }
    return nil;
}

- (NSInteger)numberOfUsers {
    return self.users.count;
}

#pragma mark - UserListInteractorOutputProtocol

- (void)usersFetched:(NSArray<UserEntity *> *)users {
    self.users = users;
    [self.view showLoading:NO];
    [self.view showUsers:users];
}

- (void)usersFetchFailed:(NSString *)errorMessage {
    [self.view showLoading:NO];
    [self.view showError:errorMessage];
}

- (void)userCreated:(UserEntity *)user {
    self.users = [self.users arrayByAddingObject:user];
    [self.view showLoading:NO];
    [self.view showUserAdded];
}

- (void)userCreationFailed:(NSString *)errorMessage {
    [self.view showLoading:NO];
    [self.view showError:errorMessage];
}

- (void)userDeleted:(NSInteger)index {
    NSMutableArray *mutableUsers = [self.users mutableCopy];
    [mutableUsers removeObjectAtIndex:index];
    self.users = [mutableUsers copy];
    
    [self.view showLoading:NO];
    [self.view showUserDeleted];
}

- (void)userDeletionFailed:(NSString *)errorMessage {
    [self.view showLoading:NO];
    [self.view showError:errorMessage];
}

@end
```