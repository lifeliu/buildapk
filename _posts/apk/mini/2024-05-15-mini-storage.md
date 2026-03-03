---
layout: post
title: 微信小程序本地存储与缓存
categories: mini
tags: [小程序, 存储]
date: 2024/5/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序提供本地存储能力，用于在设备上持久化数据。与 Web 的 localStorage 类似，但 API 分为异步和同步两套。异步版本（`wx.setStorage` / `wx.getStorage`）不会阻塞主线程，适合大多数场景；同步版本（`wx.setStorageSync` / `wx.getStorageSync`）在部分初始化场景下更方便。

存储有容量限制：单条 key 对应的数据不超过 1MB，总容量不超过 10MB。适合存储 token、用户偏好、草稿、缓存数据等。本文介绍存储 API 及使用建议。

## 异步存储

异步 API 通过 success、fail、complete 回调处理结果，不会阻塞后续代码执行。存储的数据会持久化，即使用户关闭小程序后再次打开，数据仍然存在。除非用户删除小程序或调用 clearStorage，否则数据会一直保留。

```javascript
wx.setStorage({
  key: 'userInfo',
  data: { name: '张三', id: 1 }
});

wx.getStorage({
  key: 'userInfo',
  success(res) {
    console.log(res.data);
  }
});

wx.removeStorage({ key: 'userInfo' });
wx.clearStorage(); // 清空全部
```

## 同步存储

同步 API 会阻塞执行，直到读写完成。在部分场景下必须使用同步版本，例如 app.js 的 onLaunch 中需要立即读取 token 并决定是否跳转登录页，若使用异步 getStorage，可能在回调执行前页面已经渲染，导致逻辑混乱。注意：同步 API 在数据量大时可能造成卡顿，一般仅用于读取少量关键数据。

```javascript
try {
  const token = wx.getStorageSync('token');
  if (token) {
    // 带 token 请求
  }
} catch (e) {
  console.error(e);
}
```

## 使用建议

- **安全**：敏感数据（如密码、完整手机号）勿明文存储，可加密或仅存必要字段（如脱敏手机号）。token 建议设置过期时间，定期刷新。
- **容量**：单条 1MB、总量 10MB 的限制需注意。大数据可考虑分 key 存储或使用云存储。缓存列表数据时，可只缓存前几页，避免占满空间。
- **清理**：定期清理无用数据，如过期的缓存、临时草稿等。可在 onLaunch 中做简单的过期清理逻辑。

## 总结

本地存储是小程序数据持久化的主要方式。合理使用异步/同步 API，注意容量与安全，可满足大部分离线与缓存需求。建议将存储 key 集中定义成常量，避免拼写错误和重复。
