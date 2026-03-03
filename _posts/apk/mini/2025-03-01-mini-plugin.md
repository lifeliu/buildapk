---
layout: post
title: 微信小程序插件开发
categories: mini
tags: [小程序, 插件]
date: 2025/3/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序插件是可被其他小程序复用的功能模块，如支付、地图、客服、数据统计等。插件有独立的 appid 和代码包，需单独开发、审核、发布。宿主小程序通过配置引用插件，即可使用插件提供的组件和页面。插件适合能力复用、商业化分发等场景。

本文介绍插件的创建与接入方式。插件有代码包大小限制，需精简；且不能包含 tabBar、不能直接跳转插件外页面等，需在开发时注意。

## 插件与普通小程序区别

插件无独立运行入口，必须被宿主引用才能使用。插件的 appid 在公众平台创建「小程序插件」时获得。插件代码包有大小限制，需精简。插件可对外暴露组件、页面，通过 plugin.json 的 publicComponents、pages 声明。

- 插件无独立运行入口，需被宿主小程序引用
- 插件有独立的 appid，在公众平台创建
- 插件代码包有大小限制，需精简

## 创建插件

在插件项目中，`plugin` 目录为插件代码，结构类似小程序：

```
plugin/
  components/
  pages/
  plugin.json  # 插件配置，声明对外暴露的组件与页面
```

`plugin.json` 中声明 `publicComponents`、`pages` 等供宿主使用。

## 宿主接入

在 app.json 中声明插件：

```json
{
  "plugins": {
    "myPlugin": {
      "version": "1.0.0",
      "provider": "wxid_xxx"
    }
  }
}
```

使用插件组件时，需带插件前缀：

```html
<myPlugin:my-component prop="{{ value }}"/>
```

## 总结

插件适合通用能力封装与商业化。开发时注意与宿主的隔离、版本兼容与文档说明，便于第三方接入。插件更新后，宿主需重新选择版本号才能使用新版本。
