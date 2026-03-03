---
layout: post
title: 微信小程序云存储使用
categories: mini
tags: [小程序, 云存储]
date: 2024/8/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

云存储提供文件上传、下载、删除等能力，类似对象存储（OSS）。适合存储用户头像、商品图片、文档等。文件自动接入 CDN，访问速度快；可生成临时链接，用于外部分享或 H5 展示。无需自建文件服务器，按存储量和流量计费。

本文介绍云存储的上传、下载与临时链接生成。上传前通常需先通过 `wx.chooseImage`、`wx.chooseMessageFile` 等获取本地文件路径。

## 上传文件

`cloudPath` 为文件在云端的路径，需在应用内唯一。建议按业务规则命名，如 `avatars/{openid}.png`、`products/{id}/{index}.jpg`，便于管理和清理。上传成功后返回 fileID，格式为 `cloud://env.xxx/path`，可用于 image 组件的 src 或后续下载。

```javascript
wx.chooseImage({
  count: 1,
  success(res) {
    const tempPath = res.tempFilePaths[0];
    wx.cloud.uploadFile({
      cloudPath: `avatars/${Date.now()}.png`,
      filePath: tempPath
    }).then(uploadRes => {
      console.log('fileID:', uploadRes.fileID);
    });
  }
});
```

## 下载与临时链接

`wx.cloud.downloadFile` 将云端文件下载到本地临时路径，适用于需要本地处理的场景（如预览、编辑）。若仅需展示或分享，可直接使用 fileID 作为 image 的 src，或调用 `getTempFileURL` 获取 HTTPS 临时链接。临时链接有效期为 2 小时，过期需重新获取。

```javascript
wx.cloud.downloadFile({
  fileID: 'cloud://xxx.png'
}).then(res => console.log(res.tempFilePath));

// 获取临时链接，可外部分享
wx.cloud.getTempFileURL({
  fileList: ['cloud://xxx.png']
}).then(res => console.log(res.fileList[0].tempFileURL));
```

## 删除文件

```javascript
wx.cloud.deleteFile({
  fileList: ['cloud://xxx.png']
});
```

## 总结

云存储简化文件管理，无需自建 OSS。注意控制存储用量与 CDN 流量，合理设计 cloudPath 便于管理和清理。大文件上传可考虑使用 `wx.cloud.uploadFile` 的 `cloudPath` 分片或压缩后上传，以提升成功率。
