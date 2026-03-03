---
layout: post
title: Flutter 图片加载与缓存完全指南
categories: flutter
tags: [Flutter, 图片]
date: 2024/11/25 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_012.jpg)

## 引言

Flutter 的 Image.network 支持网络图片，但无内置缓存，每次都会重新下载。cached_network_image 提供内存和磁盘缓存，基于 flutter_cache_manager，可配置缓存策略。placeholder 和 errorWidget 可提升加载体验。对于大量图片列表，需注意缓存大小和内存占用，可通过 memCacheWidth、memCacheHeight 限制解码尺寸，减少内存压力。本文将介绍图片加载、缓存配置、占位与错误处理以及性能优化。

## 为什么需要图片缓存

网络图片每次从网络加载会消耗流量、增加延迟，影响用户体验。内存缓存可避免重复解码，磁盘缓存可避免重复下载。cached_network_image 默认使用 flutter_cache_manager 的默认配置，可自定义 CacheManager 以调整缓存目录、过期策略等。

## cached_network_image

imageUrl 为必填。placeholder 在加载时显示，errorWidget 在加载失败时显示。fadeInDuration 控制淡入动画。memCacheWidth 和 memCacheHeight 可指定解码时的最大尺寸，Flutter 会按比例解码，减少内存占用，适合列表中的缩略图。fit 控制图片在容器中的适应方式。对于需要圆角、裁剪等效果，可配合 ClipRRect、BoxDecoration 等使用。

```dart
CachedNetworkImage(
  imageUrl: url,
  placeholder: (context, url) => Center(child: CircularProgressIndicator()),
  errorWidget: (context, url, error) => Icon(Icons.error, size: 48),
  memCacheWidth: 200,   // 限制解码宽度，节省内存
  memCacheHeight: 200,
  fit: BoxFit.cover,
)
```

## 总结

合理使用缓存可减少网络请求，提升图片加载速度。注意 memCacheWidth/Height 在长列表中的使用，避免大图占用过多内存。对于本地图片，使用 Image.asset 或 Image.file 即可，无需缓存库。
