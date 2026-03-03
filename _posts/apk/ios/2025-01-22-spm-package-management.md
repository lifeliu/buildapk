---
layout: post
title: Swift Package Manager 包管理完全指南
categories: ios
tags: [SPM, Swift Package, 依赖管理, Xcode]
date: 2025/1/22 09:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_063.jpg)

## 引言

Swift Package Manager（SPM）是 Apple 官方的 Swift 依赖管理和打包工具，已集成到 Xcode 中，无需单独安装。它支持从 Git 仓库添加远程依赖、本地路径添加本地包、版本与分支约束，以及创建和发布 Swift 包。相比 CocoaPods 和 Carthage，SPM 与 Xcode 原生集成、支持跨平台、配置简单，是 Apple 生态的推荐方案。本文将介绍 SPM 的依赖添加、版本管理、创建自定义包以及常见问题与最佳实践。

## SPM 与 CocoaPods 的对比

CocoaPods 依赖 Ruby 和 Podfile，支持 iOS 生态多年，库数量丰富。SPM 无需额外运行时，直接集成在 Xcode，支持 Swift 与 C 系语言混编，跨平台支持更好。许多库已同时支持 CocoaPods 和 SPM，新项目可优先考虑 SPM；老项目若已有大量 Pod 依赖，可逐步迁移或混合使用。

## 添加依赖

在 Xcode 中：File → Add Package Dependencies，输入仓库 URL（如 `https://github.com/Alamofire/Alamofire`），选择版本规则（Up to Next Major、Exact、Branch 等），Xcode 会解析并下载。或直接编辑项目的 Package.swift（若项目是 Swift Package）或通过 Project 的 Package Dependencies 面板管理。对于普通 App 项目，依赖信息存储在 project.pbxproj 和 xcshareddata 中，无需手写 Package.swift。

```swift
// Package.swift（Swift Package 项目）
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0")
],
targets: [
    .target(
        name: "MyApp",
        dependencies: ["Alamofire"]
    )
]
```

版本规则：`from: "5.8.0"` 表示 >= 5.8.0 且 < 6.0.0；`.exact("5.8.0")` 表示精确版本；`branch: "main"` 表示使用分支。

## 创建 Swift 包

Swift 包是包含 `Package.swift` 的目录，可定义多个 target 和 product。`products` 定义对外暴露的库或可执行文件；`targets` 定义模块及其依赖。包可发布到 Git 仓库，他人通过 URL 引用。本地开发时，可将包放在项目目录或通过 `path` 引用。

```swift
// Package.swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyLibrary",
    platforms: [.iOS(.v15)],
    products: [
        .library(name: "MyLibrary", targets: ["MyLibrary"])
    ],
    targets: [
        .target(name: "MyLibrary", dependencies: [])
    ]
)
```

`platforms` 限制包支持的最低系统版本；不指定则默认支持所有平台。

## 常见问题

1. **解析失败**：检查 URL、网络、版本是否存在；某些私有仓库需配置认证。
2. **二进制依赖**：SPM 主要支持源码依赖，二进制 XCFramework 需通过 `binaryTarget` 引用。
3. **与 CocoaPods 混用**：同一项目可同时使用，但需注意依赖冲突和构建顺序。

## 总结

SPM 是 Apple 生态的官方包管理方案，无需额外工具即可管理依赖。掌握依赖添加、版本规则和包创建，可以高效管理项目依赖，并可将内部模块拆分为 Swift 包，提升代码复用和模块化程度。
