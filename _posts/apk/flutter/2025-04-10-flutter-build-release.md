---
layout: post
title: Flutter 构建与发布完全指南
categories: flutter
tags: [Flutter, 构建]
date: 2025/4/10 09:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_020.jpg)

## 引言

Flutter 通过 flutter build 生成各平台产物。Android 可生成 APK（直接安装）或 App Bundle（上传 Google Play，支持动态交付，体积更优）。iOS 生成 IPA，需配置签名和描述文件。构建前需配置 android/app/build.gradle 和 ios/Runner 的签名、版本号、图标等。本文将介绍构建命令、签名配置、版本号管理以及商店发布流程的要点。

## 构建前准备

Android：在 android/app/build.gradle 中配置 signingConfigs，使用 keystore 签名。versionCode 和 versionName 需随发布递增。iOS：在 Xcode 中配置 Signing & Capabilities，使用 Apple Developer 账号的证书和 Provisioning Profile。iOS 的版本号在 pubspec.yaml 或 Xcode 的 Info.plist 中配置。图标使用 flutter_launcher_icons 等工具生成各尺寸。

## 构建命令

flutter build apk 生成 release APK，默认生成多种 ABI 的 fat APK，可加 --split-per-abi 生成分架构 APK 以减小单包体积。flutter build appbundle 生成 AAB，用于 Google Play 上传。flutter build ios 生成 iOS 产物，需在 Xcode 中 Archive 并导出 IPA。flutter build 会先执行 pub get 和代码生成，确保无错误后再构建。

```bash
flutter build apk --release
flutter build apk --split-per-abi
flutter build appbundle
flutter build ios
```

## 总结

正确配置签名和版本号是发布的前提。App Bundle 可减小 Android 应用体积并支持 Play Asset Delivery。发布前在真机上充分测试，关注各商店的审核指南和隐私政策要求。
