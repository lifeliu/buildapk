---
layout: post
title: Android 应用启动优化完全指南
categories: android
tags: [启动优化, App Startup, 冷启动, 热启动, Android]
date: 2025/1/25 11:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_047.jpg)

## 引言

应用启动速度直接影响用户的第一印象，是留存和体验的关键指标。Android 提供了多种工具和最佳实践来优化启动性能：App Startup 库用于统一管理初始化逻辑、Baseline Profile 用于 AOT 优化、Android Vitals 和 Firebase Performance 可监控启动耗时。本文介绍冷启动与温启动的差异、App Startup 的用法、延迟初始化策略以及常见优化手段。

## 启动类型

- **冷启动**：应用进程不存在，系统从零创建进程、加载 Application、启动 Activity。耗时最长，是优化重点。
- **温启动**：应用进程存在，但 Activity 已被销毁（如用户按 Home 后系统回收）。需重新创建 Activity，但可复用部分已初始化的资源。
- **热启动**：Activity 仍在栈中，从后台恢复到前台。耗时最短，通常无需特别优化。

冷启动的耗时主要包括：进程创建、Application 初始化、首屏 Activity 的创建与布局、首帧绘制。优化方向是减少主线程工作、延迟非关键初始化、减少首屏布局复杂度。

## App Startup 库

App Startup 提供统一的初始化入口，通过 ContentProvider 在应用启动时自动执行各库的初始化。各库实现 `Initializer` 接口，声明依赖关系，App Startup 会按依赖顺序执行，避免重复初始化。这样可替代各库自行注册 ContentProvider 的方式，减少 ContentProvider 数量，从而减少启动时的合并与执行开销。WorkManager、ProcessLifecycleOwner 等已支持 App Startup。

```kotlin
dependencies {
    implementation("androidx.startup:startup-runtime:1.1.1")
}
```

```kotlin
class WorkManagerInitializer : Initializer<Unit> {
    override fun create(context: Context): Unit {
        WorkManager.initialize(context, Configuration.Builder().build())
        return Unit
    }
    
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}
```

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup">
    <meta-data
        android:name="com.example.WorkManagerInitializer"
        android:value="androidx.startup" />
</provider>
```

## 延迟初始化

将非首屏必需的初始化延后执行，可缩短首帧时间。在 Application 的 `onCreate` 中仅初始化启动必需组件（如崩溃收集、基础配置）。对于 WorkManager、分析 SDK、推送等，可放到 `mainExecutor.execute { }` 或 `Handler.post` 中延迟执行，或使用 App Startup 的 `Initializer` 并设置 `dependencies` 让其在合适的时机执行。注意：延迟执行不能影响首屏功能，需评估各组件的启动依赖。

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // 仅初始化启动必需组件
        initEssentialComponents()
        
        // 延迟初始化非关键组件
        (getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager)
            .mainExecutor.execute { initNonCriticalComponents() }
    }
}
```

## 其他优化手段

- **Baseline Profile**：记录关键路径的代码，在安装时预编译，减少首次运行的 JIT 开销。
- **减少 Application 和首屏 Activity 的 work**：避免在主线程进行网络、磁盘、复杂计算。
- **简化首屏布局**：减少层级、使用 ViewStub 延迟加载非首屏内容、考虑使用 SplashScreen API 展示品牌画面。
- **使用 StrictMode 和 Profiler**：检测主线程 I/O、分析启动阶段的耗时分布。

## 总结

通过减少主线程工作、延迟初始化、使用 App Startup 和 Baseline Profile，可以显著提升应用启动速度。优先优化冷启动，关注 Application 和首屏 Activity 的初始化逻辑，持续监控启动指标并迭代优化。
