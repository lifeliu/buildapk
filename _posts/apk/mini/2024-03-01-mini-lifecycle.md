---
layout: post
title: 微信小程序生命周期完全指南
categories: mini
tags: [小程序, 生命周期]
date: 2024/3/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序有应用生命周期和页面生命周期两套机制。应用生命周期在 app.js 中通过 `App()` 定义，从小程序启动到销毁全程有效；页面生命周期在 Page 中定义，随页面的加载、显示、隐藏、卸载而触发。理解各生命周期函数的触发时机和顺序，对数据初始化、资源释放、埋点统计、权限申请等至关重要。

例如：用户从首页进入详情页再返回，详情页会经历 onHide 而非 onUnload；若用户通过右上角关闭小程序，则先触发页面的 onUnload，再触发应用的 onHide。掌握这些细节，可以避免在错误的生命周期中执行逻辑。

## 应用生命周期

应用生命周期在小程序启动时开始，贯穿整个运行过程。在 `app.js` 中通过 `App()` 注册：

```javascript
App({
  onLaunch(options) {
    // 小程序初始化完成，全局只触发一次
    console.log('onLaunch', options);
  },
  onShow(options) {
    // 小程序启动或从后台进入前台
  },
  onHide() {
    // 小程序从前台进入后台
  },
  onError(msg) {
    // 脚本错误
  }
});
```

**各函数说明：**

- `onLaunch`：小程序初始化完成时触发，全局只执行一次。options 中包含场景值（scene）、启动参数（query）、来源信息等，常用于初始化登录态、获取系统信息、预加载配置等。
- `onShow`：小程序启动或从后台切回前台时触发。可用于刷新数据、恢复播放等。
- `onHide`：小程序从前台进入后台时触发。可在此暂停计时、保存草稿等。
- `onError`：脚本执行出错时触发，可用于错误上报。

## 页面生命周期

页面生命周期与页面栈和用户操作紧密相关。当用户通过 navigateTo 进入新页面时，新页面会依次触发 onLoad、onShow、onReady；当用户返回时，当前页触发 onHide，上一页触发 onShow。

```javascript
Page({
  data: { count: 0 },
  onLoad(options) {
    // 页面加载，可获取路由参数
    const id = options.id;
  },
  onShow() {
    // 页面显示/切入前台
  },
  onReady() {
    // 页面初次渲染完成
  },
  onHide() {
    // 页面隐藏/切入后台
  },
  onUnload() {
    // 页面卸载，如 navigateBack 离开
  },
  onPullDownRefresh() {
    // 用户下拉刷新
  },
  onReachBottom() {
    // 页面上拉触底
  }
});
```

**关键区别：**

- `onLoad` 只在页面首次加载时执行一次，适合获取路由参数、请求初始数据。
- `onShow` 每次页面显示都会执行，包括从其他页面返回时，适合刷新列表、更新未读数量等。
- `onReady` 在页面初次渲染完成后触发，可在此操作 Canvas、获取节点信息等。
- `onUnload` 在页面被销毁时触发（如 navigateBack 离开、redirectTo 替换），应在此清理定时器、取消网络请求、解除监听等，避免内存泄漏。

## 总结

应用生命周期管理全局状态，页面生命周期管理页面级逻辑。合理利用 `onLoad` 初始化、`onShow` 刷新、`onUnload` 清理，可写出健壮的小程序。建议在 onUnload 中统一做资源释放，避免遗漏。
