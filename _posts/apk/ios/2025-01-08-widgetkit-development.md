---
layout: post
title: iOS WidgetKit 小组件开发完全指南
categories: ios
tags: [iOS, WidgetKit]
date: 2025-01-08
---

![title](https://image.sideproject.cn/titlex/titlex_061.jpg)

## 引言

WidgetKit 是 iOS 14 引入的框架，允许开发者在主屏幕、锁定屏幕和今日视图中展示小组件。小组件使用 SwiftUI 构建，支持小、中、大三种尺寸，通过时间线（Timeline）模型更新内容。合理设计的小组件能够提升应用曝光和用户 engagement，是应用生态的重要组成部分。本文将介绍 Widget 的创建、时间线提供者、视图实现、刷新策略以及最佳实践。

## Widget 的特点与限制

小组件是只读的信息展示，不支持按钮等复杂交互，点击会打开应用并可通过 URL 传递参数实现深度链接。小组件的刷新由系统调度，开发者提供时间线，系统在合适时机请求并展示。小组件 Extension 与主应用是独立进程，共享 App Group 可读写同一 UserDefaults 或文件，用于数据同步。注意：小组件有内存和刷新频率限制，不宜做复杂计算或频繁网络请求。

## 创建 Widget Extension

在 Xcode 中：File → New → Target → Widget Extension，输入名称并选择是否包含配置。创建后会生成 Widget 的入口结构和默认实现。若需与主应用共享数据，为两者配置相同的 App Group，并在 Widget 中通过 `UserDefaults(suiteName: "group.xxx")` 访问。

## 时间线提供者

`TimelineProvider` 是 Widget 的数据源，需实现三个方法：`placeholder` 用于占位展示（如加载中），`getSnapshot` 用于小组件库中的预览，`getTimeline` 是核心，返回包含多个 `Entry` 的时间线。每个 Entry 包含展示所需的数据和日期，系统会在对应时间切换展示。`Timeline` 的 `policy` 决定何时请求新时间线：`.atEnd` 表示最后一个 Entry 的日期到达时，`.after(date)` 表示指定时间后，`.never` 表示不自动刷新。

```swift
import WidgetKit
import SwiftUI

struct Provider: TimelineProvider {
    func placeholder(in context: Context) -> SimpleEntry {
        SimpleEntry(date: Date(), title: "Placeholder")
    }
    
    func getSnapshot(in context: Context, completion: @escaping (SimpleEntry) -> Void) {
        let entry = SimpleEntry(date: Date(), title: "Snapshot")
        completion(entry)
    }
    
    func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> Void) {
        let entries = [SimpleEntry(date: Date(), title: "Hello Widget")]
        let timeline = Timeline(entries: entries, policy: .atEnd)
        completion(timeline)
    }
}

struct SimpleEntry: TimelineEntry {
    let date: Date
    let title: String
}
```

在 `getTimeline` 中可发起网络请求或读取本地数据，构造多个 Entry 表示未来一段时间的内容，系统会按时间自动切换。

## 视图实现

根据 `context.family` 判断当前尺寸（`.systemSmall`、`.systemMedium`、`.systemLarge`），可提供不同布局。视图应简洁，避免复杂视图和大量文本，确保在各尺寸下都能良好展示。使用 `widgetURL` 或 `widgetLink` 为小组件或部分区域设置点击跳转。

```swift
struct WidgetEntryView: View {
    var entry: Provider.Entry
    
    var body: some View {
        VStack {
            Text(entry.title)
            Text(entry.date, style: .time)
        }
    }
}
```

## 刷新小组件

主应用在数据更新后，可调用 `WidgetCenter.shared.reloadTimelines(ofKind: "MyWidget")` 请求系统尽快刷新指定 Widget。系统不保证立即刷新，会综合考虑电量、使用习惯等因素。因此，时间线中提供未来一段时间的数据，可以减少对即时刷新的依赖。

```swift
WidgetCenter.shared.reloadTimelines(ofKind: "MyWidget")
```

## 总结

WidgetKit 为应用提供了主屏幕曝光机会，合理设计能够提升用户 engagement。掌握时间线模型、多尺寸适配和刷新策略，可以构建实用、美观的小组件。注意性能和数据共享方式，确保小组件轻量且数据及时。
