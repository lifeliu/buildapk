---
layout: post
title: Flutter 依赖注入与服务定位
categories: flutter
tags: [Flutter, 依赖注入]
date: 2024/10/5 09:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_009.jpg)

## 引言

依赖注入通过解耦依赖的创建与使用，提高可测试性和可维护性。Flutter 中常用的方案有 get_it（服务定位）、Provider/Riverpod（同时承担状态管理和依赖提供）、injectable（基于 get_it 的代码生成）。get_it 是纯依赖注入容器，不涉及 UI 重建，适合注入 Repository、ApiService 等无状态服务。本文将介绍 get_it 的注册、解析、作用域，以及与 injectable 的配合使用。

## 为什么需要依赖注入

当 ViewModel 内部直接 `ApiService()` 创建依赖时，ViewModel 与具体实现强耦合，单元测试时难以替换为 Mock。依赖注入将依赖的创建移到外部，由 get_it 在应用启动时注册，在需要时解析并注入。这样测试时可以 registerSingleton<ApiService>(MockApiService())，生产环境注入真实实现。此外，单例、工厂、异步初始化等生命周期管理由 get_it 统一处理。

## get_it

GetIt 是单例，通过 GetIt.instance 或自定义实例访问。registerSingleton 注册单例，registerFactory 每次解析创建新实例，registerLazySingleton 首次解析时创建并缓存。registerSingletonAsync 用于异步初始化（如数据库）。unregister 可移除注册，便于测试时替换。resolve 或 get 解析依赖，若未注册会抛出异常。

```dart
final getIt = GetIt.instance;

void setup() {
  getIt.registerSingleton<ApiService>(ApiService());
  getIt.registerFactory<Repository>(() => Repository(getIt<ApiService>()));
}

// 使用
final api = getIt<ApiService>();
```

## 与 Riverpod 的配合

Riverpod 的 Provider 本身可提供依赖，ref.watch(apiServiceProvider) 即可获取。若项目已使用 get_it，可在 Riverpod 的 Provider 中通过 getIt<T>() 获取，实现混合使用。对于纯业务服务，get_it 足够；对于需要与 UI 联动的状态，Riverpod 更合适。

## 总结

合理使用依赖注入可构建可测试、可维护的 Flutter 应用。get_it 适合无状态服务的注入，结合 injectable 可减少样板代码。根据项目规模选择 get_it、Riverpod 或两者配合。
