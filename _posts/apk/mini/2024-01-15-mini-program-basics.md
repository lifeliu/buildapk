---
layout: post
title: 微信小程序基础架构与项目结构
categories: mini
tags: [小程序, 基础]
date: 2024/1/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

微信小程序采用「逻辑层 + 视图层」分离的架构，逻辑层运行在 JSCore 中，视图层由 WebView 渲染。开发者使用 WXML 描述结构、WXSS 描述样式、JavaScript 编写逻辑，通过数据绑定实现视图与数据的同步。这种双线程架构保证了逻辑与渲染的隔离，逻辑层的数据变更会通过 Native 层转发到视图层，从而驱动界面更新。

理解小程序的目录结构、配置文件和各层职责是开发的第一步。与传统的 Web 开发不同，小程序有严格的目录规范和配置约定，例如每个页面必须由四个文件组成、app.json 中的 pages 数组决定了小程序的页面路由等。掌握这些基础概念，有助于快速搭建项目并避免常见错误。

## 项目目录结构

小程序项目采用约定式目录结构，核心文件集中在根目录，页面和组件按功能模块组织。以下是一个典型项目的目录结构：

```
miniprogram/
├── app.js          # 小程序逻辑入口
├── app.json        # 全局配置
├── app.wxss        # 全局样式
├── pages/          # 页面目录
│   ├── index/
│   │   ├── index.js
│   │   ├── index.json
│   │   ├── index.wxml
│   │   └── index.wxss
│   └── logs/
└── utils/          # 工具函数
```

- **app.js**：小程序的入口文件，通过 `App()` 注册应用，定义全局生命周期（onLaunch、onShow、onHide 等）。可在此初始化全局数据、登录态、第三方 SDK 等。
- **app.json**：全局配置文件，声明页面路径、窗口样式、tabBar、网络超时、权限等。框架会解析此文件以确定小程序的整体结构和行为。
- **app.wxss**：全局样式表，所有页面都会继承其中的样式。常用于定义主题色、字体、通用间距等，减少重复代码。

## app.json 核心配置

app.json 是小程序的核心配置文件，其结构决定了小程序的导航方式、外观和基础能力。以下示例展示了最常用的配置项：

```json
{
  "pages": ["pages/index/index", "pages/logs/logs"],
  "window": {
    "navigationBarTitleText": "我的小程序",
    "navigationBarBackgroundColor": "#ffffff"
  },
  "tabBar": {
    "list": [
      { "pagePath": "pages/index/index", "text": "首页" },
      { "pagePath": "pages/logs/logs", "text": "日志" }
    ]
  }
}
```

**配置说明：**

- `pages` 数组第一项为首页，用户打开小程序时最先展示的页面。数组顺序决定了页面在编译时的优先级。
- `window` 控制整个小程序的窗口表现，包括导航栏标题、背景色、下拉刷新等。每个页面也可在各自的 json 中覆盖这些配置。
- `tabBar` 用于底部多 tab 切换，适用于首页、分类、购物车、我的等典型多入口场景。list 最多 5 项，每项需指定 pagePath 和 text。

## 页面四件套

每个页面由四个文件组成，各司其职：

- **`.js`**：页面逻辑，通过 `Page()` 注册，包含 `data`、生命周期函数和事件处理函数。`data` 中的字段会与 WXML 绑定，修改需通过 `setData`。
- **`.json`**：页面级配置，可省略。若省略则继承 app.json 的 window 配置。常用于设置页面标题、导航栏样式等。
- **`.wxml`**：页面结构，类似 HTML，支持数据绑定、条件渲染、列表渲染等。
- **`.wxss`**：页面样式，仅作用于当前页面。会与 app.wxss 合并后生效。

## 总结

掌握项目结构和 app.json 配置是开发小程序的基础。逻辑层与视图层通过数据绑定通信，开发者只需关注数据变化，框架负责视图更新。建议在项目初期就规划好目录结构，将页面、组件、工具函数合理分层，便于后续维护和扩展。
