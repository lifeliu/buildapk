---
layout: post
title: 微信小程序云函数开发
categories: mini
tags: [小程序, 云函数]
date: 2024/7/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

云函数运行在微信云端的 Node.js 环境中，可调用微信服务端 API（如获取 openid、发送订阅消息）、访问云数据库、发起 HTTP 请求等。与小程序端不同，云函数运行在服务端，可安全地使用密钥、调用第三方 API、执行耗时逻辑。敏感逻辑、支付、解密、复杂计算等应放在云函数中，避免密钥和逻辑暴露在前端。

本文介绍云函数的创建、部署、调用与常见用法。云函数按调用次数计费，有免费额度，超出后按量付费。

## 创建云函数

在项目中 `cloudfunctions` 目录下新建文件夹，如 `login`，内含 `index.js`（入口文件）和 `config.json`（可选配置）。云函数必须导出 `main` 方法，接收 `event`（调用时传入的 data）和 `context`（运行上下文）：

```javascript
// cloudfunctions/login/index.js
const cloud = require('wx-server-sdk');
cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV });

exports.main = async (event, context) => {
  const { code } = event;
  // 调用微信接口换取 openid
  const wxacode = await cloud.callWxOpen({
    api: 'jscode2session',
    args: { js_code: code }
  });
  // 生成 token 返回
  return { openid: wxacode.openid, token: 'xxx' };
};
```


注意：上述示例中 `cloud.callWxOpen` 为简化写法，实际需使用微信开放接口。云函数需在开发者工具中右键「上传并部署」后才能在云端运行。

## 小程序端调用

小程序端通过 `wx.cloud.callFunction` 调用云函数，传入 name（函数名）和 data（参数）。返回的 res.result 即为云函数 return 的值。

```javascript
wx.cloud.callFunction({
  name: 'login',
  data: { code: 'xxx' }
}).then(res => {
  console.log(res.result);
});
```

## 云函数内访问数据库

云函数内通过 `cloud.database()` 获取数据库实例，与小程序端 API 基本一致。区别在于：云函数默认以管理员身份操作，不受集合权限限制，可执行任意增删改查。适合做批量更新、统计、跨集合查询等。注意初始化时使用 `cloud.DYNAMIC_CURRENT_ENV` 可继承调用方的环境。

```javascript
const db = cloud.database();
const users = await db.collection('users').where({ status: 1 }).get();
```

## 总结

云函数将敏感逻辑置于云端，安全可靠。掌握异步写法、错误处理、数据库操作和 HTTP 请求，可构建完整的服务端能力。建议将云函数按业务模块拆分，避免单个函数过于庞大。
