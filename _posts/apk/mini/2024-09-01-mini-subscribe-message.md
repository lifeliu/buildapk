---
layout: post
title: 微信小程序订阅消息
categories: mini
tags: [小程序, 订阅消息]
date: 2024/9/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

订阅消息用于向用户发送服务通知，如订单状态、物流更新、活动提醒等。与模板消息不同，订阅消息需用户主动点击授权，每次授权可发送一次或多次（取决于模板类型）。需在公众平台申请模板，获取 template_id，用户订阅后后端可调用接口发送。

本文介绍订阅消息的申请、唤起与发送流程。注意：唤起订阅弹窗必须在用户操作（如点击按钮）的同步回调中调用，异步回调（如 setTimeout、接口 success）中调用可能无法弹出，这是微信为防止滥用做的限制。

## 模板配置

在微信公众平台「功能 - 订阅消息」中，从模板库选择合适的模板或创建自定义模板，审核通过后获得 template_id。模板有固定格式和占位符，如「订单号：{{thing1}}，状态：{{thing2}}」。发送时需按占位符传入对应内容，且需符合字数、类型等限制。

## 唤起订阅弹窗

`wx.requestSubscribeMessage` 可一次请求多个模板，用户可选择性同意或拒绝。返回的 res 中，key 为 tmplId，value 为 'accept' 或 'reject'。用户同意后，后端可在有效期内（一次性模板为 7 天，长期性模板为长期）向该用户发送该模板消息。

```javascript
wx.requestSubscribeMessage({
  tmplIds: ['template_id_xxx'],
  success(res) {
    if (res['template_id_xxx'] === 'accept') {
      // 用户同意订阅
    }
  }
});
```

## 后端发送

后端调用微信「订阅消息」接口（subscribeMessage.send），传入 openid、template_id、data（按模板占位符填充）、page（可选，点击消息跳转的小程序页面）等。需使用 access_token 鉴权。具体格式和 data 结构参考微信官方文档。

## 总结

订阅消息需用户主动授权，模板需提前申请。合理选择唤起时机（如下单成功时、关键操作前）、优化引导文案，可提高订阅率和消息触达效果。避免在页面加载时自动弹出，易被用户拒绝。
