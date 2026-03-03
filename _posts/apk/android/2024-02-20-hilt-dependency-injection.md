---
layout: post
title: Android Hilt 依赖注入完全指南
categories: android
tags: [Hilt, 依赖注入, Dagger, Android, DI]
date: 2024/2/20 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_029.jpg)

## 引言

Hilt 是 Google 基于 Dagger 2 推出的 Android 依赖注入框架，于 2020 年正式发布。它针对 Android 的生命周期和组件模型做了大量封装，简化了 Dagger 的配置，使依赖注入在 Android 应用中更易落地。通过 Hilt，开发者可以解耦依赖创建与使用，提高代码的可测试性（便于注入 Mock）、可维护性和可扩展性。本文将深入探讨 Hilt 的核心概念、使用方式、作用域管理以及最佳实践。

## 为什么需要依赖注入

当 ViewModel 内部直接 `UserRepository()` 创建依赖时，ViewModel 与具体实现强耦合，单元测试时难以替换为 Mock。依赖注入将依赖的创建移到外部，由 Hilt 在编译时生成注入代码，在运行时将实例注入到需要的地方。这样测试时可以注入 FakeRepository，生产环境注入真实实现，代码更灵活、更易测试。此外，单例、作用域等生命周期管理由 Hilt 统一处理，避免手动管理带来的内存泄漏和重复创建问题。

## Hilt 基础配置

### 添加依赖

Hilt 需要项目级和应用级两处配置。项目级添加 Hilt Gradle 插件，应用级应用插件并添加 Hilt 库和 kapt 处理器。注意使用 kapt 时需在应用级 build.gradle 中启用 `id("org.jetbrains.kotlin.kapt")`。

```kotlin
// 项目级 build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android") version "2.48" apply false
}

// 应用级 build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.48")
    kapt("com.google.dagger:hilt-compiler:2.48")
}
```

### Application 配置

Application 类必须添加 `@HiltAndroidApp` 注解，这是 Hilt 的入口。Hilt 会为该类生成 `ApplicationComponent`，作为依赖图的根，其他组件的依赖都由此向下传递。每个应用只能有一个带 `@HiltAndroidApp` 的 Application。

```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

## 依赖注入核心概念

### @Inject 构造函数注入

构造函数注入是首选方式：在类的构造函数上添加 `@Inject`，Hilt 会自动解析构造函数参数的类型，从依赖图中获取对应实例并创建对象。这种方式无需 Module，适用于 Hilt 能直接构造的类型。参数类型必须在 Hilt 的依赖图中可解析，否则编译时报错。

```kotlin
class UserRepository @Inject constructor(
    private val api: ApiService,
    private val database: AppDatabase
) {
    suspend fun getUsers(): List<User> {
        return api.getUsers()
    }
}
```

### @Module 和 @Provides

当无法直接构造类型时（如接口、第三方库类、需要配置的对象），使用 Module 的 `@Provides` 方法提供实例。Module 用 `@InstallIn` 指定安装到哪个组件，决定实例的作用域和生命周期。`@Singleton` 表示单例，同一组件内多次请求返回同一实例。Hilt 会自动将 `provideOkHttpClient` 的返回值注入到需要 `OkHttpClient` 的地方，并解析 `provideRetrofit` 对 `OkHttpClient` 的依赖。

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

### @Binds 接口绑定

当需要将接口绑定到具体实现时，使用 `@Binds`。Module 必须是 abstract 类，`@Binds` 方法也是 abstract，返回接口类型，参数为实现类。Hilt 在编译时生成实现，比 `@Provides` 返回实现类更高效。适用于 Repository、DataSource 等面向接口编程的场景。

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

## 作用域与组件

### Hilt 组件层次

- **SingletonComponent**：应用级别单例
- **ActivityRetainedComponent**：Activity 重建时保留
- **ActivityComponent**：Activity 作用域
- **ViewModelComponent**：ViewModel 作用域
- **FragmentComponent**：Fragment 作用域

选择合适的组件可以控制实例的生命周期。例如网络层、数据库等应使用 `SingletonComponent`；Activity 级别的 Presenter 可使用 `ActivityComponent`。注意：ViewModel 的注入应使用 `@HiltViewModel`，由 Hilt 自动处理，无需在 Module 中手动 provide。

```kotlin
@Module
@InstallIn(ActivityComponent::class)
object MainActivityModule {
    
    @Provides
    fun provideMainViewModel(
        repository: UserRepository
    ): MainViewModel {
        return MainViewModel(repository)
    }
}
```

## 在 Activity 和 Fragment 中使用

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    @Inject
    lateinit var userRepository: UserRepository
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // userRepository 已自动注入
    }
}
```

```kotlin
@AndroidEntryPoint
class UserFragment : Fragment() {
    
    @Inject
    lateinit var viewModelFactory: UserViewModelFactory
    
    private val viewModel: UserViewModel by viewModels { viewModelFactory }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewModel.loadUsers()
    }
}
```

## ViewModel 注入

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    val users = repository.getUsers().asLiveData()
    
    fun loadUsers() {
        viewModelScope.launch {
            repository.refreshUsers()
        }
    }
}
```

## 限定符与命名绑定

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class OtherInterceptorOkHttpClient

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @AuthInterceptorOkHttpClient
    @Provides
    @Singleton
    fun provideAuthOkHttpClient(authInterceptor: AuthInterceptor): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .build()
    }
    
    @OtherInterceptorOkHttpClient
    @Provides
    @Singleton
    fun provideOtherOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().build()
    }
}
```

## 最佳实践与注意事项

1. **面向接口编程**：依赖接口而非实现，便于测试和替换。使用 `@Binds` 绑定接口与实现。
2. **避免循环依赖**：若 A 依赖 B、B 依赖 A，需引入中间层或重新设计依赖关系。
3. **ViewModel 使用 @HiltViewModel**：不要手动创建 ViewModelFactory，Hilt 会自动生成并注入。
4. **测试时替换依赖**：使用 `@HiltAndroidTest` 和 `@UninstallModules` 在测试中替换生产 Module，注入 Fake 实现。

## 总结

Hilt 通过简化 Dagger 的配置，为 Android 开发提供了优雅的依赖注入解决方案。合理使用 Hilt 可以显著提升代码质量和开发效率。掌握构造函数注入、Module、作用域和 ViewModel 注入，是构建可测试、可维护 Android 应用的关键。
