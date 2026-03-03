---
layout: post
title: Android WorkManager 后台任务完全指南
categories: android
tags: [WorkManager, 后台任务, 定时任务, Android, Jetpack]
date: 2024/4/10 14:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_030.jpg)

## 引言

WorkManager 是 Android Jetpack 架构组件之一，用于处理可延迟的后台任务。与 JobScheduler、AlarmManager 或直接使用 Service 不同，WorkManager 提供了统一的 API，能够保证任务一定会被执行，即使应用被关闭或设备重启。WorkManager 会根据设备状态（API 级别、Doze 模式、电池优化等）选择最佳执行方式，兼容 Android 各版本的后台限制。本文将深入探讨 WorkManager 的使用方式、约束条件、链式任务以及最佳实践。

## 为什么选择 WorkManager

Android 8.0 之后对后台执行有严格限制，直接使用 Service 或 AlarmManager 可能无法按时执行。WorkManager 由 Google 推荐，适用于"最终会执行即可"的任务，如数据同步、日志上传、图片压缩等。它不适用于需要精确时间触发的任务（如闹钟）或需要立即执行的任务（应用在前台时应使用协程等）。WorkManager 内部会选择合适的调度器（JobScheduler、AlarmManager 或 WorkManager 自带的实现），开发者无需关心底层差异。

## 添加依赖

```kotlin
dependencies {
    implementation("androidx.work:work-runtime-ktx:2.9.0")
}
```

`work-runtime-ktx` 提供 Kotlin 扩展和 CoroutineWorker 支持。若使用 Java，可仅依赖 `work-runtime`。

## 创建 Worker

Worker 是任务的执行单元。继承 `CoroutineWorker` 可使用 Kotlin 协程，在 `doWork()` 中执行挂起函数。返回 `Result.success()` 表示成功，`Result.failure()` 表示失败且不重试，`Result.retry()` 表示失败但希望 WorkManager 稍后重试。通过 `inputData` 接收输入参数，通过 `setOutputData()` 可向链式任务中的下一个 Worker 传递数据。注意：`doWork()` 在后台线程执行，不要在此进行 UI 操作。

```kotlin
class UploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val imageUri = inputData.getString("image_uri") ?: return Result.failure()
            uploadImage(imageUri)
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
    
    private suspend fun uploadImage(uri: String) {
        // 执行上传逻辑
    }
}
```

## 一次性任务

一次性任务通过 `OneTimeWorkRequest` 创建，可设置输入数据、约束条件（网络、电量、存储等）和标签。约束不满足时，WorkManager 会等待条件满足后再执行。`enqueue` 将任务加入队列，系统会选择合适的时机执行。可调用 `cancelWorkById` 或 `cancelUniqueWork` 取消任务。

```kotlin
val uploadRequest = OneTimeWorkRequestBuilder<UploadWorker>()
    .setInputData(workDataOf(
        "image_uri" to imageUri.toString()
    ))
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresCharging(false)
            .build()
    )
    .addTag("upload")
    .build()

WorkManager.getInstance(context).enqueue(uploadRequest)
```

## 周期性任务

周期性任务的最小间隔为 15 分钟，即使设置更短也会被系统调整。使用 `enqueueUniquePeriodicWork` 可确保同一名称的任务只有一个，避免重复注册。`ExistingPeriodicWorkPolicy.KEEP` 表示若已存在则保留原有任务，`REPLACE` 表示替换。周期性任务适合数据同步、定期清理等场景。

```kotlin
val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
    15, TimeUnit.MINUTES  // 最小间隔 15 分钟
)
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sync",
    ExistingPeriodicWorkPolicy.KEEP,
    syncRequest
)
```

## 链式任务

链式任务将多个 Worker 按顺序执行，前一个的输出可作为后一个的输入。`beginWith` 指定第一个任务，`then` 添加后续任务。适用于"压缩 → 上传"、"下载 → 解析 → 存储"等多步骤流程。任一 Worker 返回 `Result.failure()` 时，后续任务不会执行。

```kotlin
val compressWork = OneTimeWorkRequestBuilder<CompressWorker>()
    .setInputData(workDataOf("image_uri" to uri.toString()))
    .build()

val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>()
    .build()

WorkManager.getInstance(context)
    .beginWith(compressWork)
    .then(uploadWork)
    .enqueue()
```

## 任务状态观察

通过 `getWorkInfosForUniqueWorkLiveData` 或 `getWorkInfoByIdLiveData` 可观察任务状态变化，用于更新 UI（如显示上传进度、完成提示）。`WorkInfo.State` 包括 `ENQUEUED`、`RUNNING`、`SUCCEEDED`、`FAILED`、`CANCELLED`、`BLOCKED`。

```kotlin
WorkManager.getInstance(context).getWorkInfosForUniqueWorkLiveData("sync")
    .observe(this) { workInfos ->
        workInfos.forEach { info ->
            when (info.state) {
                WorkInfo.State.RUNNING -> showProgress()
                WorkInfo.State.SUCCEEDED -> showSuccess()
                WorkInfo.State.FAILED -> showError()
            }
        }
    }
```

## 总结

WorkManager 是处理可延迟后台任务的推荐方式，能够确保任务可靠执行，同时尊重系统资源限制。合理使用约束、链式任务和状态观察，可以构建健壮的后台处理逻辑。注意区分一次性与周期性任务，以及任务取消和重试策略。
