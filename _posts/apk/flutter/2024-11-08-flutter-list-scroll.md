---
layout: post
title: Flutter 列表与滚动完全指南
categories: flutter
tags: [Flutter, 列表]
date: 2024/11/8 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_011.jpg)

## 引言

ListView 是 Flutter 的列表组件，支持多种构造方式。ListView(children: [...]) 一次性构建所有子项，适合短列表；ListView.builder 按需构建，只构建可见项，适合长列表；ListView.separated 支持 itemBuilder 和 separatorBuilder，便于添加分隔线。CustomScrollView 配合 Sliver 可实现复杂滚动效果（如头部折叠、粘性标题）。GridView 用于网格布局，同样有 builder 和 separated 变体。本文将介绍列表的优化、懒加载、分页加载以及 Sliver 的常见用法。

## 为什么使用 ListView.builder

ListView(children: [...]) 会立即构建所有子 Widget，若列表有上千项会导致内存和性能问题。ListView.builder 的 itemBuilder 仅在子项即将进入视口时调用，离开视口后子 Widget 可被回收，从而支持无限长列表。itemCount 可省略表示无限列表，但需注意滚动到末尾时的加载更多逻辑。

## ListView.builder

itemBuilder 接收 context 和 index，返回对应项的 Widget。itemCount 限制列表长度。addAutomaticKeepAlives、addRepaintBoundaries 等可影响子项的生命周期和重绘边界，通常使用默认值即可。对于分页加载，可监听 ScrollController 的 scroll 事件，在接近底部时加载下一页，或使用 scrollable_positioned_list、infinite_scroll_pagination 等库。

```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ListTile(
      title: Text(items[index].title),
      subtitle: Text(items[index].subtitle),
    );
  },
)

// 带分隔线
ListView.separated(
  itemCount: items.length,
  separatorBuilder: (_, __) => Divider(),
  itemBuilder: (context, index) => ListTile(title: Text(items[index])),
)
```

## 总结

使用 builder 可避免一次性构建所有子项，提升长列表性能。对于分页、下拉刷新等需求，可结合 RefreshIndicator、ScrollController 或专用分页库实现。CustomScrollView 与 Sliver 可满足更复杂的滚动布局需求。
