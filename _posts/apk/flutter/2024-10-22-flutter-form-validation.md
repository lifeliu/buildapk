---
layout: post
title: Flutter 表单与验证完全指南
categories: flutter
tags: [Flutter, 表单]
date: 2024/10/22 14:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_010.jpg)

## 引言

Flutter 的 Form 和 TextFormField 提供表单验证能力。Form 持有 FormState，通过 GlobalKey<FormState> 访问。FormState.validate 会遍历所有 FormField 并执行 validator，若有验证失败则返回 false 并显示错误信息。TextFormField 的 validator 接收当前值，返回 null 表示通过，返回字符串表示错误信息。结合 TextEditingController 可编程控制文本，或使用 FormField 的自定义子类实现更复杂的表单控件。本文将介绍表单的创建、验证时机、错误展示以及与状态管理的配合。

## 表单验证流程

用户点击提交时调用 _formKey.currentState!.validate()。若所有 validator 均返回 null，validate 返回 true，可继续提交逻辑；若有 validator 返回错误字符串，validate 返回 false，对应 FormField 会显示错误文本并保持错误状态。可设置 autovalidateMode 为 AutovalidateMode.onUserInteraction，在用户输入后实时验证。注意：validate 会触发所有 FormField 的 validator，确保 validator 逻辑轻量，避免耗时操作。

## 表单实现

Form 的 key 用于在父 Widget 中获取 FormState。TextFormField 的 decoration 可设置 labelText、hintText、errorText（由 validator 返回值自动设置）。initialValue 与 controller 二选一，使用 controller 时可动态控制文本。obscureText 用于密码框。onSaved 在 FormState.save 时调用，可在此收集表单数据。

```dart
final _formKey = GlobalKey<FormState>();

Form(
  key: _formKey,
  child: Column(
    children: [
      TextFormField(
        decoration: InputDecoration(labelText: '用户名'),
        validator: (value) {
          if (value == null || value.isEmpty) return '请输入用户名';
          if (value.length < 3) return '至少3个字符';
          return null;
        },
      ),
      TextFormField(
        decoration: InputDecoration(labelText: '密码'),
        obscureText: true,
        validator: (value) {
          if (value == null || value.isEmpty) return '请输入密码';
          return null;
        },
      ),
      ElevatedButton(
        onPressed: () {
          if (_formKey.currentState!.validate()) {
            _formKey.currentState!.save();
            // 提交
          }
        },
        child: Text('提交'),
      ),
    ],
  ),
)
```

## 总结

合理使用 Form 和 validator 可构建用户友好的表单体验。注意验证时机、错误提示的友好性，以及敏感表单（如登录）的安全处理。对于复杂表单，可考虑使用 flutter_form_builder、reactive_forms 等库简化开发。
