---
layout: post
title: 微信小程序列表与滚动
categories: mini
tags: [小程序, 列表]
date: 2024/10/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序中实现列表有两种常见方式：使用 `scroll-view` 组件或依赖页面默认滚动配合 `onReachBottom`。`scroll-view` 可精确控制滚动区域高度和方向，支持横向滚动、下拉刷新、上拉触底加载，适合「局部滚动」或需要自定义滚动行为的场景。页面级滚动则更简单，适合整页列表，通过 `onReachBottom` 实现上拉加载更多。

本文介绍列表渲染、分页加载与虚拟列表优化思路。长列表需注意性能，避免一次性渲染过多节点。

## scroll-view 基础

scroll-view 需设置固定高度（如 `height: 500px` 或 `height: 100vh`），否则无法滚动。`scroll-y` 开启纵向滚动，`scroll-x` 开启横向滚动。`bindscrolltolower` 在滚动到底部时触发，可用于加载更多。`refresher-enabled` 开启下拉刷新，`bindrefresherrefresh` 处理刷新逻辑。

```html
<scroll-view
  scroll-y
  style="height: 500px"
  bindscrolltolower="onLoadMore"
  refresher-enabled
  bindrefresherrefresh="onRefresh"
>
  <view wx:for="{{ list }}" wx:key="id">{{ item.title }}</view>
</scroll-view>
```

`scroll-y` 开启纵向滚动，`scrolltolower` 触底时加载更多。`refresher-enabled` 开启下拉刷新。

## 分页加载

分页加载可避免一次性请求过多数据，提升首屏速度和体验。通常维护 page、list、hasMore 等状态，触底时请求下一页，追加到 list。注意在请求中加锁，避免重复请求；请求失败时恢复 page，便于重试。

```javascript
Page({
  data: { list: [], page: 1, hasMore: true },
  onLoadMore() {
    if (!this.data.hasMore) return;
    this.loadData(this.data.page);
  },
  loadData(page) {
    request({ url: '/api/list', data: { page } }).then(res => {
      const list = page === 1 ? res.data : [...this.data.list, ...res.data];
      this.setData({
        list,
        page: page + 1,
        hasMore: res.data.length >= 10
      });
    });
  }
});
```

## 总结

合理使用 scroll-view 与 onReachBottom，配合分页与 loading 状态，可实现流畅的列表体验。长列表（如数百条以上）可考虑虚拟列表或官方 recycle-view 组件，只渲染可视区域节点，优化滚动性能。建议在列表项上设置 `wx:key`，保证 diff 正确。
