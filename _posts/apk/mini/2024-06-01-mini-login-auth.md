---
layout: post
title: 微信小程序用户登录与授权
categories: mini
tags: [小程序, 登录]
date: 2024/6/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序登录与 Web 不同，没有传统的「账号密码」表单，而是依赖微信身份体系。标准流程为：前端调用 `wx.login` 获取临时 code（5 分钟有效），将 code 发送到后端，后端调用微信接口用 code 换取 openid 和 session_key，再根据业务生成自定义登录态（如 JWT）返回给前端。前端将 token 存入 storage，后续请求携带即可。

用户信息、手机号等敏感数据需通过 button 的 `open-type` 触发授权，不能主动弹窗获取。微信对用户隐私保护要求严格，需遵循规范，避免审核不通过。本文介绍登录流程与常见授权场景。

## wx.login 获取 code

`wx.login` 无需用户操作，静默获取 code。code 只能使用一次，且有效期为 5 分钟。后端需用 code 调用 `auth.code2Session` 接口，换取 openid（用户在当前小程序下的唯一标识）和 session_key（用于解密手机号等敏感数据）。后端根据 openid 查询或创建用户，生成业务 token 返回。

```javascript
wx.login({
  success(res) {
    if (res.code) {
      // 将 res.code 发送到后端
      wx.request({
        url: 'https://api.example.com/login',
        method: 'POST',
        data: { code: res.code },
        success(res) {
          const token = res.data.token;
          wx.setStorageSync('token', token);
        }
      });
    }
  }
});
```

后端用 code 调用微信接口换取 openid、session_key，并生成业务 token。

## 获取用户信息

微信持续收紧用户信息获取能力。基础库 2.27.1 起，`wx.getUserProfile` 已废弃。目前获取头像、昵称需使用「头像昵称填写」能力：用户点击 `button open-type="chooseAvatar"` 选择头像，在 `input type="nickname"` 中填写昵称，通过事件回调获取。敏感信息如手机号需 `open-type="getPhoneNumber"` 按钮触发，用户同意后返回加密数据，需将 code 或加密数据发往后端，用 session_key 解密得到真实手机号。

```html
<button open-type="getPhoneNumber" bindgetphonenumber="onGetPhone">
  获取手机号
</button>
```

```javascript
onGetPhone(e) {
  if (e.detail.code) {
    // 将 code 发后端解密
  } else {
    // 用户拒绝
  }
}
```

## 总结

登录依赖 wx.login + 后端校验，用户信息与手机号需通过按钮授权。遵循微信规范，避免强制授权、诱导授权，在真正需要时再申请，可提升用户体验和审核通过率。建议将登录逻辑封装成统一模块，在需要鉴权的接口 401 时自动跳转登录页。
