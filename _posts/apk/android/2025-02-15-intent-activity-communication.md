---
layout: post
title: Android Intent 与 Activity 通信详解
categories: android
tags: [Intent, Activity, 组件通信, 显式Intent, 隐式Intent]
date: 2025/2/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_048.jpg)

## 引言

Intent 是 Android 中用于组件间通信的核心机制。通过 Intent，可以启动 Activity、Service，发送广播，以及在不同组件间传递数据。Intent 分为显式（指定具体组件类）和隐式（通过 action、category、data 匹配），分别适用于应用内跳转和跨应用调用。本文深入探讨显式与隐式 Intent、Intent Filter、数据传递、startActivityForResult 的现代替代（ActivityResult API）以及深层链接（Deep Link）的实现。

## 显式 Intent

显式 Intent 指定目标组件的类名或包名，用于应用内跳转。通过 `putExtra` 传递基本类型、String、Parcelable、Serializable 等数据。注意：Intent 有大小限制（约 1MB），大数据应通过单例、数据库或文件共享。`startActivity` 启动目标 Activity，`startActivityForResult` 已废弃，推荐使用 `registerForActivityResult` 和 `ActivityResultContracts.StartActivityForResult`。

```kotlin
// 启动 Activity
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("item_id", "123")
intent.putExtra("item_name", "Product")
startActivity(intent)

// 带返回结果的启动（使用 ActivityResult API）
val launcher = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
    if (result.resultCode == RESULT_OK) {
        val data = result.data
        // 处理返回数据
    }
}
launcher.launch(Intent(this, SelectActivity::class.java))
```

## 隐式 Intent

隐式 Intent 不指定组件，通过 action、category、data 描述意图，系统根据各应用的 Intent Filter 匹配并弹出选择器或直接启动。常用于调用系统或其他应用的功能，如分享、打开链接、拨号等。使用 `Intent.createChooser` 可强制显示选择器，避免默认应用直接处理。注意：Android 11+ 对包可见性有限制，查询可处理某 Intent 的应用时需在 Manifest 声明 `queries`。

```kotlin
val intent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "分享内容")
}
startActivity(Intent.createChooser(intent, "选择分享方式"))
```

## Intent Filter

Intent Filter 在 Manifest 中声明，描述组件可响应的 Intent。`action`、`category`、`data` 组合匹配。例如声明 `VIEW` action 和 `https` scheme，可让 Activity 响应网页链接，实现 Deep Link。`android:autoVerify` 配合 `assetlinks.json` 可实现 App Links（验证域名归属后直接打开应用，不弹选择器）。Intent Filter 支持多 action、多 data，可同时支持 `myapp://` 和 `https://example.com/` 等格式。

```xml
<activity android:name=".DetailActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="myapp" android:host="detail" />
    </intent-filter>
</activity>
```

## 接收数据

目标 Activity 通过 `intent.getStringExtra("key")`、`intent.getParcelableExtra<Model>("key")` 等获取数据。使用 Parcelable 比 Serializable 性能更好，Kotlin 的 `@Parcelize` 可简化实现。注意空安全：`getStringExtra` 可能返回 null，需做空处理。对于复杂数据或大数据，考虑 ViewModel、单例或持久化存储。

```kotlin
class DetailActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val itemId = intent.getStringExtra("item_id")
        val itemName = intent.getStringExtra("item_name")
    }
}
```

## 总结

合理使用 Intent 和 Intent Filter 能够实现灵活的组件通信和深度链接功能。显式 Intent 用于应用内跳转，隐式 Intent 用于跨应用调用。使用 ActivityResult API 替代 startActivityForResult，注意数据大小限制和空安全处理。Deep Link 和 App Links 可提升从外部（如网页、推送）打开应用的体验。
