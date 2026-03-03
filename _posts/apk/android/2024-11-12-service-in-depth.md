---
layout: post
title: Android Service 服务深入解析
categories: android
tags: [Service, 后台服务, Foreground Service, Bound Service]
date: 2024/11/12 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_041.jpg)

## 引言

Service 是 Android 四大组件之一，用于在后台执行长时间运行的操作，无需 UI。与 Activity 不同，Service 没有界面，适合音乐播放、文件下载、位置追踪等场景。Android 8.0 之后对后台执行有严格限制，前台服务（Foreground Service）需显示持久通知，且需声明具体类型；普通后台服务在应用进入后台后可能被系统 quickly 终止。本文深入解析 Service 的类型、生命周期、前台服务实现、绑定服务以及与现代后台方案（WorkManager）的配合。

## Service 类型

- **Started Service**：通过 `startService` 启动，与启动者生命周期解耦，适合一次性任务（如下载）。`stopService` 或 `stopSelf` 可停止。在后台运行时，系统可能因资源紧张终止服务。
- **Bound Service**：通过 `bindService` 绑定，与客户端（如 Activity）生命周期关联，当所有客户端解绑时服务可被销毁。适合提供 C/S 接口，如音乐播放控制。
- **Foreground Service**：通过 `startForeground` 显示持久通知，系统不会轻易终止，适用于用户明确感知的长时间任务（音乐、导航、下载等）。Android 9+ 需在 Manifest 声明 `foregroundServiceType`，Android 14+ 对部分类型有额外权限要求。

## 前台服务实现

前台服务必须在启动后 5 秒内调用 `startForeground`，否则会抛出异常或服务被终止。需先创建通知渠道，再构建通知并传入 `startForeground`。`onStartCommand` 的返回值：`START_STICKY` 表示服务被杀死后会尝试重启；`START_NOT_STICKY` 表示不重启；`START_REDELIVER_INTENT` 表示重启并重新传递最后的 Intent。根据业务选择合适策略。

```kotlin
class MusicService : Service() {
    
    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(NOTIFICATION_ID, createNotification())
        // 执行音乐播放逻辑
        return START_STICKY
    }
    
    private fun createNotification(): Notification {
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("音乐播放")
            .setContentText("正在播放...")
            .setSmallIcon(R.drawable.ic_music)
            .build()
    }
}
```

## 绑定服务

绑定服务通过 `Binder` 或 `Messenger` 与客户端通信。`Binder` 适用于同进程，客户端获取 Binder 后可直接调用服务方法。`ServiceConnection` 的 `onServiceConnected` 在绑定成功时回调，`onServiceDisconnected` 在服务异常断开时回调（非正常 unbind）。使用 `BIND_AUTO_CREATE` 表示若服务未运行则自动创建。记得在 Activity 的 `onDestroy` 中 `unbindService`，避免泄漏。

```kotlin
class LocalService : Service() {
    private val binder = LocalBinder()
    
    inner class LocalBinder : Binder() {
        fun getService(): LocalService = this@LocalService
    }
    
    override fun onBind(intent: Intent): IBinder = binder
}

// 客户端
val connection = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        val binder = service as LocalService.LocalBinder
        localService = binder.getService()
    }
    override fun onServiceDisconnected(name: ComponentName?) {}
}
bindService(Intent(this, LocalService::class.java), connection, BIND_AUTO_CREATE)
```

## 总结

正确选择和使用 Service 类型，结合 WorkManager 处理可延迟任务，能够满足各类后台需求。前台服务适用于用户可感知的长时间任务，需注意通知和类型声明；绑定服务适用于与 UI 紧密交互的场景。关注 Android 版本对后台限制的演进，优先使用 WorkManager、JobScheduler 等推荐方案。
