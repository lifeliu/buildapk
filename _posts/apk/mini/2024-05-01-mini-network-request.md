---
layout: post
title: 微信小程序网络请求与 API 封装
categories: mini
tags: [小程序, 网络]
date: 2024/5/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序通过 `wx.request` 发起 HTTP 请求，与后端 API 交互获取数据。由于小程序运行在微信客户端，请求需在公众平台配置合法域名，否则正式版无法访问。实际项目中，几乎都会封装统一的请求模块，统一处理 baseURL、鉴权 token、错误提示、loading 状态等，避免在每个页面重复编写相似代码。

本文介绍 `wx.request` 的基础用法及常见的封装模式，帮助开发者快速搭建可维护的网络层。

## wx.request 基础用法

`wx.request` 支持 GET、POST 等常见方法，可设置 header、data、timeout 等。返回的 res 包含 statusCode、data、header 等。注意：wx.request 的 data 在 GET 请求中会拼接到 URL 的 query，在 POST 中会作为请求体发送。

```javascript
wx.request({
  url: 'https://api.example.com/user',
  method: 'GET',
  data: { id: 1 },
  header: {
    'content-type': 'application/json',
    'Authorization': 'Bearer ' + token
  },
  success(res) {
    console.log(res.data);
  },
  fail(err) {
    console.error(err);
  }
});
```

**域名配置说明：** 正式版小程序只能请求已配置的合法域名。需在微信公众平台「开发管理 - 开发设置 - 服务器域名」中配置 request 合法域名，且必须使用 HTTPS。本地开发时，可在开发者工具「详情 - 本地设置」中勾选「不校验合法域名、web-view（业务域名）、TLS 版本以及 HTTPS 证书」，便于调试。

## 封装请求模块

封装时通常将 baseURL、token 注入、错误统一处理、loading 等集中管理。以下示例展示了一个基础的 Promise 化封装，可根据项目需要扩展拦截器、重试、超时等逻辑。

```javascript
// utils/request.js
const BASE_URL = 'https://api.example.com';

function request(options) {
  return new Promise((resolve, reject) => {
    const token = wx.getStorageSync('token');
    wx.request({
      url: BASE_URL + options.url,
      method: options.method || 'GET',
      data: options.data,
      header: {
        'content-type': 'application/json',
        'Authorization': token ? `Bearer ${token}` : '',
        ...options.header
      },
      success(res) {
        if (res.statusCode === 200) {
          resolve(res.data);
        } else {
          reject(res);
        }
      },
      fail: reject
    });
  });
}

module.exports = { request };
```

## 总结

合理封装请求模块，统一处理域名、鉴权、错误和 loading，可提升开发效率和代码可维护性。上线前务必配置合法域名，并注意区分开发/生产环境。建议将 API 按模块拆分（如 userApi、orderApi），便于维护和 Mock。
