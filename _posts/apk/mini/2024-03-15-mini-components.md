---
layout: post
title: 微信小程序组件与自定义组件
categories: mini
tags: [小程序, 组件]
date: 2024/3/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序提供丰富的内置组件，覆盖布局、文本、图片、表单、媒体等常见场景。当内置组件无法满足业务需求时，可封装自定义组件，实现复用和模块化。自定义组件支持 properties（属性）、事件、插槽等，与页面类似，但更轻量、可组合。

合理使用组件化开发，可以显著提升代码复用率和可维护性。例如将商品卡片、订单列表项、弹窗等抽成组件，可在多个页面中复用，修改一处即可全局生效。

## 内置组件概览

- **view**：块级容器，类似 div，默认 flex 布局，常用于布局和分组。
- **text**：文本组件，支持长按复制、解码 HTML 实体。注意：只有 text 内的文本才能长按复制，view 内的文本不支持。
- **image**：图片组件，支持懒加载、多种裁剪模式（mode）、占位图等。建议始终设置宽高，避免布局抖动。
- **button**：按钮，支持多种 open-type 唤起授权、分享、客服等能力。需注意 open-type 会覆盖默认的点击行为。
- **input**：输入框，支持多种 type 和 placeholder。
- **scroll-view**：可滚动区域，可替代页面默认滚动，实现局部滚动、横向滚动等。
- **swiper**：轮播组件，常用于 Banner、图片轮播。

## 创建自定义组件

自定义组件的目录结构同页面，由四个文件组成。在 `components` 目录下创建组件，例如：

```
components/
  my-button/
    my-button.js
    my-button.json
    my-button.wxml
    my-button.wxss
```

`my-button.json` 中必须设置 `"component": true`，以区别于普通页面。在页面中引用时，需在页面的 json 中声明 `usingComponents`，例如：`"usingComponents": { "my-button": "/components/my-button/my-button" }`。

## Properties 与事件

properties 用于接收父组件传入的数据，类似于 Vue 的 props。可指定类型、默认值、是否必填等。组件内部通过 `this.properties.xxx` 访问，修改 properties 会触发视图更新。事件通过 `triggerEvent` 向父组件传递，父组件通过 `bind:事件名` 监听。

```javascript
// my-button.js
Component({
  properties: {
    text: { type: String, value: '按钮' },
    type: { type: String, value: 'default' }
  },
  methods: {
    onTap() {
      this.triggerEvent('tap', { value: 1 });
    }
  }
});
```

```html
<!-- 使用 -->
<my-button text="提交" bind:tap="onSubmit"/>
```

## 总结

合理使用内置组件和自定义组件，可提高开发效率和代码复用。自定义组件的 properties、事件、插槽与生命周期与页面类似，学习成本低。建议将可复用的 UI 块尽早抽成组件，并保持组件职责单一，便于测试和维护。
