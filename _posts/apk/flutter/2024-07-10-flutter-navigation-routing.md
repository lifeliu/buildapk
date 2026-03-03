---
layout: post
title: Flutter 路由与导航完全指南
categories: flutter
tags: [Flutter, 路由]
date: 2024/7/10 09:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_004.jpg)

## 引言

Flutter 的导航基于 Navigator 和 Route，通过栈管理页面。Navigator.push 将新页面压入栈，pop 弹出并可选返回结果。Navigator 1.0 使用命令式 API，需手动管理路由表；GoRouter、auto_route 等声明式路由库提供了基于配置的路由、深层链接和类型安全参数传递。本文将介绍基础导航、命名路由、路由传参以及 GoRouter 的用法和最佳实践。

## 为什么需要路由管理

随着页面增多，直接使用 Navigator.push 会导致路由逻辑分散、难以维护。声明式路由将路由配置集中管理，支持路径参数、查询参数、重定向、嵌套路由等。GoRouter 是 Flutter 官方推荐的声明式路由方案，支持 Web URL 与路由的映射，便于实现深层链接和 SEO（Web）。

## 基础导航

Navigator.push 传入 MaterialPageRoute 或 CupertinoPageRoute，可指定 builder、settings（含 arguments）。Navigator.pop 的第二个参数为返回给上一页的结果。带返回结果的跳转通常配合 await 使用，上一页在 push 返回的 Future 完成时收到结果。注意：push 返回的 Future 在目标页 pop 时才完成。

```dart
// Push
Navigator.push(context, MaterialPageRoute(
  builder: (context) => DetailPage(id: '123'),
));

// Pop with result
Navigator.pop(context, {'selected': true});

// 接收返回结果
final result = await Navigator.push(context, MaterialPageRoute(
  builder: (context) => SelectPage(),
));

// Named routes（需在 MaterialApp 的 routes 中配置）
Navigator.pushNamed(context, '/detail', arguments: {'id': '123'});
```

## GoRouter

GoRouter 通过 routes 配置路由树，支持 path 参数（如 /detail/:id）、redirect、refreshListenable（用于登录状态变化时重定向）。GoRouter.of(context) 或 context.go/context.push 执行导航。go 替换当前栈，push 压入新页。适合需要深层链接、统一路由管理的项目。

```dart
final router = GoRouter(
  routes: [
    GoRoute(path: '/', builder: (_, __) => HomePage()),
    GoRoute(
      path: '/detail/:id',
      builder: (_, state) => DetailPage(id: state.pathParameters['id']!),
    ),
  ],
);

// MaterialApp.router
MaterialApp.router(routerConfig: router);

// 使用
context.go('/detail/123');
context.push('/detail/123');
```

## 总结

合理选择 Navigator 或 GoRouter 可构建清晰的导航结构。简单应用可用命令式 Navigator，中大型项目推荐 GoRouter 等声明式方案，便于维护和扩展深层链接。
