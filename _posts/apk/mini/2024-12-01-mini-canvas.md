---
layout: post
title: 微信小程序 Canvas 画布
categories: mini
tags: [小程序, Canvas]
date: 2024/12/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序提供 `canvas` 组件用于绘制图形、生成海报、签名板、图表等。支持 2D 和 WebGL 两种上下文。旧版 Canvas 使用 `wx.createCanvasContext`，API 与 H5 有差异；新版 Canvas 使用 `type="2d"`（基础库 2.9.0+），兼容 H5 Canvas 2D API，推荐使用。

本文介绍 Canvas 2D 的基础用法。Canvas 绘制是异步的，需在 `draw` 回调或下一帧才能看到效果。生成图片时需注意 canvas 尺寸与设备像素比，以保证清晰度。

## 基础绘制

type="2d" 的 canvas 需通过 `SelectorQuery` 获取 canvas 节点，再调用 `getContext('2d')` 获取上下文。绘制 API 与 H5 一致，如 fillRect、fillStyle、stroke 等。注意：canvas 的宽高需在 style 和属性中同时设置，且要考虑设备像素比（pixelRatio）以获得清晰绘制。

```html
<canvas type="2d" id="myCanvas" style="width: 300px; height: 200px"/>
```

```javascript
const query = wx.createSelectorQuery();
query.select('#myCanvas')
  .fields({ node: true, size: true })
  .exec((res) => {
    const canvas = res[0].node;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = '#ff0000';
    ctx.fillRect(10, 10, 100, 50);
  });
```

type="2d" 需基础库 2.9.0+，支持更完整的 Canvas 2D API。

## 生成图片

绘制完成后，调用 `wx.canvasToTempFilePath` 导出为图片，可用于保存或分享。

```javascript
wx.canvasToTempFilePath({
  canvas: canvas,
  success(res) {
    const tempPath = res.tempFilePath;
  }
});
```

## 总结

Canvas 适合海报、图表、签名等场景。注意 type="2d" 与旧版 API 的差异，以及导出图片的尺寸与清晰度设置。复杂绘制可考虑使用第三方图表库（如 echarts 小程序版）或封装成组件复用。
