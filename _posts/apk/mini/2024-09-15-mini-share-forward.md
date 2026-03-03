---
layout: post
title: 微信小程序分享与转发
categories: mini
tags: [小程序, 分享]
date: 2024/9/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序支持分享给好友（会话）或分享到朋友圈。分享给好友通过页面中定义 `onShareAppMessage` 配置，用户点击右上角「...」或按钮 `open-type="share"` 时触发；分享到朋友圈通过 `onShareTimeline` 配置，用户点击「分享到朋友圈」时触发。可自定义标题、路径、图片等，实现带参数的分享链接，便于统计来源和个性化展示。

本文介绍两种分享方式的配置与使用。注意：分享到朋友圈需在 app.json 的 window 中开启 `enableShareTimeline`，且从朋友圈打开的小程序为「单页模式」，需适配无 tabBar、无返回键等差异。

## 分享给好友

在页面中定义 `onShareAppMessage`，返回配置对象。若不定义，则使用小程序默认分享（首页、默认标题）。通过 `button open-type="share"` 可指定在点击某按钮时触发分享，此时会执行页面的 onShareAppMessage。

```javascript
Page({
  onShareAppMessage() {
    return {
      title: '推荐给你一个好物',
      path: '/pages/detail/detail?id=' + this.data.id,
      imageUrl: this.data.shareImage
    };
  }
});
```

`path` 可带查询参数，好友打开时在 `onLoad(options)` 中获取。`imageUrl` 为自定义图片路径，建议比例 5:4，不设置则使用小程序截图。

## 分享到朋友圈

`onShareTimeline` 返回的对象中，`query` 为页面参数（格式同 URL query），`title` 为分享标题，`imageUrl` 为自定义图片。朋友圈打开时，小程序以单页模式运行，仅展示当前页，无 tabBar，需在页面内提供返回或跳转入口。

```javascript
Page({
  onShareTimeline() {
    return {
      title: '分享标题',
      query: 'id=' + this.data.id,
      imageUrl: this.data.shareImage
    };
  }
});
```

需在 app.json 的 window 中配置 `"enableShareTimeline": true`。朋友圈打开为单页模式，需适配。

## 总结

合理配置分享标题、路径和图片，可提升传播效果。注意 path/query 参数长度限制（总长度有限），避免过长导致异常。建议根据分享内容动态生成 title 和 path，便于追踪和个性化。
