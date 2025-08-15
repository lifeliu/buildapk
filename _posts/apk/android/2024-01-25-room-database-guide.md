---
layout: post
title: Android Room 数据库深入使用指南
categories: android
tags: [Room, 数据库, Android, SQLite, ORM]
date: 2024/1/25 13:40:00
---

![title](https://image.sideproject.cn/titlex/titlex_room.jpg)

## 引言

Room 是 Google 推出的 Android 架构组件之一，它在 SQLite 之上提供了一个抽象层，使得数据库操作更加简单、安全和高效。Room 通过编译时验证、强大的查询功能和与其他架构组件的无缝集成，成为了 Android 开发中首选的本地数据库解决方案。本文将深入探讨 Room 的核心概念、高级特性以及最佳实践。

## Room 架构概述

### 核心组件

Room 由三个主要组件构成：

1. **Entity（实体）**：代表数据库中的表
2. **DAO（数据访问对象）**：定义数据库操作方法
3. **Database（数据库）**：数据库持有者，作为应用程序持久化数据的主要访问点

```kotlin
// 1. Entity - 定义数据表结构
@Entity(tableName = "users")
data class User(
    @PrimaryKey
    val id: String,
    @ColumnInfo(name = "user_name")
    val name: String,
    val email: String,
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),
    @ColumnInfo(name = "is_active")
    val isActive: Boolean = true
)

// 2. DAO - 定义数据库操作
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE is_active = 1")
    suspend fun getActiveUsers(): List<User>
    
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserById(userId: String): User?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User)
    
    @Update
    suspend fun updateUser(user: User)
    
    @Delete
    suspend fun deleteUser(user: User)
}

// 3. Database - 数据库配置
@Database(
    entities = [User::class],
    version = 1,
    exportSchema = false
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

## 高级 Entity 定义

### 复杂实体结构

```kotlin
// 复杂的实体定义
@Entity(
    tableName = "articles",
    indices = [
        Index(value = ["title"], unique = true),
        Index(value = ["category_id", "published_date"])
    ],
    foreignKeys = [
        ForeignKey(
            entity = Category::class,
            parentColumns = ["id"],
            childColumns = ["category_id"],
            onDelete = ForeignKey.CASCADE
        )
    ]
)
data class Article(
    @PrimaryKey
    val id: String,
    
    @ColumnInfo(name = "title")
    val title: String,
    
    @ColumnInfo(name = "content")
    val content: String,
    
    @ColumnInfo(name = "category_id")
    val categoryId: String,
    
    @ColumnInfo(name = "author_id")
    val authorId: String,
    
    @ColumnInfo(name = "published_date")
    val publishedDate: Date,
    
    @ColumnInfo(name = "view_count")
    val viewCount: Int = 0,
    
    @ColumnInfo(name = "is_featured")
    val isFeatured: Boolean = false,
    
    @ColumnInfo(name = "tags")
    val tags: List<String> = emptyList(),
    
    @Embedded
    val metadata: ArticleMetadata
)

// 嵌入式对象
data class ArticleMetadata(
    @ColumnInfo(name = "created_at")
    val createdAt: Long,
    
    @ColumnInfo(name = "updated_at")
    val updatedAt: Long,
    
    @ColumnInfo(name = "created_by")
    val createdBy: String
)

// 分类实体
@Entity(tableName = "categories")
data class Category(
    @PrimaryKey
    val id: String,
    val name: String,
    val description: String?,
    @ColumnInfo(name = "parent_id")
    val parentId: String?
)
```

### 类型转换器

```kotlin
// 类型转换器
class Converters {
    
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? {
        return value?.let { Date(it) }
    }
    
    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? {
        return date?.time
    }
    
    @TypeConverter
    fun fromStringList(value: List<String>): String {
        return Gson().toJson(value)
    }
    
    @TypeConverter
    fun toStringList(value: String): List<String> {
        return Gson().fromJson(value, object : TypeToken<List<String>>() {}.type)
    }
    
    @TypeConverter
    fun fromArticleStatus(status: ArticleStatus): String {
        return status.name
    }
    
    @TypeConverter
    fun toArticleStatus(status: String): ArticleStatus {
        return ArticleStatus.valueOf(status)
    }
}

enum class ArticleStatus {
    DRAFT, PUBLISHED, ARCHIVED
}
```

## 高级查询操作

### 复杂查询和关联

```kotlin
@Dao
interface ArticleDao {
    
    // 基本查询
    @Query("SELECT * FROM articles WHERE category_id = :categoryId ORDER BY published_date DESC")
    suspend fun getArticlesByCategory(categoryId: String): List<Article>
    
    // 分页查询
    @Query("SELECT * FROM articles ORDER BY published_date DESC LIMIT :limit OFFSET :offset")
    suspend fun getArticlesPaged(limit: Int, offset: Int): List<Article>
    
    // 条件查询
    @Query("""
        SELECT * FROM articles 
        WHERE (:categoryId IS NULL OR category_id = :categoryId)
        AND (:searchQuery IS NULL OR title LIKE '%' || :searchQuery || '%' OR content LIKE '%' || :searchQuery || '%')
        AND published_date BETWEEN :startDate AND :endDate
        ORDER BY 
            CASE WHEN :sortBy = 'date' THEN published_date END DESC,
            CASE WHEN :sortBy = 'views' THEN view_count END DESC,
            CASE WHEN :sortBy = 'title' THEN title END ASC
    """)
    suspend fun searchArticles(
        categoryId: String?,
        searchQuery: String?,
        startDate: Long,
        endDate: Long,
        sortBy: String
    ): List<Article>
    
    // 聚合查询
    @Query("SELECT COUNT(*) FROM articles WHERE category_id = :categoryId")
    suspend fun getArticleCountByCategory(categoryId: String): Int
    
    @Query("SELECT AVG(view_count) FROM articles WHERE published_date > :date")
    suspend fun getAverageViewCount(date: Long): Double
    
    // 关联查询
    @Query("""
        SELECT a.*, c.name as category_name 
        FROM articles a 
        INNER JOIN categories c ON a.category_id = c.id 
        WHERE a.is_featured = 1
    """)
    suspend fun getFeaturedArticlesWithCategory(): List<ArticleWithCategory>
    
    // 子查询
    @Query("""
        SELECT * FROM articles 
        WHERE category_id IN (
            SELECT id FROM categories WHERE parent_id = :parentCategoryId
        )
    """)
    suspend fun getArticlesByParentCategory(parentCategoryId: String): List<Article>
    
    // 批量操作
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertArticles(articles: List<Article>)
    
    @Query("DELETE FROM articles WHERE id IN (:articleIds)")
    suspend fun deleteArticlesByIds(articleIds: List<String>)
    
    // 更新特定字段
    @Query("UPDATE articles SET view_count = view_count + 1 WHERE id = :articleId")
    suspend fun incrementViewCount(articleId: String)
    
    @Query("UPDATE articles SET is_featured = :featured WHERE id = :articleId")
    suspend fun updateFeaturedStatus(articleId: String, featured: Boolean)
}

// 查询结果数据类
data class ArticleWithCategory(
    @Embedded val article: Article,
    @ColumnInfo(name = "category_name") val categoryName: String
)
```

### Flow 和 LiveData 支持

```kotlin
@Dao
interface ReactiveArticleDao {
    
    // 使用 Flow 进行响应式查询
    @Query("SELECT * FROM articles ORDER BY published_date DESC")
    fun observeAllArticles(): Flow<List<Article>>
    
    @Query("SELECT * FROM articles WHERE category_id = :categoryId")
    fun observeArticlesByCategory(categoryId: String): Flow<List<Article>>
    
    // 使用 LiveData
    @Query("SELECT * FROM articles WHERE is_featured = 1")
    fun getFeaturedArticlesLiveData(): LiveData<List<Article>>
    
    // 复杂的响应式查询
    @Query("""
        SELECT a.*, c.name as category_name, 
               (SELECT COUNT(*) FROM articles WHERE category_id = a.category_id) as category_count
        FROM articles a 
        LEFT JOIN categories c ON a.category_id = c.id 
        ORDER BY a.published_date DESC
    """)
    fun observeArticlesWithDetails(): Flow<List<ArticleWithDetails>>
}

data class ArticleWithDetails(
    @Embedded val article: Article,
    @ColumnInfo(name = "category_name") val categoryName: String?,
    @ColumnInfo(name = "category_count") val categoryCount: Int
)
```

## 数据库迁移

### 版本迁移策略

```kotlin
// 数据库迁移
class DatabaseMigrations {
    
    companion object {
        
        // 从版本 1 到版本 2 的迁移
        val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                // 添加新列
                database.execSQL("ALTER TABLE articles ADD COLUMN summary TEXT")
                
                // 创建新表
                database.execSQL("""
                    CREATE TABLE IF NOT EXISTS tags (
                        id TEXT PRIMARY KEY NOT NULL,
                        name TEXT NOT NULL,
                        color TEXT NOT NULL
                    )
                """)
                
                // 创建关联表
                database.execSQL("""
                    CREATE TABLE IF NOT EXISTS article_tag_cross_ref (
                        article_id TEXT NOT NULL,
                        tag_id TEXT NOT NULL,
                        PRIMARY KEY(article_id, tag_id),
                        FOREIGN KEY(article_id) REFERENCES articles(id) ON DELETE CASCADE,
                        FOREIGN KEY(tag_id) REFERENCES tags(id) ON DELETE CASCADE
                    )
                """)
            }
        }
        
        // 从版本 2 到版本 3 的迁移
        val MIGRATION_2_3 = object : Migration(2, 3) {
            override fun migrate(database: SupportSQLiteDatabase) {
                // 重命名表
                database.execSQL("ALTER TABLE articles RENAME TO articles_old")
                
                // 创建新表结构
                database.execSQL("""
                    CREATE TABLE articles (
                        id TEXT PRIMARY KEY NOT NULL,
                        title TEXT NOT NULL,
                        content TEXT NOT NULL,
                        summary TEXT,
                        category_id TEXT NOT NULL,
                        author_id TEXT NOT NULL,
                        published_date INTEGER NOT NULL,
                        view_count INTEGER NOT NULL DEFAULT 0,
                        is_featured INTEGER NOT NULL DEFAULT 0,
                        status TEXT NOT NULL DEFAULT 'DRAFT',
                        created_at INTEGER NOT NULL,
                        updated_at INTEGER NOT NULL,
                        created_by TEXT NOT NULL
                    )
                """)
                
                // 迁移数据
                database.execSQL("""
                    INSERT INTO articles (id, title, content, summary, category_id, author_id, 
                                         published_date, view_count, is_featured, status, 
                                         created_at, updated_at, created_by)
                    SELECT id, title, content, summary, category_id, author_id, 
                           published_date, view_count, is_featured, 'PUBLISHED',
                           created_at, updated_at, created_by
                    FROM articles_old
                """)
                
                // 删除旧表
                database.execSQL("DROP TABLE articles_old")
                
                // 创建索引
                database.execSQL("CREATE INDEX index_articles_status ON articles(status)")
            }
        }
        
        // 复杂的数据迁移
        val MIGRATION_3_4 = object : Migration(3, 4) {
            override fun migrate(database: SupportSQLiteDatabase) {
                // 添加用户表
                database.execSQL("""
                    CREATE TABLE users (
                        id TEXT PRIMARY KEY NOT NULL,
                        username TEXT NOT NULL UNIQUE,
                        email TEXT NOT NULL UNIQUE,
                        password_hash TEXT NOT NULL,
                        full_name TEXT,
                        avatar_url TEXT,
                        created_at INTEGER NOT NULL,
                        updated_at INTEGER NOT NULL,
                        is_active INTEGER NOT NULL DEFAULT 1
                    )
                """)
                
                // 迁移现有的作者数据到用户表
                database.execSQL("""
                    INSERT INTO users (id, username, email, password_hash, full_name, created_at, updated_at)
                    SELECT DISTINCT author_id, author_id, author_id || '@example.com', 'temp_hash', 
                                   'Unknown Author', ${System.currentTimeMillis()}, ${System.currentTimeMillis()}
                    FROM articles
                """)
                
                // 添加外键约束
                database.execSQL("CREATE INDEX index_articles_author_id ON articles(author_id)")
            }
        }
    }
}

// 更新数据库配置
@Database(
    entities = [Article::class, Category::class, Tag::class, ArticleTagCrossRef::class, User::class],
    version = 4,
    exportSchema = true
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    
    abstract fun articleDao(): ArticleDao
    abstract fun categoryDao(): CategoryDao
    abstract fun tagDao(): TagDao
    abstract fun userDao(): UserDao
    
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
                .addMigrations(
                    DatabaseMigrations.MIGRATION_1_2,
                    DatabaseMigrations.MIGRATION_2_3,
                    DatabaseMigrations.MIGRATION_3_4
                )
                .build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

## 关系映射

### 一对多和多对多关系

```kotlin
// 标签实体
@Entity(tableName = "tags")
data class Tag(
    @PrimaryKey val id: String,
    val name: String,
    val color: String
)

// 文章-标签关联表
@Entity(
    tableName = "article_tag_cross_ref",
    primaryKeys = ["article_id", "tag_id"],
    foreignKeys = [
        ForeignKey(
            entity = Article::class,
            parentColumns = ["id"],
            childColumns = ["article_id"],
            onDelete = ForeignKey.CASCADE
        ),
        ForeignKey(
            entity = Tag::class,
            parentColumns = ["id"],
            childColumns = ["tag_id"],
            onDelete = ForeignKey.CASCADE
        )
    ]
)
data class ArticleTagCrossRef(
    @ColumnInfo(name = "article_id") val articleId: String,
    @ColumnInfo(name = "tag_id") val tagId: String
)

// 关系数据类
data class ArticleWithTags(
    @Embedded val article: Article,
    @Relation(
        parentColumn = "id",
        entityColumn = "id",
        associateBy = Junction(
            ArticleTagCrossRef::class,
            parentColumn = "article_id",
            entityColumn = "tag_id"
        )
    )
    val tags: List<Tag>
)

data class TagWithArticles(
    @Embedded val tag: Tag,
    @Relation(
        parentColumn = "id",
        entityColumn = "id",
        associateBy = Junction(
            ArticleTagCrossRef::class,
            parentColumn = "tag_id",
            entityColumn = "article_id"
        )
    )
    val articles: List<Article>
)

data class CategoryWithArticles(
    @Embedded val category: Category,
    @Relation(
        parentColumn = "id",
        entityColumn = "category_id"
    )
    val articles: List<Article>
)

// 复杂的嵌套关系
data class CategoryWithArticlesAndTags(
    @Embedded val category: Category,
    @Relation(
        entity = Article::class,
        parentColumn = "id",
        entityColumn = "category_id"
    )
    val articlesWithTags: List<ArticleWithTags>
)
```

### 关系查询 DAO

```kotlin
@Dao
interface RelationDao {
    
    // 获取文章及其标签
    @Transaction
    @Query("SELECT * FROM articles WHERE id = :articleId")
    suspend fun getArticleWithTags(articleId: String): ArticleWithTags?
    
    @Transaction
    @Query("SELECT * FROM articles")
    suspend fun getAllArticlesWithTags(): List<ArticleWithTags>
    
    // 获取标签及其文章
    @Transaction
    @Query("SELECT * FROM tags WHERE id = :tagId")
    suspend fun getTagWithArticles(tagId: String): TagWithArticles?
    
    // 获取分类及其文章
    @Transaction
    @Query("SELECT * FROM categories")
    suspend fun getCategoriesWithArticles(): List<CategoryWithArticles>
    
    // 复杂的嵌套查询
    @Transaction
    @Query("SELECT * FROM categories WHERE id = :categoryId")
    suspend fun getCategoryWithArticlesAndTags(categoryId: String): CategoryWithArticlesAndTags?
    
    // 管理关联关系
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertArticleTagCrossRef(crossRef: ArticleTagCrossRef)
    
    @Delete
    suspend fun deleteArticleTagCrossRef(crossRef: ArticleTagCrossRef)
    
    @Query("DELETE FROM article_tag_cross_ref WHERE article_id = :articleId")
    suspend fun deleteAllTagsForArticle(articleId: String)
    
    @Query("DELETE FROM article_tag_cross_ref WHERE tag_id = :tagId")
    suspend fun deleteAllArticlesForTag(tagId: String)
}
```

## 性能优化

### 查询优化

```kotlin
// 性能优化策略
class DatabaseOptimization {
    
    // 1. 索引优化
    @Entity(
        tableName = "optimized_articles",
        indices = [
            Index(value = ["category_id"]), // 单列索引
            Index(value = ["published_date", "view_count"]), // 复合索引
            Index(value = ["title"], unique = true), // 唯一索引
            Index(value = ["author_id", "status"]) // 查询优化索引
        ]
    )
    data class OptimizedArticle(
        @PrimaryKey val id: String,
        val title: String,
        val content: String,
        @ColumnInfo(name = "category_id") val categoryId: String,
        @ColumnInfo(name = "author_id") val authorId: String,
        @ColumnInfo(name = "published_date") val publishedDate: Long,
        @ColumnInfo(name = "view_count") val viewCount: Int,
        val status: String
    )
    
    @Dao
    interface OptimizedDao {
        
        // 2. 分页查询优化
        @Query("""
            SELECT * FROM optimized_articles 
            WHERE category_id = :categoryId 
            ORDER BY published_date DESC 
            LIMIT :pageSize OFFSET :offset
        """)
        suspend fun getArticlesPaged(
            categoryId: String,
            pageSize: Int,
            offset: Int
        ): List<OptimizedArticle>
        
        // 3. 只查询需要的列
        @Query("SELECT id, title, published_date FROM optimized_articles ORDER BY published_date DESC")
        suspend fun getArticleSummaries(): List<ArticleSummary>
        
        // 4. 使用 EXISTS 而不是 IN
        @Query("""
            SELECT * FROM optimized_articles a
            WHERE EXISTS (
                SELECT 1 FROM article_tag_cross_ref atcr 
                WHERE atcr.article_id = a.id AND atcr.tag_id = :tagId
            )
        """)
        suspend fun getArticlesByTagOptimized(tagId: String): List<OptimizedArticle>
        
        // 5. 批量操作
        @Insert(onConflict = OnConflictStrategy.REPLACE)
        suspend fun insertArticlesBatch(articles: List<OptimizedArticle>)
        
        @Query("UPDATE optimized_articles SET view_count = view_count + 1 WHERE id IN (:articleIds)")
        suspend fun incrementViewCountBatch(articleIds: List<String>)
    }
    
    data class ArticleSummary(
        val id: String,
        val title: String,
        @ColumnInfo(name = "published_date") val publishedDate: Long
    )
}
```

### 数据库配置优化

```kotlin
// 数据库配置优化
class DatabaseConfiguration {
    
    companion object {
        
        fun createOptimizedDatabase(context: Context): AppDatabase {
            return Room.databaseBuilder(
                context.applicationContext,
                AppDatabase::class.java,
                "app_database"
            )
            .addMigrations(
                DatabaseMigrations.MIGRATION_1_2,
                DatabaseMigrations.MIGRATION_2_3,
                DatabaseMigrations.MIGRATION_3_4
            )
            .setJournalMode(RoomDatabase.JournalMode.WAL) // 启用 WAL 模式
            .setQueryCallback({ sqlQuery, bindArgs ->
                if (BuildConfig.DEBUG) {
                    Log.d("RoomQuery", "SQL: $sqlQuery, Args: $bindArgs")
                }
            }, Executors.newSingleThreadExecutor())
            .addCallback(object : RoomDatabase.Callback() {
                override fun onCreate(db: SupportSQLiteDatabase) {
                    super.onCreate(db)
                    // 数据库创建时的初始化操作
                    initializeDatabase(db)
                }
                
                override fun onOpen(db: SupportSQLiteDatabase) {
                    super.onOpen(db)
                    // 数据库打开时的优化设置
                    db.execSQL("PRAGMA foreign_keys=ON")
                    db.execSQL("PRAGMA journal_mode=WAL")
                    db.execSQL("PRAGMA synchronous=NORMAL")
                    db.execSQL("PRAGMA cache_size=10000")
                    db.execSQL("PRAGMA temp_store=MEMORY")
                }
            })
            .build()
        }
        
        private fun initializeDatabase(db: SupportSQLiteDatabase) {
            // 插入初始数据
            db.execSQL("""
                INSERT INTO categories (id, name, description, parent_id) VALUES 
                ('tech', 'Technology', 'Technology related articles', null),
                ('science', 'Science', 'Science related articles', null),
                ('business', 'Business', 'Business related articles', null)
            """)
            
            db.execSQL("""
                INSERT INTO tags (id, name, color) VALUES 
                ('android', 'Android', '#3DDC84'),
                ('kotlin', 'Kotlin', '#7F52FF'),
                ('java', 'Java', '#ED8B00')
            """)
        }
    }
}
```

## 测试策略

### 数据库测试

```kotlin
// Room 数据库测试
@RunWith(AndroidJUnit4::class)
class DatabaseTest {
    
    private lateinit var database: AppDatabase
    private lateinit var articleDao: ArticleDao
    private lateinit var categoryDao: CategoryDao
    
    @Before
    fun createDb() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(
            context,
            AppDatabase::class.java
        ).build()
        articleDao = database.articleDao()
        categoryDao = database.categoryDao()
    }
    
    @After
    fun closeDb() {
        database.close()
    }
    
    @Test
    fun insertAndRetrieveArticle() = runTest {
        // Given
        val category = Category("tech", "Technology", "Tech articles", null)
        val article = Article(
            id = "1",
            title = "Test Article",
            content = "Test content",
            categoryId = "tech",
            authorId = "author1",
            publishedDate = Date(),
            metadata = ArticleMetadata(
                createdAt = System.currentTimeMillis(),
                updatedAt = System.currentTimeMillis(),
                createdBy = "author1"
            )
        )
        
        // When
        categoryDao.insertCategory(category)
        articleDao.insertArticle(article)
        
        // Then
        val retrievedArticle = articleDao.getUserById("1")
        assertThat(retrievedArticle).isNotNull()
        assertThat(retrievedArticle?.title).isEqualTo("Test Article")
    }
    
    @Test
    fun testArticleWithTags() = runTest {
        // Given
        val article = createTestArticle()
        val tag1 = Tag("android", "Android", "#3DDC84")
        val tag2 = Tag("kotlin", "Kotlin", "#7F52FF")
        
        // When
        articleDao.insertArticle(article)
        database.tagDao().insertTags(listOf(tag1, tag2))
        database.relationDao().insertArticleTagCrossRef(
            ArticleTagCrossRef(article.id, tag1.id)
        )
        database.relationDao().insertArticleTagCrossRef(
            ArticleTagCrossRef(article.id, tag2.id)
        )
        
        // Then
        val articleWithTags = database.relationDao().getArticleWithTags(article.id)
        assertThat(articleWithTags).isNotNull()
        assertThat(articleWithTags?.tags).hasSize(2)
        assertThat(articleWithTags?.tags?.map { it.name }).containsExactly("Android", "Kotlin")
    }
    
    @Test
    fun testDatabaseMigration() {
        // 测试数据库迁移
        val migrationTestHelper = MigrationTestHelper(
            InstrumentationRegistry.getInstrumentation(),
            AppDatabase::class.java
        )
        
        // 创建版本 1 的数据库
        var db = migrationTestHelper.createDatabase("test_db", 1)
        
        // 插入测试数据
        db.execSQL("""
            INSERT INTO articles (id, title, content, category_id, author_id, published_date, view_count, is_featured, created_at, updated_at, created_by)
            VALUES ('1', 'Test', 'Content', 'tech', 'author1', ${System.currentTimeMillis()}, 0, 0, ${System.currentTimeMillis()}, ${System.currentTimeMillis()}, 'author1')
        """)
        
        db.close()
        
        // 迁移到版本 2
        db = migrationTestHelper.runMigrationsAndValidate(
            "test_db",
            2,
            true,
            DatabaseMigrations.MIGRATION_1_2
        )
        
        // 验证迁移后的数据
        val cursor = db.query("SELECT * FROM articles WHERE id = '1'")
        assertThat(cursor.moveToFirst()).isTrue()
        assertThat(cursor.getString(cursor.getColumnIndex("title"))).isEqualTo("Test")
        cursor.close()
    }
    
    private fun createTestArticle(): Article {
        return Article(
            id = "test1",
            title = "Test Article",
            content = "Test content",
            categoryId = "tech",
            authorId = "author1",
            publishedDate = Date(),
            metadata = ArticleMetadata(
                createdAt = System.currentTimeMillis(),
                updatedAt = System.currentTimeMillis(),
                createdBy = "author1"
            )
        )
    }
}
```

## 最佳实践

### Repository 模式集成

```kotlin
// Repository 模式与 Room 集成
class ArticleRepository(
    private val articleDao: ArticleDao,
    private val relationDao: RelationDao,
    private val apiService: ApiService
) {
    
    // 响应式数据流
    fun observeArticles(): Flow<List<Article>> {
        return articleDao.observeAllArticles()
    }
    
    // 缓存策略
    suspend fun getArticles(forceRefresh: Boolean = false): Result<List<Article>> {
        return try {
            if (forceRefresh || shouldRefreshData()) {
                refreshArticlesFromNetwork()
            }
            
            val articles = articleDao.getAllArticles()
            Result.success(articles)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    private suspend fun refreshArticlesFromNetwork() {
        try {
            val networkArticles = apiService.getArticles()
            articleDao.insertArticles(networkArticles)
            updateLastRefreshTime()
        } catch (e: Exception) {
            // 网络错误时使用本地数据
            Log.w("ArticleRepository", "Failed to refresh from network", e)
        }
    }
    
    private fun shouldRefreshData(): Boolean {
        val lastRefresh = getLastRefreshTime()
        val now = System.currentTimeMillis()
        return (now - lastRefresh) > TimeUnit.HOURS.toMillis(1) // 1小时刷新一次
    }
    
    // 搜索功能
    suspend fun searchArticles(
        query: String,
        categoryId: String? = null,
        sortBy: String = "date"
    ): List<Article> {
        return articleDao.searchArticles(
            categoryId = categoryId,
            searchQuery = query.takeIf { it.isNotBlank() },
            startDate = 0,
            endDate = System.currentTimeMillis(),
            sortBy = sortBy
        )
    }
    
    // 事务操作
    @Transaction
    suspend fun createArticleWithTags(
        article: Article,
        tagIds: List<String>
    ): Result<Unit> {
        return try {
            articleDao.insertArticle(article)
            
            tagIds.forEach { tagId ->
                relationDao.insertArticleTagCrossRef(
                    ArticleTagCrossRef(article.id, tagId)
                )
            }
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // 批量操作
    suspend fun syncArticles(articles: List<Article>): Result<Unit> {
        return try {
            articleDao.insertArticles(articles)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

## 总结

Room 数据库为 Android 开发提供了强大而灵活的本地数据存储解决方案。通过本文的深入探讨，我们了解了：

1. **核心架构**：Entity、DAO、Database 三大组件的设计和使用
2. **高级特性**：复杂查询、关系映射、类型转换器等
3. **性能优化**：索引策略、查询优化、配置调优
4. **数据迁移**：版本升级和数据迁移的最佳实践
5. **测试策略**：全面的数据库测试方法
6. **架构集成**：与 Repository 模式和 MVVM 架构的结合

Room 的强大之处在于它提供了类型安全的数据库操作、编译时查询验证、与协程和 Flow 的无缝集成，以及对复杂关系映射的支持。通过遵循最佳实践和合理的架构设计，开发者可以构建出高性能、可维护的数据持久化层，为应用程序提供可靠的数据支撑。