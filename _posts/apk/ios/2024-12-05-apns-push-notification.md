---
layout: post
title: iOS APNs 推送通知完全指南
categories: ios
tags: [APNs, 推送通知, 远程通知, iOS, UserNotifications]
date: 2024/12/5 09:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_057.jpg)

## 引言

APNs（Apple Push Notification service）是 Apple 的推送通知服务，允许服务器向已安装应用的设备发送通知。通过 UserNotifications 框架，应用可以请求权限、注册远程推送、接收并展示通知、处理用户点击，以及发送本地通知。推送通知是提升用户留存和参与度的重要手段，但需注意权限申请时机、通知内容和频率，避免打扰用户。本文将介绍推送通知的完整配置、接收处理流程、本地通知以及最佳实践。

## 推送通知的流程概述

推送流程涉及应用、APNs 和你的服务器。应用向系统请求权限并注册远程推送，获得 device token 后上传到你的服务器。当需要推送时，服务器构造 payload，通过 APNs 的 HTTP/2 接口发送，APNs 将通知推送到设备。设备收到后，若应用在前台，可通过 `UNUserNotificationCenterDelegate` 决定是否展示；若在后台或未运行，系统按 payload 展示。用户点击通知时，可通过 `didReceive response` 跳转到对应界面。

## 请求权限

在向用户展示推送前，必须请求授权。`requestAuthorization` 会弹出系统权限对话框，用户可选择允许或拒绝。建议在合适的业务场景（如完成注册、开启消息提醒功能）时再请求，而非启动即请求，以提高通过率。授权成功后，调用 `registerForRemoteNotifications()` 向 APNs 注册，系统会回调 `didRegisterForRemoteNotificationsWithDeviceToken` 或 `didFailToRegisterForRemoteNotificationsWithError`。

```swift
import UserNotifications

UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, error in
    if granted {
        DispatchQueue.main.async {
            UIApplication.shared.registerForRemoteNotifications()
        }
    }
}
```

`options` 指定需要的权限类型：`.alert` 横幅/列表展示，`.badge` 角标，`.sound` 声音。可根据业务需要选择。

## 注册设备 Token

Device Token 是 APNs 用于标识设备与应用组合的唯一标识，服务器推送时必须携带。Token 在应用安装、重装或系统升级后可能变化，因此每次获得新 Token 都应上传到服务器并更新。Token 是二进制数据，通常转为十六进制字符串传输。注意：模拟器上 `registerForRemoteNotifications` 可能失败，真机调试才能获得有效 Token。

```swift
func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    print("Device Token: \(token)")
    // 发送 token 到服务器
}
```

## 处理推送

设置 `UNUserNotificationCenter.current().delegate` 以接收通知相关回调。`willPresent` 在应用前台收到通知时调用，返回的 `UNNotificationPresentationOptions` 决定是否展示横幅、声音、角标。若不实现或返回空，前台通知默认不展示。`didReceive` 在用户点击通知时调用，可通过 `response.notification.request.content.userInfo` 获取自定义数据，实现跳转逻辑。

```swift
extension AppDelegate: UNUserNotificationCenterDelegate {
    func userNotificationCenter(_ center: UNUserNotificationCenter, 
                               willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
        return [.banner, .sound, .badge]
    }
    
    func userNotificationCenter(_ center: UNUserNotificationCenter, 
                               didReceive response: UNNotificationResponse) async {
        let userInfo = response.notification.request.content.userInfo
        // 处理用户点击通知
    }
}
```

## 本地通知

本地通知由应用在设备上创建，无需服务器。适用于提醒、定时任务、地理围栏触发等场景。`UNMutableNotificationContent` 设置标题、正文、声音等；`UNTimeIntervalNotificationTrigger`、`UNCalendarNotificationTrigger`、`UNLocationTrigger` 等定义触发条件。创建 `UNNotificationRequest` 并 `add` 到 `UNUserNotificationCenter` 即可。注意：应用被系统终止后，已添加的本地通知仍会按计划触发。

```swift
let content = UNMutableNotificationContent()
content.title = "提醒"
content.body = "您有一条新消息"
content.sound = .default

let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 5, repeats: false)
let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: trigger)
UNUserNotificationCenter.current().add(request)
```

## 总结

合理使用推送通知能够提升用户参与度，需注意权限申请时机、通知内容和频率，避免过度打扰。实现时确保 Token 正确上传、点击跳转逻辑完善，并处理好前台与后台的展示策略，以提供良好的用户体验。
