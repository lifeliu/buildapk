---
layout: post
title: "Flutter数据持久化与本地存储深度解析"
categories: flutter
tags: [Flutter, SharedPreferences, SQLite, Hive, 数据持久化, 本地存储, 数据库]
date: 2022-12-03 14:30:00 +0800
image: https://flutter.dev/assets/images/shared/brand/flutter/logo/flutter-lockup.png
---

在Flutter应用开发中，数据持久化是一个至关重要的功能。无论是用户偏好设置、应用状态保存，还是复杂的业务数据存储，选择合适的本地存储方案直接影响应用的性能和用户体验。本文将深入探讨Flutter中各种数据持久化方案的原理、实现和最佳实践。

## Flutter数据持久化概述

### 数据持久化的重要性

数据持久化是指将应用程序中的数据保存到非易失性存储介质中，使得数据在应用重启或设备重启后仍然可用。在移动应用开发中，数据持久化具有以下重要意义：

1. **用户体验连续性**：保存用户的操作状态和偏好设置
2. **离线功能支持**：在网络不可用时提供基本功能
3. **性能优化**：减少网络请求，提升应用响应速度
4. **数据安全**：防止数据丢失，提供数据备份机制

### Flutter存储方案分类

Flutter提供了多种数据持久化方案，可以根据数据类型和使用场景进行分类：

#### 按数据复杂度分类

1. **简单键值对存储**：SharedPreferences
2. **结构化数据存储**：SQLite、Hive、Isar
3. **文件存储**：File I/O操作
4. **安全存储**：flutter_secure_storage

#### 按存储位置分类

1. **应用私有目录**：getApplicationDocumentsDirectory()
2. **临时目录**：getTemporaryDirectory()
3. **外部存储**：getExternalStorageDirectory()
4. **系统偏好**：SharedPreferences

## SharedPreferences详解

### SharedPreferences原理

SharedPreferences是Flutter中最简单的数据持久化方案，适用于存储简单的键值对数据。它在不同平台上有不同的实现：

- **Android**：基于Android的SharedPreferences API
- **iOS**：基于NSUserDefaults
- **Web**：基于localStorage
- **Desktop**：基于平台特定的偏好存储

### SharedPreferences实现

```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';

// SharedPreferences管理器
class PreferencesManager {
  static PreferencesManager? _instance;
  static PreferencesManager get instance => _instance ??= PreferencesManager._();
  
  SharedPreferences? _prefs;
  
  PreferencesManager._();
  
  // 初始化
  Future<void> init() async {
    _prefs ??= await SharedPreferences.getInstance();
  }
  
  // 确保已初始化
  SharedPreferences get prefs {
    if (_prefs == null) {
      throw StateError('PreferencesManager not initialized. Call init() first.');
    }
    return _prefs!;
  }
  
  // 基础数据类型存储
  Future<bool> setString(String key, String value) async {
    return await prefs.setString(key, value);
  }
  
  String? getString(String key, {String? defaultValue}) {
    return prefs.getString(key) ?? defaultValue;
  }
  
  Future<bool> setInt(String key, int value) async {
    return await prefs.setInt(key, value);
  }
  
  int? getInt(String key, {int? defaultValue}) {
    return prefs.getInt(key) ?? defaultValue;
  }
  
  Future<bool> setBool(String key, bool value) async {
    return await prefs.setBool(key, value);
  }
  
  bool? getBool(String key, {bool? defaultValue}) {
    return prefs.getBool(key) ?? defaultValue;
  }
  
  Future<bool> setDouble(String key, double value) async {
    return await prefs.setDouble(key, value);
  }
  
  double? getDouble(String key, {double? defaultValue}) {
    return prefs.getDouble(key) ?? defaultValue;
  }
  
  Future<bool> setStringList(String key, List<String> value) async {
    return await prefs.setStringList(key, value);
  }
  
  List<String>? getStringList(String key, {List<String>? defaultValue}) {
    return prefs.getStringList(key) ?? defaultValue;
  }
  
  // 复杂对象存储（JSON序列化）
  Future<bool> setObject<T>(String key, T object) async {
    try {
      final jsonString = jsonEncode(object);
      return await setString(key, jsonString);
    } catch (e) {
      print('Failed to serialize object: $e');
      return false;
    }
  }
  
  T? getObject<T>(String key, T Function(Map<String, dynamic>) fromJson, {T? defaultValue}) {
    try {
      final jsonString = getString(key);
      if (jsonString == null) return defaultValue;
      
      final jsonMap = jsonDecode(jsonString) as Map<String, dynamic>;
      return fromJson(jsonMap);
    } catch (e) {
      print('Failed to deserialize object: $e');
      return defaultValue;
    }
  }
  
  // 列表对象存储
  Future<bool> setObjectList<T>(String key, List<T> objects) async {
    try {
      final jsonList = objects.map((obj) => jsonEncode(obj)).toList();
      return await setStringList(key, jsonList);
    } catch (e) {
      print('Failed to serialize object list: $e');
      return false;
    }
  }
  
  List<T>? getObjectList<T>(String key, T Function(Map<String, dynamic>) fromJson, {List<T>? defaultValue}) {
    try {
      final jsonStringList = getStringList(key);
      if (jsonStringList == null) return defaultValue;
      
      return jsonStringList.map((jsonString) {
        final jsonMap = jsonDecode(jsonString) as Map<String, dynamic>;
        return fromJson(jsonMap);
      }).toList();
    } catch (e) {
      print('Failed to deserialize object list: $e');
      return defaultValue;
    }
  }
  
  // 删除数据
  Future<bool> remove(String key) async {
    return await prefs.remove(key);
  }
  
  // 清空所有数据
  Future<bool> clear() async {
    return await prefs.clear();
  }
  
  // 检查键是否存在
  bool containsKey(String key) {
    return prefs.containsKey(key);
  }
  
  // 获取所有键
  Set<String> getKeys() {
    return prefs.getKeys();
  }
  
  // 重新加载数据
  Future<void> reload() async {
    await prefs.reload();
  }
}

// 用户偏好设置管理
class UserPreferences {
  static const String _keyThemeMode = 'theme_mode';
  static const String _keyLanguage = 'language';
  static const String _keyFontSize = 'font_size';
  static const String _keyNotificationsEnabled = 'notifications_enabled';
  static const String _keyUserProfile = 'user_profile';
  static const String _keyRecentSearches = 'recent_searches';
  
  final PreferencesManager _prefs = PreferencesManager.instance;
  
  // 主题模式
  Future<void> setThemeMode(ThemeMode mode) async {
    await _prefs.setString(_keyThemeMode, mode.name);
  }
  
  ThemeMode getThemeMode() {
    final modeString = _prefs.getString(_keyThemeMode, defaultValue: 'system');
    return ThemeMode.values.firstWhere(
      (mode) => mode.name == modeString,
      orElse: () => ThemeMode.system,
    );
  }
  
  // 语言设置
  Future<void> setLanguage(String languageCode) async {
    await _prefs.setString(_keyLanguage, languageCode);
  }
  
  String getLanguage() {
    return _prefs.getString(_keyLanguage, defaultValue: 'en') ?? 'en';
  }
  
  // 字体大小
  Future<void> setFontSize(double fontSize) async {
    await _prefs.setDouble(_keyFontSize, fontSize);
  }
  
  double getFontSize() {
    return _prefs.getDouble(_keyFontSize, defaultValue: 14.0) ?? 14.0;
  }
  
  // 通知设置
  Future<void> setNotificationsEnabled(bool enabled) async {
    await _prefs.setBool(_keyNotificationsEnabled, enabled);
  }
  
  bool getNotificationsEnabled() {
    return _prefs.getBool(_keyNotificationsEnabled, defaultValue: true) ?? true;
  }
  
  // 用户资料
  Future<void> setUserProfile(UserProfile profile) async {
    await _prefs.setObject(_keyUserProfile, profile.toJson());
  }
  
  UserProfile? getUserProfile() {
    return _prefs.getObject(
      _keyUserProfile,
      (json) => UserProfile.fromJson(json),
    );
  }
  
  // 最近搜索
  Future<void> addRecentSearch(String query) async {
    final searches = getRecentSearches();
    searches.remove(query); // 移除重复项
    searches.insert(0, query); // 添加到开头
    
    // 限制数量
    if (searches.length > 10) {
      searches.removeRange(10, searches.length);
    }
    
    await _prefs.setStringList(_keyRecentSearches, searches);
  }
  
  List<String> getRecentSearches() {
    return _prefs.getStringList(_keyRecentSearches, defaultValue: []) ?? [];
  }
  
  Future<void> clearRecentSearches() async {
    await _prefs.remove(_keyRecentSearches);
  }
}

// 用户资料模型
class UserProfile {
  final String id;
  final String name;
  final String email;
  final String? avatarUrl;
  final DateTime lastLoginTime;
  
  UserProfile({
    required this.id,
    required this.name,
    required this.email,
    this.avatarUrl,
    required this.lastLoginTime,
  });
  
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'email': email,
      'avatarUrl': avatarUrl,
      'lastLoginTime': lastLoginTime.toIso8601String(),
    };
  }
  
  factory UserProfile.fromJson(Map<String, dynamic> json) {
    return UserProfile(
      id: json['id'],
      name: json['name'],
      email: json['email'],
      avatarUrl: json['avatarUrl'],
      lastLoginTime: DateTime.parse(json['lastLoginTime']),
    );
  }
}
```

## SQLite数据库存储

### SQLite在Flutter中的应用

SQLite是一个轻量级的关系型数据库，非常适合移动应用的本地数据存储。Flutter通过`sqflite`包提供SQLite支持，具有以下特点：

1. **ACID事务支持**：保证数据一致性
2. **SQL查询语言**：强大的数据查询和操作能力
3. **跨平台支持**：在所有Flutter支持的平台上运行
4. **高性能**：针对移动设备优化

### SQLite数据库设计与实现

```dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import 'dart:async';

// 数据库管理器
class DatabaseManager {
  static DatabaseManager? _instance;
  static DatabaseManager get instance => _instance ??= DatabaseManager._();
  
  static Database? _database;
  
  DatabaseManager._();
  
  // 数据库版本
  static const int _databaseVersion = 1;
  static const String _databaseName = 'app_database.db';
  
  // 获取数据库实例
  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
  
  // 初始化数据库
  Future<Database> _initDatabase() async {
    final databasesPath = await getDatabasesPath();
    final path = join(databasesPath, _databaseName);
    
    return await openDatabase(
      path,
      version: _databaseVersion,
      onCreate: _onCreate,
      onUpgrade: _onUpgrade,
      onConfigure: _onConfigure,
    );
  }
  
  // 配置数据库
  Future<void> _onConfigure(Database db) async {
    // 启用外键约束
    await db.execute('PRAGMA foreign_keys = ON');
  }
  
  // 创建数据库表
  Future<void> _onCreate(Database db, int version) async {
    await _createTables(db);
  }
  
  // 升级数据库
  Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
    // 根据版本号执行相应的升级脚本
    for (int version = oldVersion + 1; version <= newVersion; version++) {
      await _upgradeToVersion(db, version);
    }
  }
  
  // 创建所有表
  Future<void> _createTables(Database db) async {
    // 用户表
    await db.execute('''
      CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        avatar_url TEXT,
        created_at INTEGER NOT NULL,
        updated_at INTEGER NOT NULL
      )
    ''');
    
    // 笔记表
    await db.execute('''
      CREATE TABLE notes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        title TEXT NOT NULL,
        content TEXT NOT NULL,
        category_id INTEGER,
        is_favorite INTEGER NOT NULL DEFAULT 0,
        created_at INTEGER NOT NULL,
        updated_at INTEGER NOT NULL,
        FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE,
        FOREIGN KEY (category_id) REFERENCES categories (id) ON DELETE SET NULL
      )
    ''');
    
    // 分类表
    await db.execute('''
      CREATE TABLE categories (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        color TEXT NOT NULL,
        icon TEXT,
        created_at INTEGER NOT NULL
      )
    ''');
    
    // 标签表
    await db.execute('''
      CREATE TABLE tags (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT UNIQUE NOT NULL,
        created_at INTEGER NOT NULL
      )
    ''');
    
    // 笔记标签关联表
    await db.execute('''
      CREATE TABLE note_tags (
        note_id INTEGER NOT NULL,
        tag_id INTEGER NOT NULL,
        PRIMARY KEY (note_id, tag_id),
        FOREIGN KEY (note_id) REFERENCES notes (id) ON DELETE CASCADE,
        FOREIGN KEY (tag_id) REFERENCES tags (id) ON DELETE CASCADE
      )
    ''');
    
    // 创建索引
    await _createIndexes(db);
  }
  
  // 创建索引
  Future<void> _createIndexes(Database db) async {
    await db.execute('CREATE INDEX idx_notes_user_id ON notes (user_id)');
    await db.execute('CREATE INDEX idx_notes_category_id ON notes (category_id)');
    await db.execute('CREATE INDEX idx_notes_created_at ON notes (created_at)');
    await db.execute('CREATE INDEX idx_notes_title ON notes (title)');
  }
  
  // 版本升级
  Future<void> _upgradeToVersion(Database db, int version) async {
    switch (version) {
      case 2:
        // 添加新列或表
        await db.execute('ALTER TABLE notes ADD COLUMN sync_status INTEGER DEFAULT 0');
        break;
      case 3:
        // 其他升级操作
        break;
    }
  }
  
  // 关闭数据库
  Future<void> close() async {
    final db = _database;
    if (db != null) {
      await db.close();
      _database = null;
    }
  }
  
  // 删除数据库
  Future<void> deleteDatabase() async {
    final databasesPath = await getDatabasesPath();
    final path = join(databasesPath, _databaseName);
    await databaseFactory.deleteDatabase(path);
    _database = null;
  }
}

// 基础DAO类
abstract class BaseDao<T> {
  final DatabaseManager _dbManager = DatabaseManager.instance;
  
  String get tableName;
  T fromMap(Map<String, dynamic> map);
  Map<String, dynamic> toMap(T entity);
  
  Future<Database> get _db async => await _dbManager.database;
  
  // 插入
  Future<int> insert(T entity) async {
    final db = await _db;
    final map = toMap(entity);
    map['created_at'] = DateTime.now().millisecondsSinceEpoch;
    map['updated_at'] = DateTime.now().millisecondsSinceEpoch;
    return await db.insert(tableName, map);
  }
  
  // 批量插入
  Future<void> insertBatch(List<T> entities) async {
    final db = await _db;
    final batch = db.batch();
    
    for (final entity in entities) {
      final map = toMap(entity);
      map['created_at'] = DateTime.now().millisecondsSinceEpoch;
      map['updated_at'] = DateTime.now().millisecondsSinceEpoch;
      batch.insert(tableName, map);
    }
    
    await batch.commit(noResult: true);
  }
  
  // 更新
  Future<int> update(T entity, String whereClause, List<dynamic> whereArgs) async {
    final db = await _db;
    final map = toMap(entity);
    map['updated_at'] = DateTime.now().millisecondsSinceEpoch;
    return await db.update(tableName, map, where: whereClause, whereArgs: whereArgs);
  }
  
  // 删除
  Future<int> delete(String whereClause, List<dynamic> whereArgs) async {
    final db = await _db;
    return await db.delete(tableName, where: whereClause, whereArgs: whereArgs);
  }
  
  // 查询单个
  Future<T?> findOne(String whereClause, List<dynamic> whereArgs) async {
    final db = await _db;
    final maps = await db.query(
      tableName,
      where: whereClause,
      whereArgs: whereArgs,
      limit: 1,
    );
    
    if (maps.isNotEmpty) {
      return fromMap(maps.first);
    }
    return null;
  }
  
  // 查询多个
  Future<List<T>> findAll({
    String? where,
    List<dynamic>? whereArgs,
    String? orderBy,
    int? limit,
    int? offset,
  }) async {
    final db = await _db;
    final maps = await db.query(
      tableName,
      where: where,
      whereArgs: whereArgs,
      orderBy: orderBy,
      limit: limit,
      offset: offset,
    );
    
    return maps.map((map) => fromMap(map)).toList();
  }
  
  // 计数
  Future<int> count({String? where, List<dynamic>? whereArgs}) async {
    final db = await _db;
    final result = await db.query(
      tableName,
      columns: ['COUNT(*) as count'],
      where: where,
      whereArgs: whereArgs,
    );
    
    return Sqflite.firstIntValue(result) ?? 0;
  }
  
  // 检查是否存在
  Future<bool> exists(String whereClause, List<dynamic> whereArgs) async {
    final count = await this.count(where: whereClause, whereArgs: whereArgs);
    return count > 0;
  }
}

// 笔记模型
class Note {
  final int? id;
  final int userId;
  final String title;
  final String content;
  final int? categoryId;
  final bool isFavorite;
  final DateTime createdAt;
  final DateTime updatedAt;
  
  Note({
    this.id,
    required this.userId,
    required this.title,
    required this.content,
    this.categoryId,
    this.isFavorite = false,
    required this.createdAt,
    required this.updatedAt,
  });
  
  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'user_id': userId,
      'title': title,
      'content': content,
      'category_id': categoryId,
      'is_favorite': isFavorite ? 1 : 0,
      'created_at': createdAt.millisecondsSinceEpoch,
      'updated_at': updatedAt.millisecondsSinceEpoch,
    };
  }
  
  factory Note.fromMap(Map<String, dynamic> map) {
    return Note(
      id: map['id'],
      userId: map['user_id'],
      title: map['title'],
      content: map['content'],
      categoryId: map['category_id'],
      isFavorite: map['is_favorite'] == 1,
      createdAt: DateTime.fromMillisecondsSinceEpoch(map['created_at']),
      updatedAt: DateTime.fromMillisecondsSinceEpoch(map['updated_at']),
    );
  }
  
  Note copyWith({
    int? id,
    int? userId,
    String? title,
    String? content,
    int? categoryId,
    bool? isFavorite,
    DateTime? createdAt,
    DateTime? updatedAt,
  }) {
    return Note(
      id: id ?? this.id,
      userId: userId ?? this.userId,
      title: title ?? this.title,
      content: content ?? this.content,
      categoryId: categoryId ?? this.categoryId,
      isFavorite: isFavorite ?? this.isFavorite,
      createdAt: createdAt ?? this.createdAt,
      updatedAt: updatedAt ?? this.updatedAt,
    );
  }
}

// 笔记DAO
class NoteDao extends BaseDao<Note> {
  @override
  String get tableName => 'notes';
  
  @override
  Note fromMap(Map<String, dynamic> map) => Note.fromMap(map);
  
  @override
  Map<String, dynamic> toMap(Note entity) => entity.toMap();
  
  // 根据ID查找笔记
  Future<Note?> findById(int id) async {
    return await findOne('id = ?', [id]);
  }
  
  // 根据用户ID查找笔记
  Future<List<Note>> findByUserId(int userId, {
    String? orderBy = 'updated_at DESC',
    int? limit,
    int? offset,
  }) async {
    return await findAll(
      where: 'user_id = ?',
      whereArgs: [userId],
      orderBy: orderBy,
      limit: limit,
      offset: offset,
    );
  }
  
  // 搜索笔记
  Future<List<Note>> search(int userId, String query, {
    int? limit = 50,
    int? offset,
  }) async {
    return await findAll(
      where: 'user_id = ? AND (title LIKE ? OR content LIKE ?)',
      whereArgs: [userId, '%$query%', '%$query%'],
      orderBy: 'updated_at DESC',
      limit: limit,
      offset: offset,
    );
  }
  
  // 获取收藏的笔记
  Future<List<Note>> findFavorites(int userId) async {
    return await findAll(
      where: 'user_id = ? AND is_favorite = 1',
      whereArgs: [userId],
      orderBy: 'updated_at DESC',
    );
  }
  
  // 根据分类查找笔记
  Future<List<Note>> findByCategory(int userId, int categoryId) async {
    return await findAll(
      where: 'user_id = ? AND category_id = ?',
      whereArgs: [userId, categoryId],
      orderBy: 'updated_at DESC',
    );
  }
  
  // 更新笔记
  Future<int> updateNote(Note note) async {
    if (note.id == null) {
      throw ArgumentError('Note ID cannot be null for update');
    }
    
    final updatedNote = note.copyWith(updatedAt: DateTime.now());
    return await update(updatedNote, 'id = ?', [note.id]);
  }
  
  // 删除笔记
  Future<int> deleteById(int id) async {
    return await delete('id = ?', [id]);
  }
  
  // 切换收藏状态
  Future<int> toggleFavorite(int id) async {
    final db = await _db;
    return await db.rawUpdate(
      'UPDATE notes SET is_favorite = NOT is_favorite, updated_at = ? WHERE id = ?',
      [DateTime.now().millisecondsSinceEpoch, id],
    );
  }
  
  // 获取统计信息
  Future<Map<String, int>> getStatistics(int userId) async {
    final db = await _db;
    
    final totalResult = await db.rawQuery(
      'SELECT COUNT(*) as total FROM notes WHERE user_id = ?',
      [userId],
    );
    
    final favoritesResult = await db.rawQuery(
      'SELECT COUNT(*) as favorites FROM notes WHERE user_id = ? AND is_favorite = 1',
      [userId],
    );
    
    final categoriesResult = await db.rawQuery(
      'SELECT COUNT(DISTINCT category_id) as categories FROM notes WHERE user_id = ? AND category_id IS NOT NULL',
      [userId],
    );
    
    return {
      'total': Sqflite.firstIntValue(totalResult) ?? 0,
      'favorites': Sqflite.firstIntValue(favoritesResult) ?? 0,
      'categories': Sqflite.firstIntValue(categoriesResult) ?? 0,
    };
  }
}
```

## Hive数据库详解

### Hive的优势与特点

Hive是一个轻量级、快速的NoSQL数据库，专为Flutter设计。相比SQLite，Hive具有以下优势：

1. **纯Dart实现**：无需原生代码，支持所有平台
2. **高性能**：比SQLite快数倍
3. **类型安全**：强类型支持，编译时检查
4. **简单易用**：API简洁，学习成本低
5. **加密支持**：内置数据加密功能

### Hive实现详解

```dart
import 'package:hive/hive.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:crypto/crypto.dart';
import 'dart:convert';
import 'dart:typed_data';

// Hive数据库管理器
class HiveManager {
  static HiveManager? _instance;
  static HiveManager get instance => _instance ??= HiveManager._();
  
  HiveManager._();
  
  // 初始化Hive
  Future<void> init() async {
    await Hive.initFlutter();
    
    // 注册适配器
    _registerAdapters();
    
    // 打开必要的盒子
    await _openBoxes();
  }
  
  // 注册类型适配器
  void _registerAdapters() {
    if (!Hive.isAdapterRegistered(0)) {
      Hive.registerAdapter(TaskAdapter());
    }
    if (!Hive.isAdapterRegistered(1)) {
      Hive.registerAdapter(TaskPriorityAdapter());
    }
    if (!Hive.isAdapterRegistered(2)) {
      Hive.registerAdapter(TaskStatusAdapter());
    }
    if (!Hive.isAdapterRegistered(3)) {
      Hive.registerAdapter(UserSettingsAdapter());
    }
  }
  
  // 打开数据盒子
  Future<void> _openBoxes() async {
    // 普通盒子
    await Hive.openBox<Task>('tasks');
    await Hive.openBox<UserSettings>('user_settings');
    
    // 加密盒子
    final encryptionKey = await _getEncryptionKey();
    await Hive.openBox<String>('secure_data', encryptionCipher: HiveAesCipher(encryptionKey));
  }
  
  // 获取加密密钥
  Future<Uint8List> _getEncryptionKey() async {
    // 在实际应用中，应该从安全存储中获取密钥
    const keyString = 'your-32-character-secret-key-here';
    final key = sha256.convert(utf8.encode(keyString)).bytes;
    return Uint8List.fromList(key);
  }
  
  // 获取盒子
  Box<T> getBox<T>(String name) {
    return Hive.box<T>(name);
  }
  
  // 关闭所有盒子
  Future<void> closeAll() async {
    await Hive.close();
  }
  
  // 删除盒子
  Future<void> deleteBox(String name) async {
    await Hive.deleteBoxFromDisk(name);
  }
  
  // 压缩盒子
  Future<void> compactBox(String name) async {
    final box = Hive.box(name);
    await box.compact();
  }
}

// 任务优先级枚举
enum TaskPriority {
  low,
  medium,
  high,
  urgent,
}

// 任务状态枚举
enum TaskStatus {
  pending,
  inProgress,
  completed,
  cancelled,
}

// 任务模型
@HiveType(typeId: 0)
class Task extends HiveObject {
  @HiveField(0)
  String id;
  
  @HiveField(1)
  String title;
  
  @HiveField(2)
  String description;
  
  @HiveField(3)
  TaskPriority priority;
  
  @HiveField(4)
  TaskStatus status;
  
  @HiveField(5)
  DateTime createdAt;
  
  @HiveField(6)
  DateTime updatedAt;
  
  @HiveField(7)
  DateTime? dueDate;
  
  @HiveField(8)
  List<String> tags;
  
  @HiveField(9)
  bool isCompleted;
  
  Task({
    required this.id,
    required this.title,
    required this.description,
    required this.priority,
    required this.status,
    required this.createdAt,
    required this.updatedAt,
    this.dueDate,
    required this.tags,
    required this.isCompleted,
  });
  
  // 复制方法
  Task copyWith({
    String? id,
    String? title,
    String? description,
    TaskPriority? priority,
    TaskStatus? status,
    DateTime? createdAt,
    DateTime? updatedAt,
    DateTime? dueDate,
    List<String>? tags,
    bool? isCompleted,
  }) {
    return Task(
      id: id ?? this.id,
      title: title ?? this.title,
      description: description ?? this.description,
      priority: priority ?? this.priority,
      status: status ?? this.status,
      createdAt: createdAt ?? this.createdAt,
      updatedAt: updatedAt ?? this.updatedAt,
      dueDate: dueDate ?? this.dueDate,
      tags: tags ?? List.from(this.tags),
      isCompleted: isCompleted ?? this.isCompleted,
    );
  }
  
  @override
  String toString() {
    return 'Task(id: $id, title: $title, priority: $priority, status: $status)';
  }
}

// 用户设置模型
@HiveType(typeId: 3)
class UserSettings extends HiveObject {
  @HiveField(0)
  String theme;
  
  @HiveField(1)
  String language;
  
  @HiveField(2)
  bool notificationsEnabled;
  
  @HiveField(3)
  double fontSize;
  
  @HiveField(4)
  Map<String, dynamic> customSettings;
  
  UserSettings({
    required this.theme,
    required this.language,
    required this.notificationsEnabled,
    required this.fontSize,
    required this.customSettings,
  });
}

// 任务仓库
class TaskRepository {
  final Box<Task> _taskBox;
  
  TaskRepository() : _taskBox = Hive.box<Task>('tasks');
  
  // 添加任务
  Future<void> addTask(Task task) async {
    await _taskBox.put(task.id, task);
  }
  
  // 获取任务
  Task? getTask(String id) {
    return _taskBox.get(id);
  }
  
  // 获取所有任务
  List<Task> getAllTasks() {
    return _taskBox.values.toList();
  }
  
  // 更新任务
  Future<void> updateTask(Task task) async {
    task.updatedAt = DateTime.now();
    await task.save(); // 使用HiveObject的save方法
  }
  
  // 删除任务
  Future<void> deleteTask(String id) async {
    await _taskBox.delete(id);
  }
  
  // 根据状态筛选任务
  List<Task> getTasksByStatus(TaskStatus status) {
    return _taskBox.values.where((task) => task.status == status).toList();
  }
  
  // 根据优先级筛选任务
  List<Task> getTasksByPriority(TaskPriority priority) {
    return _taskBox.values.where((task) => task.priority == priority).toList();
  }
  
  // 搜索任务
  List<Task> searchTasks(String query) {
    final lowerQuery = query.toLowerCase();
    return _taskBox.values.where((task) {
      return task.title.toLowerCase().contains(lowerQuery) ||
             task.description.toLowerCase().contains(lowerQuery) ||
             task.tags.any((tag) => tag.toLowerCase().contains(lowerQuery));
    }).toList();
  }
  
  // 获取即将到期的任务
  List<Task> getUpcomingTasks({int days = 7}) {
    final now = DateTime.now();
    final futureDate = now.add(Duration(days: days));
    
    return _taskBox.values.where((task) {
      if (task.dueDate == null || task.isCompleted) return false;
      return task.dueDate!.isAfter(now) && task.dueDate!.isBefore(futureDate);
    }).toList();
  }
  
  // 获取过期任务
  List<Task> getOverdueTasks() {
    final now = DateTime.now();
    return _taskBox.values.where((task) {
      if (task.dueDate == null || task.isCompleted) return false;
      return task.dueDate!.isBefore(now);
    }).toList();
  }
  
  // 标记任务完成
  Future<void> markTaskCompleted(String id) async {
    final task = getTask(id);
    if (task != null) {
      task.isCompleted = true;
      task.status = TaskStatus.completed;
      task.updatedAt = DateTime.now();
      await task.save();
    }
  }
  
  // 获取统计信息
  Map<String, int> getStatistics() {
    final tasks = getAllTasks();
    return {
      'total': tasks.length,
      'completed': tasks.where((t) => t.isCompleted).length,
      'pending': tasks.where((t) => t.status == TaskStatus.pending).length,
      'inProgress': tasks.where((t) => t.status == TaskStatus.inProgress).length,
      'overdue': getOverdueTasks().length,
    };
  }
  
  // 清空所有任务
  Future<void> clearAllTasks() async {
    await _taskBox.clear();
  }
  
  // 监听任务变化
  Stream<BoxEvent> watchTasks() {
    return _taskBox.watch();
  }
  
  // 批量操作
  Future<void> batchUpdate(List<Task> tasks) async {
    final Map<String, Task> taskMap = {};
    for (final task in tasks) {
      task.updatedAt = DateTime.now();
      taskMap[task.id] = task;
    }
    await _taskBox.putAll(taskMap);
  }
  
  // 导出数据
  Map<String, dynamic> exportData() {
    final tasks = getAllTasks();
    return {
      'tasks': tasks.map((task) => {
        'id': task.id,
        'title': task.title,
        'description': task.description,
        'priority': task.priority.name,
        'status': task.status.name,
        'createdAt': task.createdAt.toIso8601String(),
        'updatedAt': task.updatedAt.toIso8601String(),
        'dueDate': task.dueDate?.toIso8601String(),
        'tags': task.tags,
        'isCompleted': task.isCompleted,
      }).toList(),
      'exportedAt': DateTime.now().toIso8601String(),
    };
  }
  
  // 导入数据
  Future<void> importData(Map<String, dynamic> data) async {
    final tasksData = data['tasks'] as List<dynamic>;
    final tasks = tasksData.map((taskData) {
      return Task(
        id: taskData['id'],
        title: taskData['title'],
        description: taskData['description'],
        priority: TaskPriority.values.firstWhere(
          (p) => p.name == taskData['priority'],
          orElse: () => TaskPriority.medium,
        ),
        status: TaskStatus.values.firstWhere(
          (s) => s.name == taskData['status'],
          orElse: () => TaskStatus.pending,
        ),
        createdAt: DateTime.parse(taskData['createdAt']),
        updatedAt: DateTime.parse(taskData['updatedAt']),
        dueDate: taskData['dueDate'] != null ? DateTime.parse(taskData['dueDate']) : null,
        tags: List<String>.from(taskData['tags']),
        isCompleted: taskData['isCompleted'],
      );
    }).toList();
    
    await batchUpdate(tasks);
  }
}
```

## 文件存储系统

### 文件存储的应用场景

文件存储适用于以下场景：

1. **大型数据文件**：图片、视频、音频等媒体文件
2. **配置文件**：JSON、XML等格式的配置数据
3. **缓存文件**：临时数据和缓存内容
4. **日志文件**：应用运行日志和错误记录
5. **导入导出**：数据的备份和恢复

### 文件存储实现

```dart
import 'dart:io';
import 'dart:convert';
import 'dart:typed_data';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as path;
import 'package:crypto/crypto.dart';

// 文件存储管理器
class FileStorageManager {
  static FileStorageManager? _instance;
  static FileStorageManager get instance => _instance ??= FileStorageManager._();
  
  FileStorageManager._();
  
  // 获取应用文档目录
  Future<Directory> get _documentsDirectory async {
    return await getApplicationDocumentsDirectory();
  }
  
  // 获取临时目录
  Future<Directory> get _temporaryDirectory async {
    return await getTemporaryDirectory();
  }
  
  // 获取缓存目录
  Future<Directory> get _cacheDirectory async {
    return await getTemporaryDirectory(); // 在某些平台上可能不同
  }
  
  // 创建目录
  Future<Directory> createDirectory(String relativePath, {bool inTemp = false}) async {
    final baseDir = inTemp ? await _temporaryDirectory : await _documentsDirectory;
    final dir = Directory(path.join(baseDir.path, relativePath));
    
    if (!await dir.exists()) {
      await dir.create(recursive: true);
    }
    
    return dir;
  }
  
  // 获取文件路径
  Future<String> getFilePath(String fileName, {String? subDirectory, bool inTemp = false}) async {
    final baseDir = inTemp ? await _temporaryDirectory : await _documentsDirectory;
    
    if (subDirectory != null) {
      final dir = await createDirectory(subDirectory, inTemp: inTemp);
      return path.join(dir.path, fileName);
    }
    
    return path.join(baseDir.path, fileName);
  }
  
  // 写入文本文件
  Future<File> writeTextFile(String fileName, String content, {
    String? subDirectory,
    bool inTemp = false,
    Encoding encoding = utf8,
  }) async {
    final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
    final file = File(filePath);
    return await file.writeAsString(content, encoding: encoding);
  }
  
  // 读取文本文件
  Future<String?> readTextFile(String fileName, {
    String? subDirectory,
    bool inTemp = false,
    Encoding encoding = utf8,
  }) async {
    try {
      final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
      final file = File(filePath);
      
      if (await file.exists()) {
        return await file.readAsString(encoding: encoding);
      }
      return null;
    } catch (e) {
      print('Error reading text file: $e');
      return null;
    }
  }
  
  // 写入二进制文件
  Future<File> writeBinaryFile(String fileName, Uint8List data, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
    final file = File(filePath);
    return await file.writeAsBytes(data);
  }
  
  // 读取二进制文件
  Future<Uint8List?> readBinaryFile(String fileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    try {
      final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
      final file = File(filePath);
      
      if (await file.exists()) {
        return await file.readAsBytes();
      }
      return null;
    } catch (e) {
      print('Error reading binary file: $e');
      return null;
    }
  }
  
  // 写入JSON文件
  Future<File> writeJsonFile(String fileName, Map<String, dynamic> data, {
    String? subDirectory,
    bool inTemp = false,
    bool prettyPrint = false,
  }) async {
    final jsonString = prettyPrint 
        ? JsonEncoder.withIndent('  ').convert(data)
        : jsonEncode(data);
    return await writeTextFile(fileName, jsonString, subDirectory: subDirectory, inTemp: inTemp);
  }
  
  // 读取JSON文件
  Future<Map<String, dynamic>?> readJsonFile(String fileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    try {
      final content = await readTextFile(fileName, subDirectory: subDirectory, inTemp: inTemp);
      if (content != null) {
        return jsonDecode(content) as Map<String, dynamic>;
      }
      return null;
    } catch (e) {
      print('Error reading JSON file: $e');
      return null;
    }
  }
  
  // 复制文件
  Future<File> copyFile(String sourcePath, String destinationFileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    final sourceFile = File(sourcePath);
    final destPath = await getFilePath(destinationFileName, subDirectory: subDirectory, inTemp: inTemp);
    return await sourceFile.copy(destPath);
  }
  
  // 移动文件
  Future<File> moveFile(String sourcePath, String destinationFileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    final sourceFile = File(sourcePath);
    final destPath = await getFilePath(destinationFileName, subDirectory: subDirectory, inTemp: inTemp);
    return await sourceFile.rename(destPath);
  }
  
  // 删除文件
  Future<bool> deleteFile(String fileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    try {
      final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
      final file = File(filePath);
      
      if (await file.exists()) {
        await file.delete();
        return true;
      }
      return false;
    } catch (e) {
      print('Error deleting file: $e');
      return false;
    }
  }
  
  // 检查文件是否存在
  Future<bool> fileExists(String fileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
    final file = File(filePath);
    return await file.exists();
  }
  
  // 获取文件大小
  Future<int?> getFileSize(String fileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    try {
      final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
      final file = File(filePath);
      
      if (await file.exists()) {
        return await file.length();
      }
      return null;
    } catch (e) {
      print('Error getting file size: $e');
      return null;
    }
  }
  
  // 获取文件修改时间
  Future<DateTime?> getFileModifiedTime(String fileName, {
    String? subDirectory,
    bool inTemp = false,
  }) async {
    try {
      final filePath = await getFilePath(fileName, subDirectory: subDirectory, inTemp: inTemp);
      final file = File(filePath);
      
      if (await file.exists()) {
        final stat = await file.stat();
        return stat.modified;
      }
      return null;
    } catch (e) {
      print('Error getting file modified time: $e');
      return null;
    }
  }
  
  // 列出目录中的文件
  Future<List<FileSystemEntity>> listFiles({
    String? subDirectory,
    bool inTemp = false,
    bool recursive = false,
  }) async {
    try {
      final baseDir = inTemp ? await _temporaryDirectory : await _documentsDirectory;
      final dir = subDirectory != null 
          ? Directory(path.join(baseDir.path, subDirectory))
          : baseDir;
      
      if (await dir.exists()) {
        return await dir.list(recursive: recursive).toList();
      }
      return [];
    } catch (e) {
      print('Error listing files: $e');
      return [];
    }
  }
  
  // 计算文件哈希值
  Future<String?> calculateFileHash(String fileName, {
    String? subDirectory,
    bool inTemp = false,
    Hash hashAlgorithm = sha256,
  }) async {
    try {
      final data = await readBinaryFile(fileName, subDirectory: subDirectory, inTemp: inTemp);
      if (data != null) {
        final digest = hashAlgorithm.convert(data);
        return digest.toString();
      }
      return null;
    } catch (e) {
      print('Error calculating file hash: $e');
      return null;
    }
  }
  
  // 清理临时文件
  Future<void> cleanupTempFiles({Duration? olderThan}) async {
    try {
      final tempDir = await _temporaryDirectory;
      final files = await tempDir.list().toList();
      
      for (final file in files) {
        if (file is File) {
          if (olderThan != null) {
            final stat = await file.stat();
            final age = DateTime.now().difference(stat.modified);
            if (age > olderThan) {
              await file.delete();
            }
          } else {
            await file.delete();
          }
        }
      }
    } catch (e) {
      print('Error cleaning up temp files: $e');
    }
  }
  
  // 获取目录大小
  Future<int> getDirectorySize({
    String? subDirectory,
    bool inTemp = false,
  }) async {
    try {
      final files = await listFiles(subDirectory: subDirectory, inTemp: inTemp, recursive: true);
      int totalSize = 0;
      
      for (final file in files) {
        if (file is File) {
          final stat = await file.stat();
          totalSize += stat.size;
        }
      }
      
      return totalSize;
    } catch (e) {
      print('Error calculating directory size: $e');
      return 0;
    }
  }
}

// 缓存管理器
class CacheManager {
  final FileStorageManager _fileManager = FileStorageManager.instance;
  static const String _cacheSubDirectory = 'cache';
  
  // 缓存数据
  Future<void> cacheData(String key, Map<String, dynamic> data, {
    Duration? expiry,
  }) async {
    final cacheData = {
      'data': data,
      'cachedAt': DateTime.now().toIso8601String(),
      'expiryAt': expiry != null ? DateTime.now().add(expiry).toIso8601String() : null,
    };
    
    await _fileManager.writeJsonFile(
      '$key.cache',
      cacheData,
      subDirectory: _cacheSubDirectory,
      inTemp: true,
    );
  }
  
  // 获取缓存数据
  Future<Map<String, dynamic>?> getCachedData(String key) async {
    try {
      final cacheData = await _fileManager.readJsonFile(
        '$key.cache',
        subDirectory: _cacheSubDirectory,
        inTemp: true,
      );
      
      if (cacheData == null) return null;
      
      // 检查是否过期
      final expiryString = cacheData['expiryAt'] as String?;
      if (expiryString != null) {
        final expiry = DateTime.parse(expiryString);
        if (DateTime.now().isAfter(expiry)) {
          await removeCachedData(key);
          return null;
        }
      }
      
      return cacheData['data'] as Map<String, dynamic>;
    } catch (e) {
      print('Error getting cached data: $e');
      return null;
    }
  }
  
  // 删除缓存数据
  Future<void> removeCachedData(String key) async {
    await _fileManager.deleteFile(
      '$key.cache',
      subDirectory: _cacheSubDirectory,
      inTemp: true,
    );
  }
  
  // 清理过期缓存
  Future<void> cleanupExpiredCache() async {
    try {
      final files = await _fileManager.listFiles(
        subDirectory: _cacheSubDirectory,
        inTemp: true,
      );
      
      for (final file in files) {
        if (file is File && file.path.endsWith('.cache')) {
          final fileName = path.basename(file.path);
          final key = fileName.replaceAll('.cache', '');
          
          // 尝试读取缓存数据检查过期时间
          final cachedData = await getCachedData(key);
          // getCachedData会自动删除过期的缓存
        }
      }
    } catch (e) {
      print('Error cleaning up expired cache: $e');
    }
  }
  
  // 清理所有缓存
  Future<void> clearAllCache() async {
    try {
      final files = await _fileManager.listFiles(
        subDirectory: _cacheSubDirectory,
        inTemp: true,
      );
      
      for (final file in files) {
        if (file is File) {
          await file.delete();
        }
      }
    } catch (e) {
      print('Error clearing all cache: $e');
    }
  }
  
  // 获取缓存大小
  Future<int> getCacheSize() async {
    return await _fileManager.getDirectorySize(
      subDirectory: _cacheSubDirectory,
      inTemp: true,
    );
  }
}
```

## 安全存储解决方案

### flutter_secure_storage详解

对于敏感数据（如密码、令牌、密钥等），需要使用安全存储方案。`flutter_secure_storage`提供了跨平台的安全存储功能：

- **Android**：使用Android Keystore
- **iOS**：使用iOS Keychain
- **Web**：使用加密的localStorage
- **Desktop**：使用平台特定的安全存储

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'dart:convert';

// 安全存储管理器
class SecureStorageManager {
  static SecureStorageManager? _instance;
  static SecureStorageManager get instance => _instance ??= SecureStorageManager._();
  
  late final FlutterSecureStorage _storage;
  
  SecureStorageManager._() {
    _storage = FlutterSecureStorage(
      aOptions: AndroidOptions(
        encryptedSharedPreferences: true,
        sharedPreferencesName: 'secure_prefs',
        preferencesKeyPrefix: 'secure_',
      ),
      iOptions: IOSOptions(
        groupId: 'group.com.yourapp.secure',
        accountName: 'secure_storage',
        accessibility: IOSAccessibility.first_unlock_this_device,
      ),
      webOptions: WebOptions(
        dbName: 'secure_storage',
        publicKey: 'your_public_key_here',
      ),
    );
  }
  
  // 存储字符串
  Future<void> writeString(String key, String value) async {
    await _storage.write(key: key, value: value);
  }
  
  // 读取字符串
  Future<String?> readString(String key) async {
    return await _storage.read(key: key);
  }
  
  // 存储对象（JSON序列化）
  Future<void> writeObject(String key, Map<String, dynamic> object) async {
    final jsonString = jsonEncode(object);
    await writeString(key, jsonString);
  }
  
  // 读取对象
  Future<Map<String, dynamic>?> readObject(String key) async {
    final jsonString = await readString(key);
    if (jsonString != null) {
      try {
        return jsonDecode(jsonString) as Map<String, dynamic>;
      } catch (e) {
        print('Error decoding JSON: $e');
      }
    }
    return null;
  }
  
  // 删除数据
  Future<void> delete(String key) async {
    await _storage.delete(key: key);
  }
  
  // 删除所有数据
  Future<void> deleteAll() async {
    await _storage.deleteAll();
  }
  
  // 检查键是否存在
  Future<bool> containsKey(String key) async {
    return await _storage.containsKey(key: key);
  }
  
  // 获取所有键
  Future<Map<String, String>> readAll() async {
    return await _storage.readAll();
  }
}

// 认证信息管理
class AuthCredentialsManager {
  static const String _accessTokenKey = 'access_token';
  static const String _refreshTokenKey = 'refresh_token';
  static const String _userCredentialsKey = 'user_credentials';
  static const String _biometricKey = 'biometric_enabled';
  
  final SecureStorageManager _secureStorage = SecureStorageManager.instance;
  
  // 保存访问令牌
  Future<void> saveAccessToken(String token) async {
    await _secureStorage.writeString(_accessTokenKey, token);
  }
  
  // 获取访问令牌
  Future<String?> getAccessToken() async {
    return await _secureStorage.readString(_accessTokenKey);
  }
  
  // 保存刷新令牌
  Future<void> saveRefreshToken(String token) async {
    await _secureStorage.writeString(_refreshTokenKey, token);
  }
  
  // 获取刷新令牌
  Future<String?> getRefreshToken() async {
    return await _secureStorage.readString(_refreshTokenKey);
  }
  
  // 保存用户凭据
  Future<void> saveUserCredentials(UserCredentials credentials) async {
    await _secureStorage.writeObject(_userCredentialsKey, credentials.toJson());
  }
  
  // 获取用户凭据
  Future<UserCredentials?> getUserCredentials() async {
    final data = await _secureStorage.readObject(_userCredentialsKey);
    if (data != null) {
      return UserCredentials.fromJson(data);
    }
    return null;
  }
  
  // 清除所有认证信息
  Future<void> clearAllCredentials() async {
    await _secureStorage.delete(_accessTokenKey);
    await _secureStorage.delete(_refreshTokenKey);
    await _secureStorage.delete(_userCredentialsKey);
  }
  
  // 检查是否已登录
  Future<bool> isLoggedIn() async {
    final token = await getAccessToken();
    return token != null && token.isNotEmpty;
  }
}

// 用户凭据模型
class UserCredentials {
  final String username;
  final String email;
  final String? encryptedPassword;
  final DateTime lastLoginTime;
  
  UserCredentials({
    required this.username,
    required this.email,
    this.encryptedPassword,
    required this.lastLoginTime,
  });
  
  Map<String, dynamic> toJson() {
    return {
      'username': username,
      'email': email,
      'encryptedPassword': encryptedPassword,
      'lastLoginTime': lastLoginTime.toIso8601String(),
    };
  }
  
  factory UserCredentials.fromJson(Map<String, dynamic> json) {
    return UserCredentials(
      username: json['username'],
      email: json['email'],
      encryptedPassword: json['encryptedPassword'],
      lastLoginTime: DateTime.parse(json['lastLoginTime']),
    );
  }
}
```

## 数据同步与备份策略

### 数据同步架构

在现代移动应用中，数据同步是确保用户数据一致性和可用性的关键功能。以下是一个完整的数据同步解决方案：

```dart
import 'dart:async';
import 'package:connectivity_plus/connectivity_plus.dart';

// 同步状态枚举
enum SyncStatus {
  idle,
  syncing,
  success,
  failed,
  conflict,
}

// 同步策略枚举
enum SyncStrategy {
  manual,
  automatic,
  wifiOnly,
  backgroundSync,
}

// 数据同步管理器
class DataSyncManager {
  static DataSyncManager? _instance;
  static DataSyncManager get instance => _instance ??= DataSyncManager._();
  
  final StreamController<SyncStatus> _statusController = StreamController<SyncStatus>.broadcast();
  final StreamController<double> _progressController = StreamController<double>.broadcast();
  
  SyncStatus _currentStatus = SyncStatus.idle;
  SyncStrategy _syncStrategy = SyncStrategy.automatic;
  Timer? _autoSyncTimer;
  
  DataSyncManager._() {
    _initializeSync();
  }
  
  // 同步状态流
  Stream<SyncStatus> get statusStream => _statusController.stream;
  
  // 同步进度流
  Stream<double> get progressStream => _progressController.stream;
  
  // 当前同步状态
  SyncStatus get currentStatus => _currentStatus;
  
  // 同步策略
  SyncStrategy get syncStrategy => _syncStrategy;
  
  void _initializeSync() {
    // 监听网络状态变化
    Connectivity().onConnectivityChanged.listen(_onConnectivityChanged);
    
    // 设置自动同步
    _setupAutoSync();
  }
  
  void _onConnectivityChanged(ConnectivityResult result) {
    if (result != ConnectivityResult.none && _syncStrategy == SyncStrategy.automatic) {
      // 网络恢复时自动同步
      scheduleMicrotask(() => syncData());
    }
  }
  
  void _setupAutoSync() {
    _autoSyncTimer?.cancel();
    
    if (_syncStrategy == SyncStrategy.automatic || _syncStrategy == SyncStrategy.backgroundSync) {
      _autoSyncTimer = Timer.periodic(Duration(minutes: 15), (_) {
        if (_currentStatus == SyncStatus.idle) {
          syncData();
        }
      });
    }
  }
  
  // 设置同步策略
  void setSyncStrategy(SyncStrategy strategy) {
    _syncStrategy = strategy;
    _setupAutoSync();
  }
  
  // 执行数据同步
  Future<bool> syncData({bool force = false}) async {
    if (_currentStatus == SyncStatus.syncing && !force) {
      return false;
    }
    
    // 检查网络连接
    final connectivity = await Connectivity().checkConnectivity();
    if (connectivity == ConnectivityResult.none) {
      _updateStatus(SyncStatus.failed);
      return false;
    }
    
    // 检查WiFi限制
    if (_syncStrategy == SyncStrategy.wifiOnly && connectivity != ConnectivityResult.wifi) {
      return false;
    }
    
    _updateStatus(SyncStatus.syncing);
    _updateProgress(0.0);
    
    try {
      // 1. 上传本地更改
      await _uploadLocalChanges();
      _updateProgress(0.3);
      
      // 2. 下载远程更改
      await _downloadRemoteChanges();
      _updateProgress(0.6);
      
      // 3. 解决冲突
      await _resolveConflicts();
      _updateProgress(0.8);
      
      // 4. 更新本地数据
      await _updateLocalData();
      _updateProgress(1.0);
      
      _updateStatus(SyncStatus.success);
      return true;
    } catch (e) {
      print('Sync failed: $e');
      _updateStatus(SyncStatus.failed);
      return false;
    }
  }
  
  Future<void> _uploadLocalChanges() async {
    // 获取本地未同步的更改
    final localChanges = await _getLocalChanges();
    
    for (int i = 0; i < localChanges.length; i++) {
      final change = localChanges[i];
      await _uploadChange(change);
      
      // 更新进度
      final progress = (i + 1) / localChanges.length * 0.3;
      _updateProgress(progress);
    }
  }
  
  Future<void> _downloadRemoteChanges() async {
    // 获取远程更改
    final remoteChanges = await _getRemoteChanges();
    
    for (int i = 0; i < remoteChanges.length; i++) {
      final change = remoteChanges[i];
      await _applyRemoteChange(change);
      
      // 更新进度
      final progress = 0.3 + (i + 1) / remoteChanges.length * 0.3;
      _updateProgress(progress);
    }
  }
  
  Future<void> _resolveConflicts() async {
    final conflicts = await _getConflicts();
    
    if (conflicts.isNotEmpty) {
      _updateStatus(SyncStatus.conflict);
      
      for (final conflict in conflicts) {
        await _resolveConflict(conflict);
      }
    }
  }
  
  Future<void> _updateLocalData() async {
    // 更新本地数据库的同步状态
    await _markDataAsSynced();
  }
  
  void _updateStatus(SyncStatus status) {
    _currentStatus = status;
    _statusController.add(status);
  }
  
  void _updateProgress(double progress) {
    _progressController.add(progress);
  }
  
  // 获取本地更改（需要根据具体业务实现）
  Future<List<DataChange>> _getLocalChanges() async {
    // 实现获取本地未同步更改的逻辑
    return [];
  }
  
  // 获取远程更改（需要根据具体业务实现）
  Future<List<DataChange>> _getRemoteChanges() async {
    // 实现获取远程更改的逻辑
    return [];
  }
  
  // 上传更改（需要根据具体业务实现）
  Future<void> _uploadChange(DataChange change) async {
    // 实现上传更改的逻辑
  }
  
  // 应用远程更改（需要根据具体业务实现）
  Future<void> _applyRemoteChange(DataChange change) async {
    // 实现应用远程更改的逻辑
  }
  
  // 获取冲突（需要根据具体业务实现）
  Future<List<DataConflict>> _getConflicts() async {
    // 实现获取冲突的逻辑
    return [];
  }
  
  // 解决冲突（需要根据具体业务实现）
  Future<void> _resolveConflict(DataConflict conflict) async {
    // 实现冲突解决的逻辑
  }
  
  // 标记数据为已同步（需要根据具体业务实现）
  Future<void> _markDataAsSynced() async {
    // 实现标记数据为已同步的逻辑
  }
  
  // 清理资源
  void dispose() {
    _autoSyncTimer?.cancel();
    _statusController.close();
    _progressController.close();
  }
}

// 数据更改模型
class DataChange {
  final String id;
  final String type;
  final String operation; // create, update, delete
  final Map<String, dynamic> data;
  final DateTime timestamp;
  
  DataChange({
    required this.id,
    required this.type,
    required this.operation,
    required this.data,
    required this.timestamp,
  });
}

// 数据冲突模型
class DataConflict {
  final String id;
  final Map<String, dynamic> localData;
  final Map<String, dynamic> remoteData;
  final DateTime localTimestamp;
  final DateTime remoteTimestamp;
  
  DataConflict({
    required this.id,
    required this.localData,
    required this.remoteData,
    required this.localTimestamp,
    required this.remoteTimestamp,
  });
}
```

## 性能优化与最佳实践

### 数据库性能优化

```dart
// 数据库性能优化工具
class DatabasePerformanceOptimizer {
  final DatabaseManager _dbManager = DatabaseManager.instance;
  
  // 批量操作优化
  Future<void> batchInsert<T>(String tableName, List<Map<String, dynamic>> records) async {
    final db = await _dbManager.database;
    final batch = db.batch();
    
    for (final record in records) {
      batch.insert(tableName, record);
    }
    
    await batch.commit(noResult: true);
  }
  
  // 分页查询优化
  Future<List<Map<String, dynamic>>> paginatedQuery(
    String tableName, {
    String? where,
    List<dynamic>? whereArgs,
    String? orderBy,
    required int page,
    required int pageSize,
  }) async {
    final db = await _dbManager.database;
    final offset = (page - 1) * pageSize;
    
    return await db.query(
      tableName,
      where: where,
      whereArgs: whereArgs,
      orderBy: orderBy,
      limit: pageSize,
      offset: offset,
    );
  }
  
  // 索引优化建议
  Future<void> analyzeQueryPerformance(String query) async {
    final db = await _dbManager.database;
    final result = await db.rawQuery('EXPLAIN QUERY PLAN $query');
    
    print('Query Plan:');
    for (final row in result) {
      print(row);
    }
  }
  
  // 数据库统计信息
  Future<Map<String, dynamic>> getDatabaseStats() async {
    final db = await _dbManager.database;
    
    final pageCount = await db.rawQuery('PRAGMA page_count');
    final pageSize = await db.rawQuery('PRAGMA page_size');
    final freePages = await db.rawQuery('PRAGMA freelist_count');
    
    final totalPages = pageCount.first['page_count'] as int;
    final pageSizeBytes = pageSize.first['page_size'] as int;
    final freePageCount = freePages.first['freelist_count'] as int;
    
    return {
      'totalSize': totalPages * pageSizeBytes,
      'usedSize': (totalPages - freePageCount) * pageSizeBytes,
      'freeSize': freePageCount * pageSizeBytes,
      'pageCount': totalPages,
      'pageSize': pageSizeBytes,
      'freePages': freePageCount,
    };
  }
  
  // 数据库优化
  Future<void> optimizeDatabase() async {
    final db = await _dbManager.database;
    
    // 分析数据库
    await db.execute('ANALYZE');
    
    // 重建索引
    await db.execute('REINDEX');
    
    // 清理碎片
    await db.execute('VACUUM');
  }
}
```

### 内存管理优化

```dart
// 内存管理优化器
class MemoryOptimizer {
  static const int _maxCacheSize = 100;
  final Map<String, dynamic> _cache = {};
  final List<String> _cacheKeys = [];
  
  // LRU缓存实现
  void cacheData(String key, dynamic data) {
    if (_cache.containsKey(key)) {
      _cacheKeys.remove(key);
    } else if (_cache.length >= _maxCacheSize) {
      final oldestKey = _cacheKeys.removeAt(0);
      _cache.remove(oldestKey);
    }
    
    _cache[key] = data;
    _cacheKeys.add(key);
  }
  
  dynamic getCachedData(String key) {
    if (_cache.containsKey(key)) {
      // 移动到最后（最近使用）
      _cacheKeys.remove(key);
      _cacheKeys.add(key);
      return _cache[key];
    }
    return null;
  }
  
  void clearCache() {
    _cache.clear();
    _cacheKeys.clear();
  }
  
  // 内存使用监控
  void logMemoryUsage() {
    print('Cache size: ${_cache.length}');
    print('Cache keys: ${_cacheKeys.length}');
  }
}
```

## 总结

Flutter数据持久化是构建高质量移动应用的基础。通过深入理解各种存储方案的特点和适用场景，开发者可以为不同的业务需求选择最合适的解决方案：

1. **SharedPreferences**：适用于简单的键值对存储，如用户偏好设置
2. **SQLite**：适用于复杂的关系型数据存储，支持事务和复杂查询
3. **Hive**：适用于高性能的NoSQL存储，类型安全且易于使用
4. **文件存储**：适用于大型文件和非结构化数据
5. **安全存储**：适用于敏感数据的加密存储

在实际开发中，通常需要结合多种存储方案来满足不同的需求。同时，还需要考虑数据同步、性能优化、内存管理等方面，以确保应用的稳定性和用户体验。

随着Flutter生态系统的不断发展，新的存储解决方案如Isar、ObjectBox等也在不断涌现，为开发者提供了更多的选择。开发者应该根据项目的具体需求，选择最适合的数据持久化方案，并遵循最佳实践来确保数据的安全性和应用的性能。