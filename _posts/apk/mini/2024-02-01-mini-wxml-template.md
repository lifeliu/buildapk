---
layout: post
title: 微信小程序 WXML 模板语法详解
categories: mini
tags: [小程序, WXML]
date: 2024/2/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

WXML（WeiXin Markup Language）是小程序的标记语言，语法上类似 HTML，但针对小程序场景做了定制和扩展。它支持数据绑定、条件渲染、列表渲染、模板引用等特性，能够以声明式的方式描述界面结构，并与逻辑层的数据保持同步。

掌握 WXML 是构建小程序界面的核心技能。与 HTML 不同，WXML 中的动态内容必须通过 `{{ }}` 显式绑定，不能直接写 JavaScript 表达式；同时，WXML 提供了一套以 `wx:` 为前缀的指令，用于控制渲染逻辑。理解这些差异，可以避免从 Web 开发迁移时的常见误区。

## 数据绑定

数据绑定是 WXML 与逻辑层通信的桥梁。使用双花括号 `{{ }}` 将 Page 或 Component 的 data 中的字段绑定到视图层，支持简单的表达式和运算。当 data 通过 `setData` 更新时，绑定了该字段的视图会自动刷新。

```html
<view>{{ message }}</view>
<view>{{ a + b }}</view>
<view>{{ flag ? '是' : '否' }}</view>
```

**属性绑定**时，整个属性值需用双引号包裹，内部使用 `{{ }}` 引用变量。注意：属性值为布尔型时，`{{ false }}` 会解析为 false，若写 `checked="false"` 则会被当作字符串，导致 checkbox 仍显示为选中。正确写法是 `checked="{{ false }}"`。

```html
<view id="item-{{ id }}">内容</view>
<checkbox checked="{{ checked }}"/>
```

## 条件渲染

条件渲染用于根据数据状态显示或隐藏不同的 UI 块。`wx:if`、`wx:elif`、`wx:else` 是块级指令，不满足条件的节点不会被渲染到 DOM 中，因此适合条件切换不频繁的场景，可减少不必要的节点。

```html
<view wx:if="{{ score >= 90 }}">优秀</view>
<view wx:elif="{{ score >= 60 }}">及格</view>
<view wx:else>不及格</view>
```

相比之下，`hidden` 通过 CSS 的 `display: none` 隐藏元素，节点始终存在于 DOM 中，仅切换显示状态。适合需要频繁切换的场景（如 Tab 切换），因为不需要反复创建和销毁节点。选择 `wx:if` 还是 `hidden`，需根据切换频率和列表长度权衡。

## 列表渲染

列表渲染通过 `wx:for` 指令遍历数组，为每个元素生成对应的节点。`wx:key` 用于指定列表中每项的唯一标识，帮助框架在 diff 时正确复用节点，避免错位和状态混乱。当列表项有动态增删或排序时，`wx:key` 尤为重要。

```html
<view wx:for="{{ list }}" wx:key="id">
  {{ index }}: {{ item.name }}
</view>
```

默认情况下，循环变量名为 `item`，索引名为 `index`。通过 `wx:for-item` 和 `wx:for-index` 可自定义变量名，避免与作用域内其他变量冲突。例如在嵌套循环中，可设置 `wx:for-item="subItem"` 以区分内外层。

## 模板与引用

当同一块结构需要在多处复用时，可使用 `<template>` 定义模板。模板本身不会渲染，只有通过 `is` 属性引用并传入 `data` 时才会生成实际节点。模板可以定义在当前文件，也可通过 `import` 或 `include` 从其他文件引入，适合卡片、列表项等重复结构的抽象。

```html
<template name="userCard">
  <view>{{ name }} - {{ age }}</view>
</template>
<template is="userCard" data="{{ user }}"/>
```

## 总结

WXML 通过数据绑定、条件与列表渲染、模板等语法，实现声明式 UI。合理使用 `wx:key` 保证列表 diff 正确，根据场景选择 `wx:if` 与 `hidden`，可提升渲染性能。建议将复杂结构抽成模板，保持 WXML 简洁易读。
