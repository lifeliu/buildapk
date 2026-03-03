---
layout: post
title: 微信小程序支付接入
categories: mini
tags: [小程序, 支付]
date: 2024/8/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序支付需在微信支付商户平台完成商户号注册、与小程序 AppID 关联等配置。支付流程为：用户点击支付后，小程序请求后端创建订单；后端调用微信「JSAPI 支付」统一下单接口，获取 prepay_id；后端使用商户私钥对参数签名，生成 timeStamp、nonceStr、package、signType、paySign 返回给前端；前端调用 `wx.requestPayment` 调起微信支付界面；支付结果通过微信异步通知（推荐）或主动查询订单状态获取。

本文介绍支付流程与小程序端关键代码。V3 版本接口使用 RSA 签名，需后端正确实现签名逻辑。

## 支付流程

1. 用户点击支付，小程序请求后端创建订单并传入商品信息、金额等
2. 后端调用微信统一下单 API，获取 prepay_id
3. 后端使用商户 API 密钥对支付参数签名，生成五参数返回给小程序
4. 小程序调用 `wx.requestPayment` 调起支付，用户完成支付或取消
5. 支付成功后，微信会向后端配置的 notify_url 发送异步通知，后端验证签名后更新订单状态；小程序可在 success 回调中跳转订单详情或轮询查询

## 小程序端调用

```javascript
wx.requestPayment({
  timeStamp: res.timeStamp,
  nonceStr: res.nonceStr,
  package: res.package,
  signType: 'RSA',
  paySign: res.paySign,
  success() {
    wx.showToast({ title: '支付成功' });
  },
  fail(err) {
    if (err.errMsg.includes('cancel')) {
      // 用户取消
    }
  }
});
```

**参数说明：** 所有参数由后端生成，前端不可篡改或伪造。V3 接口使用 RSA 签名，signType 为 'RSA'。package 格式为 `prepay_id=wx...`，由微信统一下单接口返回。

## 注意事项

- **真机测试**：支付只能在真机上调起，模拟器无法唤起微信支付界面。开发时可在真机预览或体验版中测试。
- **商户关联**：商户号需在微信支付商户平台中与小程序 AppID 关联，否则无法完成支付。
- **密钥安全**：API 密钥、证书等仅在后端使用，切勿泄露到前端或代码仓库。

## 总结

小程序支付依赖后端配合，前端主要负责调起支付和结果展示。严格按微信文档实现签名与参数传递，可确保支付流程正确。建议在后端做好幂等和异常处理，避免重复扣款或状态不一致。
