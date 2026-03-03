---
layout: post
title: Android 通知系统完全指南
categories: android
tags: [Notification, 通知, NotificationChannel, Android]
date: 2024/10/25 09:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_040.jpg)

## 引言

Android 通知系统是应用与用户沟通的重要渠道，用于提醒、进度展示、快捷操作等。从 Android 8.0（API 26）开始，通知渠道（NotificationChannel）成为必需配置，用户可按渠道管理通知行为（声音、振动、重要性等）。Android 13 引入了运行时通知权限（POST_NOTIFICATIONS），需在发送通知前请求。本文介绍通知渠道创建、基础通知、进度通知、点击处理以及权限与最佳实践。

## 通知渠道的作用

通知渠道允许用户按类别管理通知：例如将"营销推送"静音，保留"订单状态"的声音。应用在首次发送某渠道的通知前，必须创建该渠道；渠道创建后其重要性等设置可由用户修改，应用无法再更改。合理划分渠道（如按功能：消息、订单、系统）可以提升用户对通知的掌控感，减少关闭应用通知的情况。

## 创建通知渠道

渠道 ID 需唯一，名称和描述会展示给用户。重要性（IMPORTANCE_HIGH、IMPORTANCE_DEFAULT 等）影响通知的默认行为。创建渠道后，后续发送到该 channelId 的通知都会使用该渠道。渠道只需创建一次，通常放在 Application 或首次需要通知时；重复创建不会报错，但也不会更新已有渠道的设置。

```kotlin
fun createNotificationChannel(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val channel = NotificationChannel(
            CHANNEL_ID,
            "重要通知",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description = "应用重要消息"
            enableVibration(true)
            enableLights(true)
        }
        
        val manager = context.getSystemService(NotificationManager::class.java)
        manager.createNotificationChannel(channel)
    }
}
```

## 基础通知

使用 `NotificationCompat.Builder` 保证低版本兼容。`setSmallIcon` 是必需的，使用 alpha 通道的图标以适配不同主题。`setContentIntent` 设置点击后的跳转，使用 `PendingIntent`。`setAutoCancel(true)` 表示点击后自动移除通知。Android 13+ 需在 Manifest 声明 `POST_NOTIFICATIONS` 权限并在运行时请求，否则无法发送通知。使用 `NotificationManagerCompat.from(context).notify()` 可自动处理权限检查（在部分版本上）。

```kotlin
fun showNotification(context: Context, title: String, content: String) {
    val intent = Intent(context, MainActivity::class.java)
    val pendingIntent = PendingIntent.getActivity(
        context, 0, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )
    
    val notification = NotificationCompat.Builder(context, CHANNEL_ID)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(content)
        .setPriority(NotificationCompat.PRIORITY_HIGH)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .build()
    
    NotificationManagerCompat.from(context).notify(NOTIFICATION_ID, notification)
}
```

## 进度通知

进度通知通过不断更新同一 ID 的 notification 实现，`setProgress(max, progress, indeterminate)` 设置进度条。下载完成后更新为完成状态并移除进度条，或调用 `cancel` 移除通知。`indeterminate` 为 true 时显示无限进度条，适用于无法确定进度的场景。

```kotlin
val builder = NotificationCompat.Builder(context, CHANNEL_ID)
    .setContentTitle("下载中")
    .setSmallIcon(R.drawable.ic_download)
    .setProgress(100, progress, false)

notificationManager.notify(NOTIFICATION_ID, builder.build())
```

## 总结

合理使用通知能够提升用户体验，同时需注意 Android 13+ 的权限申请、渠道划分和通知频率。避免过度推送，提供清晰的渠道名称和描述，让用户能够按需管理，是通知设计的最佳实践。
