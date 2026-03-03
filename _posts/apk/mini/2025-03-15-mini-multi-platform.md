---
layout: post
title: 微信小程序多端适配
categories: mini
tags: [小程序, 多端]
date: 2025/3/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

除微信外，小程序还可运行于支付宝、百度、字节跳动、QQ 等平台，以及微信/支付宝的 PC 端。各平台 API 前缀、组件、配置有差异，直接写多套代码维护成本高。多端适配可通过条件编译、跨端框架（如 Taro、uni-app）或运行时判断实现，一套代码多端输出。

本文介绍多端差异与适配思路。若只 targeting 微信，可忽略；若需多端发布，建议使用成熟框架，减少适配工作量。

## 平台差异

各平台 API 前缀不同：微信 `wx`、支付宝 `my`、百度 `swan`、字节 `tt` 等。组件和属性也有差异，如 button 的 open-type 支持的能力不同。审核规则、类目、支付等也有差异。需针对各平台分别测试和适配。

- API 前缀不同：微信 `wx`，支付宝 `my`，百度 `swan` 等
- 组件与属性有差异
- 审核与配置规则不同

## 条件编译

使用 `#ifdef`、`#ifndef`、`#endif` 按平台编译不同代码：

```javascript
// #ifdef MP-WEIXIN
wx.showToast({ title: '微信' });
// #endif

// #ifdef MP-ALIPAY
my.showToast({ content: '支付宝' });
// #endif
```

## 框架方案

Taro、uni-app 等支持一套代码多端输出，通过抽象层统一 API，编译时生成各端代码。适合多端同时维护的项目。

## 总结

多端适配需权衡开发成本与维护成本。简单差异可用条件编译，复杂多端建议采用成熟框架，统一开发体验。框架会抽象各平台差异，编译时生成对应平台代码，但部分平台特性可能需单独处理。
