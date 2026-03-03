---
layout: post
title: 微信小程序图片与媒体
categories: mini
tags: [小程序, 媒体]
date: 2024/11/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序中图片通过 `image` 组件展示，支持懒加载、多种裁剪模式（mode）、占位图、错误处理等。媒体方面有 `video`（视频播放）、`camera`（相机）、`live-player`（直播）等组件。图片和媒体是小程序中最常见的资源类型，合理使用可提升加载速度和用户体验。

本文介绍图片与常用媒体组件的用法。注意：图片和视频会占用较多流量和内存，建议控制尺寸、使用懒加载、压缩图片。

## image 组件

image 的 `mode` 决定图片如何裁剪和显示。常用 scaleToFill（拉伸填满）、aspectFit（保持比例完整显示）、aspectFill（保持比例填满裁剪）、widthFix（宽度固定，高度自适应）。`lazy-load` 开启懒加载，图片进入视口时再加载，可节省流量和提升首屏速度。

```html
<image
  src="{{ url }}"
  mode="aspectFill"
  lazy-load
  binderror="onError"
/>
```

`mode` 常用值：scaleToFill、aspectFit、aspectFill、widthFix。`lazy-load` 开启懒加载，进入视口再加载。

## 选择与预览图片

`wx.chooseImage` 可从相册选择或拍照，返回临时文件路径。`wx.previewImage` 可预览图片，支持左右滑动查看多张。预览时传入 urls 数组和当前 current，常用于商品详情、相册等场景。

```javascript
wx.chooseImage({
  count: 9,
  sizeType: ['compressed'],
  sourceType: ['album', 'camera'],
  success(res) {
    const tempPaths = res.tempFilePaths;
  }
});

wx.previewImage({
  current: url,
  urls: [url1, url2, url3]
});
```

## video 组件

```html
<video src="{{ videoUrl }}" controls show-center-play-btn/>
```

支持全屏、进度、弹幕等配置，详见文档。

## 总结

合理使用 image 的 mode 和 lazy-load，可优化加载与展示效果。媒体组件需注意真机兼容与性能，大文件建议使用云存储。图片建议使用 CDN 或云存储，并设置合适的缓存策略。
