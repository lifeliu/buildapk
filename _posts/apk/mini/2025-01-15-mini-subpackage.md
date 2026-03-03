---
layout: post
title: 微信小程序分包加载
categories: mini
tags: [小程序, 分包]
date: 2025/1/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序主包大小限制 2MB，总包（主包+所有分包）20MB。当业务增多时，主包容易超限，导致无法上传。分包机制可将非首屏页面、组件、静态资源拆分到子包，用户进入对应页面时再下载，从而减小主包体积，加快首屏加载。

本文介绍分包配置与使用方式。分包有独立 root 目录，主包不能引用分包内资源，分包可以引用主包。注意：tabBar 页面必须在主包。

## 分包配置

在 app.json 中通过 `subpackages` 或 `subPackages` 配置。每个分包有 root（根目录）、name（可选，用于独立分包）、pages（页面路径数组）。主包 pages 中的页面不能与分包重复。

在 app.json 中配置：

```json
{
  "pages": ["pages/index/index"],
  "subpackages": [
    {
      "root": "packageA",
      "name": "packA",
      "pages": ["pages/cat/cat", "pages/dog/dog"]
    },
    {
      "root": "packageB",
      "pages": ["pages/list/list"]
    }
  ]
}
```

主包 `pages` 为首页及核心页面，`subpackages` 为分包。分包有独立 root，其下 pages 为相对路径。

## 分包大小限制

- 主包 ≤ 2MB
- 单个分包 ≤ 2MB
- 总包 ≤ 20MB（含主包与所有分包）

## 跳转分包页面

```javascript
wx.navigateTo({
  url: '/packageA/pages/cat/cat'
});
```

路径为 `root + 页面路径`。分包页面首次进入时下载，之后缓存。

## 独立分包

`"independent": true` 的分包可独立于主包运行，适合从分享等场景直接进入的页面。独立分包不能依赖主包资源。

## 总结

合理使用分包可显著降低首屏体积，提升加载速度。将低频页面、插件放入分包，主包保留核心路径（首页、tabBar 等）。独立分包适合分享页等可独立运行的场景，但不能依赖主包资源。
