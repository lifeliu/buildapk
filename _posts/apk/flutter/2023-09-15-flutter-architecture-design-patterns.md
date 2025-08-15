---
layout: post
title: "Flutter架构设计模式深度解析：构建可维护的大型应用"
date: 2023-09-15
categories: [Flutter, 架构设计, 设计模式]
tags: [Flutter, Architecture, Design Patterns, MVVM, BLoC, Clean Architecture, Provider, Riverpod]
author: "Flutter开发团队"
description: "深入探讨Flutter应用的架构设计模式，包括MVVM、BLoC、Clean Architecture等主流架构模式的实现与最佳实践，帮助开发者构建可维护、可扩展的大型Flutter应用。"
---

在Flutter应用开发中，选择合适的架构设计模式对于构建可维护、可扩展的大型应用至关重要。本文将深入探讨Flutter中的主流架构设计模式，包括MVVM、BLoC、Clean Architecture等，并提供详细的实现示例和最佳实践指导。

## 架构设计的重要性

### 为什么需要架构设计

良好的架构设计能够带来以下好处：

1. **代码可维护性**：清晰的代码结构便于理解和修改
2. **可扩展性**：易于添加新功能和模块
3. **可测试性**：各层职责分离，便于单元测试
4. **团队协作**：统一的架构标准提高团队开发效率
5. **代码复用**：模块化设计提高代码复用率

### Flutter架构设计原则

```dart
// lib/src/core/architecture/principles.dart

/// 架构设计原则
abstract class ArchitecturePrinciples {
  /// 单一职责原则 (Single Responsibility Principle)
  /// 每个类应该只有一个引起它变化的原因
  static const String singleResponsibility = 
      "Each class should have only one reason to change";
  
  /// 开闭原则 (Open-Closed Principle)
  /// 软件实体应该对扩展开放，对修改关闭
  static const String openClosed = 
      "Software entities should be open for extension, but closed for modification";
  
  /// 里氏替换原则 (Liskov Substitution Principle)
  /// 子类应该能够替换其基类
  static const String liskovSubstitution = 
      "Subtypes must be substitutable for their base types";
  
  /// 接口隔离原则 (Interface Segregation Principle)
  /// 客户端不应该依赖它不需要的接口
  static const String interfaceSegregation = 
      "Clients should not be forced to depend on interfaces they do not use";
  
  /// 依赖倒置原则 (Dependency Inversion Principle)
  /// 高层模块不应该依赖低层模块，两者都应该依赖抽象
  static const String dependencyInversion = 
      "High-level modules should not depend on low-level modules. Both should depend on abstractions";
}

/// 架构层级定义
enum ArchitectureLayer {
  presentation,  // 表现层
  domain,       // 业务逻辑层
  data,         // 数据层
  infrastructure, // 基础设施层
}

/// 架构组件接口
abstract class ArchitectureComponent {
  ArchitectureLayer get layer;
  String get name;
  List<String> get dependencies;
  
  void initialize();
  void dispose();
}

/// 架构验证器
class ArchitectureValidator {
  static bool validateDependencies(List<ArchitectureComponent> components) {
    // 验证依赖关系是否符合架构原则
    for (final component in components) {
      if (!_validateComponentDependencies(component, components)) {
        return false;
      }
    }
    return true;
  }
  
  static bool _validateComponentDependencies(
    ArchitectureComponent component,
    List<ArchitectureComponent> allComponents,
  ) {
    final componentLayer = component.layer;
    
    for (final dependency in component.dependencies) {
      final dependentComponent = allComponents.firstWhere(
        (c) => c.name == dependency,
        orElse: () => throw Exception('Dependency $dependency not found'),
      );
      
      // 检查是否违反了分层架构原则
      if (!_isValidLayerDependency(componentLayer, dependentComponent.layer)) {
        return false;
      }
    }
    
    return true;
  }
  
  static bool _isValidLayerDependency(
    ArchitectureLayer from,
    ArchitectureLayer to,
  ) {
    // 定义允许的依赖关系
    const allowedDependencies = {
      ArchitectureLayer.presentation: [
        ArchitectureLayer.domain,
        ArchitectureLayer.infrastructure,
      ],
      ArchitectureLayer.domain: [
        ArchitectureLayer.infrastructure,
      ],
      ArchitectureLayer.data: [
        ArchitectureLayer.domain,
        ArchitectureLayer.infrastructure,
      ],
      ArchitectureLayer.infrastructure: [],
    };
    
    return allowedDependencies[from]?.contains(to) ?? false;
  }
}
```

## MVVM架构模式

### MVVM模式概述

MVVM（Model-View-ViewModel）是一种将用户界面与业务逻辑分离的架构模式：

```dart
// lib/src/architecture/mvvm/base_view_model.dart

import 'package:flutter/foundation.dart';

/// ViewModel基类
abstract class BaseViewModel extends ChangeNotifier {
  bool _isLoading = false;
  String? _errorMessage;
  bool _isDisposed = false;
  
  /// 是否正在加载
  bool get isLoading => _isLoading;
  
  /// 错误信息
  String? get errorMessage => _errorMessage;
  
  /// 是否已释放
  bool get isDisposed => _isDisposed;
  
  /// 设置加载状态
  void setLoading(bool loading) {
    if (_isDisposed) return;
    
    _isLoading = loading;
    if (loading) {
      _errorMessage = null;
    }
    notifyListeners();
  }
  
  /// 设置错误信息
  void setError(String? error) {
    if (_isDisposed) return;
    
    _errorMessage = error;
    _isLoading = false;
    notifyListeners();
  }
  
  /// 清除错误
  void clearError() {
    if (_isDisposed) return;
    
    _errorMessage = null;
    notifyListeners();
  }
  
  /// 安全的通知监听器
  void safeNotifyListeners() {
    if (!_isDisposed) {
      notifyListeners();
    }
  }
  
  /// 执行异步操作
  Future<T?> executeAsync<T>(
    Future<T> Function() operation, {
    bool showLoading = true,
    String? errorPrefix,
  }) async {
    try {
      if (showLoading) setLoading(true);
      
      final result = await operation();
      
      if (showLoading) setLoading(false);
      return result;
    } catch (e) {
      final errorMsg = errorPrefix != null 
          ? '$errorPrefix: ${e.toString()}'
          : e.toString();
      setError(errorMsg);
      return null;
    }
  }
  
  @override
  void dispose() {
    _isDisposed = true;
    super.dispose();
  }
}

/// 用户ViewModel示例
class UserViewModel extends BaseViewModel {
  final UserRepository _userRepository;
  
  List<User> _users = [];
  User? _selectedUser;
  
  UserViewModel(this._userRepository);
  
  /// 用户列表
  List<User> get users => List.unmodifiable(_users);
  
  /// 选中的用户
  User? get selectedUser => _selectedUser;
  
  /// 加载用户列表
  Future<void> loadUsers() async {
    await executeAsync(
      () async {
        _users = await _userRepository.getUsers();
      },
      errorPrefix: '加载用户列表失败',
    );
  }
  
  /// 选择用户
  void selectUser(User user) {
    _selectedUser = user;
    safeNotifyListeners();
  }
  
  /// 添加用户
  Future<void> addUser(User user) async {
    await executeAsync(
      () async {
        final newUser = await _userRepository.createUser(user);
        _users.add(newUser);
      },
      errorPrefix: '添加用户失败',
    );
  }
  
  /// 更新用户
  Future<void> updateUser(User user) async {
    await executeAsync(
      () async {
        final updatedUser = await _userRepository.updateUser(user);
        final index = _users.indexWhere((u) => u.id == user.id);
        if (index != -1) {
          _users[index] = updatedUser;
        }
      },
      errorPrefix: '更新用户失败',
    );
  }
  
  /// 删除用户
  Future<void> deleteUser(String userId) async {
    await executeAsync(
      () async {
        await _userRepository.deleteUser(userId);
        _users.removeWhere((user) => user.id == userId);
        
        if (_selectedUser?.id == userId) {
          _selectedUser = null;
        }
      },
      errorPrefix: '删除用户失败',
    );
  }
  
  /// 搜索用户
  Future<void> searchUsers(String query) async {
    if (query.isEmpty) {
      await loadUsers();
      return;
    }
    
    await executeAsync(
      () async {
        _users = await _userRepository.searchUsers(query);
      },
      errorPrefix: '搜索用户失败',
    );
  }
}

/// MVVM View基类
abstract class MVVMView<T extends BaseViewModel> extends StatefulWidget {
  const MVVMView({Key? key}) : super(key: key);
  
  /// 创建ViewModel
  T createViewModel();
  
  /// 构建UI
  Widget buildView(BuildContext context, T viewModel);
  
  @override
  State<MVVMView<T>> createState() => _MVVMViewState<T>();
}

class _MVVMViewState<T extends BaseViewModel> extends State<MVVMView<T>> {
  late T _viewModel;
  
  @override
  void initState() {
    super.initState();
    _viewModel = widget.createViewModel();
  }
  
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider<T>.value(
      value: _viewModel,
      child: Consumer<T>(
        builder: (context, viewModel, child) {
          return widget.buildView(context, viewModel);
        },
      ),
    );
  }
  
  @override
  void dispose() {
    _viewModel.dispose();
    super.dispose();
  }
}

/// 用户列表页面
class UserListPage extends MVVMView<UserViewModel> {
  const UserListPage({Key? key}) : super(key: key);
  
  @override
  UserViewModel createViewModel() {
    return UserViewModel(GetIt.instance<UserRepository>());
  }
  
  @override
  Widget buildView(BuildContext context, UserViewModel viewModel) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('用户列表'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: viewModel.loadUsers,
          ),
        ],
      ),
      body: _buildBody(context, viewModel),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddUserDialog(context, viewModel),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  Widget _buildBody(BuildContext context, UserViewModel viewModel) {
    if (viewModel.isLoading) {
      return const Center(child: CircularProgressIndicator());
    }
    
    if (viewModel.errorMessage != null) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              viewModel.errorMessage!,
              style: const TextStyle(color: Colors.red),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: viewModel.loadUsers,
              child: const Text('重试'),
            ),
          ],
        ),
      );
    }
    
    if (viewModel.users.isEmpty) {
      return const Center(
        child: Text('暂无用户数据'),
      );
    }
    
    return ListView.builder(
      itemCount: viewModel.users.length,
      itemBuilder: (context, index) {
        final user = viewModel.users[index];
        return ListTile(
          leading: CircleAvatar(
            backgroundImage: NetworkImage(user.avatar),
          ),
          title: Text(user.name),
          subtitle: Text(user.email),
          trailing: PopupMenuButton<String>(
            onSelected: (value) {
              switch (value) {
                case 'edit':
                  _showEditUserDialog(context, viewModel, user);
                  break;
                case 'delete':
                  _showDeleteConfirmDialog(context, viewModel, user);
                  break;
              }
            },
            itemBuilder: (context) => [
              const PopupMenuItem(
                value: 'edit',
                child: Text('编辑'),
              ),
              const PopupMenuItem(
                value: 'delete',
                child: Text('删除'),
              ),
            ],
          ),
          onTap: () => viewModel.selectUser(user),
        );
      },
    );
  }
  
  void _showAddUserDialog(BuildContext context, UserViewModel viewModel) {
    // 显示添加用户对话框的实现
  }
  
  void _showEditUserDialog(BuildContext context, UserViewModel viewModel, User user) {
    // 显示编辑用户对话框的实现
  }
  
  void _showDeleteConfirmDialog(BuildContext context, UserViewModel viewModel, User user) {
    // 显示删除确认对话框的实现
  }
}
```

## BLoC架构模式

### BLoC模式实现

BLoC（Business Logic Component）模式使用Stream来管理状态：

```dart
// lib/src/architecture/bloc/base_bloc.dart

import 'dart:async';
import 'package:flutter_bloc/flutter_bloc.dart';

/// BLoC事件基类
abstract class BaseEvent {
  const BaseEvent();
}

/// BLoC状态基类
abstract class BaseState {
  const BaseState();
}

/// 加载状态
class LoadingState extends BaseState {
  const LoadingState();
}

/// 成功状态
class SuccessState<T> extends BaseState {
  final T data;
  
  const SuccessState(this.data);
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is SuccessState<T> && other.data == data;
  }
  
  @override
  int get hashCode => data.hashCode;
}

/// 错误状态
class ErrorState extends BaseState {
  final String message;
  final dynamic error;
  
  const ErrorState(this.message, [this.error]);
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is ErrorState && 
           other.message == message && 
           other.error == error;
  }
  
  @override
  int get hashCode => message.hashCode ^ error.hashCode;
}

/// 初始状态
class InitialState extends BaseState {
  const InitialState();
}

/// 基础BLoC
abstract class BaseBloc<Event extends BaseEvent, State extends BaseState>
    extends Bloc<Event, State> {
  
  BaseBloc(State initialState) : super(initialState);
  
  /// 处理异步操作
  Future<void> handleAsync<T>(
    Future<T> Function() operation,
    Emitter<State> emit,
    State Function(T data) onSuccess,
    {String? errorPrefix}
  ) async {
    try {
      emit(const LoadingState() as State);
      final result = await operation();
      emit(onSuccess(result));
    } catch (e) {
      final errorMessage = errorPrefix != null 
          ? '$errorPrefix: ${e.toString()}'
          : e.toString();
      emit(ErrorState(errorMessage, e) as State);
    }
  }
}

/// 用户BLoC事件
abstract class UserEvent extends BaseEvent {
  const UserEvent();
}

class LoadUsersEvent extends UserEvent {
  const LoadUsersEvent();
}

class AddUserEvent extends UserEvent {
  final User user;
  
  const AddUserEvent(this.user);
}

class UpdateUserEvent extends UserEvent {
  final User user;
  
  const UpdateUserEvent(this.user);
}

class DeleteUserEvent extends UserEvent {
  final String userId;
  
  const DeleteUserEvent(this.userId);
}

class SearchUsersEvent extends UserEvent {
  final String query;
  
  const SearchUsersEvent(this.query);
}

class SelectUserEvent extends UserEvent {
  final User user;
  
  const SelectUserEvent(this.user);
}

/// 用户BLoC状态
abstract class UserState extends BaseState {
  const UserState();
}

class UserInitialState extends UserState {
  const UserInitialState();
}

class UserLoadingState extends UserState {
  const UserLoadingState();
}

class UserLoadedState extends UserState {
  final List<User> users;
  final User? selectedUser;
  
  const UserLoadedState(this.users, [this.selectedUser]);
  
  UserLoadedState copyWith({
    List<User>? users,
    User? selectedUser,
  }) {
    return UserLoadedState(
      users ?? this.users,
      selectedUser ?? this.selectedUser,
    );
  }
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is UserLoadedState &&
           listEquals(other.users, users) &&
           other.selectedUser == selectedUser;
  }
  
  @override
  int get hashCode => users.hashCode ^ selectedUser.hashCode;
}

class UserErrorState extends UserState {
  final String message;
  
  const UserErrorState(this.message);
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is UserErrorState && other.message == message;
  }
  
  @override
  int get hashCode => message.hashCode;
}

/// 用户BLoC
class UserBloc extends BaseBloc<UserEvent, UserState> {
  final UserRepository _userRepository;
  
  UserBloc(this._userRepository) : super(const UserInitialState()) {
    on<LoadUsersEvent>(_onLoadUsers);
    on<AddUserEvent>(_onAddUser);
    on<UpdateUserEvent>(_onUpdateUser);
    on<DeleteUserEvent>(_onDeleteUser);
    on<SearchUsersEvent>(_onSearchUsers);
    on<SelectUserEvent>(_onSelectUser);
  }
  
  Future<void> _onLoadUsers(
    LoadUsersEvent event,
    Emitter<UserState> emit,
  ) async {
    await handleAsync(
      () => _userRepository.getUsers(),
      emit,
      (users) => UserLoadedState(users),
      errorPrefix: '加载用户列表失败',
    );
  }
  
  Future<void> _onAddUser(
    AddUserEvent event,
    Emitter<UserState> emit,
  ) async {
    final currentState = state;
    if (currentState is UserLoadedState) {
      await handleAsync(
        () => _userRepository.createUser(event.user),
        emit,
        (newUser) {
          final updatedUsers = List<User>.from(currentState.users)..add(newUser);
          return UserLoadedState(updatedUsers, currentState.selectedUser);
        },
        errorPrefix: '添加用户失败',
      );
    }
  }
  
  Future<void> _onUpdateUser(
    UpdateUserEvent event,
    Emitter<UserState> emit,
  ) async {
    final currentState = state;
    if (currentState is UserLoadedState) {
      await handleAsync(
        () => _userRepository.updateUser(event.user),
        emit,
        (updatedUser) {
          final updatedUsers = currentState.users.map((user) {
            return user.id == updatedUser.id ? updatedUser : user;
          }).toList();
          
          final selectedUser = currentState.selectedUser?.id == updatedUser.id
              ? updatedUser
              : currentState.selectedUser;
          
          return UserLoadedState(updatedUsers, selectedUser);
        },
        errorPrefix: '更新用户失败',
      );
    }
  }
  
  Future<void> _onDeleteUser(
    DeleteUserEvent event,
    Emitter<UserState> emit,
  ) async {
    final currentState = state;
    if (currentState is UserLoadedState) {
      await handleAsync(
        () => _userRepository.deleteUser(event.userId),
        emit,
        (_) {
          final updatedUsers = currentState.users
              .where((user) => user.id != event.userId)
              .toList();
          
          final selectedUser = currentState.selectedUser?.id == event.userId
              ? null
              : currentState.selectedUser;
          
          return UserLoadedState(updatedUsers, selectedUser);
        },
        errorPrefix: '删除用户失败',
      );
    }
  }
  
  Future<void> _onSearchUsers(
    SearchUsersEvent event,
    Emitter<UserState> emit,
  ) async {
    if (event.query.isEmpty) {
      add(const LoadUsersEvent());
      return;
    }
    
    await handleAsync(
      () => _userRepository.searchUsers(event.query),
      emit,
      (users) => UserLoadedState(users),
      errorPrefix: '搜索用户失败',
    );
  }
  
  void _onSelectUser(
    SelectUserEvent event,
    Emitter<UserState> emit,
  ) {
    final currentState = state;
    if (currentState is UserLoadedState) {
      emit(currentState.copyWith(selectedUser: event.user));
    }
  }
}

/// BLoC页面基类
abstract class BlocPage<B extends BlocBase<S>, S> extends StatelessWidget {
  const BlocPage({Key? key}) : super(key: key);
  
  /// 创建BLoC
  B createBloc(BuildContext context);
  
  /// 构建页面内容
  Widget buildPage(BuildContext context, S state);
  
  /// 监听状态变化
  void onStateChanged(BuildContext context, S state) {}
  
  @override
  Widget build(BuildContext context) {
    return BlocProvider<B>(
      create: createBloc,
      child: BlocConsumer<B, S>(
        listener: onStateChanged,
        builder: (context, state) => buildPage(context, state),
      ),
    );
  }
}

/// 用户列表页面（BLoC版本）
class UserListBlocPage extends BlocPage<UserBloc, UserState> {
  const UserListBlocPage({Key? key}) : super(key: key);
  
  @override
  UserBloc createBloc(BuildContext context) {
    return UserBloc(GetIt.instance<UserRepository>())
      ..add(const LoadUsersEvent());
  }
  
  @override
  Widget buildPage(BuildContext context, UserState state) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('用户列表 (BLoC)'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () {
              context.read<UserBloc>().add(const LoadUsersEvent());
            },
          ),
        ],
      ),
      body: _buildBody(context, state),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddUserDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  Widget _buildBody(BuildContext context, UserState state) {
    if (state is UserLoadingState) {
      return const Center(child: CircularProgressIndicator());
    }
    
    if (state is UserErrorState) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              state.message,
              style: const TextStyle(color: Colors.red),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () {
                context.read<UserBloc>().add(const LoadUsersEvent());
              },
              child: const Text('重试'),
            ),
          ],
        ),
      );
    }
    
    if (state is UserLoadedState) {
      if (state.users.isEmpty) {
        return const Center(
          child: Text('暂无用户数据'),
        );
      }
      
      return ListView.builder(
        itemCount: state.users.length,
        itemBuilder: (context, index) {
          final user = state.users[index];
          final isSelected = state.selectedUser?.id == user.id;
          
          return ListTile(
            leading: CircleAvatar(
              backgroundImage: NetworkImage(user.avatar),
            ),
            title: Text(user.name),
            subtitle: Text(user.email),
            selected: isSelected,
            trailing: PopupMenuButton<String>(
              onSelected: (value) {
                switch (value) {
                  case 'edit':
                    _showEditUserDialog(context, user);
                    break;
                  case 'delete':
                    _showDeleteConfirmDialog(context, user);
                    break;
                }
              },
              itemBuilder: (context) => [
                const PopupMenuItem(
                  value: 'edit',
                  child: Text('编辑'),
                ),
                const PopupMenuItem(
                  value: 'delete',
                  child: Text('删除'),
                ),
              ],
            ),
            onTap: () {
              context.read<UserBloc>().add(SelectUserEvent(user));
            },
          );
        },
      );
    }
    
    return const Center(
      child: Text('未知状态'),
    );
  }
  
  @override
  void onStateChanged(BuildContext context, UserState state) {
    if (state is UserErrorState) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(state.message),
          backgroundColor: Colors.red,
        ),
      );
    }
  }
  
  void _showAddUserDialog(BuildContext context) {
    // 显示添加用户对话框的实现
  }
  
  void _showEditUserDialog(BuildContext context, User user) {
    // 显示编辑用户对话框的实现
  }
  
  void _showDeleteConfirmDialog(BuildContext context, User user) {
    // 显示删除确认对话框的实现
  }
}
```

## Clean Architecture架构

### Clean Architecture实现

Clean Architecture强调依赖倒置和分层设计：

```dart
// lib/src/architecture/clean/domain/entities/user.dart

/// 用户实体
class User {
  final String id;
  final String name;
  final String email;
  final String avatar;
  final DateTime createdAt;
  final DateTime updatedAt;
  
  const User({
    required this.id,
    required this.name,
    required this.email,
    required this.avatar,
    required this.createdAt,
    required this.updatedAt,
  });
  
  User copyWith({
    String? id,
    String? name,
    String? email,
    String? avatar,
    DateTime? createdAt,
    DateTime? updatedAt,
  }) {
    return User(
      id: id ?? this.id,
      name: name ?? this.name,
      email: email ?? this.email,
      avatar: avatar ?? this.avatar,
      createdAt: createdAt ?? this.createdAt,
      updatedAt: updatedAt ?? this.updatedAt,
    );
  }
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is User &&
           other.id == id &&
           other.name == name &&
           other.email == email &&
           other.avatar == avatar &&
           other.createdAt == createdAt &&
           other.updatedAt == updatedAt;
  }
  
  @override
  int get hashCode {
    return id.hashCode ^
           name.hashCode ^
           email.hashCode ^
           avatar.hashCode ^
           createdAt.hashCode ^
           updatedAt.hashCode;
  }
  
  @override
  String toString() {
    return 'User(id: $id, name: $name, email: $email, avatar: $avatar, createdAt: $createdAt, updatedAt: $updatedAt)';
  }
}

// lib/src/architecture/clean/domain/repositories/user_repository.dart

/// 用户仓库接口
abstract class UserRepository {
  Future<List<User>> getUsers();
  Future<User> getUserById(String id);
  Future<User> createUser(User user);
  Future<User> updateUser(User user);
  Future<void> deleteUser(String id);
  Future<List<User>> searchUsers(String query);
}

// lib/src/architecture/clean/domain/usecases/get_users_usecase.dart

/// 获取用户列表用例
class GetUsersUseCase {
  final UserRepository _repository;
  
  GetUsersUseCase(this._repository);
  
  Future<Either<Failure, List<User>>> call() async {
    try {
      final users = await _repository.getUsers();
      return Right(users);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }
}

/// 创建用户用例
class CreateUserUseCase {
  final UserRepository _repository;
  
  CreateUserUseCase(this._repository);
  
  Future<Either<Failure, User>> call(CreateUserParams params) async {
    try {
      // 验证用户数据
      final validationResult = _validateUserData(params);
      if (validationResult != null) {
        return Left(ValidationFailure(validationResult));
      }
      
      final user = User(
        id: '', // 由服务器生成
        name: params.name,
        email: params.email,
        avatar: params.avatar,
        createdAt: DateTime.now(),
        updatedAt: DateTime.now(),
      );
      
      final createdUser = await _repository.createUser(user);
      return Right(createdUser);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }
  
  String? _validateUserData(CreateUserParams params) {
    if (params.name.trim().isEmpty) {
      return '用户名不能为空';
    }
    
    if (params.email.trim().isEmpty) {
      return '邮箱不能为空';
    }
    
    if (!_isValidEmail(params.email)) {
      return '邮箱格式不正确';
    }
    
    return null;
  }
  
  bool _isValidEmail(String email) {
    return RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(email);
  }
}

/// 创建用户参数
class CreateUserParams {
  final String name;
  final String email;
  final String avatar;
  
  const CreateUserParams({
    required this.name,
    required this.email,
    required this.avatar,
  });
}

/// 更新用户用例
class UpdateUserUseCase {
  final UserRepository _repository;
  
  UpdateUserUseCase(this._repository);
  
  Future<Either<Failure, User>> call(UpdateUserParams params) async {
    try {
      final user = params.user.copyWith(
        name: params.name ?? params.user.name,
        email: params.email ?? params.user.email,
        avatar: params.avatar ?? params.user.avatar,
        updatedAt: DateTime.now(),
      );
      
      final updatedUser = await _repository.updateUser(user);
      return Right(updatedUser);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }
}

/// 更新用户参数
class UpdateUserParams {
  final User user;
  final String? name;
  final String? email;
  final String? avatar;
  
  const UpdateUserParams({
    required this.user,
    this.name,
    this.email,
    this.avatar,
  });
}

/// 删除用户用例
class DeleteUserUseCase {
  final UserRepository _repository;
  
  DeleteUserUseCase(this._repository);
  
  Future<Either<Failure, void>> call(String userId) async {
    try {
      await _repository.deleteUser(userId);
      return const Right(null);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }
}

/// 搜索用户用例
class SearchUsersUseCase {
  final UserRepository _repository;
  
  SearchUsersUseCase(this._repository);
  
  Future<Either<Failure, List<User>>> call(String query) async {
    try {
      if (query.trim().isEmpty) {
        return const Right([]);
      }
      
      final users = await _repository.searchUsers(query);
      return Right(users);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }
}

// lib/src/architecture/clean/domain/failures/failure.dart

/// 失败基类
abstract class Failure {
  final String message;
  
  const Failure(this.message);
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Failure && other.message == message;
  }
  
  @override
  int get hashCode => message.hashCode;
}

/// 服务器失败
class ServerFailure extends Failure {
  const ServerFailure(String message) : super(message);
}

/// 网络失败
class NetworkFailure extends Failure {
  const NetworkFailure(String message) : super(message);
}

/// 验证失败
class ValidationFailure extends Failure {
  const ValidationFailure(String message) : super(message);
}

/// 缓存失败
class CacheFailure extends Failure {
  const CacheFailure(String message) : super(message);
}

// lib/src/architecture/clean/data/repositories/user_repository_impl.dart

/// 用户仓库实现
class UserRepositoryImpl implements UserRepository {
  final UserRemoteDataSource _remoteDataSource;
  final UserLocalDataSource _localDataSource;
  final NetworkInfo _networkInfo;
  
  UserRepositoryImpl(
    this._remoteDataSource,
    this._localDataSource,
    this._networkInfo,
  );
  
  @override
  Future<List<User>> getUsers() async {
    if (await _networkInfo.isConnected) {
      try {
        final remoteUsers = await _remoteDataSource.getUsers();
        await _localDataSource.cacheUsers(remoteUsers);
        return remoteUsers;
      } catch (e) {
        // 网络请求失败，尝试从缓存获取
        return await _localDataSource.getCachedUsers();
      }
    } else {
      return await _localDataSource.getCachedUsers();
    }
  }
  
  @override
  Future<User> getUserById(String id) async {
    if (await _networkInfo.isConnected) {
      try {
        final user = await _remoteDataSource.getUserById(id);
        await _localDataSource.cacheUser(user);
        return user;
      } catch (e) {
        return await _localDataSource.getCachedUser(id);
      }
    } else {
      return await _localDataSource.getCachedUser(id);
    }
  }
  
  @override
  Future<User> createUser(User user) async {
    if (await _networkInfo.isConnected) {
      final createdUser = await _remoteDataSource.createUser(user);
      await _localDataSource.cacheUser(createdUser);
      return createdUser;
    } else {
      throw const NetworkFailure('无网络连接，无法创建用户');
    }
  }
  
  @override
  Future<User> updateUser(User user) async {
    if (await _networkInfo.isConnected) {
      final updatedUser = await _remoteDataSource.updateUser(user);
      await _localDataSource.cacheUser(updatedUser);
      return updatedUser;
    } else {
      throw const NetworkFailure('无网络连接，无法更新用户');
    }
  }
  
  @override
  Future<void> deleteUser(String id) async {
    if (await _networkInfo.isConnected) {
      await _remoteDataSource.deleteUser(id);
      await _localDataSource.deleteCachedUser(id);
    } else {
      throw const NetworkFailure('无网络连接，无法删除用户');
    }
  }
  
  @override
  Future<List<User>> searchUsers(String query) async {
    if (await _networkInfo.isConnected) {
      return await _remoteDataSource.searchUsers(query);
    } else {
      return await _localDataSource.searchCachedUsers(query);
    }
  }
}

// lib/src/architecture/clean/data/datasources/user_remote_data_source.dart

/// 用户远程数据源接口
abstract class UserRemoteDataSource {
  Future<List<User>> getUsers();
  Future<User> getUserById(String id);
  Future<User> createUser(User user);
  Future<User> updateUser(User user);
  Future<void> deleteUser(String id);
  Future<List<User>> searchUsers(String query);
}

/// 用户远程数据源实现
class UserRemoteDataSourceImpl implements UserRemoteDataSource {
  final Dio _dio;
  
  UserRemoteDataSourceImpl(this._dio);
  
  @override
  Future<List<User>> getUsers() async {
    final response = await _dio.get('/users');
    
    if (response.statusCode == 200) {
      final List<dynamic> data = response.data['data'];
      return data.map((json) => UserModel.fromJson(json).toEntity()).toList();
    } else {
      throw ServerException('获取用户列表失败');
    }
  }
  
  @override
  Future<User> getUserById(String id) async {
    final response = await _dio.get('/users/$id');
    
    if (response.statusCode == 200) {
      return UserModel.fromJson(response.data['data']).toEntity();
    } else {
      throw ServerException('获取用户详情失败');
    }
  }
  
  @override
  Future<User> createUser(User user) async {
    final userModel = UserModel.fromEntity(user);
    final response = await _dio.post(
      '/users',
      data: userModel.toJson(),
    );
    
    if (response.statusCode == 201) {
      return UserModel.fromJson(response.data['data']).toEntity();
    } else {
      throw ServerException('创建用户失败');
    }
  }
  
  @override
  Future<User> updateUser(User user) async {
    final userModel = UserModel.fromEntity(user);
    final response = await _dio.put(
      '/users/${user.id}',
      data: userModel.toJson(),
    );
    
    if (response.statusCode == 200) {
      return UserModel.fromJson(response.data['data']).toEntity();
    } else {
      throw ServerException('更新用户失败');
    }
  }
  
  @override
  Future<void> deleteUser(String id) async {
    final response = await _dio.delete('/users/$id');
    
    if (response.statusCode != 200) {
      throw ServerException('删除用户失败');
    }
  }
  
  @override
  Future<List<User>> searchUsers(String query) async {
    final response = await _dio.get(
      '/users/search',
      queryParameters: {'q': query},
    );
    
    if (response.statusCode == 200) {
      final List<dynamic> data = response.data['data'];
      return data.map((json) => UserModel.fromJson(json).toEntity()).toList();
    } else {
      throw ServerException('搜索用户失败');
    }
  }
}

/// 服务器异常
class ServerException implements Exception {
  final String message;
  
  const ServerException(this.message);
}

// lib/src/architecture/clean/data/models/user_model.dart

/// 用户数据模型
class UserModel {
  final String id;
  final String name;
  final String email;
  final String avatar;
  final String createdAt;
  final String updatedAt;
  
  const UserModel({
    required this.id,
    required this.name,
    required this.email,
    required this.avatar,
    required this.createdAt,
    required this.updatedAt,
  });
  
  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'] as String,
      name: json['name'] as String,
      email: json['email'] as String,
      avatar: json['avatar'] as String,
      createdAt: json['created_at'] as String,
      updatedAt: json['updated_at'] as String,
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'email': email,
      'avatar': avatar,
      'created_at': createdAt,
      'updated_at': updatedAt,
    };
  }
  
  factory UserModel.fromEntity(User user) {
    return UserModel(
      id: user.id,
      name: user.name,
      email: user.email,
      avatar: user.avatar,
      createdAt: user.createdAt.toIso8601String(),
      updatedAt: user.updatedAt.toIso8601String(),
    );
  }
  
  User toEntity() {
    return User(
      id: id,
      name: name,
      email: email,
      avatar: avatar,
      createdAt: DateTime.parse(createdAt),
      updatedAt: DateTime.parse(updatedAt),
    );
  }
}
```

## Provider状态管理

### Provider架构实现

Provider是Flutter官方推荐的状态管理解决方案：

```dart
// lib/src/architecture/provider/providers/user_provider.dart

import 'package:flutter/foundation.dart';

/// 用户Provider
class UserProvider extends ChangeNotifier {
  final UserRepository _userRepository;
  
  UserProvider(this._userRepository);
  
  List<User> _users = [];
  User? _selectedUser;
  bool _isLoading = false;
  String? _errorMessage;
  
  /// 用户列表
  List<User> get users => List.unmodifiable(_users);
  
  /// 选中的用户
  User? get selectedUser => _selectedUser;
  
  /// 是否正在加载
  bool get isLoading => _isLoading;
  
  /// 错误信息
  String? get errorMessage => _errorMessage;
  
  /// 是否有错误
  bool get hasError => _errorMessage != null;
  
  /// 设置加载状态
  void _setLoading(bool loading) {
    _isLoading = loading;
    if (loading) {
      _errorMessage = null;
    }
    notifyListeners();
  }
  
  /// 设置错误信息
  void _setError(String? error) {
    _errorMessage = error;
    _isLoading = false;
    notifyListeners();
  }
  
  /// 加载用户列表
  Future<void> loadUsers() async {
    try {
      _setLoading(true);
      _users = await _userRepository.getUsers();
      _setLoading(false);
    } catch (e) {
      _setError('加载用户列表失败: ${e.toString()}');
    }
  }
  
  /// 添加用户
  Future<void> addUser(User user) async {
    try {
      _setLoading(true);
      final newUser = await _userRepository.createUser(user);
      _users.add(newUser);
      _setLoading(false);
    } catch (e) {
      _setError('添加用户失败: ${e.toString()}');
    }
  }
  
  /// 更新用户
  Future<void> updateUser(User user) async {
    try {
      _setLoading(true);
      final updatedUser = await _userRepository.updateUser(user);
      
      final index = _users.indexWhere((u) => u.id == user.id);
      if (index != -1) {
        _users[index] = updatedUser;
      }
      
      if (_selectedUser?.id == user.id) {
        _selectedUser = updatedUser;
      }
      
      _setLoading(false);
    } catch (e) {
      _setError('更新用户失败: ${e.toString()}');
    }
  }
  
  /// 删除用户
  Future<void> deleteUser(String userId) async {
    try {
      _setLoading(true);
      await _userRepository.deleteUser(userId);
      
      _users.removeWhere((user) => user.id == userId);
      
      if (_selectedUser?.id == userId) {
        _selectedUser = null;
      }
      
      _setLoading(false);
    } catch (e) {
      _setError('删除用户失败: ${e.toString()}');
    }
  }
  
  /// 搜索用户
  Future<void> searchUsers(String query) async {
    try {
      _setLoading(true);
      
      if (query.isEmpty) {
        await loadUsers();
      } else {
        _users = await _userRepository.searchUsers(query);
      }
      
      _setLoading(false);
    } catch (e) {
      _setError('搜索用户失败: ${e.toString()}');
    }
  }
  
  /// 选择用户
  void selectUser(User user) {
    _selectedUser = user;
    notifyListeners();
  }
  
  /// 清除选择
  void clearSelection() {
    _selectedUser = null;
    notifyListeners();
  }
  
  /// 清除错误
  void clearError() {
    _errorMessage = null;
    notifyListeners();
  }
  
  /// 刷新数据
  Future<void> refresh() async {
    await loadUsers();
  }
}

/// 应用状态Provider
class AppStateProvider extends ChangeNotifier {
  ThemeMode _themeMode = ThemeMode.system;
  Locale _locale = const Locale('zh', 'CN');
  bool _isFirstLaunch = true;
  
  /// 主题模式
  ThemeMode get themeMode => _themeMode;
  
  /// 当前语言
  Locale get locale => _locale;
  
  /// 是否首次启动
  bool get isFirstLaunch => _isFirstLaunch;
  
  /// 设置主题模式
  void setThemeMode(ThemeMode mode) {
    _themeMode = mode;
    notifyListeners();
  }
  
  /// 设置语言
  void setLocale(Locale locale) {
    _locale = locale;
    notifyListeners();
  }
  
  /// 标记已完成首次启动
  void completeFirstLaunch() {
    _isFirstLaunch = false;
    notifyListeners();
  }
}

/// Provider配置
class ProviderConfig {
  static List<ChangeNotifierProvider> getProviders() {
    return [
      ChangeNotifierProvider<AppStateProvider>(
        create: (_) => AppStateProvider(),
      ),
      ChangeNotifierProvider<UserProvider>(
        create: (_) => UserProvider(GetIt.instance<UserRepository>()),
      ),
    ];
  }
  
  static List<ProxyProvider> getProxyProviders() {
    return [
      ProxyProvider<AppStateProvider, ThemeData>(
        update: (context, appState, _) {
          return appState.themeMode == ThemeMode.dark
              ? ThemeData.dark()
              : ThemeData.light();
        },
      ),
    ];
  }
}

/// Provider页面基类
abstract class ProviderPage<T extends ChangeNotifier> extends StatelessWidget {
  const ProviderPage({Key? key}) : super(key: key);
  
  /// 构建页面内容
  Widget buildPage(BuildContext context, T provider);
  
  @override
  Widget build(BuildContext context) {
    return Consumer<T>(
      builder: (context, provider, child) {
        return buildPage(context, provider);
      },
    );
  }
}

/// 用户列表页面（Provider版本）
class UserListProviderPage extends ProviderPage<UserProvider> {
  const UserListProviderPage({Key? key}) : super(key: key);
  
  @override
  Widget buildPage(BuildContext context, UserProvider provider) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('用户列表 (Provider)'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: provider.refresh,
          ),
        ],
      ),
      body: _buildBody(context, provider),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddUserDialog(context, provider),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  Widget _buildBody(BuildContext context, UserProvider provider) {
    if (provider.isLoading) {
      return const Center(child: CircularProgressIndicator());
    }
    
    if (provider.hasError) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              provider.errorMessage!,
              style: const TextStyle(color: Colors.red),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () {
                provider.clearError();
                provider.loadUsers();
              },
              child: const Text('重试'),
            ),
          ],
        ),
      );
    }
    
    if (provider.users.isEmpty) {
      return const Center(
        child: Text('暂无用户数据'),
      );
    }
    
    return ListView.builder(
      itemCount: provider.users.length,
      itemBuilder: (context, index) {
        final user = provider.users[index];
        final isSelected = provider.selectedUser?.id == user.id;
        
        return ListTile(
          leading: CircleAvatar(
            backgroundImage: NetworkImage(user.avatar),
          ),
          title: Text(user.name),
          subtitle: Text(user.email),
          selected: isSelected,
          trailing: PopupMenuButton<String>(
            onSelected: (value) {
              switch (value) {
                case 'edit':
                  _showEditUserDialog(context, provider, user);
                  break;
                case 'delete':
                  _showDeleteConfirmDialog(context, provider, user);
                  break;
              }
            },
            itemBuilder: (context) => [
              const PopupMenuItem(
                value: 'edit',
                child: Text('编辑'),
              ),
              const PopupMenuItem(
                value: 'delete',
                child: Text('删除'),
              ),
            ],
          ),
          onTap: () => provider.selectUser(user),
        );
      },
    );
  }
  
  void _showAddUserDialog(BuildContext context, UserProvider provider) {
    // 显示添加用户对话框的实现
  }
  
  void _showEditUserDialog(BuildContext context, UserProvider provider, User user) {
    // 显示编辑用户对话框的实现
  }
  
  void _showDeleteConfirmDialog(BuildContext context, UserProvider provider, User user) {
    // 显示删除确认对话框的实现
  }
}
```

## 架构模式对比与选择

### 各架构模式对比

| 特性 | MVVM | BLoC | Clean Architecture | Provider |
|------|------|------|-------------------|----------|
| 学习曲线 | 中等 | 较高 | 高 | 低 |
| 代码复杂度 | 中等 | 高 | 高 | 低 |
| 可测试性 | 好 | 很好 | 很好 | 好 |
| 状态管理 | ChangeNotifier | Stream | 灵活 | ChangeNotifier |
| 性能 | 好 | 很好 | 好 | 好 |
| 团队协作 | 好 | 很好 | 很好 | 中等 |
| 维护性 | 好 | 很好 | 很好 | 中等 |
| 扩展性 | 好 | 很好 | 很好 | 中等 |

### 架构选择指南

```dart
// lib/src/architecture/selection/architecture_selector.dart

/// 架构选择器
class ArchitectureSelector {
  /// 根据项目特征推荐架构
  static ArchitectureRecommendation recommend(ProjectCharacteristics characteristics) {
    final score = _calculateArchitectureScores(characteristics);
    
    final bestArchitecture = score.entries
        .reduce((a, b) => a.value > b.value ? a : b)
        .key;
    
    return ArchitectureRecommendation(
      recommendedArchitecture: bestArchitecture,
      scores: score,
      reasons: _getRecommendationReasons(bestArchitecture, characteristics),
    );
  }
  
  static Map<ArchitectureType, double> _calculateArchitectureScores(
    ProjectCharacteristics characteristics,
  ) {
    final scores = <ArchitectureType, double>{};
    
    // MVVM评分
    double mvvmScore = 0.0;
    if (characteristics.teamSize <= 5) mvvmScore += 2.0;
    if (characteristics.complexity == ProjectComplexity.medium) mvvmScore += 2.0;
    if (characteristics.testingRequirement == TestingLevel.medium) mvvmScore += 1.5;
    if (characteristics.developmentSpeed == DevelopmentSpeed.fast) mvvmScore += 2.0;
    scores[ArchitectureType.mvvm] = mvvmScore;
    
    // BLoC评分
    double blocScore = 0.0;
    if (characteristics.teamSize > 5) blocScore += 2.0;
    if (characteristics.complexity == ProjectComplexity.high) blocScore += 2.5;
    if (characteristics.testingRequirement == TestingLevel.high) blocScore += 2.0;
    if (characteristics.stateComplexity == StateComplexity.high) blocScore += 2.0;
    scores[ArchitectureType.bloc] = blocScore;
    
    // Clean Architecture评分
    double cleanScore = 0.0;
    if (characteristics.teamSize > 8) cleanScore += 2.0;
    if (characteristics.complexity == ProjectComplexity.high) cleanScore += 2.5;
    if (characteristics.maintainabilityRequirement == MaintainabilityLevel.high) cleanScore += 2.0;
    if (characteristics.scalabilityRequirement == ScalabilityLevel.high) cleanScore += 2.0;
    scores[ArchitectureType.clean] = cleanScore;
    
    // Provider评分
    double providerScore = 0.0;
    if (characteristics.teamSize <= 3) providerScore += 2.0;
    if (characteristics.complexity == ProjectComplexity.low) providerScore += 2.0;
    if (characteristics.developmentSpeed == DevelopmentSpeed.fast) providerScore += 2.5;
    if (characteristics.learningCurve == LearningCurve.easy) providerScore += 2.0;
    scores[ArchitectureType.provider] = providerScore;
    
    return scores;
  }
  
  static List<String> _getRecommendationReasons(
    ArchitectureType architecture,
    ProjectCharacteristics characteristics,
  ) {
    switch (architecture) {
      case ArchitectureType.mvvm:
        return [
          '适合中等规模团队开发',
          '学习曲线适中，易于上手',
          '代码结构清晰，便于维护',
          '支持数据绑定，开发效率高',
        ];
      case ArchitectureType.bloc:
        return [
          '强大的状态管理能力',
          '优秀的可测试性',
          '适合复杂的业务逻辑',
          '团队协作效果好',
        ];
      case ArchitectureType.clean:
        return [
          '高度模块化，易于扩展',
          '依赖倒置，便于测试',
          '适合大型项目',
          '长期维护成本低',
        ];
      case ArchitectureType.provider:
        return [
          '官方推荐，生态完善',
          '学习成本低',
          '开发速度快',
          '适合小型项目',
        ];
    }
  }
}

/// 项目特征
class ProjectCharacteristics {
  final int teamSize;
  final ProjectComplexity complexity;
  final StateComplexity stateComplexity;
  final TestingLevel testingRequirement;
  final MaintainabilityLevel maintainabilityRequirement;
  final ScalabilityLevel scalabilityRequirement;
  final DevelopmentSpeed developmentSpeed;
  final LearningCurve learningCurve;
  
  const ProjectCharacteristics({
    required this.teamSize,
    required this.complexity,
    required this.stateComplexity,
    required this.testingRequirement,
    required this.maintainabilityRequirement,
    required this.scalabilityRequirement,
    required this.developmentSpeed,
    required this.learningCurve,
  });
}

/// 架构推荐结果
class ArchitectureRecommendation {
  final ArchitectureType recommendedArchitecture;
  final Map<ArchitectureType, double> scores;
  final List<String> reasons;
  
  const ArchitectureRecommendation({
    required this.recommendedArchitecture,
    required this.scores,
    required this.reasons,
  });
}

/// 架构类型枚举
enum ArchitectureType {
  mvvm,
  bloc,
  clean,
  provider,
}

/// 项目复杂度
enum ProjectComplexity {
  low,
  medium,
  high,
}

/// 状态复杂度
enum StateComplexity {
  low,
  medium,
  high,
}

/// 测试要求级别
enum TestingLevel {
  low,
  medium,
  high,
}

/// 可维护性要求级别
enum MaintainabilityLevel {
  low,
  medium,
  high,
}

/// 可扩展性要求级别
enum ScalabilityLevel {
  low,
  medium,
  high,
}

/// 开发速度要求
enum DevelopmentSpeed {
  slow,
  medium,
  fast,
}

/// 学习曲线
enum LearningCurve {
  easy,
  medium,
  hard,
}
```

## Riverpod状态管理

### Riverpod架构实现

Riverpod是Provider的进化版本，提供了更好的类型安全和性能：

```dart
// lib/src/architecture/riverpod/providers/user_providers.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';

/// 用户仓库Provider
final userRepositoryProvider = Provider<UserRepository>((ref) {
  return GetIt.instance<UserRepository>();
});

/// 用户列表Provider
final userListProvider = StateNotifierProvider<UserListNotifier, AsyncValue<List<User>>>((ref) {
  final repository = ref.watch(userRepositoryProvider);
  return UserListNotifier(repository);
});

/// 选中用户Provider
final selectedUserProvider = StateProvider<User?>((ref) => null);

/// 搜索查询Provider
final searchQueryProvider = StateProvider<String>((ref) => '');

/// 过滤后的用户列表Provider
final filteredUserListProvider = Provider<AsyncValue<List<User>>>((ref) {
  final userList = ref.watch(userListProvider);
  final searchQuery = ref.watch(searchQueryProvider);
  
  return userList.when(
    data: (users) {
      if (searchQuery.isEmpty) {
        return AsyncValue.data(users);
      }
      
      final filteredUsers = users.where((user) {
        return user.name.toLowerCase().contains(searchQuery.toLowerCase()) ||
               user.email.toLowerCase().contains(searchQuery.toLowerCase());
      }).toList();
      
      return AsyncValue.data(filteredUsers);
    },
    loading: () => const AsyncValue.loading(),
    error: (error, stackTrace) => AsyncValue.error(error, stackTrace),
  );
});

/// 用户统计Provider
final userStatsProvider = Provider<UserStats>((ref) {
  final userList = ref.watch(userListProvider);
  
  return userList.when(
    data: (users) => UserStats(
      totalUsers: users.length,
      activeUsers: users.where((u) => u.isActive).length,
      newUsersToday: users.where((u) => _isToday(u.createdAt)).length,
    ),
    loading: () => const UserStats(totalUsers: 0, activeUsers: 0, newUsersToday: 0),
    error: (_, __) => const UserStats(totalUsers: 0, activeUsers: 0, newUsersToday: 0),
  );
});

bool _isToday(DateTime date) {
  final now = DateTime.now();
  return date.year == now.year && date.month == now.month && date.day == now.day;
}

/// 用户列表状态管理器
class UserListNotifier extends StateNotifier<AsyncValue<List<User>>> {
  final UserRepository _repository;
  
  UserListNotifier(this._repository) : super(const AsyncValue.loading()) {
    loadUsers();
  }
  
  /// 加载用户列表
  Future<void> loadUsers() async {
    state = const AsyncValue.loading();
    
    try {
      final users = await _repository.getUsers();
      state = AsyncValue.data(users);
    } catch (error, stackTrace) {
      state = AsyncValue.error(error, stackTrace);
    }
  }
  
  /// 添加用户
  Future<void> addUser(User user) async {
    try {
      final newUser = await _repository.createUser(user);
      
      state = state.when(
        data: (users) => AsyncValue.data([...users, newUser]),
        loading: () => state,
        error: (error, stackTrace) => state,
      );
    } catch (error, stackTrace) {
      state = AsyncValue.error(error, stackTrace);
    }
  }
  
  /// 更新用户
  Future<void> updateUser(User user) async {
    try {
      final updatedUser = await _repository.updateUser(user);
      
      state = state.when(
        data: (users) {
          final updatedUsers = users.map((u) {
            return u.id == updatedUser.id ? updatedUser : u;
          }).toList();
          return AsyncValue.data(updatedUsers);
        },
        loading: () => state,
        error: (error, stackTrace) => state,
      );
    } catch (error, stackTrace) {
      state = AsyncValue.error(error, stackTrace);
    }
  }
  
  /// 删除用户
  Future<void> deleteUser(String userId) async {
    try {
      await _repository.deleteUser(userId);
      
      state = state.when(
        data: (users) {
          final updatedUsers = users.where((u) => u.id != userId).toList();
          return AsyncValue.data(updatedUsers);
        },
        loading: () => state,
        error: (error, stackTrace) => state,
      );
    } catch (error, stackTrace) {
      state = AsyncValue.error(error, stackTrace);
    }
  }
  
  /// 刷新数据
  Future<void> refresh() async {
    await loadUsers();
  }
}

/// 用户统计数据
class UserStats {
  final int totalUsers;
  final int activeUsers;
  final int newUsersToday;
  
  const UserStats({
    required this.totalUsers,
    required this.activeUsers,
    required this.newUsersToday,
  });
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is UserStats &&
           other.totalUsers == totalUsers &&
           other.activeUsers == activeUsers &&
           other.newUsersToday == newUsersToday;
  }
  
  @override
  int get hashCode {
    return totalUsers.hashCode ^
           activeUsers.hashCode ^
           newUsersToday.hashCode;
  }
}

/// 用户列表页面（Riverpod版本）
class UserListRiverpodPage extends ConsumerWidget {
  const UserListRiverpodPage({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userList = ref.watch(filteredUserListProvider);
    final selectedUser = ref.watch(selectedUserProvider);
    final userStats = ref.watch(userStatsProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('用户列表 (Riverpod)'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () {
              ref.read(userListProvider.notifier).refresh();
            },
          ),
        ],
        bottom: PreferredSize(
          preferredSize: const Size.fromHeight(60),
          child: _buildSearchBar(ref),
        ),
      ),
      body: Column(
        children: [
          _buildStatsCard(userStats),
          Expanded(
            child: _buildUserList(context, ref, userList, selectedUser),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddUserDialog(context, ref),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  Widget _buildSearchBar(WidgetRef ref) {
    return Padding(
      padding: const EdgeInsets.all(8.0),
      child: TextField(
        decoration: const InputDecoration(
          hintText: '搜索用户...',
          prefixIcon: Icon(Icons.search),
          border: OutlineInputBorder(),
        ),
        onChanged: (value) {
          ref.read(searchQueryProvider.notifier).state = value;
        },
      ),
    );
  }
  
  Widget _buildStatsCard(UserStats stats) {
    return Card(
      margin: const EdgeInsets.all(8.0),
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Row(
          mainAxisAlignment: MainAxisAlignment.spaceAround,
          children: [
            _buildStatItem('总用户', stats.totalUsers.toString()),
            _buildStatItem('活跃用户', stats.activeUsers.toString()),
            _buildStatItem('今日新增', stats.newUsersToday.toString()),
          ],
        ),
      ),
    );
  }
  
  Widget _buildStatItem(String label, String value) {
    return Column(
      children: [
        Text(
          value,
          style: const TextStyle(
            fontSize: 24,
            fontWeight: FontWeight.bold,
          ),
        ),
        Text(label),
      ],
    );
  }
  
  Widget _buildUserList(
    BuildContext context,
    WidgetRef ref,
    AsyncValue<List<User>> userList,
    User? selectedUser,
  ) {
    return userList.when(
      data: (users) {
        if (users.isEmpty) {
          return const Center(
            child: Text('暂无用户数据'),
          );
        }
        
        return ListView.builder(
          itemCount: users.length,
          itemBuilder: (context, index) {
            final user = users[index];
            final isSelected = selectedUser?.id == user.id;
            
            return ListTile(
              leading: CircleAvatar(
                backgroundImage: NetworkImage(user.avatar),
              ),
              title: Text(user.name),
              subtitle: Text(user.email),
              selected: isSelected,
              trailing: PopupMenuButton<String>(
                onSelected: (value) {
                  switch (value) {
                    case 'edit':
                      _showEditUserDialog(context, ref, user);
                      break;
                    case 'delete':
                      _showDeleteConfirmDialog(context, ref, user);
                      break;
                  }
                },
                itemBuilder: (context) => [
                  const PopupMenuItem(
                    value: 'edit',
                    child: Text('编辑'),
                  ),
                  const PopupMenuItem(
                    value: 'delete',
                    child: Text('删除'),
                  ),
                ],
              ),
              onTap: () {
                ref.read(selectedUserProvider.notifier).state = user;
              },
            );
          },
        );
      },
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (error, stackTrace) => Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              '加载失败: ${error.toString()}',
              style: const TextStyle(color: Colors.red),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () {
                ref.read(userListProvider.notifier).refresh();
              },
              child: const Text('重试'),
            ),
          ],
        ),
      ),
    );
  }
  
  void _showAddUserDialog(BuildContext context, WidgetRef ref) {
    // 显示添加用户对话框的实现
  }
  
  void _showEditUserDialog(BuildContext context, WidgetRef ref, User user) {
    // 显示编辑用户对话框的实现
  }
  
  void _showDeleteConfirmDialog(BuildContext context, WidgetRef ref, User user) {
    // 显示删除确认对话框的实现
  }
}
```

## 架构测试策略

### 单元测试

```dart
// test/architecture/mvvm/user_view_model_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:mockito/annotations.dart';

@GenerateMocks([UserRepository])
void main() {
  group('UserViewModel Tests', () {
    late UserViewModel viewModel;
    late MockUserRepository mockRepository;
    
    setUp(() {
      mockRepository = MockUserRepository();
      viewModel = UserViewModel(mockRepository);
    });
    
    tearDown(() {
      viewModel.dispose();
    });
    
    test('初始状态应该是空的', () {
      expect(viewModel.users, isEmpty);
      expect(viewModel.isLoading, false);
      expect(viewModel.error, isNull);
    });
    
    test('加载用户成功', () async {
      // Arrange
      final users = [
        User(id: '1', name: 'Test User 1', email: 'test1@example.com'),
        User(id: '2', name: 'Test User 2', email: 'test2@example.com'),
      ];
      when(mockRepository.getUsers()).thenAnswer((_) async => users);
      
      // Act
      await viewModel.loadUsers();
      
      // Assert
      expect(viewModel.users, equals(users));
      expect(viewModel.isLoading, false);
      expect(viewModel.error, isNull);
      verify(mockRepository.getUsers()).called(1);
    });
    
    test('加载用户失败', () async {
      // Arrange
      final exception = Exception('Network error');
      when(mockRepository.getUsers()).thenThrow(exception);
      
      // Act
      await viewModel.loadUsers();
      
      // Assert
      expect(viewModel.users, isEmpty);
      expect(viewModel.isLoading, false);
      expect(viewModel.error, equals(exception.toString()));
      verify(mockRepository.getUsers()).called(1);
    });
    
    test('添加用户成功', () async {
      // Arrange
      final newUser = User(id: '3', name: 'New User', email: 'new@example.com');
      when(mockRepository.createUser(any)).thenAnswer((_) async => newUser);
      
      // Act
      await viewModel.addUser(newUser);
      
      // Assert
      expect(viewModel.users, contains(newUser));
      verify(mockRepository.createUser(newUser)).called(1);
    });
    
    test('搜索用户功能', () {
      // Arrange
      viewModel.users.addAll([
        User(id: '1', name: 'John Doe', email: 'john@example.com'),
        User(id: '2', name: 'Jane Smith', email: 'jane@example.com'),
        User(id: '3', name: 'Bob Johnson', email: 'bob@example.com'),
      ]);
      
      // Act
      viewModel.searchQuery = 'john';
      
      // Assert
      expect(viewModel.filteredUsers.length, equals(2));
      expect(viewModel.filteredUsers.any((u) => u.name.contains('John')), true);
      expect(viewModel.filteredUsers.any((u) => u.email.contains('john')), true);
    });
  });
}
```

### BLoC测试

```dart
// test/architecture/bloc/user_bloc_test.dart

import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';

void main() {
  group('UserBloc Tests', () {
    late UserBloc userBloc;
    late MockUserRepository mockRepository;
    
    setUp(() {
      mockRepository = MockUserRepository();
      userBloc = UserBloc(mockRepository);
    });
    
    tearDown(() {
      userBloc.close();
    });
    
    test('初始状态应该是UserInitial', () {
      expect(userBloc.state, equals(UserInitial()));
    });
    
    blocTest<UserBloc, UserState>(
      '加载用户成功时发出UserLoading然后UserLoaded',
      build: () {
        final users = [
          User(id: '1', name: 'Test User', email: 'test@example.com'),
        ];
        when(mockRepository.getUsers()).thenAnswer((_) async => users);
        return userBloc;
      },
      act: (bloc) => bloc.add(LoadUsers()),
      expect: () => [
        UserLoading(),
        UserLoaded([
          User(id: '1', name: 'Test User', email: 'test@example.com'),
        ]),
      ],
      verify: (_) {
        verify(mockRepository.getUsers()).called(1);
      },
    );
    
    blocTest<UserBloc, UserState>(
      '加载用户失败时发出UserLoading然后UserError',
      build: () {
        when(mockRepository.getUsers()).thenThrow(Exception('Network error'));
        return userBloc;
      },
      act: (bloc) => bloc.add(LoadUsers()),
      expect: () => [
        UserLoading(),
        UserError('Exception: Network error'),
      ],
      verify: (_) {
        verify(mockRepository.getUsers()).called(1);
      },
    );
    
    blocTest<UserBloc, UserState>(
      '添加用户成功',
      build: () {
        final existingUsers = [
          User(id: '1', name: 'Existing User', email: 'existing@example.com'),
        ];
        final newUser = User(id: '2', name: 'New User', email: 'new@example.com');
        
        when(mockRepository.createUser(any)).thenAnswer((_) async => newUser);
        
        userBloc.emit(UserLoaded(existingUsers));
        return userBloc;
      },
      act: (bloc) => bloc.add(AddUser(
        User(id: '2', name: 'New User', email: 'new@example.com'),
      )),
      expect: () => [
        UserLoaded([
          User(id: '1', name: 'Existing User', email: 'existing@example.com'),
          User(id: '2', name: 'New User', email: 'new@example.com'),
        ]),
      ],
    );
  });
}
```

### 集成测试

```dart
// integration_test/architecture_integration_test.dart

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('架构集成测试', () {
    testWidgets('MVVM架构用户列表页面测试', (WidgetTester tester) async {
      // 启动应用
      await tester.pumpWidget(MyApp());
      
      // 等待页面加载
      await tester.pumpAndSettle();
      
      // 验证页面标题
      expect(find.text('用户列表 (MVVM)'), findsOneWidget);
      
      // 验证加载指示器
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      
      // 等待数据加载完成
      await tester.pumpAndSettle(Duration(seconds: 3));
      
      // 验证用户列表
      expect(find.byType(ListTile), findsWidgets);
      
      // 测试搜索功能
      await tester.enterText(find.byType(TextField), 'test');
      await tester.pumpAndSettle();
      
      // 验证搜索结果
      expect(find.textContaining('test'), findsWidgets);
      
      // 测试添加用户
      await tester.tap(find.byType(FloatingActionButton));
      await tester.pumpAndSettle();
      
      // 验证添加用户对话框
      expect(find.byType(Dialog), findsOneWidget);
    });
    
    testWidgets('BLoC架构状态管理测试', (WidgetTester tester) async {
      await tester.pumpWidget(MyApp());
      
      // 切换到BLoC页面
      await tester.tap(find.text('BLoC'));
      await tester.pumpAndSettle();
      
      // 验证BLoC页面
      expect(find.text('用户列表 (BLoC)'), findsOneWidget);
      
      // 测试刷新功能
      await tester.tap(find.byIcon(Icons.refresh));
      await tester.pumpAndSettle();
      
      // 验证加载状态
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      
      // 等待加载完成
      await tester.pumpAndSettle(Duration(seconds: 3));
      
      // 验证数据加载成功
      expect(find.byType(ListTile), findsWidgets);
    });
    
    testWidgets('Provider架构状态共享测试', (WidgetTester tester) async {
      await tester.pumpWidget(MyApp());
      
      // 切换到Provider页面
      await tester.tap(find.text('Provider'));
      await tester.pumpAndSettle();
      
      // 选择一个用户
      await tester.tap(find.byType(ListTile).first);
      await tester.pumpAndSettle();
      
      // 验证用户选择状态
      expect(find.byType(ListTile).first, findsOneWidget);
      
      // 切换到详情页面
      await tester.tap(find.text('详情'));
      await tester.pumpAndSettle();
      
      // 验证选中用户信息在详情页面显示
      expect(find.textContaining('用户详情'), findsOneWidget);
    });
  });
}
```

## 架构最佳实践

### 1. 代码组织结构

```
lib/
├── src/
│   ├── core/                    # 核心功能
│   │   ├── error/               # 错误处理
│   │   ├── network/             # 网络层
│   │   ├── storage/             # 存储层
│   │   └── utils/               # 工具类
│   ├── features/                # 功能模块
│   │   ├── user/                # 用户模块
│   │   │   ├── data/            # 数据层
│   │   │   │   ├── datasources/ # 数据源
│   │   │   │   ├── models/      # 数据模型
│   │   │   │   └── repositories/ # 仓库实现
│   │   │   ├── domain/          # 领域层
│   │   │   │   ├── entities/    # 实体
│   │   │   │   ├── repositories/ # 仓库接口
│   │   │   │   └── usecases/    # 用例
│   │   │   └── presentation/    # 表现层
│   │   │       ├── bloc/        # BLoC
│   │   │       ├── pages/       # 页面
│   │   │       └── widgets/     # 组件
│   │   └── auth/                # 认证模块
│   └── shared/                  # 共享组件
│       ├── widgets/             # 通用组件
│       ├── themes/              # 主题
│       └── constants/           # 常量
└── main.dart
```

### 2. 依赖注入最佳实践

```dart
// lib/src/core/di/injection_container.dart

import 'package:get_it/get_it.dart';
import 'package:dio/dio.dart';
import 'package:shared_preferences/shared_preferences.dart';

final sl = GetIt.instance;

class InjectionContainer {
  static Future<void> init() async {
    // 外部依赖
    final sharedPreferences = await SharedPreferences.getInstance();
    sl.registerLazySingleton(() => sharedPreferences);
    
    final dio = Dio();
    sl.registerLazySingleton(() => dio);
    
    // 核心服务
    sl.registerLazySingleton<NetworkService>(
      () => NetworkServiceImpl(sl()),
    );
    
    sl.registerLazySingleton<StorageService>(
      () => StorageServiceImpl(sl()),
    );
    
    // 数据源
    sl.registerLazySingleton<UserRemoteDataSource>(
      () => UserRemoteDataSourceImpl(sl()),
    );
    
    sl.registerLazySingleton<UserLocalDataSource>(
      () => UserLocalDataSourceImpl(sl()),
    );
    
    // 仓库
    sl.registerLazySingleton<UserRepository>(
      () => UserRepositoryImpl(
        remoteDataSource: sl(),
        localDataSource: sl(),
        networkService: sl(),
      ),
    );
    
    // 用例
    sl.registerLazySingleton(() => GetUsers(sl()));
    sl.registerLazySingleton(() => CreateUser(sl()));
    sl.registerLazySingleton(() => UpdateUser(sl()));
    sl.registerLazySingleton(() => DeleteUser(sl()));
    
    // BLoC
    sl.registerFactory(() => UserBloc(
      getUsers: sl(),
      createUser: sl(),
      updateUser: sl(),
      deleteUser: sl(),
    ));
  }
}
```

### 3. 错误处理策略

```dart
// lib/src/core/error/error_handler.dart

abstract class ErrorHandler {
  static String getErrorMessage(dynamic error) {
    if (error is NetworkException) {
      return _getNetworkErrorMessage(error);
    } else if (error is ValidationException) {
      return error.message;
    } else if (error is AuthException) {
      return '认证失败，请重新登录';
    } else if (error is ServerException) {
      return '服务器错误，请稍后重试';
    } else {
      return '未知错误，请联系客服';
    }
  }
  
  static String _getNetworkErrorMessage(NetworkException error) {
    switch (error.type) {
      case NetworkExceptionType.connectTimeout:
        return '连接超时，请检查网络';
      case NetworkExceptionType.sendTimeout:
        return '发送超时，请重试';
      case NetworkExceptionType.receiveTimeout:
        return '接收超时，请重试';
      case NetworkExceptionType.cancel:
        return '请求已取消';
      case NetworkExceptionType.other:
        return '网络错误：${error.message}';
    }
  }
  
  static void logError(dynamic error, StackTrace? stackTrace) {
    // 记录错误日志
    print('Error: $error');
    if (stackTrace != null) {
      print('StackTrace: $stackTrace');
    }
    
    // 发送错误报告到服务器
    _sendErrorReport(error, stackTrace);
  }
  
  static void _sendErrorReport(dynamic error, StackTrace? stackTrace) {
    // 实现错误报告逻辑
  }
}
```

### 4. 性能优化建议

```dart
// lib/src/core/performance/performance_monitor.dart

class PerformanceMonitor {
  static final Map<String, DateTime> _startTimes = {};
  
  /// 开始监控
  static void startMonitoring(String operation) {
    _startTimes[operation] = DateTime.now();
  }
  
  /// 结束监控
  static void endMonitoring(String operation) {
    final startTime = _startTimes[operation];
    if (startTime != null) {
      final duration = DateTime.now().difference(startTime);
      print('Operation "$operation" took ${duration.inMilliseconds}ms');
      
      // 记录性能数据
      _recordPerformanceData(operation, duration);
      
      _startTimes.remove(operation);
    }
  }
  
  static void _recordPerformanceData(String operation, Duration duration) {
    // 实现性能数据记录逻辑
    if (duration.inMilliseconds > 1000) {
      print('Warning: Slow operation detected - $operation');
    }
  }
}

/// 性能监控装饰器
class PerformanceDecorator<T> {
  final String operationName;
  final Future<T> Function() operation;
  
  PerformanceDecorator({
    required this.operationName,
    required this.operation,
  });
  
  Future<T> execute() async {
    PerformanceMonitor.startMonitoring(operationName);
    try {
      final result = await operation();
      return result;
    } finally {
      PerformanceMonitor.endMonitoring(operationName);
    }
  }
}
```

### 5. 代码质量保证

```dart
// lib/src/core/quality/code_analyzer.dart

/// 代码质量分析器
class CodeAnalyzer {
  /// 检查类的复杂度
  static bool isClassTooComplex(Type classType) {
    // 实现类复杂度检查逻辑
    return false;
  }
  
  /// 检查方法的复杂度
  static bool isMethodTooComplex(Function method) {
    // 实现方法复杂度检查逻辑
    return false;
  }
  
  /// 检查依赖关系
  static List<String> checkDependencies() {
    // 实现依赖关系检查逻辑
    return [];
  }
}

/// 架构规则验证器
class ArchitectureValidator {
  /// 验证层级依赖规则
  static bool validateLayerDependencies() {
    // 验证表现层不能直接依赖数据层
    // 验证领域层不能依赖外部框架
    return true;
  }
  
  /// 验证命名约定
  static List<String> validateNamingConventions() {
    // 验证文件命名、类命名、方法命名等
    return [];
  }
}
```

## 总结

### 核心要点

1. **架构选择**：根据项目规模、团队经验和业务复杂度选择合适的架构模式
2. **分层设计**：明确各层职责，保持层间依赖的单向性
3. **状态管理**：选择适合的状态管理方案，确保状态的可预测性
4. **测试策略**：建立完善的测试体系，包括单元测试、集成测试和端到端测试
5. **代码质量**：遵循SOLID原则，保持代码的可读性和可维护性

### 最佳实践总结

- **单一职责**：每个类和方法只负责一个功能
- **依赖倒置**：依赖抽象而不是具体实现
- **开闭原则**：对扩展开放，对修改关闭
- **接口隔离**：使用小而专一的接口
- **里氏替换**：子类可以替换父类

### 未来发展趋势

1. **微前端架构**：将大型应用拆分为多个独立的小应用
2. **响应式编程**：使用Stream和RxDart处理异步数据流
3. **函数式编程**：引入函数式编程概念，提高代码的可预测性
4. **AI辅助开发**：利用AI工具提高开发效率和代码质量
5. **跨平台统一**：实现真正的一套代码多平台运行

通过合理的架构设计和最佳实践的应用，我们可以构建出高质量、可维护、可扩展的Flutter应用。架构设计不是一成不变的，需要根据项目的发展不断调整和优化，以适应业务需求的变化。
```