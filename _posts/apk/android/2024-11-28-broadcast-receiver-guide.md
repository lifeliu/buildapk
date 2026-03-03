---
layout: post
title: Android BroadcastReceiver 广播接收器详解
categories: android
tags: [BroadcastReceiver, 广播, 系统广播, Android]
date: 2024/11/28 10:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_042.jpg)

## 引言

BroadcastReceiver 用于接收来自系统或其他应用的广播消息。它可以响应系统事件（如开机完成、网络变化、电量低、屏幕开关）或应用内自定义广播。广播分为标准广播（异步、无序）和有序广播（按优先级顺序传递，可中止）。Android 8.0 对隐式广播有严格限制，许多系统广播无法静态注册，需动态注册。对于应用内组件通信，LocalBroadcastManager 已废弃，推荐使用 LiveData、Flow 或事件总线。本文介绍 BroadcastReceiver 的静态与动态注册、系统广播限制以及最佳实践。

## 静态注册与动态注册

静态注册在 AndroidManifest 中声明，应用安装后即可接收广播，即使应用未运行（系统会启动进程）。动态注册在代码中 `registerReceiver`，需在适当时机 `unregisterReceiver`，通常与 Activity/Fragment 生命周期绑定。静态注册的 Receiver 在 `onReceive` 中不应执行耗时操作，超时会导致 ANR；动态注册的 Receiver 同样需快速返回，耗时任务应交给 Service 或 WorkManager。

## 静态注册

仅部分广播支持静态注册，如 `BOOT_COMPLETED`、`MY_PACKAGE_REPLACED` 等。`android:exported` 表示是否接收其他应用发出的广播，接收系统广播时通常为 true。注意：Android 8.0+ 对隐式广播限制较多，许多广播需改为动态注册。具体可接收的静态广播列表请参考官方文档。

```xml
<receiver
    android:name=".BootReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // 设备启动完成后的处理
        }
    }
}
```

## 动态注册

动态注册可指定 `RECEIVER_EXPORTED` 或 `RECEIVER_NOT_EXPORTED`，控制是否接收其他应用的广播。注册时传入 `IntentFilter` 指定要接收的 action。务必在生命周期结束时 `unregisterReceiver`，否则会导致内存泄漏。例如在 Activity 的 `onCreate` 中注册，在 `onDestroy` 中注销；或使用 `LifecycleEventObserver` 在合适的生命周期阶段管理。

```kotlin
val filter = IntentFilter().apply {
    addAction(ConnectivityManager.CONNECTIVITY_ACTION)
}

val receiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 处理网络变化
    }
}

registerReceiver(receiver, filter, RECEIVER_NOT_EXPORTED)

// 记得在 onDestroy 中注销
unregisterReceiver(receiver)
```

## 发送有序广播

`sendOrderedBroadcast` 按优先级顺序将广播传递给 Receiver，高优先级的 Receiver 可调用 `abortBroadcast` 中止传递。应用内广播可使用 `sendBroadcast` 或 `LocalBroadcastManager`（已废弃，可自建基于 LiveData 的轻量方案）。发送时建议 `intent.setPackage(packageName)` 限制接收范围，避免广播泄露。

```kotlin
val intent = Intent("com.example.CUSTOM_ACTION")
intent.setPackage(packageName)
sendOrderedBroadcast(intent, null)
```

## 总结

BroadcastReceiver 适合响应系统事件，对于应用内通信，建议优先使用 LiveData、Flow 或单例事件总线。注意 Android 8.0+ 的隐式广播限制，合理选择静态与动态注册，并在动态注册时及时注销，避免泄漏。
