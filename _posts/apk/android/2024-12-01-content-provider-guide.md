---
layout: post
title: Android ContentProvider 内容提供者详解
categories: android
tags: [ContentProvider, 内容提供者, 数据共享, Android]
date: 2024/12/1 15:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_043.jpg)

## 引言

ContentProvider 是 Android 四大组件之一，用于在不同应用间安全地共享数据。它封装数据访问逻辑，通过 URI 和 CRUD 操作（query、insert、update、delete）提供统一的数据接口。调用方通过 ContentResolver 访问，无需关心数据存储细节。ContentProvider 常用于通讯录、媒体库、自定义数据共享等场景。对于应用内数据访问，Room 或 DataStore 更简单；跨应用共享时，ContentProvider 是标准方案。本文介绍 ContentProvider 的实现、URI 设计、ContentResolver 使用以及权限控制。

## ContentProvider 的作用

ContentProvider 将数据访问抽象为 URI 和标准 CRUD 操作，调用方通过 ContentResolver 传入 URI 和参数即可读写数据。Provider 可配置权限，实现细粒度的访问控制。系统自带的通讯录、媒体库等均通过 ContentProvider 暴露，第三方应用通过 ContentResolver 访问。自定义 Provider 可用于导出应用数据供其他应用使用，或封装对数据库、文件的访问。

## 实现 ContentProvider

继承 `ContentProvider`，实现 `onCreate`、`query`、`insert`、`update`、`delete`、`getType`。`onCreate` 在 Provider 首次被访问时调用，可在此初始化数据库等。`query` 返回 Cursor，支持 projection、selection、sortOrder 等参数，与 SQL 查询对应。`insert` 返回新插入行的 URI。`getType` 返回 MIME 类型，用于描述单条或集合。使用 `UriMatcher` 可根据 URI 路径分发到不同处理逻辑。注意：Provider 方法可能被多线程调用，需保证线程安全。

```kotlin
class UserProvider : ContentProvider() {
    
    private lateinit var dbHelper: DatabaseHelper
    
    override fun onCreate(): Boolean {
        dbHelper = DatabaseHelper(context!!)
        return true
    }
    
    override fun query(
        uri: Uri,
        projection: Array<out String>?,
        selection: String?,
        selectionArgs: Array<out String>?,
        sortOrder: String?
    ): Cursor? {
        val db = dbHelper.readableDatabase
        return db.query("users", projection, selection, selectionArgs, null, null, sortOrder)
    }
    
    override fun insert(uri: Uri, values: ContentValues?): Uri? {
        val db = dbHelper.writableDatabase
        val id = db.insert("users", null, values)
        return ContentUris.withAppendedId(uri, id)
    }
    
    override fun getType(uri: Uri): String? = "vnd.android.cursor.item/user"
}
```

## 在 AndroidManifest 中声明

`authorities` 是 Provider 的唯一标识，通常使用应用包名或子路径。`exported` 为 true 时其他应用可访问；为 false 时仅本应用可访问。可配置 `readPermission`、`writePermission` 限制访问。

```xml
<provider
    android:name=".UserProvider"
    android:authorities="com.example.provider"
    android:exported="true" />
```

## 使用 ContentResolver 访问

通过 `contentResolver.query(uri, ...)` 查询，返回 Cursor，需在使用完毕后关闭。URI 格式通常为 `content://authorities/path`，如 `content://com.example.provider/users`。可结合 CursorLoader 或协程在后台加载，避免阻塞主线程。

```kotlin
val uri = Uri.parse("content://com.example.provider/users")
val cursor = contentResolver.query(uri, null, null, null, null)
cursor?.use {
    while (it.moveToNext()) {
        val name = it.getString(it.getColumnIndex("name"))
    }
}
```

## 总结

ContentProvider 适用于跨应用数据共享，对于应用内数据访问，Room 或 DataStore 是更简单的选择。实现时注意 URI 设计、权限配置和线程安全，可构建安全、规范的数据共享接口。
