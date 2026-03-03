---
layout: post
title: Flutter 推送通知完全指南
categories: flutter
tags: [Flutter, 推送]
date: 2025/1/15 14:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_015.jpg)

## 引言

Flutter 推送通常使用 Firebase Cloud Messaging（FCM），通过 firebase_messaging 插件实现。FCM 支持 Android 和 iOS，需在 Firebase 控制台配置项目并下载配置文件。flutter_local_notifications 用于本地通知，可在应用内调度通知而无需服务器。本文将介绍 FCM 的配置、权限请求、接收消息、点击处理以及本地通知的基本用法。

## FCM 流程概述

应用启动时调用 requestPermission 请求通知权限（iOS 必需，Android 13+ 也需）。getToken 获取 FCM token，需上传到服务器，服务器推送时携带 token。onMessage 在应用前台时收到消息，可自定义展示；onMessageOpenedApp 在用户通过通知打开应用时触发；getInitialMessage 处理应用从终止状态被通知启动的情况。后台消息需在原生层处理，或使用 flutter_local_notifications 配合后台 isolate。

## firebase_messaging

确保已配置 Firebase（google-services.json、GoogleService-Info.plist）、Firebase.initializeApp。requestPermission 在 iOS 上会弹出权限对话框。onMessage 收到的 RemoteMessage 包含 notification 和 data，可据此更新 UI 或显示自定义通知。onMessageOpenedApp 和 getInitialMessage 可用于深度链接，根据 data 跳转到对应页面。

```dart
await FirebaseMessaging.instance.requestPermission();
final token = await FirebaseMessaging.instance.getToken();
// 将 token 发送到服务器

FirebaseMessaging.onMessage.listen((RemoteMessage message) {
  // 前台接收，可显示自定义 In-App 通知
});

FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
  // 用户点击通知打开应用
  final route = message.data['route'];
  if (route != null) navigatorKey.currentState?.pushNamed(route);
});
```

## 总结

合理使用推送可提升用户留存，需注意权限申请时机和通知内容的质量。FCM 需配置 Firebase 和原生端，本地通知可满足应用内提醒需求。注意 Android 和 iOS 的通道、权限差异，做好多平台适配。
