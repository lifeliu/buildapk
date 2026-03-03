---
layout: post
title: 微信小程序地图与定位
categories: mini
tags: [小程序, 地图]
date: 2024/11/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

小程序提供 `map` 组件展示地图，支持标记点（markers）、路线（polyline）、缩放、旋转等。定位通过 `wx.getLocation` 获取，需用户授权位置权限。地图和定位是 LBS（基于位置的服务）类小程序的核心能力，如外卖、打车、找店等。

本文介绍地图展示与定位的用法。注意：需在 app.json 中声明 `scope.userLocation` 权限，用户拒绝时需做好引导。地图组件使用腾讯地图或高德地图（需配置），不同平台可能有差异。

## map 组件

map 通过 `longitude`、`latitude` 指定中心点，`scale` 控制缩放级别（5-18）。`markers` 为标记点数组，每项含 id、latitude、longitude、title、iconPath 等。`bindmarkertap` 可监听标记点点击。`polyline` 可绘制路线，用于导航、轨迹展示等。

```html
<map
  longitude="{{ lng }}"
  latitude="{{ lat }}"
  scale="16"
  markers="{{ markers }}"
  bindmarkertap="onMarkerTap"
/>
```

`markers` 为标记点数组，含 id、latitude、longitude、title、iconPath 等。可配置 polyline 绘制路线。

## 获取定位

`wx.getLocation` 的 type 可选 `wgs84`（GPS 坐标）或 `gcj02`（国测局坐标，国内地图常用）。返回的 latitude、longitude 可直接用于 map 组件的 center。用户拒绝授权时，fail 回调会触发，可引导用户去设置中开启。

```javascript
wx.getLocation({
  type: 'gcj02',
  success(res) {
    const { latitude, longitude } = res;
    this.setData({ lat: latitude, lng: longitude });
  },
  fail() {
    wx.showToast({ title: '请授权定位', icon: 'none' });
  }
});
```

`type` 为 wgs84 或 gcj02，国内地图一般用 gcj02。需在 app.json 中声明 `scope.userLocation` 权限。

## 选点与逆地理编码

`wx.chooseLocation` 可打开地图选点。逆地理编码（坐标转地址）需调用第三方或微信接口。

## 总结

地图与定位是 LBS 类小程序的必备能力。注意权限申请与拒绝场景的引导，以及不同坐标系的选择。逆地理编码（坐标转地址）需调用第三方服务或微信相关接口，需在服务端完成。
