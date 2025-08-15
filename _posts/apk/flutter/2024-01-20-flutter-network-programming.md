---
layout: post
title: Flutter网络编程与HTTP请求处理深度解析
categories: flutter
tags: [Flutter, 网络编程, HTTP, Dio, 异步请求, API]
date: 2024/1/20 14:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_flutter_network.jpg)

## 引言

在现代移动应用开发中，网络编程是不可或缺的核心功能。Flutter提供了强大的网络编程能力，从基础的HTTP请求到复杂的WebSocket连接，都能够高效处理。本文将深入探讨Flutter网络编程的各个方面，包括HTTP客户端的使用、请求拦截、错误处理、缓存策略、以及网络安全等关键主题，帮助开发者构建健壮可靠的网络层架构。

## Flutter网络编程基础

### HTTP客户端概述

Flutter提供了多种HTTP客户端选择：

1. **dart:io HttpClient**：Dart标准库提供的底层HTTP客户端
2. **http包**：Flutter官方推荐的高级HTTP客户端
3. **dio包**：功能丰富的第三方HTTP客户端
4. **chopper包**：基于注解的HTTP客户端生成器

```dart
// HTTP客户端比较和选择
import 'dart:io';
import 'package:http/http.dart' as http;
import 'package:dio/dio.dart';

// 1. 使用dart:io HttpClient
class DartHttpClient {
  final HttpClient _client = HttpClient();
  
  Future<String> get(String url) async {
    try {
      final uri = Uri.parse(url);
      final request = await _client.getUrl(uri);
      final response = await request.close();
      
      if (response.statusCode == 200) {
        final responseBody = await response.transform(utf8.decoder).join();
        return responseBody;
      } else {
        throw HttpException('HTTP ${response.statusCode}: ${response.reasonPhrase}');
      }
    } catch (e) {
      throw NetworkException('Network request failed: $e');
    }
  }
  
  Future<String> post(String url, Map<String, dynamic> data) async {
    try {
      final uri = Uri.parse(url);
      final request = await _client.postUrl(uri);
      
      // 设置请求头
      request.headers.contentType = ContentType.json;
      
      // 写入请求体
      final jsonData = jsonEncode(data);
      request.write(jsonData);
      
      final response = await request.close();
      
      if (response.statusCode == 200 || response.statusCode == 201) {
        final responseBody = await response.transform(utf8.decoder).join();
        return responseBody;
      } else {
        throw HttpException('HTTP ${response.statusCode}: ${response.reasonPhrase}');
      }
    } catch (e) {
      throw NetworkException('Network request failed: $e');
    }
  }
  
  void dispose() {
    _client.close();
  }
}

// 2. 使用http包
class HttpPackageClient {
  static const Duration _timeout = Duration(seconds: 30);
  
  Future<Map<String, dynamic>> get(String url, {Map<String, String>? headers}) async {
    try {
      final response = await http.get(
        Uri.parse(url),
        headers: headers,
      ).timeout(_timeout);
      
      return _handleResponse(response);
    } catch (e) {
      throw _handleError(e);
    }
  }
  
  Future<Map<String, dynamic>> post(
    String url,
    Map<String, dynamic> data, {
    Map<String, String>? headers,
  }) async {
    try {
      final response = await http.post(
        Uri.parse(url),
        headers: {
          'Content-Type': 'application/json',
          ...?headers,
        },
        body: jsonEncode(data),
      ).timeout(_timeout);
      
      return _handleResponse(response);
    } catch (e) {
      throw _handleError(e);
    }
  }
  
  Future<Map<String, dynamic>> put(
    String url,
    Map<String, dynamic> data, {
    Map<String, String>? headers,
  }) async {
    try {
      final response = await http.put(
        Uri.parse(url),
        headers: {
          'Content-Type': 'application/json',
          ...?headers,
        },
        body: jsonEncode(data),
      ).timeout(_timeout);
      
      return _handleResponse(response);
    } catch (e) {
      throw _handleError(e);
    }
  }
  
  Future<Map<String, dynamic>> delete(
    String url, {
    Map<String, String>? headers,
  }) async {
    try {
      final response = await http.delete(
        Uri.parse(url),
        headers: headers,
      ).timeout(_timeout);
      
      return _handleResponse(response);
    } catch (e) {
      throw _handleError(e);
    }
  }
  
  Map<String, dynamic> _handleResponse(http.Response response) {
    if (response.statusCode >= 200 && response.statusCode < 300) {
      try {
        return jsonDecode(response.body) as Map<String, dynamic>;
      } catch (e) {
        throw ParseException('Failed to parse response: $e');
      }
    } else {
      throw HttpException(
        'HTTP ${response.statusCode}: ${response.reasonPhrase}',
        statusCode: response.statusCode,
        responseBody: response.body,
      );
    }
  }
  
  Exception _handleError(dynamic error) {
    if (error is TimeoutException) {
      return NetworkTimeoutException('Request timeout');
    } else if (error is SocketException) {
      return NetworkException('Network connection failed: ${error.message}');
    } else if (error is HttpException) {
      return error;
    } else {
      return NetworkException('Unknown network error: $error');
    }
  }
}

// 3. 使用Dio包（推荐）
class DioClient {
  late final Dio _dio;
  
  DioClient({String? baseUrl}) {
    _dio = Dio(BaseOptions(
      baseUrl: baseUrl ?? '',
      connectTimeout: Duration(seconds: 30),
      receiveTimeout: Duration(seconds: 30),
      sendTimeout: Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    ));
    
    _setupInterceptors();
  }
  
  void _setupInterceptors() {
    // 请求拦截器
    _dio.interceptors.add(
      InterceptorsWrapper(
        onRequest: (options, handler) {
          print('Request: ${options.method} ${options.path}');
          print('Headers: ${options.headers}');
          print('Data: ${options.data}');
          
          // 添加认证token
          final token = _getAuthToken();
          if (token != null) {
            options.headers['Authorization'] = 'Bearer $token';
          }
          
          handler.next(options);
        },
        onResponse: (response, handler) {
          print('Response: ${response.statusCode} ${response.statusMessage}');
          print('Data: ${response.data}');
          handler.next(response);
        },
        onError: (error, handler) {
          print('Error: ${error.message}');
          print('Response: ${error.response?.data}');
          
          // 处理token过期
          if (error.response?.statusCode == 401) {
            _handleTokenExpired();
          }
          
          handler.next(error);
        },
      ),
    );
    
    // 日志拦截器
    _dio.interceptors.add(LogInterceptor(
      requestBody: true,
      responseBody: true,
      requestHeader: true,
      responseHeader: true,
    ));
  }
  
  String? _getAuthToken() {
    // 从本地存储获取token
    return null; // 实际实现中应该从SharedPreferences或Secure Storage获取
  }
  
  void _handleTokenExpired() {
    // 处理token过期，例如跳转到登录页面
    print('Token expired, redirecting to login');
  }
  
  Future<T> get<T>(
    String path, {
    Map<String, dynamic>? queryParameters,
    Options? options,
  }) async {
    try {
      final response = await _dio.get<T>(
        path,
        queryParameters: queryParameters,
        options: options,
      );
      return response.data!;
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }
  
  Future<T> post<T>(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    Options? options,
  }) async {
    try {
      final response = await _dio.post<T>(
        path,
        data: data,
        queryParameters: queryParameters,
        options: options,
      );
      return response.data!;
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }
  
  Future<T> put<T>(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    Options? options,
  }) async {
    try {
      final response = await _dio.put<T>(
        path,
        data: data,
        queryParameters: queryParameters,
        options: options,
      );
      return response.data!;
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }
  
  Future<T> delete<T>(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    Options? options,
  }) async {
    try {
      final response = await _dio.delete<T>(
        path,
        data: data,
        queryParameters: queryParameters,
        options: options,
      );
      return response.data!;
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }
  
  // 文件上传
  Future<T> uploadFile<T>(
    String path,
    String filePath, {
    String? fileName,
    Map<String, dynamic>? data,
    ProgressCallback? onSendProgress,
  }) async {
    try {
      final formData = FormData.fromMap({
        'file': await MultipartFile.fromFile(
          filePath,
          filename: fileName ?? filePath.split('/').last,
        ),
        ...?data,
      });
      
      final response = await _dio.post<T>(
        path,
        data: formData,
        onSendProgress: onSendProgress,
      );
      
      return response.data!;
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }
  
  // 文件下载
  Future<void> downloadFile(
    String url,
    String savePath, {
    ProgressCallback? onReceiveProgress,
    CancelToken? cancelToken,
  }) async {
    try {
      await _dio.download(
        url,
        savePath,
        onReceiveProgress: onReceiveProgress,
        cancelToken: cancelToken,
      );
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }
  
  Exception _handleDioError(DioException error) {
    switch (error.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return NetworkTimeoutException('Request timeout: ${error.message}');
      
      case DioExceptionType.badResponse:
        final statusCode = error.response?.statusCode ?? 0;
        final message = error.response?.statusMessage ?? 'Unknown error';
        return HttpException(
          'HTTP $statusCode: $message',
          statusCode: statusCode,
          responseBody: error.response?.data?.toString(),
        );
      
      case DioExceptionType.cancel:
        return NetworkException('Request cancelled');
      
      case DioExceptionType.connectionError:
        return NetworkException('Connection error: ${error.message}');
      
      case DioExceptionType.badCertificate:
        return NetworkException('SSL certificate error: ${error.message}');
      
      case DioExceptionType.unknown:
      default:
        return NetworkException('Unknown network error: ${error.message}');
    }
  }
}

// 自定义异常类
class NetworkException implements Exception {
  final String message;
  
  NetworkException(this.message);
  
  @override
  String toString() => 'NetworkException: $message';
}

class NetworkTimeoutException extends NetworkException {
  NetworkTimeoutException(String message) : super(message);
  
  @override
  String toString() => 'NetworkTimeoutException: $message';
}

class HttpException extends NetworkException {
  final int? statusCode;
  final String? responseBody;
  
  HttpException(
    String message, {
    this.statusCode,
    this.responseBody,
  }) : super(message);
  
  @override
  String toString() => 'HttpException: $message (Status: $statusCode)';
}

class ParseException extends NetworkException {
  ParseException(String message) : super(message);
  
  @override
  String toString() => 'ParseException: $message';
}
```

### 网络请求封装和管理

```dart
// 网络请求管理器
class NetworkManager {
  static NetworkManager? _instance;
  static NetworkManager get instance => _instance ??= NetworkManager._();
  
  late final DioClient _client;
  final Map<String, CancelToken> _cancelTokens = {};
  
  NetworkManager._() {
    _client = DioClient(baseUrl: 'https://api.example.com');
  }
  
  // API端点常量
  static const String _authEndpoint = '/auth';
  static const String _usersEndpoint = '/users';
  static const String _postsEndpoint = '/posts';
  static const String _uploadEndpoint = '/upload';
  
  // 认证相关API
  Future<AuthResponse> login(String email, String password) async {
    final data = {
      'email': email,
      'password': password,
    };
    
    final response = await _client.post<Map<String, dynamic>>(
      '$_authEndpoint/login',
      data: data,
    );
    
    return AuthResponse.fromJson(response);
  }
  
  Future<AuthResponse> register(String email, String password, String name) async {
    final data = {
      'email': email,
      'password': password,
      'name': name,
    };
    
    final response = await _client.post<Map<String, dynamic>>(
      '$_authEndpoint/register',
      data: data,
    );
    
    return AuthResponse.fromJson(response);
  }
  
  Future<void> logout() async {
    await _client.post('$_authEndpoint/logout');
  }
  
  Future<AuthResponse> refreshToken(String refreshToken) async {
    final data = {'refresh_token': refreshToken};
    
    final response = await _client.post<Map<String, dynamic>>(
      '$_authEndpoint/refresh',
      data: data,
    );
    
    return AuthResponse.fromJson(response);
  }
  
  // 用户相关API
  Future<User> getCurrentUser() async {
    final response = await _client.get<Map<String, dynamic>>('$_usersEndpoint/me');
    return User.fromJson(response);
  }
  
  Future<User> updateUser(Map<String, dynamic> userData) async {
    final response = await _client.put<Map<String, dynamic>>(
      '$_usersEndpoint/me',
      data: userData,
    );
    return User.fromJson(response);
  }
  
  Future<List<User>> getUsers({
    int page = 1,
    int limit = 20,
    String? search,
  }) async {
    final queryParams = {
      'page': page,
      'limit': limit,
      if (search != null) 'search': search,
    };
    
    final response = await _client.get<Map<String, dynamic>>(
      _usersEndpoint,
      queryParameters: queryParams,
    );
    
    final List<dynamic> usersJson = response['data'] ?? [];
    return usersJson.map((json) => User.fromJson(json)).toList();
  }
  
  // 帖子相关API
  Future<List<Post>> getPosts({
    int page = 1,
    int limit = 20,
    String? category,
    String requestId = 'posts',
  }) async {
    // 取消之前的相同请求
    _cancelPreviousRequest(requestId);
    
    final cancelToken = CancelToken();
    _cancelTokens[requestId] = cancelToken;
    
    try {
      final queryParams = {
        'page': page,
        'limit': limit,
        if (category != null) 'category': category,
      };
      
      final response = await _client.get<Map<String, dynamic>>(
        _postsEndpoint,
        queryParameters: queryParams,
        options: Options(cancelToken: cancelToken),
      );
      
      final List<dynamic> postsJson = response['data'] ?? [];
      return postsJson.map((json) => Post.fromJson(json)).toList();
    } finally {
      _cancelTokens.remove(requestId);
    }
  }
  
  Future<Post> getPost(int postId) async {
    final response = await _client.get<Map<String, dynamic>>('$_postsEndpoint/$postId');
    return Post.fromJson(response);
  }
  
  Future<Post> createPost(Map<String, dynamic> postData) async {
    final response = await _client.post<Map<String, dynamic>>(
      _postsEndpoint,
      data: postData,
    );
    return Post.fromJson(response);
  }
  
  Future<Post> updatePost(int postId, Map<String, dynamic> postData) async {
    final response = await _client.put<Map<String, dynamic>>(
      '$_postsEndpoint/$postId',
      data: postData,
    );
    return Post.fromJson(response);
  }
  
  Future<void> deletePost(int postId) async {
    await _client.delete('$_postsEndpoint/$postId');
  }
  
  // 文件上传
  Future<UploadResponse> uploadImage(
    String filePath, {
    ProgressCallback? onProgress,
  }) async {
    final response = await _client.uploadFile<Map<String, dynamic>>(
      _uploadEndpoint,
      filePath,
      onSendProgress: onProgress,
    );
    
    return UploadResponse.fromJson(response);
  }
  
  // 批量上传
  Future<List<UploadResponse>> uploadMultipleImages(
    List<String> filePaths, {
    ProgressCallback? onProgress,
  }) async {
    final List<UploadResponse> results = [];
    
    for (int i = 0; i < filePaths.length; i++) {
      final result = await uploadImage(
        filePaths[i],
        onProgress: (sent, total) {
          final overallProgress = (i * total + sent) / (filePaths.length * total);
          onProgress?.call((overallProgress * total).round(), total);
        },
      );
      results.add(result);
    }
    
    return results;
  }
  
  // 取消请求
  void cancelRequest(String requestId) {
    _cancelTokens[requestId]?.cancel('Request cancelled by user');
    _cancelTokens.remove(requestId);
  }
  
  void _cancelPreviousRequest(String requestId) {
    if (_cancelTokens.containsKey(requestId)) {
      _cancelTokens[requestId]?.cancel('Cancelled by new request');
      _cancelTokens.remove(requestId);
    }
  }
  
  // 取消所有请求
  void cancelAllRequests() {
    for (final token in _cancelTokens.values) {
      token.cancel('Cancelled all requests');
    }
    _cancelTokens.clear();
  }
}

// 数据模型
class AuthResponse {
  final String accessToken;
  final String refreshToken;
  final User user;
  final DateTime expiresAt;
  
  AuthResponse({
    required this.accessToken,
    required this.refreshToken,
    required this.user,
    required this.expiresAt,
  });
  
  factory AuthResponse.fromJson(Map<String, dynamic> json) {
    return AuthResponse(
      accessToken: json['access_token'] ?? '',
      refreshToken: json['refresh_token'] ?? '',
      user: User.fromJson(json['user'] ?? {}),
      expiresAt: DateTime.parse(json['expires_at'] ?? DateTime.now().toIso8601String()),
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'access_token': accessToken,
      'refresh_token': refreshToken,
      'user': user.toJson(),
      'expires_at': expiresAt.toIso8601String(),
    };
  }
}

class User {
  final int id;
  final String name;
  final String email;
  final String? avatar;
  final DateTime createdAt;
  final DateTime updatedAt;
  
  User({
    required this.id,
    required this.name,
    required this.email,
    this.avatar,
    required this.createdAt,
    required this.updatedAt,
  });
  
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] ?? 0,
      name: json['name'] ?? '',
      email: json['email'] ?? '',
      avatar: json['avatar'],
      createdAt: DateTime.parse(json['created_at'] ?? DateTime.now().toIso8601String()),
      updatedAt: DateTime.parse(json['updated_at'] ?? DateTime.now().toIso8601String()),
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'email': email,
      'avatar': avatar,
      'created_at': createdAt.toIso8601String(),
      'updated_at': updatedAt.toIso8601String(),
    };
  }
}

class Post {
  final int id;
  final String title;
  final String content;
  final String? image;
  final String category;
  final User author;
  final DateTime createdAt;
  final DateTime updatedAt;
  
  Post({
    required this.id,
    required this.title,
    required this.content,
    this.image,
    required this.category,
    required this.author,
    required this.createdAt,
    required this.updatedAt,
  });
  
  factory Post.fromJson(Map<String, dynamic> json) {
    return Post(
      id: json['id'] ?? 0,
      title: json['title'] ?? '',
      content: json['content'] ?? '',
      image: json['image'],
      category: json['category'] ?? '',
      author: User.fromJson(json['author'] ?? {}),
      createdAt: DateTime.parse(json['created_at'] ?? DateTime.now().toIso8601String()),
      updatedAt: DateTime.parse(json['updated_at'] ?? DateTime.now().toIso8601String()),
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'content': content,
      'image': image,
      'category': category,
      'author': author.toJson(),
      'created_at': createdAt.toIso8601String(),
      'updated_at': updatedAt.toIso8601String(),
    };
  }
}

class UploadResponse {
  final String url;
  final String filename;
  final int size;
  final String mimeType;
  
  UploadResponse({
    required this.url,
    required this.filename,
    required this.size,
    required this.mimeType,
  });
  
  factory UploadResponse.fromJson(Map<String, dynamic> json) {
    return UploadResponse(
      url: json['url'] ?? '',
      filename: json['filename'] ?? '',
      size: json['size'] ?? 0,
      mimeType: json['mime_type'] ?? '',
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'url': url,
      'filename': filename,
      'size': size,
      'mime_type': mimeType,
    };
  }
}
```

## 网络缓存策略

### HTTP缓存实现

```dart
// 网络缓存管理器
class NetworkCacheManager {
  static NetworkCacheManager? _instance;
  static NetworkCacheManager get instance => _instance ??= NetworkCacheManager._();
  
  final Map<String, CacheEntry> _memoryCache = {};
  final int _maxMemoryCacheSize = 100;
  final Duration _defaultCacheDuration = Duration(minutes: 30);
  
  NetworkCacheManager._();
  
  // 生成缓存键
  String _generateCacheKey(String url, Map<String, dynamic>? params) {
    final uri = Uri.parse(url);
    final queryParams = {...uri.queryParameters, ...?params};
    final sortedParams = Map.fromEntries(
      queryParams.entries.toList()..sort((a, b) => a.key.compareTo(b.key)),
    );
    
    final cacheKey = '${uri.path}?${Uri(queryParameters: sortedParams).query}';
    return cacheKey;
  }
  
  // 获取缓存
  T? getCache<T>(String url, {Map<String, dynamic>? params}) {
    final key = _generateCacheKey(url, params);
    final entry = _memoryCache[key];
    
    if (entry != null && !entry.isExpired) {
      print('Cache hit for: $key');
      return entry.data as T?;
    }
    
    if (entry != null && entry.isExpired) {
      _memoryCache.remove(key);
      print('Cache expired for: $key');
    }
    
    return null;
  }
  
  // 设置缓存
  void setCache<T>(
    String url,
    T data, {
    Map<String, dynamic>? params,
    Duration? cacheDuration,
  }) {
    final key = _generateCacheKey(url, params);
    final duration = cacheDuration ?? _defaultCacheDuration;
    
    // 如果缓存已满，移除最旧的条目
    if (_memoryCache.length >= _maxMemoryCacheSize) {
      _removeOldestEntry();
    }
    
    _memoryCache[key] = CacheEntry(
      data: data,
      expiresAt: DateTime.now().add(duration),
      createdAt: DateTime.now(),
    );
    
    print('Cache set for: $key');
  }
  
  // 移除缓存
  void removeCache(String url, {Map<String, dynamic>? params}) {
    final key = _generateCacheKey(url, params);
    _memoryCache.remove(key);
    print('Cache removed for: $key');
  }
  
  // 清除所有缓存
  void clearCache() {
    _memoryCache.clear();
    print('All cache cleared');
  }
  
  // 清除过期缓存
  void clearExpiredCache() {
    final expiredKeys = _memoryCache.entries
        .where((entry) => entry.value.isExpired)
        .map((entry) => entry.key)
        .toList();
    
    for (final key in expiredKeys) {
      _memoryCache.remove(key);
    }
    
    print('Cleared ${expiredKeys.length} expired cache entries');
  }
  
  // 移除最旧的缓存条目
  void _removeOldestEntry() {
    if (_memoryCache.isEmpty) return;
    
    String? oldestKey;
    DateTime? oldestTime;
    
    for (final entry in _memoryCache.entries) {
      if (oldestTime == null || entry.value.createdAt.isBefore(oldestTime)) {
        oldestTime = entry.value.createdAt;
        oldestKey = entry.key;
      }
    }
    
    if (oldestKey != null) {
      _memoryCache.remove(oldestKey);
      print('Removed oldest cache entry: $oldestKey');
    }
  }
  
  // 获取缓存统计信息
  CacheStats getCacheStats() {
    final totalEntries = _memoryCache.length;
    final expiredEntries = _memoryCache.values.where((entry) => entry.isExpired).length;
    final validEntries = totalEntries - expiredEntries;
    
    return CacheStats(
      totalEntries: totalEntries,
      validEntries: validEntries,
      expiredEntries: expiredEntries,
      maxSize: _maxMemoryCacheSize,
    );
  }
}

// 缓存条目
class CacheEntry {
  final dynamic data;
  final DateTime expiresAt;
  final DateTime createdAt;
  
  CacheEntry({
    required this.data,
    required this.expiresAt,
    required this.createdAt,
  });
  
  bool get isExpired => DateTime.now().isAfter(expiresAt);
  
  Duration get timeToLive => expiresAt.difference(DateTime.now());
  
  Duration get age => DateTime.now().difference(createdAt);
}

// 缓存统计信息
class CacheStats {
  final int totalEntries;
  final int validEntries;
  final int expiredEntries;
  final int maxSize;
  
  CacheStats({
    required this.totalEntries,
    required this.validEntries,
    required this.expiredEntries,
    required this.maxSize,
  });
  
  double get hitRatio => totalEntries > 0 ? validEntries / totalEntries : 0.0;
  
  double get usageRatio => maxSize > 0 ? totalEntries / maxSize : 0.0;
  
  @override
  String toString() {
    return 'CacheStats(total: $totalEntries, valid: $validEntries, expired: $expiredEntries, hitRatio: ${(hitRatio * 100).toStringAsFixed(1)}%, usage: ${(usageRatio * 100).toStringAsFixed(1)}%)';
  }
}

// 带缓存的网络客户端
class CachedNetworkClient {
  final DioClient _client;
  final NetworkCacheManager _cacheManager;
  
  CachedNetworkClient({
    required DioClient client,
    NetworkCacheManager? cacheManager,
  }) : _client = client,
       _cacheManager = cacheManager ?? NetworkCacheManager.instance;
  
  Future<T> get<T>(
    String path, {
    Map<String, dynamic>? queryParameters,
    bool useCache = true,
    Duration? cacheDuration,
    bool forceRefresh = false,
  }) async {
    // 如果不使用缓存或强制刷新，直接请求
    if (!useCache || forceRefresh) {
      final result = await _client.get<T>(path, queryParameters: queryParameters);
      
      if (useCache) {
        _cacheManager.setCache(
          path,
          result,
          params: queryParameters,
          cacheDuration: cacheDuration,
        );
      }
      
      return result;
    }
    
    // 尝试从缓存获取
    final cachedResult = _cacheManager.getCache<T>(path, params: queryParameters);
    if (cachedResult != null) {
      return cachedResult;
    }
    
    // 缓存未命中，发起网络请求
    final result = await _client.get<T>(path, queryParameters: queryParameters);
    
    // 缓存结果
    _cacheManager.setCache(
      path,
      result,
      params: queryParameters,
      cacheDuration: cacheDuration,
    );
    
    return result;
  }
  
  Future<T> post<T>(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    bool invalidateCache = true,
  }) async {
    final result = await _client.post<T>(
      path,
      data: data,
      queryParameters: queryParameters,
    );
    
    // POST请求通常会修改数据，需要清除相关缓存
    if (invalidateCache) {
      _invalidateRelatedCache(path);
    }
    
    return result;
  }
  
  Future<T> put<T>(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    bool invalidateCache = true,
  }) async {
    final result = await _client.put<T>(
      path,
      data: data,
      queryParameters: queryParameters,
    );
    
    if (invalidateCache) {
      _invalidateRelatedCache(path);
    }
    
    return result;
  }
  
  Future<T> delete<T>(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    bool invalidateCache = true,
  }) async {
    final result = await _client.delete<T>(
      path,
      data: data,
      queryParameters: queryParameters,
    );
    
    if (invalidateCache) {
      _invalidateRelatedCache(path);
    }
    
    return result;
  }
  
  // 清除相关缓存
  void _invalidateRelatedCache(String path) {
    // 简单实现：清除所有缓存
    // 实际应用中可以根据路径模式清除相关缓存
    _cacheManager.clearCache();
    print('Cache invalidated for path: $path');
  }
  
  // 预加载数据
  Future<void> preloadData(List<String> paths) async {
    final futures = paths.map((path) => get(path, useCache: true));
    await Future.wait(futures, eagerError: false);
    print('Preloaded ${paths.length} endpoints');
  }
}
```

## WebSocket连接管理

```dart
// WebSocket连接管理器
class WebSocketManager {
  static WebSocketManager? _instance;
  static WebSocketManager get instance => _instance ??= WebSocketManager._();
  
  WebSocket? _socket;
  StreamController<dynamic>? _messageController;
  StreamController<WebSocketConnectionState>? _stateController;
  
  Timer? _heartbeatTimer;
  Timer? _reconnectTimer;
  
  String? _url;
  Map<String, String>? _headers;
  Duration _heartbeatInterval = Duration(seconds: 30);
  Duration _reconnectInterval = Duration(seconds: 5);
  int _maxReconnectAttempts = 5;
  int _reconnectAttempts = 0;
  bool _shouldReconnect = true;
  
  WebSocketConnectionState _state = WebSocketConnectionState.disconnected;
  
  WebSocketManager._();
  
  // 连接状态流
  Stream<WebSocketConnectionState> get stateStream {
    _stateController ??= StreamController<WebSocketConnectionState>.broadcast();
    return _stateController!.stream;
  }
  
  // 消息流
  Stream<dynamic> get messageStream {
    _messageController ??= StreamController<dynamic>.broadcast();
    return _messageController!.stream;
  }
  
  // 当前连接状态
  WebSocketConnectionState get state => _state;
  
  // 是否已连接
  bool get isConnected => _state == WebSocketConnectionState.connected;
  
  // 连接WebSocket
  Future<void> connect(
    String url, {
    Map<String, String>? headers,
    Duration? heartbeatInterval,
    Duration? reconnectInterval,
    int? maxReconnectAttempts,
  }) async {
    if (_state == WebSocketConnectionState.connecting) {
      print('WebSocket is already connecting');
      return;
    }
    
    _url = url;
    _headers = headers;
    _heartbeatInterval = heartbeatInterval ?? _heartbeatInterval;
    _reconnectInterval = reconnectInterval ?? _reconnectInterval;
    _maxReconnectAttempts = maxReconnectAttempts ?? _maxReconnectAttempts;
    
    await _performConnect();
  }
  
  Future<void> _performConnect() async {
    try {
      _setState(WebSocketConnectionState.connecting);
      
      _socket = await WebSocket.connect(
        _url!,
        headers: _headers,
      );
      
      _setState(WebSocketConnectionState.connected);
      _reconnectAttempts = 0;
      
      // 监听消息
      _socket!.listen(
        _onMessage,
        onError: _onError,
        onDone: _onDisconnected,
      );
      
      // 启动心跳
      _startHeartbeat();
      
      print('WebSocket connected to $_url');
    } catch (e) {
      print('WebSocket connection failed: $e');
      _setState(WebSocketConnectionState.error);
      _scheduleReconnect();
    }
  }
  
  // 断开连接
  Future<void> disconnect() async {
    _shouldReconnect = false;
    _stopHeartbeat();
    _stopReconnectTimer();
    
    if (_socket != null) {
      await _socket!.close();
      _socket = null;
    }
    
    _setState(WebSocketConnectionState.disconnected);
    print('WebSocket disconnected');
  }
  
  // 发送消息
  void send(dynamic message) {
    if (!isConnected) {
      throw StateError('WebSocket is not connected');
    }
    
    try {
      if (message is String) {
        _socket!.add(message);
      } else {
        _socket!.add(jsonEncode(message));
      }
      
      print('WebSocket message sent: $message');
    } catch (e) {
      print('Failed to send WebSocket message: $e');
      throw WebSocketException('Failed to send message: $e');
    }
  }
  
  // 发送JSON消息
  void sendJson(Map<String, dynamic> data) {
    send(jsonEncode(data));
  }
  
  // 发送心跳
  void sendHeartbeat() {
    if (isConnected) {
      send({'type': 'heartbeat', 'timestamp': DateTime.now().millisecondsSinceEpoch});
    }
  }
  
  void _onMessage(dynamic message) {
    print('WebSocket message received: $message');
    
    try {
      dynamic parsedMessage;
      
      if (message is String) {
        try {
          parsedMessage = jsonDecode(message);
        } catch (e) {
          parsedMessage = message;
        }
      } else {
        parsedMessage = message;
      }
      
      _messageController?.add(parsedMessage);
    } catch (e) {
      print('Error processing WebSocket message: $e');
    }
  }
  
  void _onError(dynamic error) {
    print('WebSocket error: $error');
    _setState(WebSocketConnectionState.error);
    _scheduleReconnect();
  }
  
  void _onDisconnected() {
    print('WebSocket disconnected');
    _setState(WebSocketConnectionState.disconnected);
    _stopHeartbeat();
    
    if (_shouldReconnect) {
      _scheduleReconnect();
    }
  }
  
  void _setState(WebSocketConnectionState newState) {
    if (_state != newState) {
      _state = newState;
      _stateController?.add(newState);
      print('WebSocket state changed to: $newState');
    }
  }
  
  void _startHeartbeat() {
    _stopHeartbeat();
    
    _heartbeatTimer = Timer.periodic(_heartbeatInterval, (timer) {
      if (isConnected) {
        sendHeartbeat();
      } else {
        timer.cancel();
      }
    });
  }
  
  void _stopHeartbeat() {
    _heartbeatTimer?.cancel();
    _heartbeatTimer = null;
  }
  
  void _scheduleReconnect() {
    if (!_shouldReconnect || _reconnectAttempts >= _maxReconnectAttempts) {
      print('Max reconnect attempts reached or reconnect disabled');
      return;
    }
    
    _stopReconnectTimer();
    
    _reconnectAttempts++;
    print('Scheduling reconnect attempt $_reconnectAttempts in ${_reconnectInterval.inSeconds} seconds');
    
    _reconnectTimer = Timer(_reconnectInterval, () {
      if (_shouldReconnect) {
        _performConnect();
      }
    });
  }
  
  void _stopReconnectTimer() {
    _reconnectTimer?.cancel();
    _reconnectTimer = null;
  }
  
  // 清理资源
  void dispose() {
    _shouldReconnect = false;
    _stopHeartbeat();
    _stopReconnectTimer();
    
    _socket?.close();
    _messageController?.close();
    _stateController?.close();
    
    _socket = null;
    _messageController = null;
    _stateController = null;
  }
}

// WebSocket连接状态
enum WebSocketConnectionState {
  disconnected,
  connecting,
  connected,
  error,
}

// WebSocket异常
class WebSocketException implements Exception {
  final String message;
  
  WebSocketException(this.message);
  
  @override
  String toString() => 'WebSocketException: $message';
}

// WebSocket消息类型
class WebSocketMessage {
  final String type;
  final Map<String, dynamic> data;
  final DateTime timestamp;
  
  WebSocketMessage({
    required this.type,
    required this.data,
    DateTime? timestamp,
  }) : timestamp = timestamp ?? DateTime.now();
  
  factory WebSocketMessage.fromJson(Map<String, dynamic> json) {
    return WebSocketMessage(
      type: json['type'] ?? '',
      data: json['data'] ?? {},
      timestamp: json['timestamp'] != null
          ? DateTime.fromMillisecondsSinceEpoch(json['timestamp'])
          : DateTime.now(),
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'type': type,
      'data': data,
      'timestamp': timestamp.millisecondsSinceEpoch,
    };
  }
}

// WebSocket使用示例
class WebSocketDemo extends StatefulWidget {
  @override
  _WebSocketDemoState createState() => _WebSocketDemoState();
}

class _WebSocketDemoState extends State<WebSocketDemo> {
  final WebSocketManager _wsManager = WebSocketManager.instance;
  final List<String> _messages = [];
  final TextEditingController _messageController = TextEditingController();
  
  StreamSubscription<dynamic>? _messageSubscription;
  StreamSubscription<WebSocketConnectionState>? _stateSubscription;
  
  WebSocketConnectionState _connectionState = WebSocketConnectionState.disconnected;
  
  @override
  void initState() {
    super.initState();
    
    // 监听连接状态
    _stateSubscription = _wsManager.stateStream.listen((state) {
      setState(() {
        _connectionState = state;
      });
    });
    
    // 监听消息
    _messageSubscription = _wsManager.messageStream.listen((message) {
      setState(() {
        _messages.add('Received: ${message.toString()}');
      });
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('WebSocket Demo'),
        actions: [
          IconButton(
            icon: Icon(_connectionState == WebSocketConnectionState.connected
                ? Icons.wifi
                : Icons.wifi_off),
            onPressed: _toggleConnection,
          ),
        ],
      ),
      body: Column(
        children: [
          // 连接状态
          Container(
            width: double.infinity,
            padding: EdgeInsets.all(16),
            color: _getStateColor(),
            child: Text(
              'Status: ${_connectionState.toString().split('.').last}',
              style: TextStyle(
                color: Colors.white,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
          
          // 消息列表
          Expanded(
            child: ListView.builder(
              itemCount: _messages.length,
              itemBuilder: (context, index) {
                return ListTile(
                  title: Text(_messages[index]),
                  dense: true,
                );
              },
            ),
          ),
          
          // 消息输入
          Container(
            padding: EdgeInsets.all(16),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _messageController,
                    decoration: InputDecoration(
                      hintText: 'Enter message...',
                      border: OutlineInputBorder(),
                    ),
                    onSubmitted: (_) => _sendMessage(),
                  ),
                ),
                SizedBox(width: 8),
                ElevatedButton(
                  onPressed: _connectionState == WebSocketConnectionState.connected
                      ? _sendMessage
                      : null,
                  child: Text('Send'),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
  
  Color _getStateColor() {
    switch (_connectionState) {
      case WebSocketConnectionState.connected:
        return Colors.green;
      case WebSocketConnectionState.connecting:
        return Colors.orange;
      case WebSocketConnectionState.error:
        return Colors.red;
      case WebSocketConnectionState.disconnected:
      default:
        return Colors.grey;
    }
  }
  
  void _toggleConnection() {
    if (_connectionState == WebSocketConnectionState.connected) {
      _wsManager.disconnect();
    } else {
      _wsManager.connect(
        'wss://echo.websocket.org',
        heartbeatInterval: Duration(seconds: 30),
        reconnectInterval: Duration(seconds: 5),
        maxReconnectAttempts: 3,
      );
    }
  }
  
  void _sendMessage() {
    final message = _messageController.text.trim();
    if (message.isNotEmpty) {
      _wsManager.send(message);
      setState(() {
        _messages.add('Sent: $message');
      });
      _messageController.clear();
    }
  }
  
  @override
  void dispose() {
    _messageSubscription?.cancel();
    _stateSubscription?.cancel();
    _messageController.dispose();
    super.dispose();
  }
}
```

## 网络安全和认证

### JWT认证管理

```dart
// JWT认证管理器
class AuthManager {
  static AuthManager? _instance;
  static AuthManager get instance => _instance ??= AuthManager._();
  
  static const String _accessTokenKey = 'access_token';
  static const String _refreshTokenKey = 'refresh_token';
  static const String _userKey = 'user_data';
  
  String? _accessToken;
  String? _refreshToken;
  User? _currentUser;
  DateTime? _tokenExpiresAt;
  
  final StreamController<AuthState> _authStateController = StreamController<AuthState>.broadcast();
  Timer? _tokenRefreshTimer;
  
  AuthManager._() {
    _loadAuthData();
  }
  
  // 认证状态流
  Stream<AuthState> get authStateStream => _authStateController.stream;
  
  // 当前用户
  User? get currentUser => _currentUser;
  
  // 是否已认证
  bool get isAuthenticated => _accessToken != null && !_isTokenExpired();
  
  // 访问令牌
  String? get accessToken => _accessToken;
  
  // 登录
  Future<void> login(String email, String password) async {
    try {
      final response = await NetworkManager.instance.login(email, password);
      
      await _saveAuthData(response);
      _scheduleTokenRefresh();
      
      _authStateController.add(AuthState.authenticated);
    } catch (e) {
      _authStateController.add(AuthState.unauthenticated);
      rethrow;
    }
  }
  
  // 注册
  Future<void> register(String email, String password, String name) async {
    try {
      final response = await NetworkManager.instance.register(email, password, name);
      
      await _saveAuthData(response);
      _scheduleTokenRefresh();
      
      _authStateController.add(AuthState.authenticated);
    } catch (e) {
      _authStateController.add(AuthState.unauthenticated);
      rethrow;
    }
  }
  
  // 登出
  Future<void> logout() async {
    try {
      if (_accessToken != null) {
        await NetworkManager.instance.logout();
      }
    } catch (e) {
      print('Logout request failed: $e');
    } finally {
      await _clearAuthData();
      _cancelTokenRefresh();
      _authStateController.add(AuthState.unauthenticated);
    }
  }
  
  // 刷新令牌
  Future<bool> refreshToken() async {
    if (_refreshToken == null) {
      await logout();
      return false;
    }
    
    try {
      final response = await NetworkManager.instance.refreshToken(_refreshToken!);
      
      await _saveAuthData(response);
      _scheduleTokenRefresh();
      
      return true;
    } catch (e) {
      print('Token refresh failed: $e');
      await logout();
      return false;
    }
  }
  
  // 检查令牌是否过期
  bool _isTokenExpired() {
    if (_tokenExpiresAt == null) return true;
    
    // 提前5分钟认为令牌过期，以便有时间刷新
    final buffer = Duration(minutes: 5);
    return DateTime.now().add(buffer).isAfter(_tokenExpiresAt!);
  }
  
  // 保存认证数据
  Future<void> _saveAuthData(AuthResponse response) async {
    _accessToken = response.accessToken;
    _refreshToken = response.refreshToken;
    _currentUser = response.user;
    _tokenExpiresAt = response.expiresAt;
    
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_userKey, jsonEncode(_currentUser!.toJson()));
  }
  
  // 清理资源
  void dispose() {
    _cancelTokenRefresh();
    _authStateController.close();
  }
}

// 认证状态
enum AuthState {
  authenticated,
  unauthenticated,
  loading,
}

// 安全的HTTP拦截器
class SecurityInterceptor extends Interceptor {
  final AuthManager _authManager;
  
  SecurityInterceptor(this._authManager);
  
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // 添加认证头
    final token = _authManager.accessToken;
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    
    // 添加安全头
    options.headers['X-Requested-With'] = 'XMLHttpRequest';
    options.headers['X-Client-Version'] = '1.0.0';
    
    // 添加CSRF保护
    if (options.method.toUpperCase() != 'GET') {
      options.headers['X-CSRF-Token'] = _generateCSRFToken();
    }
    
    super.onRequest(options, handler);
  }
  
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // 处理认证错误
    if (err.response?.statusCode == 401) {
      _handleUnauthorized();
    }
    
    super.onError(err, handler);
  }
  
  String _generateCSRFToken() {
    // 生成CSRF令牌
    final random = Random.secure();
    final bytes = List<int>.generate(32, (i) => random.nextInt(256));
    return base64Encode(bytes);
  }
  
  void _handleUnauthorized() {
    // 处理未授权错误
    _authManager.logout();
  }
}
```

## 网络请求优化策略

### 请求队列和并发控制

```dart
// 网络请求队列管理器
class RequestQueueManager {
  static RequestQueueManager? _instance;
  static RequestQueueManager get instance => _instance ??= RequestQueueManager._();
  
  final Queue<RequestTask> _requestQueue = Queue<RequestTask>();
  final Set<RequestTask> _activeRequests = <RequestTask>{};
  final int _maxConcurrentRequests = 6;
  final Duration _requestTimeout = Duration(seconds: 30);
  
  RequestQueueManager._();
  
  // 添加请求到队列
  Future<T> enqueue<T>(RequestTask<T> task) async {
    final completer = Completer<T>();
    task._completer = completer;
    
    _requestQueue.add(task);
    _processQueue();
    
    return completer.future;
  }
  
  // 处理队列
  void _processQueue() {
    while (_requestQueue.isNotEmpty && _activeRequests.length < _maxConcurrentRequests) {
      final task = _requestQueue.removeFirst();
      _executeTask(task);
    }
  }
  
  // 执行任务
  Future<void> _executeTask(RequestTask task) async {
    _activeRequests.add(task);
    
    try {
      final result = await task._execute().timeout(_requestTimeout);
      task._completer?.complete(result);
    } catch (e) {
      task._completer?.completeError(e);
    } finally {
      _activeRequests.remove(task);
      _processQueue(); // 处理下一个任务
    }
  }
  
  // 取消所有请求
  void cancelAll() {
    // 取消队列中的请求
    while (_requestQueue.isNotEmpty) {
      final task = _requestQueue.removeFirst();
      task._completer?.completeError(RequestCancelledException('Request cancelled'));
    }
    
    // 取消活跃的请求
    for (final task in _activeRequests) {
      task.cancel();
    }
    _activeRequests.clear();
  }
  
  // 获取队列状态
  RequestQueueStatus getStatus() {
    return RequestQueueStatus(
      queuedRequests: _requestQueue.length,
      activeRequests: _activeRequests.length,
      maxConcurrentRequests: _maxConcurrentRequests,
    );
  }
}

// 请求任务
abstract class RequestTask<T> {
  final String id;
  final RequestPriority priority;
  CancelToken? _cancelToken;
  Completer<T>? _completer;
  
  RequestTask({
    required this.id,
    this.priority = RequestPriority.normal,
  }) {
    _cancelToken = CancelToken();
  }
  
  Future<T> _execute();
  
  void cancel() {
    _cancelToken?.cancel('Request cancelled');
    _completer?.completeError(RequestCancelledException('Request cancelled'));
  }
  
  bool get isCancelled => _cancelToken?.isCancelled ?? false;
}

// HTTP请求任务
class HttpRequestTask<T> extends RequestTask<T> {
  final String method;
  final String url;
  final dynamic data;
  final Map<String, dynamic>? queryParameters;
  final Map<String, String>? headers;
  final DioClient client;
  
  HttpRequestTask({
    required String id,
    required this.method,
    required this.url,
    required this.client,
    this.data,
    this.queryParameters,
    this.headers,
    RequestPriority priority = RequestPriority.normal,
  }) : super(id: id, priority: priority);
  
  @override
  Future<T> _execute() async {
    final options = Options(
      method: method,
      headers: headers,
      cancelToken: _cancelToken,
    );
    
    switch (method.toUpperCase()) {
      case 'GET':
        return await client.get<T>(url, queryParameters: queryParameters, options: options);
      case 'POST':
        return await client.post<T>(url, data: data, queryParameters: queryParameters, options: options);
      case 'PUT':
        return await client.put<T>(url, data: data, queryParameters: queryParameters, options: options);
      case 'DELETE':
        return await client.delete<T>(url, data: data, queryParameters: queryParameters, options: options);
      default:
        throw UnsupportedError('HTTP method $method is not supported');
    }
  }
}

// 请求优先级
enum RequestPriority {
  low,
  normal,
  high,
  critical,
}

// 请求队列状态
class RequestQueueStatus {
  final int queuedRequests;
  final int activeRequests;
  final int maxConcurrentRequests;
  
  RequestQueueStatus({
    required this.queuedRequests,
    required this.activeRequests,
    required this.maxConcurrentRequests,
  });
  
  double get utilizationRatio => maxConcurrentRequests > 0 ? activeRequests / maxConcurrentRequests : 0.0;
  
  @override
  String toString() {
    return 'RequestQueueStatus(queued: $queuedRequests, active: $activeRequests, max: $maxConcurrentRequests, utilization: ${(utilizationRatio * 100).toStringAsFixed(1)}%)';
  }
}

// 请求取消异常
class RequestCancelledException implements Exception {
  final String message;
  
  RequestCancelledException(this.message);
  
  @override
  String toString() => 'RequestCancelledException: $message';
}
```

### 网络状态监控

```dart
// 网络状态监控器
class NetworkStatusMonitor {
  static NetworkStatusMonitor? _instance;
  static NetworkStatusMonitor get instance => _instance ??= NetworkStatusMonitor._();
  
  final StreamController<NetworkStatus> _statusController = StreamController<NetworkStatus>.broadcast();
  StreamSubscription<ConnectivityResult>? _connectivitySubscription;
  
  NetworkStatus _currentStatus = NetworkStatus.unknown;
  Timer? _statusCheckTimer;
  
  NetworkStatusMonitor._() {
    _initializeMonitoring();
  }
  
  // 网络状态流
  Stream<NetworkStatus> get statusStream => _statusController.stream;
  
  // 当前网络状态
  NetworkStatus get currentStatus => _currentStatus;
  
  // 是否有网络连接
  bool get isConnected => _currentStatus != NetworkStatus.disconnected;
  
  // 是否是WiFi连接
  bool get isWiFi => _currentStatus == NetworkStatus.wifi;
  
  // 是否是移动网络
  bool get isMobile => _currentStatus == NetworkStatus.mobile;
  
  void _initializeMonitoring() {
    // 监听连接状态变化
    _connectivitySubscription = Connectivity().onConnectivityChanged.listen(_onConnectivityChanged);
    
    // 初始状态检查
    _checkNetworkStatus();
    
    // 定期检查网络状态
    _statusCheckTimer = Timer.periodic(Duration(seconds: 30), (_) {
      _checkNetworkStatus();
    });
  }
  
  void _onConnectivityChanged(ConnectivityResult result) {
    _updateNetworkStatus(_mapConnectivityResult(result));
  }
  
  Future<void> _checkNetworkStatus() async {
    try {
      final connectivityResult = await Connectivity().checkConnectivity();
      final status = _mapConnectivityResult(connectivityResult);
      
      // 如果显示有连接，进一步验证网络可达性
      if (status != NetworkStatus.disconnected) {
        final isReachable = await _checkInternetReachability();
        if (!isReachable) {
          _updateNetworkStatus(NetworkStatus.disconnected);
          return;
        }
      }
      
      _updateNetworkStatus(status);
    } catch (e) {
      print('Failed to check network status: $e');
      _updateNetworkStatus(NetworkStatus.unknown);
    }
  }
  
  NetworkStatus _mapConnectivityResult(ConnectivityResult result) {
    switch (result) {
      case ConnectivityResult.wifi:
        return NetworkStatus.wifi;
      case ConnectivityResult.mobile:
        return NetworkStatus.mobile;
      case ConnectivityResult.ethernet:
        return NetworkStatus.ethernet;
      case ConnectivityResult.none:
        return NetworkStatus.disconnected;
      default:
        return NetworkStatus.unknown;
    }
  }
  
  Future<bool> _checkInternetReachability() async {
    try {
      final result = await InternetAddress.lookup('google.com');
      return result.isNotEmpty && result[0].rawAddress.isNotEmpty;
    } catch (e) {
      return false;
    }
  }
  
  void _updateNetworkStatus(NetworkStatus newStatus) {
    if (_currentStatus != newStatus) {
      _currentStatus = newStatus;
      _statusController.add(newStatus);
      print('Network status changed to: $newStatus');
    }
  }
  
  // 手动刷新网络状态
  Future<void> refresh() async {
    await _checkNetworkStatus();
  }
  
  // 清理资源
  void dispose() {
    _connectivitySubscription?.cancel();
    _statusCheckTimer?.cancel();
    _statusController.close();
  }
}

// 网络状态枚举
enum NetworkStatus {
  wifi,
  mobile,
  ethernet,
  disconnected,
  unknown,
}

// 网络状态扩展
extension NetworkStatusExtension on NetworkStatus {
  String get displayName {
    switch (this) {
      case NetworkStatus.wifi:
        return 'WiFi';
      case NetworkStatus.mobile:
        return 'Mobile Data';
      case NetworkStatus.ethernet:
        return 'Ethernet';
      case NetworkStatus.disconnected:
        return 'Disconnected';
      case NetworkStatus.unknown:
        return 'Unknown';
    }
  }
  
  IconData get icon {
    switch (this) {
      case NetworkStatus.wifi:
        return Icons.wifi;
      case NetworkStatus.mobile:
        return Icons.signal_cellular_4_bar;
      case NetworkStatus.ethernet:
        return Icons.ethernet;
      case NetworkStatus.disconnected:
        return Icons.wifi_off;
      case NetworkStatus.unknown:
        return Icons.help_outline;
    }
  }
  
  Color get color {
    switch (this) {
      case NetworkStatus.wifi:
      case NetworkStatus.mobile:
      case NetworkStatus.ethernet:
        return Colors.green;
      case NetworkStatus.disconnected:
        return Colors.red;
      case NetworkStatus.unknown:
        return Colors.orange;
    }
  }
}
```

## 总结

Flutter网络编程是构建现代移动应用的核心技能。通过深入理解HTTP客户端的选择和使用、网络缓存策略、WebSocket连接管理、安全认证机制以及性能优化技术，开发者可以构建出高效、安全、用户体验良好的网络层架构。

网络编程的关键要点包括：

1. **选择合适的HTTP客户端**：根据项目需求选择dart:io、http包或dio包
2. **实现有效的缓存策略**：减少网络请求，提升应用性能
3. **处理网络错误和异常**：提供良好的错误处理和用户反馈
4. **实现安全的认证机制**：保护用户数据和API安全
5. **优化网络性能**：通过请求队列、并发控制等技术提升效率
6. **监控网络状态**：根据网络状况调整应用行为

随着移动互联网技术的不断发展，网络编程也在持续演进。开发者应该关注最新的网络技术趋势，如HTTP/3、GraphQL、gRPC等，为用户提供更快速、更可靠的网络体验。setString(_accessTokenKey, _accessToken!);
    await prefs.setString(_refreshTokenKey, _refreshToken!);
    await prefs.setString(_userKey, jsonEncode(_currentUser!.toJson()));
    await prefs.setString('token_expires_at', _tokenExpiresAt!.toIso8601String());
  }
  
  // 加载认证数据
  Future<void> _loadAuthData() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      
      _accessToken = prefs.getString(_accessTokenKey);
      _refreshToken = prefs.getString(_refreshTokenKey);
      
      final userJson = prefs.getString(_userKey);
      if (userJson != null) {
        _currentUser = User.fromJson(jsonDecode(userJson));
      }
      
      final expiresAtString = prefs.getString('token_expires_at');
      if (expiresAtString != null) {
        _tokenExpiresAt = DateTime.parse(expiresAtString);
      }
      
      // 检查令牌状态
      if (isAuthenticated) {
        _authStateController.add(AuthState.authenticated);
        _scheduleTokenRefresh();
      } else if (_refreshToken != null) {
        // 尝试刷新令牌
        final refreshed = await refreshToken();
        if (refreshed) {
          _authStateController.add(AuthState.authenticated);
        } else {
          _authStateController.add(AuthState.unauthenticated);
        }
      } else {
        _authStateController.add(AuthState.unauthenticated);
      }
    } catch (e) {
      print('Failed to load auth data: $e');
      _authStateController.add(AuthState.unauthenticated);
    }
  }
  
  // 清除认证数据
  Future<void> _clearAuthData() async {
    _accessToken = null;
    _refreshToken = null;
    _currentUser = null;
    _tokenExpiresAt = null;
    
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove(_accessTokenKey);
    await prefs.remove(_refreshTokenKey);
    await prefs.remove(_userKey);
    await prefs.remove('token_expires_at');
  }
  
  // 安排令牌刷新
  void _scheduleTokenRefresh() {
    _cancelTokenRefresh();
    
    if (_tokenExpiresAt == null) return;
    
    // 在令牌过期前10分钟刷新
    final refreshTime = _tokenExpiresAt!.subtract(Duration(minutes: 10));
    final delay = refreshTime.difference(DateTime.now());
    
    if (delay.isNegative) {
      // 令牌即将过期，立即刷新
      refreshToken();
    } else {
      _tokenRefreshTimer = Timer(delay, () {
        refreshToken();
      });
    }
  }
  
  // 取消令牌刷新
  void _cancelTokenRefresh() {
    _tokenRefreshTimer?.cancel();
    _tokenRefreshTimer = null;
  }
  
  // 更新用户信息
  Future<void> updateUserProfile(Map<String, dynamic> userData) async {
    final updatedUser = await NetworkManager.instance.updateUser(userData);
    _currentUser = updatedUser;
    
    final prefs = await SharedPreferences.getInstance();
    await prefs.