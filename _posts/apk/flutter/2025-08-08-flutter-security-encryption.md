---
layout: post
title: Flutter 安全与加密完全指南
categories: flutter
tags: [Flutter, 安全]
date: 2025/8/8 14:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_027.jpg)

## 引言

Flutter 应用需保护敏感数据，如 Token、密钥、用户凭证。SharedPreferences 以明文存储，不适合敏感信息。flutter_secure_storage 使用 Android 的 Keystore 和 iOS 的 Keychain 安全存储，数据加密保存。encrypt 包支持 AES、RSA 等加密算法，可用于加密字符串或文件。本文介绍安全存储、加密使用以及最佳实践（如避免硬编码密钥、HTTPS、证书校验等）。

## 为什么需要安全存储

Token、密码等若存储在 SharedPreferences 或文件中，可能被 rooted 设备、备份或调试工具读取。Keychain/Keystore 由系统保护，加密存储在安全区域，普通应用无法直接访问。flutter_secure_storage 提供跨平台的统一 API，底层使用平台安全存储。

## flutter_secure_storage

FlutterSecureStorage 的 write 和 read 为异步操作。可配置 aOptions 指定 Android 的加密模式、iOS 的 accessibility 等。delete 和 deleteAll 可移除数据。注意：在部分 Android 设备上，若用户未设置锁屏密码，Keystore 可能不可用，需处理异常。iOS 模拟器上 Keychain 行为可能与真机有差异。

```dart
final storage = FlutterSecureStorage();

await storage.write(key: 'token', value: token);
final token = await storage.read(key: 'token');
await storage.delete(key: 'token');
await storage.deleteAll();
```

## 加密

encrypt 包提供 Encrypter、Key、IV 等，支持 AES。加密时需妥善管理密钥，避免硬编码。密钥可来自用户密码派生（如 PBKDF2）或从安全存储读取。对于网络传输，使用 HTTPS 并正确校验证书，避免中间人攻击。

## 总结

敏感数据应使用安全存储，避免明文保存。flutter_secure_storage 是 Token、密钥等存储的推荐方案。结合 HTTPS、证书校验、代码混淆等，可构建多层安全防护。注意密钥管理和平台差异，做好异常处理。
