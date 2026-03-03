---
layout: post
title: iOS 相机与相册开发完全指南
categories: ios
tags: [iOS, 相机]
date: 2025-01-29
---

![title](https://image.sideproject.cn/titlex/titlex_064.jpg)

## 引言

iOS 提供了多种方式访问相机和相册：`UIImagePickerController` 是传统方案，支持拍照和选择照片/视频，配置简单；`PHPicker`（PhotosUI）是 iOS 14 引入的现代相册选择器，无需完整相册权限即可选择有限照片，隐私友好；`AVFoundation` 提供完整的相机控制，适合需要自定义 UI、滤镜、多摄像头的场景。本文将介绍各类 API 的用法、权限配置以及选择建议。

## 权限与隐私

访问相机需 `NSCameraUsageDescription`，访问相册需 `NSPhotoLibraryUsageDescription`（完整访问）或使用 PHPicker 实现无需权限的有限选择。iOS 14+ 的"有限照片库"模式允许用户只授权部分照片，`UIImagePickerController` 会尊重此选择；若需访问全部照片，需申请 `NSPhotoLibraryAddUsageDescription`（仅写入）或完整库权限。权限描述应清晰说明用途，以提高用户授权率。

## PHPicker（推荐）

PHPicker 是 iOS 14+ 的相册选择组件，用户选择照片时无需授予完整相册权限，系统会提供选择界面并仅将用户选中的资源暴露给应用。在 SwiftUI 中可使用 `PhotosPicker`，绑定 `PhotosPickerItem`，通过 `loadTransferable` 异步加载数据。支持多选、类型过滤（`.images`、`.videos` 等），是相册选择的现代推荐方案。

```swift
import PhotosUI

var selectedItem: PhotosPickerItem?

PhotosPicker(selection: $selectedItem, 
             matching: .images,
             photoLibrary: .shared()) {
    Text("选择照片")
}
.onChange(of: selectedItem) { _, newValue in
    Task {
        if let data = try? await newValue?.loadTransferable(type: Data.self) {
            // 处理图片数据
        }
    }
}
```

`loadTransferable` 支持 `Data`、`UIImage`（通过 `Image` 的 Transferable）、`URL` 等类型，根据 `matching` 过滤可选择的资源类型。

## UIImagePickerController

`UIImagePickerController` 支持 `sourceType` 为 `.camera`（拍照）或 `.photoLibrary`（相册）。设置 `delegate` 实现 `imagePickerController(_:didFinishPickingMediaWithInfo:)` 获取选中的图片或视频，以及 `imagePickerControllerDidCancel` 处理取消。可配置 `mediaTypes`、`allowsEditing` 等。适合需要兼容老系统或简单拍照/选图场景。

```swift
let picker = UIImagePickerController()
picker.sourceType = .camera  // 或 .photoLibrary
picker.delegate = self
present(picker, animated: true)
```

## 权限配置

```xml
<key>NSCameraUsageDescription</key>
<string>需要相机以拍摄照片</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>需要访问相册以选择照片</string>
```

若仅使用 PHPicker 且不需要完整相册权限，部分场景可能无需 `NSPhotoLibraryUsageDescription`，具体以系统行为和审核要求为准。

## 总结

根据需求选择合适的 API：相册选择优先考虑 PHPicker，简单拍照可用 UIImagePickerController，自定义相机需 AVFoundation。注意权限配置和用户隐私，在合适的时机请求权限，并提供清晰的说明。
