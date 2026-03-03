---
layout: post
title: Android DataStore 数据存储完全指南
categories: android
tags: [DataStore, Preferences, 数据存储, Android, Jetpack]
date: 2024/7/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_034.jpg)

## 引言

DataStore 是 Android 推荐的替代 SharedPreferences 的现代数据存储解决方案，于 2020 年发布。SharedPreferences 存在同步 I/O 阻塞主线程、无类型安全、缺乏事务支持等问题。DataStore 基于 Kotlin 协程和 Flow，提供类型安全、事务性 API，并且完全异步，不会阻塞主线程。DataStore 分为 Preferences DataStore（类似 SharedPreferences 的键值对）和 Proto DataStore（基于 Protocol Buffers 的强类型存储）。本文将深入探讨 DataStore Preferences 的使用方式、迁移策略以及最佳实践。

## 为什么迁移到 DataStore

SharedPreferences 的 `getString`、`commit` 等可能触发磁盘 I/O，在主线程调用会卡顿。DataStore 的读写都是挂起函数，需在协程中调用，天然避免主线程阻塞。DataStore 支持事务：`edit` 块内的多次修改要么全部成功，要么全部回滚。Preferences DataStore 使用类型化的 Key（如 `intPreferencesKey`），避免键名拼写错误和类型转换异常。Google 已明确推荐新项目使用 DataStore，SharedPreferences 将逐步被取代。

## 添加依赖

```kotlin
dependencies {
    implementation("androidx.datastore:datastore-preferences:1.0.0")
    // 或使用 Proto DataStore
    implementation("androidx.datastore:datastore:1.0.0")
}
```

## Preferences DataStore

通过 `preferencesDataStore` 委托创建 DataStore 实例，name 指定文件名（不含路径）。Key 使用 `booleanPreferencesKey`、`stringPreferencesKey`、`intPreferencesKey` 等创建，保证类型安全。`data` 属性是 Flow，可多次收集，每次数据变化会发射新值。`edit` 是挂起函数，在块内通过 `preferences[key] = value` 修改，块结束时自动提交。

```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

// 写入数据
suspend fun saveTheme(isDark: Boolean) {
    context.dataStore.edit { preferences ->
        preferences[PreferencesKeys.BOOLEAN_KEY] = isDark
    }
}

// 读取数据
val themeFlow: Flow<Boolean> = context.dataStore.data
    .map { preferences ->
        preferences[PreferencesKeys.BOOLEAN_KEY] ?: false
    }

// 定义 Key
object PreferencesKeys {
    val BOOLEAN_KEY = booleanPreferencesKey("theme_dark")
    val STRING_KEY = stringPreferencesKey("user_name")
    val INT_KEY = intPreferencesKey("user_age")
}
```

## 在 ViewModel 中使用

ViewModel 通过构造函数注入 DataStore，或从 Application 的 dataStore 获取。将 `data` 的 Flow 通过 `map` 转换为业务需要的类型，在 UI 层收集并更新界面。写入时在 `viewModelScope.launch` 中调用 `edit`。注意：DataStore 的 `data` Flow 是热流，多次收集会多次读取；若需共享，可在 Repository 层缓存或使用 `stateIn`。

```kotlin
class SettingsViewModel(private val dataStore: DataStore<Preferences>) : ViewModel() {
    
    val theme: Flow<Boolean> = dataStore.data
        .map { it[PreferencesKeys.BOOLEAN_KEY] ?: false }
    
    fun setTheme(isDark: Boolean) {
        viewModelScope.launch {
            dataStore.edit { it[PreferencesKeys.BOOLEAN_KEY] = isDark }
        }
    }
}
```

## 迁移 SharedPreferences

DataStore 提供 `SharedPreferencesMigration`，可在首次访问时从 SharedPreferences 读取数据并写入 DataStore，然后删除旧文件。在 `preferencesDataStore` 的 `produceMigrations` 参数中提供 migration 列表。迁移在 DataStore 初始化时执行，会阻塞直到完成，因此迁移逻辑应尽快完成，避免复杂计算。

```kotlin
val dataStore = PreferenceDataStoreFactory.create(
    migrations = { context, _ ->
        context.getSharedPreferences("old_prefs", Context.MODE_PRIVATE)
    }
) {
    context.dataStore
}
```

## 总结

DataStore 提供了更安全、更现代的数据存储方式，是 SharedPreferences 的推荐替代方案。掌握 Preferences API、Flow 收集和迁移策略，可以平滑迁移现有项目，并享受类型安全和异步 I/O 带来的好处。
