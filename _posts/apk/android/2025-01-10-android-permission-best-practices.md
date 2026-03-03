---
layout: post
title: Android 权限请求最佳实践
categories: android
tags: [权限, Permission, 运行时权限, Android, 隐私]
date: 2025/1/10 09:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_046.jpg)

## 引言

Android 6.0（API 23）引入了运行时权限机制，危险权限（如相机、位置、存储）需在运行时动态申请，用户可随时在设置中撤销。Android 10 对存储权限进行了分区限制（Scoped Storage），Android 11 进一步收紧，Android 13 引入了细粒度的媒体权限（READ_MEDIA_IMAGES、READ_MEDIA_VIDEO 等）。本文介绍权限请求的最佳实践、ActivityResult API、多权限处理以及兼容性策略。

## 权限分类与变化

- **普通权限**：在 Manifest 声明即可，无需运行时请求（如网络、振动）。
- **危险权限**：需在 Manifest 声明，并在使用前运行时请求（如相机、位置、存储、麦克风）。
- **特殊权限**：如悬浮窗、修改系统设置，需跳转设置页由用户手动开启。

存储权限演变：Android 9 及以下使用 READ/WRITE_EXTERNAL_STORAGE；Android 10–12 使用 READ_EXTERNAL_STORAGE 和 MANAGE_EXTERNAL_STORAGE（后者需特殊审批）；Android 13+ 使用 READ_MEDIA_IMAGES、READ_MEDIA_VIDEO、READ_MEDIA_AUDIO 等细分权限。需根据 targetSdk 和 minSdk 做兼容处理。

## 检查权限

使用 `ContextCompat.checkSelfPermission` 检查是否已授权。若未授权，可先调用 `shouldShowRequestPermissionRationale` 判断是否应显示说明（用户曾拒绝且未勾选"不再询问"时返回 true）。根据情况决定是直接请求、显示说明后再请求，还是引导用户去设置页手动开启。注意：用户选择"不再询问"后，`shouldShowRequestPermissionRationale` 返回 false，且再次请求不会弹窗，只能引导去设置。

```kotlin
when {
    ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA) 
        == PackageManager.PERMISSION_GRANTED -> {
        // 已有权限
        openCamera()
    }
    ActivityCompat.shouldShowRequestPermissionRationale(activity, Manifest.permission.CAMERA) -> {
        // 用户曾拒绝，显示说明
        showPermissionRationale()
    }
    else -> {
        // 请求权限
        requestPermissions(arrayOf(Manifest.permission.CAMERA), REQUEST_CAMERA)
    }
}
```

## 使用 ActivityResult API

`registerForActivityResult` 是替代 `startActivityForResult` 和 `requestPermissions` 的现代 API，无需重写 `onRequestPermissionsResult`。`ActivityResultContracts.RequestPermission` 用于单个权限，`RequestMultiplePermissions` 用于多个权限。Launcher 在 Activity/Fragment 中注册，在需要时 `launch`。结果在回调中处理，代码更清晰。

```kotlin
private val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) {
        openCamera()
    } else {
        showPermissionDeniedMessage()
    }
}

requestPermissionLauncher.launch(Manifest.permission.CAMERA)
```

## 多权限请求

多权限时使用 `RequestMultiplePermissions`，回调收到 Map<权限, 是否授权>。可遍历检查每个权限，分别处理。若需全部授权才继续，可判断 `permissions.values.all { it }`。注意：系统可能一次弹窗请求多个权限，用户可能部分授权部分拒绝，需分别处理。

```kotlin
private val requestMultiplePermissions = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    permissions.entries.forEach { (permission, granted) ->
        when (permission) {
            Manifest.permission.CAMERA -> if (granted) setupCamera()
            Manifest.permission.RECORD_AUDIO -> if (granted) setupMicrophone()
        }
    }
}

requestMultiplePermissions.launch(
    arrayOf(Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO)
)
```

## 总结

遵循最小权限原则，在用户即将使用需要权限的功能时再请求，而非启动时批量请求。提供清晰的权限说明（Rationale），尊重用户拒绝的选择，必要时引导去设置页。使用 ActivityResult API 简化代码，并根据 Android 版本做好存储等权限的兼容处理。
