---
layout: post
title: Android 数据存储技术深入指南
categories: android
tags: [数据存储, Android, SharedPreferences, SQLite, 缓存]
date: 2024/9/5 12:10:00
---

![title](https://image.sideproject.cn/titlex/titlex_storage.jpg)

## 引言

数据存储是 Android 应用开发中的基础且关键的技术领域。随着移动应用功能的日益复杂，如何高效、安全地存储和管理数据成为开发者必须掌握的核心技能。Android 平台提供了多种数据存储方案，从简单的 SharedPreferences 到复杂的 SQLite 数据库，再到现代化的 Room 持久化库，每种方案都有其适用场景和最佳实践。本文将深入探讨 Android 数据存储的各种技术，帮助开发者选择合适的存储方案并实现高效的数据管理。

## SharedPreferences 轻量级存储

### 基础使用

SharedPreferences 是 Android 提供的轻量级键值对存储方案，适用于存储应用配置、用户偏好设置等简单数据。

```kotlin
class PreferencesManager(private val context: Context) {
    
    companion object {
        private const val PREF_NAME = "app_preferences"
        private const val KEY_USER_ID = "user_id"
        private const val KEY_USERNAME = "username"
        private const val KEY_IS_LOGGED_IN = "is_logged_in"
        private const val KEY_THEME_MODE = "theme_mode"
        private const val KEY_LANGUAGE = "language"
        private const val KEY_NOTIFICATION_ENABLED = "notification_enabled"
        private const val KEY_LAST_SYNC_TIME = "last_sync_time"
    }
    
    private val sharedPreferences: SharedPreferences by lazy {
        context.getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
    }
    
    private val editor: SharedPreferences.Editor
        get() = sharedPreferences.edit()
    
    // 用户相关
    var userId: String?
        get() = sharedPreferences.getString(KEY_USER_ID, null)
        set(value) = editor.putString(KEY_USER_ID, value).apply()
    
    var username: String?
        get() = sharedPreferences.getString(KEY_USERNAME, null)
        set(value) = editor.putString(KEY_USERNAME, value).apply()
    
    var isLoggedIn: Boolean
        get() = sharedPreferences.getBoolean(KEY_IS_LOGGED_IN, false)
        set(value) = editor.putBoolean(KEY_IS_LOGGED_IN, value).apply()
    
    // 应用设置
    var themeMode: String
        get() = sharedPreferences.getString(KEY_THEME_MODE, "system") ?: "system"
        set(value) = editor.putString(KEY_THEME_MODE, value).apply()
    
    var language: String
        get() = sharedPreferences.getString(KEY_LANGUAGE, "zh") ?: "zh"
        set(value) = editor.putString(KEY_LANGUAGE, value).apply()
    
    var notificationEnabled: Boolean
        get() = sharedPreferences.getBoolean(KEY_NOTIFICATION_ENABLED, true)
        set(value) = editor.putBoolean(KEY_NOTIFICATION_ENABLED, value).apply()
    
    var lastSyncTime: Long
        get() = sharedPreferences.getLong(KEY_LAST_SYNC_TIME, 0L)
        set(value) = editor.putLong(KEY_LAST_SYNC_TIME, value).apply()
    
    // 批量操作
    fun saveUserInfo(userId: String, username: String, isLoggedIn: Boolean) {
        editor.apply {
            putString(KEY_USER_ID, userId)
            putString(KEY_USERNAME, username)
            putBoolean(KEY_IS_LOGGED_IN, isLoggedIn)
            apply()
        }
    }
    
    fun clearUserInfo() {
        editor.apply {
            remove(KEY_USER_ID)
            remove(KEY_USERNAME)
            remove(KEY_IS_LOGGED_IN)
            apply()
        }
    }
    
    fun clearAll() {
        editor.clear().apply()
    }
    
    // 监听变化
    fun registerOnSharedPreferenceChangeListener(
        listener: SharedPreferences.OnSharedPreferenceChangeListener
    ) {
        sharedPreferences.registerOnSharedPreferenceChangeListener(listener)
    }
    
    fun unregisterOnSharedPreferenceChangeListener(
        listener: SharedPreferences.OnSharedPreferenceChangeListener
    ) {
        sharedPreferences.unregisterOnSharedPreferenceChangeListener(listener)
    }
}

// 类型安全的 Preferences 封装
class TypedPreferences(private val context: Context) {
    
    private val preferences = context.getSharedPreferences("typed_prefs", Context.MODE_PRIVATE)
    
    // 泛型方法
    inline fun <reified T> get(key: String, defaultValue: T): T {
        return when (T::class) {
            String::class -> preferences.getString(key, defaultValue as String) as T
            Int::class -> preferences.getInt(key, defaultValue as Int) as T
            Long::class -> preferences.getLong(key, defaultValue as Long) as T
            Float::class -> preferences.getFloat(key, defaultValue as Float) as T
            Boolean::class -> preferences.getBoolean(key, defaultValue as Boolean) as T
            else -> throw IllegalArgumentException("Unsupported type: ${T::class}")
        }
    }
    
    inline fun <reified T> put(key: String, value: T) {
        val editor = preferences.edit()
        when (T::class) {
            String::class -> editor.putString(key, value as String)
            Int::class -> editor.putInt(key, value as Int)
            Long::class -> editor.putLong(key, value as Long)
            Float::class -> editor.putFloat(key, value as Float)
            Boolean::class -> editor.putBoolean(key, value as Boolean)
            else -> throw IllegalArgumentException("Unsupported type: ${T::class}")
        }
        editor.apply()
    }
    
    // 对象存储（使用 JSON 序列化）
    inline fun <reified T> putObject(key: String, obj: T) {
        val json = Gson().toJson(obj)
        preferences.edit().putString(key, json).apply()
    }
    
    inline fun <reified T> getObject(key: String, defaultValue: T? = null): T? {
        val json = preferences.getString(key, null)
        return if (json != null) {
            try {
                Gson().fromJson(json, T::class.java)
            } catch (e: Exception) {
                defaultValue
            }
        } else {
            defaultValue
        }
    }
}
```

### 加密 SharedPreferences

```kotlin
class EncryptedPreferencesManager(private val context: Context) {
    
    private val masterKey: MasterKey by lazy {
        MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()
    }
    
    private val encryptedPreferences: SharedPreferences by lazy {
        EncryptedSharedPreferences.create(
            context,
            "encrypted_prefs",
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )
    }
    
    // 敏感数据存储
    var authToken: String?
        get() = encryptedPreferences.getString("auth_token", null)
        set(value) = encryptedPreferences.edit().putString("auth_token", value).apply()
    
    var refreshToken: String?
        get() = encryptedPreferences.getString("refresh_token", null)
        set(value) = encryptedPreferences.edit().putString("refresh_token", value).apply()
    
    var userPassword: String?
        get() = encryptedPreferences.getString("user_password", null)
        set(value) = encryptedPreferences.edit().putString("user_password", value).apply()
    
    var biometricKey: String?
        get() = encryptedPreferences.getString("biometric_key", null)
        set(value) = encryptedPreferences.edit().putString("biometric_key", value).apply()
    
    fun clearSensitiveData() {
        encryptedPreferences.edit().apply {
            remove("auth_token")
            remove("refresh_token")
            remove("user_password")
            remove("biometric_key")
            apply()
        }
    }
}
```

## 文件存储

### 内部存储

```kotlin
class InternalStorageManager(private val context: Context) {
    
    // 写入文件
    fun writeToFile(filename: String, content: String): Boolean {
        return try {
            context.openFileOutput(filename, Context.MODE_PRIVATE).use { output ->
                output.write(content.toByteArray())
            }
            true
        } catch (e: Exception) {
            Log.e("InternalStorage", "Error writing file: $filename", e)
            false
        }
    }
    
    // 读取文件
    fun readFromFile(filename: String): String? {
        return try {
            context.openFileInput(filename).use { input ->
                input.bufferedReader().use { reader ->
                    reader.readText()
                }
            }
        } catch (e: Exception) {
            Log.e("InternalStorage", "Error reading file: $filename", e)
            null
        }
    }
    
    // 删除文件
    fun deleteFile(filename: String): Boolean {
        return context.deleteFile(filename)
    }
    
    // 检查文件是否存在
    fun fileExists(filename: String): Boolean {
        val file = File(context.filesDir, filename)
        return file.exists()
    }
    
    // 获取文件列表
    fun getFileList(): Array<String> {
        return context.fileList()
    }
    
    // 获取文件大小
    fun getFileSize(filename: String): Long {
        val file = File(context.filesDir, filename)
        return if (file.exists()) file.length() else 0L
    }
    
    // 创建子目录
    fun createDirectory(dirName: String): File? {
        val dir = File(context.filesDir, dirName)
        return if (dir.mkdirs() || dir.exists()) dir else null
    }
    
    // 写入对象（JSON 序列化）
    fun <T> writeObject(filename: String, obj: T): Boolean {
        val json = Gson().toJson(obj)
        return writeToFile(filename, json)
    }
    
    // 读取对象
    inline fun <reified T> readObject(filename: String): T? {
        val json = readFromFile(filename)
        return if (json != null) {
            try {
                Gson().fromJson(json, T::class.java)
            } catch (e: Exception) {
                Log.e("InternalStorage", "Error parsing JSON from file: $filename", e)
                null
            }
        } else {
            null
        }
    }
    
    // 缓存管理
    fun getCacheSize(): Long {
        return calculateDirectorySize(context.cacheDir)
    }
    
    fun clearCache(): Boolean {
        return deleteDirectory(context.cacheDir)
    }
    
    private fun calculateDirectorySize(directory: File): Long {
        var size = 0L
        if (directory.exists()) {
            directory.listFiles()?.forEach { file ->
                size += if (file.isDirectory) {
                    calculateDirectorySize(file)
                } else {
                    file.length()
                }
            }
        }
        return size
    }
    
    private fun deleteDirectory(directory: File): Boolean {
        if (directory.exists()) {
            directory.listFiles()?.forEach { file ->
                if (file.isDirectory) {
                    deleteDirectory(file)
                } else {
                    file.delete()
                }
            }
        }
        return directory.delete()
    }
}
```

### 外部存储

```kotlin
class ExternalStorageManager(private val context: Context) {
    
    // 检查外部存储状态
    fun isExternalStorageWritable(): Boolean {
        return Environment.getExternalStorageState() == Environment.MEDIA_MOUNTED
    }
    
    fun isExternalStorageReadable(): Boolean {
        val state = Environment.getExternalStorageState()
        return state == Environment.MEDIA_MOUNTED || state == Environment.MEDIA_MOUNTED_READ_ONLY
    }
    
    // 获取公共目录
    fun getPublicDirectory(type: String): File? {
        return if (isExternalStorageWritable()) {
            Environment.getExternalStoragePublicDirectory(type)
        } else {
            null
        }
    }
    
    // 获取应用专用外部目录
    fun getAppExternalFilesDir(type: String? = null): File? {
        return context.getExternalFilesDir(type)
    }
    
    fun getAppExternalCacheDir(): File? {
        return context.externalCacheDir
    }
    
    // 保存图片到相册
    fun saveImageToGallery(bitmap: Bitmap, filename: String): Uri? {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            saveImageToGalleryQ(bitmap, filename)
        } else {
            saveImageToGalleryLegacy(bitmap, filename)
        }
    }
    
    @RequiresApi(Build.VERSION_CODES.Q)
    private fun saveImageToGalleryQ(bitmap: Bitmap, filename: String): Uri? {
        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, filename)
            put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
            put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_PICTURES)
        }
        
        val resolver = context.contentResolver
        val uri = resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues)
        
        return uri?.let {
            try {
                resolver.openOutputStream(it)?.use { outputStream ->
                    bitmap.compress(Bitmap.CompressFormat.JPEG, 90, outputStream)
                }
                it
            } catch (e: Exception) {
                Log.e("ExternalStorage", "Error saving image", e)
                null
            }
        }
    }
    
    private fun saveImageToGalleryLegacy(bitmap: Bitmap, filename: String): Uri? {
        val picturesDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES)
        val imageFile = File(picturesDir, filename)
        
        return try {
            imageFile.outputStream().use { outputStream ->
                bitmap.compress(Bitmap.CompressFormat.JPEG, 90, outputStream)
            }
            
            // 通知媒体扫描器
            MediaScannerConnection.scanFile(
                context,
                arrayOf(imageFile.absolutePath),
                arrayOf("image/jpeg"),
                null
            )
            
            Uri.fromFile(imageFile)
        } catch (e: Exception) {
            Log.e("ExternalStorage", "Error saving image", e)
            null
        }
    }
    
    // 下载文件
    fun downloadFile(url: String, filename: String): File? {
        val downloadsDir = getAppExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)
        return if (downloadsDir != null) {
            val file = File(downloadsDir, filename)
            try {
                URL(url).openStream().use { input ->
                    file.outputStream().use { output ->
                        input.copyTo(output)
                    }
                }
                file
            } catch (e: Exception) {
                Log.e("ExternalStorage", "Error downloading file", e)
                null
            }
        } else {
            null
        }
    }
}
```

## SQLite 数据库

### 原生 SQLite 使用

```kotlin
class DatabaseHelper(context: Context) : SQLiteOpenHelper(
    context, DATABASE_NAME, null, DATABASE_VERSION
) {
    
    companion object {
        private const val DATABASE_NAME = "app_database.db"
        private const val DATABASE_VERSION = 1
        
        // 用户表
        private const val TABLE_USERS = "users"
        private const val COLUMN_USER_ID = "id"
        private const val COLUMN_USERNAME = "username"
        private const val COLUMN_EMAIL = "email"
        private const val COLUMN_CREATED_AT = "created_at"
        
        // 文章表
        private const val TABLE_ARTICLES = "articles"
        private const val COLUMN_ARTICLE_ID = "id"
        private const val COLUMN_TITLE = "title"
        private const val COLUMN_CONTENT = "content"
        private const val COLUMN_AUTHOR_ID = "author_id"
        private const val COLUMN_PUBLISHED_AT = "published_at"
    }
    
    override fun onCreate(db: SQLiteDatabase) {
        // 创建用户表
        val createUsersTable = """
            CREATE TABLE $TABLE_USERS (
                $COLUMN_USER_ID INTEGER PRIMARY KEY AUTOINCREMENT,
                $COLUMN_USERNAME TEXT NOT NULL UNIQUE,
                $COLUMN_EMAIL TEXT NOT NULL UNIQUE,
                $COLUMN_CREATED_AT INTEGER NOT NULL
            )
        """.trimIndent()
        
        // 创建文章表
        val createArticlesTable = """
            CREATE TABLE $TABLE_ARTICLES (
                $COLUMN_ARTICLE_ID INTEGER PRIMARY KEY AUTOINCREMENT,
                $COLUMN_TITLE TEXT NOT NULL,
                $COLUMN_CONTENT TEXT NOT NULL,
                $COLUMN_AUTHOR_ID INTEGER NOT NULL,
                $COLUMN_PUBLISHED_AT INTEGER NOT NULL,
                FOREIGN KEY ($COLUMN_AUTHOR_ID) REFERENCES $TABLE_USERS($COLUMN_USER_ID)
            )
        """.trimIndent()
        
        db.execSQL(createUsersTable)
        db.execSQL(createArticlesTable)
        
        // 创建索引
        db.execSQL("CREATE INDEX idx_users_username ON $TABLE_USERS($COLUMN_USERNAME)")
        db.execSQL("CREATE INDEX idx_articles_author ON $TABLE_ARTICLES($COLUMN_AUTHOR_ID)")
    }
    
    override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
        when (oldVersion) {
            1 -> {
                // 升级到版本 2 的逻辑
                // db.execSQL("ALTER TABLE $TABLE_USERS ADD COLUMN avatar_url TEXT")
            }
        }
    }
    
    override fun onOpen(db: SQLiteDatabase) {
        super.onOpen(db)
        // 启用外键约束
        db.execSQL("PRAGMA foreign_keys=ON")
    }
}

// 用户数据访问对象
class UserDao(private val dbHelper: DatabaseHelper) {
    
    data class User(
        val id: Long = 0,
        val username: String,
        val email: String,
        val createdAt: Long = System.currentTimeMillis()
    )
    
    fun insertUser(user: User): Long {
        val db = dbHelper.writableDatabase
        val values = ContentValues().apply {
            put("username", user.username)
            put("email", user.email)
            put("created_at", user.createdAt)
        }
        
        return db.insert("users", null, values)
    }
    
    fun getUserById(id: Long): User? {
        val db = dbHelper.readableDatabase
        val cursor = db.query(
            "users",
            null,
            "id = ?",
            arrayOf(id.toString()),
            null,
            null,
            null
        )
        
        return cursor.use {
            if (it.moveToFirst()) {
                User(
                    id = it.getLong(it.getColumnIndexOrThrow("id")),
                    username = it.getString(it.getColumnIndexOrThrow("username")),
                    email = it.getString(it.getColumnIndexOrThrow("email")),
                    createdAt = it.getLong(it.getColumnIndexOrThrow("created_at"))
                )
            } else {
                null
            }
        }
    }
    
    fun getUserByUsername(username: String): User? {
        val db = dbHelper.readableDatabase
        val cursor = db.query(
            "users",
            null,
            "username = ?",
            arrayOf(username),
            null,
            null,
            null
        )
        
        return cursor.use {
            if (it.moveToFirst()) {
                User(
                    id = it.getLong(it.getColumnIndexOrThrow("id")),
                    username = it.getString(it.getColumnIndexOrThrow("username")),
                    email = it.getString(it.getColumnIndexOrThrow("email")),
                    createdAt = it.getLong(it.getColumnIndexOrThrow("created_at"))
                )
            } else {
                null
            }
        }
    }
    
    fun getAllUsers(): List<User> {
        val users = mutableListOf<User>()
        val db = dbHelper.readableDatabase
        val cursor = db.query(
            "users",
            null,
            null,
            null,
            null,
            null,
            "created_at DESC"
        )
        
        cursor.use {
            while (it.moveToNext()) {
                users.add(
                    User(
                        id = it.getLong(it.getColumnIndexOrThrow("id")),
                        username = it.getString(it.getColumnIndexOrThrow("username")),
                        email = it.getString(it.getColumnIndexOrThrow("email")),
                        createdAt = it.getLong(it.getColumnIndexOrThrow("created_at"))
                    )
                )
            }
        }
        
        return users
    }
    
    fun updateUser(user: User): Int {
        val db = dbHelper.writableDatabase
        val values = ContentValues().apply {
            put("username", user.username)
            put("email", user.email)
        }
        
        return db.update(
            "users",
            values,
            "id = ?",
            arrayOf(user.id.toString())
        )
    }
    
    fun deleteUser(id: Long): Int {
        val db = dbHelper.writableDatabase
        return db.delete(
            "users",
            "id = ?",
            arrayOf(id.toString())
        )
    }
    
    fun searchUsers(query: String): List<User> {
        val users = mutableListOf<User>()
        val db = dbHelper.readableDatabase
        val cursor = db.query(
            "users",
            null,
            "username LIKE ? OR email LIKE ?",
            arrayOf("%$query%", "%$query%"),
            null,
            null,
            "username ASC"
        )
        
        cursor.use {
            while (it.moveToNext()) {
                users.add(
                    User(
                        id = it.getLong(it.getColumnIndexOrThrow("id")),
                        username = it.getString(it.getColumnIndexOrThrow("username")),
                        email = it.getString(it.getColumnIndexOrThrow("email")),
                        createdAt = it.getLong(it.getColumnIndexOrThrow("created_at"))
                    )
                )
            }
        }
        
        return users
    }
}
```

## Room 持久化库

### Room 基础配置

```kotlin
// 实体类
@Entity(
    tableName = "users",
    indices = [
        Index(value = ["username"], unique = true),
        Index(value = ["email"], unique = true)
    ]
)
data class User(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "username")
    val username: String,
    
    @ColumnInfo(name = "email")
    val email: String,
    
    @ColumnInfo(name = "full_name")
    val fullName: String,
    
    @ColumnInfo(name = "avatar_url")
    val avatarUrl: String? = null,
    
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),
    
    @ColumnInfo(name = "updated_at")
    val updatedAt: Long = System.currentTimeMillis()
)

@Entity(
    tableName = "articles",
    foreignKeys = [
        ForeignKey(
            entity = User::class,
            parentColumns = ["id"],
            childColumns = ["author_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [
        Index(value = ["author_id"]),
        Index(value = ["published_at"])
    ]
)
data class Article(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "title")
    val title: String,
    
    @ColumnInfo(name = "content")
    val content: String,
    
    @ColumnInfo(name = "author_id")
    val authorId: Long,
    
    @ColumnInfo(name = "published_at")
    val publishedAt: Long = System.currentTimeMillis(),
    
    @ColumnInfo(name = "updated_at")
    val updatedAt: Long = System.currentTimeMillis()
)

// 关系数据类
data class UserWithArticles(
    @Embedded val user: User,
    @Relation(
        parentColumn = "id",
        entityColumn = "author_id"
    )
    val articles: List<Article>
)

data class ArticleWithAuthor(
    @Embedded val article: Article,
    @Relation(
        parentColumn = "author_id",
        entityColumn = "id"
    )
    val author: User
)

// DAO 接口
@Dao
interface UserDao {
    
    @Query("SELECT * FROM users ORDER BY created_at DESC")
    suspend fun getAllUsers(): List<User>
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserById(id: Long): User?
    
    @Query("SELECT * FROM users WHERE username = :username")
    suspend fun getUserByUsername(username: String): User?
    
    @Query("SELECT * FROM users WHERE email = :email")
    suspend fun getUserByEmail(email: String): User?
    
    @Query("""
        SELECT * FROM users 
        WHERE username LIKE '%' || :query || '%' 
        OR email LIKE '%' || :query || '%' 
        OR full_name LIKE '%' || :query || '%'
        ORDER BY username ASC
    """)
    suspend fun searchUsers(query: String): List<User>
    
    @Query("SELECT * FROM users WHERE created_at >= :startTime AND created_at <= :endTime")
    suspend fun getUsersByDateRange(startTime: Long, endTime: Long): List<User>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User): Long
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUsers(users: List<User>): List<Long>
    
    @Update
    suspend fun updateUser(user: User): Int
    
    @Delete
    suspend fun deleteUser(user: User): Int
    
    @Query("DELETE FROM users WHERE id = :id")
    suspend fun deleteUserById(id: Long): Int
    
    @Query("DELETE FROM users")
    suspend fun deleteAllUsers(): Int
    
    // 关系查询
    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithArticles(userId: Long): UserWithArticles?
    
    @Transaction
    @Query("SELECT * FROM users")
    suspend fun getAllUsersWithArticles(): List<UserWithArticles>
    
    // Flow 支持
    @Query("SELECT * FROM users ORDER BY created_at DESC")
    fun getAllUsersFlow(): Flow<List<User>>
    
    @Query("SELECT * FROM users WHERE id = :id")
    fun getUserByIdFlow(id: Long): Flow<User?>
    
    @Query("SELECT COUNT(*) FROM users")
    fun getUserCountFlow(): Flow<Int>
}

@Dao
interface ArticleDao {
    
    @Query("SELECT * FROM articles ORDER BY published_at DESC")
    suspend fun getAllArticles(): List<Article>
    
    @Query("SELECT * FROM articles WHERE id = :id")
    suspend fun getArticleById(id: Long): Article?
    
    @Query("SELECT * FROM articles WHERE author_id = :authorId ORDER BY published_at DESC")
    suspend fun getArticlesByAuthor(authorId: Long): List<Article>
    
    @Query("""
        SELECT * FROM articles 
        WHERE title LIKE '%' || :query || '%' 
        OR content LIKE '%' || :query || '%'
        ORDER BY published_at DESC
    """)
    suspend fun searchArticles(query: String): List<Article>
    
    @Query("""
        SELECT * FROM articles 
        WHERE published_at >= :startTime AND published_at <= :endTime
        ORDER BY published_at DESC
    """)
    suspend fun getArticlesByDateRange(startTime: Long, endTime: Long): List<Article>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertArticle(article: Article): Long
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertArticles(articles: List<Article>): List<Long>
    
    @Update
    suspend fun updateArticle(article: Article): Int
    
    @Delete
    suspend fun deleteArticle(article: Article): Int
    
    @Query("DELETE FROM articles WHERE id = :id")
    suspend fun deleteArticleById(id: Long): Int
    
    @Query("DELETE FROM articles WHERE author_id = :authorId")
    suspend fun deleteArticlesByAuthor(authorId: Long): Int
    
    // 关系查询
    @Transaction
    @Query("SELECT * FROM articles WHERE id = :articleId")
    suspend fun getArticleWithAuthor(articleId: Long): ArticleWithAuthor?
    
    @Transaction
    @Query("SELECT * FROM articles ORDER BY published_at DESC")
    suspend fun getAllArticlesWithAuthors(): List<ArticleWithAuthor>
    
    // 分页查询
    @Query("SELECT * FROM articles ORDER BY published_at DESC LIMIT :limit OFFSET :offset")
    suspend fun getArticlesPaged(limit: Int, offset: Int): List<Article>
    
    // Flow 支持
    @Query("SELECT * FROM articles ORDER BY published_at DESC")
    fun getAllArticlesFlow(): Flow<List<Article>>
    
    @Query("SELECT * FROM articles WHERE author_id = :authorId ORDER BY published_at DESC")
    fun getArticlesByAuthorFlow(authorId: Long): Flow<List<Article>>
}

// 数据库类
@Database(
    entities = [User::class, Article::class],
    version = 1,
    exportSchema = false
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    
    abstract fun userDao(): UserDao
    abstract fun articleDao(): ArticleDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                    .addMigrations(MIGRATION_1_2)
                    .fallbackToDestructiveMigration()
                    .build()
                
                INSTANCE = instance
                instance
            }
        }
        
        private val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                // 数据库迁移逻辑
                database.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")
            }
        }
    }
}

// 类型转换器
class Converters {
    
    @TypeConverter
    fun fromStringList(value: List<String>): String {
        return Gson().toJson(value)
    }
    
    @TypeConverter
    fun toStringList(value: String): List<String> {
        return Gson().fromJson(value, object : TypeToken<List<String>>() {}.type)
    }
    
    @TypeConverter
    fun fromDate(date: Date?): Long? {
        return date?.time
    }
    
    @TypeConverter
    fun toDate(timestamp: Long?): Date? {
        return timestamp?.let { Date(it) }
    }
}
```

### Repository 模式

```kotlin
class UserRepository(
    private val userDao: UserDao,
    private val articleDao: ArticleDao,
    private val apiService: ApiService
) {
    
    // 获取用户（本地优先，网络同步）
    suspend fun getUser(userId: Long, forceRefresh: Boolean = false): Result<User> {
        return try {
            if (!forceRefresh) {
                val localUser = userDao.getUserById(userId)
                if (localUser != null) {
                    return Result.success(localUser)
                }
            }
            
            // 从网络获取
            val response = apiService.getUser(userId)
            if (response.isSuccessful) {
                val user = response.body()!!
                userDao.insertUser(user)
                Result.success(user)
            } else {
                Result.failure(Exception("Network error: ${response.code()}"))
            }
            
        } catch (e: Exception) {
            // 网络错误时尝试从本地获取
            val localUser = userDao.getUserById(userId)
            if (localUser != null) {
                Result.success(localUser)
            } else {
                Result.failure(e)
            }
        }
    }
    
    // 获取用户列表
    suspend fun getUsers(forceRefresh: Boolean = false): Result<List<User>> {
        return try {
            if (!forceRefresh) {
                val localUsers = userDao.getAllUsers()
                if (localUsers.isNotEmpty()) {
                    return Result.success(localUsers)
                }
            }
            
            val response = apiService.getUsers()
            if (response.isSuccessful) {
                val users = response.body()!!
                userDao.insertUsers(users)
                Result.success(users)
            } else {
                Result.failure(Exception("Network error: ${response.code()}"))
            }
            
        } catch (e: Exception) {
            val localUsers = userDao.getAllUsers()
            if (localUsers.isNotEmpty()) {
                Result.success(localUsers)
            } else {
                Result.failure(e)
            }
        }
    }
    
    // 创建用户
    suspend fun createUser(user: User): Result<User> {
        return try {
            val response = apiService.createUser(user)
            if (response.isSuccessful) {
                val createdUser = response.body()!!
                userDao.insertUser(createdUser)
                Result.success(createdUser)
            } else {
                Result.failure(Exception("Failed to create user: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // 更新用户
    suspend fun updateUser(user: User): Result<User> {
        return try {
            val response = apiService.updateUser(user)
            if (response.isSuccessful) {
                val updatedUser = response.body()!!
                userDao.updateUser(updatedUser)
                Result.success(updatedUser)
            } else {
                Result.failure(Exception("Failed to update user: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // 删除用户
    suspend fun deleteUser(userId: Long): Result<Unit> {
        return try {
            val response = apiService.deleteUser(userId)
            if (response.isSuccessful) {
                userDao.deleteUserById(userId)
                Result.success(Unit)
            } else {
                Result.failure(Exception("Failed to delete user: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // 搜索用户
    suspend fun searchUsers(query: String): Result<List<User>> {
        return try {
            val localResults = userDao.searchUsers(query)
            Result.success(localResults)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // 获取用户及其文章
    suspend fun getUserWithArticles(userId: Long): Result<UserWithArticles> {
        return try {
            val userWithArticles = userDao.getUserWithArticles(userId)
            if (userWithArticles != null) {
                Result.success(userWithArticles)
            } else {
                Result.failure(Exception("User not found"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // Flow 数据流
    fun getUsersFlow(): Flow<List<User>> = userDao.getAllUsersFlow()
    
    fun getUserFlow(userId: Long): Flow<User?> = userDao.getUserByIdFlow(userId)
    
    fun getUserCountFlow(): Flow<Int> = userDao.getUserCountFlow()
    
    // 数据同步
    suspend fun syncUsers(): Result<Unit> {
        return try {
            val response = apiService.getUsers()
            if (response.isSuccessful) {
                val users = response.body()!!
                userDao.deleteAllUsers()
                userDao.insertUsers(users)
                Result.success(Unit)
            } else {
                Result.failure(Exception("Sync failed: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

## 数据缓存策略

### 多级缓存实现

```kotlin
class CacheManager {
    
    // 内存缓存
    private val memoryCache = LruCache<String, Any>(100)
    
    // 磁盘缓存
    private val diskCache: DiskLruCache by lazy {
        DiskLruCache.open(
            File(context.cacheDir, "disk_cache"),
            1,
            1,
            10 * 1024 * 1024 // 10MB
        )
    }
    
    // 缓存策略
    enum class CacheStrategy {
        MEMORY_ONLY,
        DISK_ONLY,
        MEMORY_FIRST,
        DISK_FIRST,
        BOTH
    }
    
    // 缓存项
    data class CacheItem<T>(
        val data: T,
        val timestamp: Long = System.currentTimeMillis(),
        val ttl: Long = 0 // Time to live in milliseconds
    ) {
        fun isExpired(): Boolean {
            return ttl > 0 && (System.currentTimeMillis() - timestamp) > ttl
        }
    }
    
    // 存储到缓存
    suspend fun <T> put(
        key: String,
        value: T,
        strategy: CacheStrategy = CacheStrategy.BOTH,
        ttl: Long = 0
    ) {
        val cacheItem = CacheItem(value, ttl = ttl)
        
        when (strategy) {
            CacheStrategy.MEMORY_ONLY, CacheStrategy.BOTH -> {
                memoryCache.put(key, cacheItem)
            }
            CacheStrategy.DISK_ONLY, CacheStrategy.DISK_FIRST -> {
                putToDisk(key, cacheItem)
            }
            CacheStrategy.MEMORY_FIRST -> {
                memoryCache.put(key, cacheItem)
                putToDisk(key, cacheItem)
            }
        }
    }
    
    // 从缓存获取
    suspend fun <T> get(
        key: String,
        strategy: CacheStrategy = CacheStrategy.MEMORY_FIRST,
        clazz: Class<T>
    ): T? {
        return when (strategy) {
            CacheStrategy.MEMORY_ONLY -> {
                getFromMemory(key, clazz)
            }
            CacheStrategy.DISK_ONLY -> {
                getFromDisk(key, clazz)
            }
            CacheStrategy.MEMORY_FIRST -> {
                getFromMemory(key, clazz) ?: getFromDisk(key, clazz)
            }
            CacheStrategy.DISK_FIRST -> {
                getFromDisk(key, clazz) ?: getFromMemory(key, clazz)
            }
            CacheStrategy.BOTH -> {
                getFromMemory(key, clazz) ?: getFromDisk(key, clazz)
            }
        }
    }
    
    // 从内存获取
    private fun <T> getFromMemory(key: String, clazz: Class<T>): T? {
        val cacheItem = memoryCache.get(key) as? CacheItem<*>
        return if (cacheItem != null && !cacheItem.isExpired()) {
            try {
                clazz.cast(cacheItem.data)
            } catch (e: ClassCastException) {
                null
            }
        } else {
            if (cacheItem?.isExpired() == true) {
                memoryCache.remove(key)
            }
            null
        }
    }
    
    // 从磁盘获取
    private suspend fun <T> getFromDisk(key: String, clazz: Class<T>): T? {
        return withContext(Dispatchers.IO) {
            try {
                val snapshot = diskCache.get(key)
                if (snapshot != null) {
                    val json = snapshot.getInputStream(0).bufferedReader().use { it.readText() }
                    val cacheItem = Gson().fromJson(json, object : TypeToken<CacheItem<T>>() {}.type) as CacheItem<T>
                    
                    if (!cacheItem.isExpired()) {
                        // 同步到内存缓存
                        memoryCache.put(key, cacheItem)
                        cacheItem.data
                    } else {
                        diskCache.remove(key)
                        null
                    }
                } else {
                    null
                }
            } catch (e: Exception) {
                Log.e("CacheManager", "Error reading from disk cache", e)
                null
            }
        }
    }
    
    // 存储到磁盘
    private suspend fun <T> putToDisk(key: String, cacheItem: CacheItem<T>) {
        withContext(Dispatchers.IO) {
            try {
                val editor = diskCache.edit(key)
                if (editor != null) {
                    val json = Gson().toJson(cacheItem)
                    editor.newOutputStream(0).bufferedWriter().use { writer ->
                        writer.write(json)
                    }
                    editor.commit()
                }
            } catch (e: Exception) {
                Log.e("CacheManager", "Error writing to disk cache", e)
            }
        }
    }
    
    // 删除缓存
    suspend fun remove(key: String) {
        memoryCache.remove(key)
        withContext(Dispatchers.IO) {
            diskCache.remove(key)
        }
    }
    
    // 清空缓存
    suspend fun clear() {
        memoryCache.evictAll()
        withContext(Dispatchers.IO) {
            diskCache.delete()
        }
    }
    
    // 获取缓存大小
    suspend fun getCacheSize(): Long {
        return withContext(Dispatchers.IO) {
            diskCache.size()
        }
    }
}

// 缓存装饰器
class CachedRepository<T>(
    private val repository: Repository<T>,
    private val cacheManager: CacheManager,
    private val cacheKeyPrefix: String,
    private val defaultTtl: Long = TimeUnit.HOURS.toMillis(1)
) {
    
    suspend fun get(id: String, forceRefresh: Boolean = false): Result<T> {
        val cacheKey = "$cacheKeyPrefix:$id"
        
        if (!forceRefresh) {
            val cached = cacheManager.get(cacheKey, CacheStrategy.MEMORY_FIRST, Any::class.java) as? T
            if (cached != null) {
                return Result.success(cached)
            }
        }
        
        return repository.get(id).also { result ->
            if (result.isSuccess) {
                cacheManager.put(cacheKey, result.getOrNull()!!, ttl = defaultTtl)
            }
        }
    }
    
    suspend fun getList(forceRefresh: Boolean = false): Result<List<T>> {
        val cacheKey = "$cacheKeyPrefix:list"
        
        if (!forceRefresh) {
            val cached = cacheManager.get(cacheKey, CacheStrategy.MEMORY_FIRST, List::class.java) as? List<T>
            if (cached != null) {
                return Result.success(cached)
            }
        }
        
        return repository.getList().also { result ->
            if (result.isSuccess) {
                cacheManager.put(cacheKey, result.getOrNull()!!, ttl = defaultTtl)
            }
        }
    }
    
    suspend fun invalidate(id: String? = null) {
        if (id != null) {
            cacheManager.remove("$cacheKeyPrefix:$id")
        } else {
            cacheManager.remove("$cacheKeyPrefix:list")
        }
    }
}

interface Repository<T> {
    suspend fun get(id: String): Result<T>
    suspend fun getList(): Result<List<T>>
}
```

## 总结

Android 数据存储技术涵盖了从简单的键值对存储到复杂的关系型数据库管理。通过本文的深入探讨，我们学习了：

1. **SharedPreferences**：轻量级键值对存储，适用于应用配置和用户偏好
2. **文件存储**：内部和外部存储的使用，包括文件操作和媒体文件管理
3. **SQLite 数据库**：原生数据库操作和 DAO 模式实现
4. **Room 持久化库**：现代化的数据库抽象层，提供类型安全和协程支持
5. **缓存策略**：多级缓存实现和缓存管理最佳实践

选择合适的存储方案需要考虑数据类型、访问频率、数据量大小、安全性要求等因素。在实际开发中，通常会组合使用多种存储技术，构建完整的数据管理架构，为用户提供流畅的应用体验。