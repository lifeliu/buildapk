---
layout: post
title: 微信小程序开放能力
categories: mini
tags: [小程序, 开放能力]
date: 2024/12/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序提供丰富的开放能力，如打开 App、打开其他小程序、获取发票、生物认证、客服会话等。部分能力通过 `button` 的 `open-type` 触发（如 getPhoneNumber、share、openSetting），部分通过 API 调用（如 wx.navigateToMiniProgram）。开放能力需在公众平台或代码中配置，部分需用户授权。

本文介绍常用开放能力的使用方式。开放能力随微信版本迭代，使用前请查阅最新文档，注意兼容性和审核要求。

## 打开其他小程序

`wx.navigateToMiniProgram` 可打开其他小程序，需在公众平台配置业务域名关联。`button open-type="launchApp"` 可打开关联的 App，需在 App 中配置。适用于小程序与 App 互通、多小程序跳转等场景。

```html
<button open-type="launchApp" app-id="xxx" extra-data="{{ data }}">
  打开 App
</button>
```

需在公众平台配置关联的 App。`wx.navigateToMiniProgram` 可打开其他小程序。

## 获取用户信息与手机号

`open-type="getPhoneNumber"` 获取手机号，`open-type="getUserInfo"` 已调整，用户信息通过头像昵称填写能力获取。

## 生物认证

```javascript
wx.startSoterAuthentication({
  requestAuthModes: ['fingerPrint'],
  challenge: 'xxx',
  success() {
    // 认证成功
  }
});
```

## 总结

开放能力随微信版本迭代，使用前请查阅最新文档。注意各能力的权限与使用场景限制，避免滥用导致审核不通过。合理使用开放能力可丰富小程序功能，提升用户体验。
