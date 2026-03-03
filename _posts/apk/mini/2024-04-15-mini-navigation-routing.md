---
layout: post
title: 微信小程序路由与页面导航
categories: mini
tags: [小程序, 路由]
date: 2024/4/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序的路由由框架统一管理，采用页面栈机制。用户通过 `wx.navigateTo`、`wx.redirectTo`、`wx.reLaunch`、`wx.navigateBack` 等 API 实现页面跳转，每种方式对页面栈的影响不同。理解各 API 的差异和页面栈行为，可正确设计导航流程，避免「返回键行为异常」「页面栈溢出」等问题。

例如：登录页通常使用 `redirectTo` 或 `reLaunch`，避免用户登录后按返回键回到登录页；支付完成页也常用 `redirectTo`，防止重复支付。

## 跳转 API 对比

| API | 行为 | 页面栈 |
|-----|------|--------|
| wx.navigateTo | 保留当前页，打开新页 | 压栈 |
| wx.redirectTo | 关闭当前页，打开新页 | 替换 |
| wx.reLaunch | 关闭所有页，打开新页 | 清空后压栈 |
| wx.navigateBack | 返回上一页 | 出栈 |

跳转时可通过 URL 的 query 参数传递数据。目标页在 `onLoad(options)` 中获取，options 为解析后的对象。注意：参数值会转为字符串，复杂对象需 JSON 序列化后传递。

```javascript
// 跳转并传参
wx.navigateTo({
  url: '/pages/detail/detail?id=1&type=product'
});

// 目标页 onLoad(options) 中获取
// options.id === '1', options.type === 'product'
```

## 页面栈限制

小程序页面栈最多 10 层，超过时 `navigateTo` 会失败并报错。若业务存在深层级跳转（如分类->列表->详情->评论->回复），可考虑在中间使用 `redirectTo` 替换当前页，或使用 `reLaunch` 清空栈后跳转。同时，注意 URL 长度限制（不能超过 2048 字符）。

## 总结

根据业务选择合适跳转方式：需返回用 `navigateTo`，不需返回用 `redirectTo`，重置整个应用用 `reLaunch`。注意页面栈限制，避免深层嵌套。合理设计页面层级，可提升用户体验和开发效率。
