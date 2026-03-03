---
layout: post
title: Android Gradle 构建配置完全指南
categories: android
tags: [Gradle, 构建配置, Kotlin DSL, Android, 依赖管理]
date: 2024/9/20 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_038.jpg)

## 引言

Gradle 是 Android 项目的标准构建系统，负责编译、打包、依赖管理和多渠道构建。通过 Kotlin DSL（build.gradle.kts）或 Groovy DSL（build.gradle），开发者可以配置 SDK 版本、构建类型、产品风味、依赖等。理解 Gradle 配置有助于优化构建速度、管理多环境构建、统一依赖版本。本文将深入探讨项目级与应用级配置、构建变体、依赖管理以及常见优化技巧。

## Gradle 配置结构

Android 项目通常包含项目级 `build.gradle.kts`（或 `settings.gradle.kts`）和应用级 `app/build.gradle.kts`。项目级定义插件版本、仓库和子项目；应用级配置具体模块的 Android 选项和依赖。Kotlin DSL 提供类型安全和 IDE 支持，是当前推荐方式。版本目录（Version Catalog）可将依赖版本集中管理，在 `libs.versions.toml` 中定义，通过 `libs.xxx` 引用。

## 项目级 build.gradle.kts

项目级使用 `apply false` 声明插件及版本，不在根项目应用，由子项目按需应用。这样可统一各模块的插件版本，避免冲突。Hilt、Kotlin、Navigation 等插件的版本在此集中管理。

```kotlin
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.20" apply false
    id("com.google.dagger.hilt.android") version "2.48" apply false
}
```

## 应用级 build.gradle.kts

`namespace` 替代了 Manifest 中的 package，用于 R 类和 BuildConfig 的包名。`compileSdk` 决定可用的 API 级别，`minSdk` 和 `targetSdk` 影响兼容性和行为。`buildTypes` 定义 debug、release 等类型，release 通常开启混淆（minifyEnabled）以减小体积。`buildFeatures` 启用 ViewBinding、BuildConfig、Compose 等。`kotlin-kapt` 用于注解处理（如 Hilt、Room）。

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
}

android {
    namespace = "com.example.app"
    compileSdk = 34
    
    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }
    
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    
    buildFeatures {
        viewBinding = true
        buildConfig = true
    }
}
```

## 构建变体 (Build Variants)

Product Flavors 用于多环境、多版本构建。定义 `flavorDimensions` 和 `productFlavors`，Gradle 会生成 flavor 与 buildType 的笛卡尔积（如 devDebug、devRelease、prodDebug、prodRelease）。每个变体可有独立的 applicationIdSuffix、versionNameSuffix、依赖、源码目录等。适合区分开发/测试/生产环境，或免费版/付费版等。

```kotlin
android {
    flavorDimensions += "environment"
    
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
        }
        create("prod") {
            dimension = "environment"
        }
    }
}
```

## 依赖管理

`implementation` 将依赖对当前模块可见，对下游模块不可见，是默认选择。`api` 会传递依赖，慎用。`testImplementation` 和 `androidTestImplementation` 分别用于单元测试和仪器测试。使用版本目录时，`libs.bundles.retrofit` 可引用预定义的依赖组，简化配置并统一版本。

```kotlin
dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    
    // 使用版本目录
    implementation(libs.bundles.retrofit)
}
```

## 总结

掌握 Gradle 配置能够优化构建流程，提高开发效率。合理使用构建变体、版本目录和构建缓存，可以支持多环境发布、统一依赖管理并加速构建。关注 `buildCache`、`parallel`、`configureOnDemand` 等配置，可进一步优化大型项目的构建性能。
