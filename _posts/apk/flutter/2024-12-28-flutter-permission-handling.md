---
layout: post
title: Flutter 权限管理完全指南
categories: flutter
tags: [Flutter, 权限]
date: 2024/12/28 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_014.jpg)

## 引言

Flutter 通过平台通道与原生通信，权限请求需依赖原生能力。permission_handler 插件封装了 Android 和 iOS 的权限 API，提供统一的 Dart 接口。使用前需在 Android 的 AndroidManifest.xml 和 iOS 的 Info.plist 中声明权限及用途描述，否则无法请求或审核会被拒。本文将介绍 permission_handler 的用法、权限状态检查、请求时机以及引导用户开启权限的策略。

## 权限与平台配置

不同权限对应不同的 Manifest 和 Info.plist 配置。例如相机需 Android 的 CAMERA 和 iOS 的 NSCameraUsageDescription。permission_handler 的 Permission 枚举对应各权限，request 会弹出系统权限对话框。status 包括 granted、denied、permanentlyDenied（用户选择"不再询问"）、restricted（系统限制）等。permanentlyDenied 时需引导用户前往应用设置页手动开启，openAppSettings 可打开设置。

## permission_handler

在适当时机请求权限，而非应用启动时批量请求。例如用户点击"拍照"时再请求相机权限，这样用户能理解权限用途，授权率更高。先 checkPermissionStatus 检查当前状态，若 denied 再 request。注意：request 可能抛出异常，需 try-catch。部分权限（如通知）在 Android 13+ 需单独处理。

```dart
import 'package:permission_handler/permission_handler.dart';

Future<void> requestCamera() async {
  final status = await Permission.camera.status;
  if (status.isGranted) {
    // 已授权，打开相机
    openCamera();
    return;
  }
  if (status.isDenied) {
    final result = await Permission.camera.request();
    if (result.isGranted) {
      openCamera();
    } else if (result.isPermanentlyDenied) {
      // 引导用户去设置
      await openAppSettings();
    }
  }
}
```

## 总结

合理请求权限并在适当时机展示说明，可提高用户授权率。遵循最小权限原则，仅在需要时请求，并妥善处理拒绝和永久拒绝的情况。注意 Android 和 iOS 的权限差异，做好平台特定配置。
