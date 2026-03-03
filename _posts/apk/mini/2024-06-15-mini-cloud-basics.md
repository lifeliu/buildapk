---
layout: post
title: 微信小程序云开发基础
categories: mini
tags: [小程序, 云开发]
date: 2024/6/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

微信云开发是微信提供的一站式后端服务，包含云函数、云数据库、云存储等能力。开发者无需自建服务器、购买域名、配置 SSL，即可完成后端逻辑、数据存储、文件上传等。特别适合快速验证想法、个人项目、中小型应用。

开通云开发后，在 app.js 中调用 `wx.cloud.init()` 初始化，即可在小程序端直接操作云数据库、上传文件，或调用云函数执行服务端逻辑。本文介绍云开发开通与基础用法。

## 开通与初始化

1. 在微信开发者工具中点击顶部「云开发」按钮，按指引开通（需主体认证）
2. 创建环境，建议开发和生产分开，如 `dev-xxx`、`prod-xxx`
3. 在 app.js 的 onLaunch 中初始化，传入 env 指定环境 ID：

```javascript
App({
  onLaunch() {
    wx.cloud.init({ env: 'your-env-id' });
  }
});
```

## 云数据库

云数据库基于 MongoDB，为 NoSQL 文档型数据库。支持增删改查、聚合、索引等。在小程序端可直接操作，但受权限控制；敏感操作（如管理员逻辑、复杂统计）建议放在云函数中，云函数可以管理员身份绕过权限，且不暴露逻辑。

```javascript
const db = wx.cloud.database();
db.collection('users').add({
  data: { name: '张三', age: 20 }
}).then(res => console.log(res));

db.collection('users').where({ age: 20 }).get()
  .then(res => console.log(res.data));
```

## 云存储

云存储用于存储用户上传的图片、文件等。`cloudPath` 为云端路径，需唯一，建议按业务规则命名（如 `avatars/{openid}.png`）。上传成功后返回 fileID，可用于展示或分享。下载时使用 `wx.cloud.downloadFile`，获取临时链接使用 `wx.cloud.getTempFileURL`。

```javascript
wx.cloud.uploadFile({
  cloudPath: 'images/avatar.png',
  filePath: tempFilePath
}).then(res => console.log(res.fileID));
```

## 总结

云开发降低后端门槛，适合快速验证和中小型项目。掌握云函数、云数据库、云存储的基本用法，可快速搭建全栈小程序。建议开发与生产环境分离，避免误操作影响线上数据。
