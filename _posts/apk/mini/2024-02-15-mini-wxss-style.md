---
layout: post
title: 微信小程序 WXSS 样式与布局
categories: mini
tags: [小程序, WXSS]
date: 2024/2/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

WXSS（WeiXin Style Sheets）是小程序的样式语言，基于 CSS 规范并做了扩展。它支持 rpx 响应式单位、样式导入、选择器增强等特性，能够帮助开发者快速实现多端适配和统一的视觉风格。与 Web 开发中的 CSS 相比，WXSS 在单位、选择器等方面有部分限制，但核心用法一致，前端开发者可以快速上手。

掌握 WXSS 是实现精美界面的关键。在实际开发中，建议将主题色、字体、间距等提取为变量或公共样式，通过 `@import` 引入，便于维护和主题切换。

## rpx 响应式单位

小程序运行在不同尺寸的设备上，固定 px 会导致大屏元素过小、小屏元素过大。rpx（responsive pixel）是微信提供的响应式单位，规定屏幕宽度为 750rpx，框架会根据实际屏幕宽度自动换算。当设计稿按 750px 宽度出图时，可直接将 px 数值改为 rpx，实现「1:1 还原」。

```css
.container {
  width: 750rpx;   /* 全屏宽 */
  font-size: 28rpx; /* 约 14px @ 375pt */
}
```

例如在 375pt 宽度的 iPhone 上，750rpx 会换算为 375px；在 414pt 的设备上则换算为 414px。不同设备会自动换算，实现多端适配。注意：rpx 不适合边框等需要精细控制的场景，边框建议使用 1px 或配合 transform 实现 0.5px 细线。

## 样式导入

当多个页面或组件需要共享样式时，可将公共样式抽离到独立文件，使用 `@import` 引入。引入路径可以是相对路径或绝对路径（以 `/` 开头）。被引入的样式会合并到当前文件中，与直接书写效果相同。

```css
@import "/common/theme.wxss";
```

## 选择器

WXSS 支持类选择器（`.class`）、id 选择器（`#id`）、元素选择器（`view`、`text` 等）、后代选择器、伪类（`:active`、`:focus` 等）。但小程序对选择器做了限制：不支持 `*` 通配符，且选择器层级不宜过深（建议不超过三层），以避免样式污染和性能问题。组件内的样式默认具有隔离性，不会影响外部，除非使用 `externalClasses` 或 `~` 选择器。

```css
.page {
  padding: 20rpx;
}
#header {
  height: 88rpx;
}
view::after {
  content: '';
}
```

## Flex 布局

小程序官方推荐使用 Flex 布局实现页面排版。Flex 可以方便地实现水平/垂直居中、等分布局、两端对齐等常见需求，且兼容性好。`display: flex` 在小程序中默认生效，部分组件（如 `view`）的默认 display 即为 flex，可直接使用 `flex-direction`、`justify-content`、`align-items` 等属性。

```css
.flex-row {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}
.flex-item {
  flex: 1;
}
```

## 总结

WXSS 通过 rpx、导入、选择器和 Flex 布局，实现响应式与可维护的样式。合理组织全局与页面样式，将主题和通用样式抽离，可提升开发效率和后期维护成本。建议在项目初期建立统一的设计规范（如 8rpx 栅格、标准色板），并在 app.wxss 中定义基础变量。
