---
layout: post
title: 微信小程序表单与输入组件
categories: mini
tags: [小程序, 表单]
date: 2024/10/1 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序提供 input、textarea、picker、slider、switch 等表单组件，覆盖单行输入、多行输入、选择器、滑块、开关等场景。配合 `form` 组件可统一收集所有带 name 的子组件数据，在 submit 时一次性获取。本文介绍常用表单组件的用法与 form 的 submit 流程。

注意：input 和 textarea 在 iOS 上可能存在键盘遮挡问题，可考虑使用 `adjust-position` 或监听 focus 时滚动到合适位置。

## input 与 textarea

input 支持 type 为 text、number、idcard、digit 等，可限制输入类型。textarea 为多行输入，可设置 auto-height 自动增高。通过 `bindinput` 可实时获取输入内容，或使用 `model:value` 双向绑定（基础库 2.9.3+）。

```html
<input placeholder="请输入" value="{{ keyword }}" bindinput="onInput"/>
<textarea placeholder="多行输入" bindinput="onTextareaInput"/>
```

```javascript
onInput(e) {
  this.setData({ keyword: e.detail.value });
}
```

## picker 选择器

picker 支持多种 mode：selector（普通选择器）、multiSelector（多列选择）、time（时间）、date（日期）、region（省市区）。根据 mode 不同，range 的数据格式和 bindchange 的 detail 结构不同，需参考文档。

```html
<picker mode="selector" range="{{ cities }}" value="{{ cityIndex }}" bindchange="onCityChange">
  <view>当前选择：{{ cities[cityIndex] }}</view>
</picker>
```

## form 表单

form 的 `bindsubmit` 在用户点击 `form-type="submit"` 的按钮时触发。`e.detail.value` 为对象，key 为各子组件的 name，value 为对应值。可在此做校验、提交到后端等。`form-type="reset"` 可重置表单。

```html
<form bindsubmit="onSubmit">
  <input name="username" placeholder="用户名"/>
  <input name="password" type="password" placeholder="密码"/>
  <button form-type="submit">提交</button>
</form>
```

```javascript
onSubmit(e) {
  const { username, password } = e.detail.value;
}
```

## 总结

合理使用表单组件与 form，可简化数据收集与校验。注意 input 的 focus、blur 等事件，以及键盘遮挡问题的处理。建议将校验逻辑封装成工具函数，在 submit 时统一校验并提示。
