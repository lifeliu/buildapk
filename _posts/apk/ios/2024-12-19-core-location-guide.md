---
layout: post
title: iOS Core Location 定位服务完全指南
categories: ios
tags: [Core Location, 定位, CLLocationManager, 地理围栏, iOS]
date: 2024/12/19 10:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_059.jpg)

## 引言

Core Location 框架提供设备的定位能力，包括 GPS 定位、Wi-Fi 与基站辅助定位、区域监控（地理围栏）、信标检测等。通过 CLLocationManager，应用可以获取用户位置、监听位置变化、在进入或离开某区域时收到回调。定位涉及用户隐私和电量消耗，需在 Info.plist 中声明用途，并在合适的业务场景下请求权限。本文将介绍 CLLocationManager 的使用、权限配置、持续定位与地理围栏，以及后台定位的注意事项。

## 权限与隐私

iOS 对定位权限有严格要求。必须在 Info.plist 中提供 `NSLocationWhenInUseUsageDescription`（使用期间定位）和/或 `NSLocationAlwaysAndWhenInUseUsageDescription`（始终定位），描述为何需要定位，否则审核会被拒。用户可选择"使用期间"或"始终"授权，也可拒绝。请求权限时应选择与功能匹配的级别：仅需使用时定位的，用 `requestWhenInUseAuthorization()`；需要后台定位的（如导航、运动追踪），再考虑 `requestAlwaysAuthorization()`，并准备向用户解释原因。

## 配置 Info.plist

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>需要定位以展示附近服务</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>需要后台定位以提供位置相关功能</string>
```

若使用后台定位，还需在 Background Modes 中勾选 Location updates。

## 获取定位

CLLocationManager 通过 delegate 回调位置更新。`requestWhenInUseAuthorization()` 触发权限弹窗（若尚未决定），`requestLocation()` 请求一次定位，适合只需获取当前位置的场景。`startUpdatingLocation()` 会持续回调，适合导航等需要连续位置的场景，但会增加电量消耗，用完后应 `stopUpdatingLocation()`。`desiredAccuracy` 影响精度和耗电，按需选择从 `kCLLocationAccuracyBest` 到 `kCLLocationAccuracyThreeKilometers` 的级别。

```swift
import CoreLocation

class LocationManager: NSObject, ObservableObject {
    private let manager = CLLocationManager()
    
    @Published var location: CLLocation?
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined
    
    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }
    
    func requestLocation() {
        manager.requestWhenInUseAuthorization()
        manager.requestLocation()
    }
}

extension LocationManager: CLLocationManagerDelegate {
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        location = locations.last
    }
}
```

实现 `locationManager(_:didChangeAuthorization:)` 可监听权限变化，根据新状态更新 UI 或调整行为。

## 地理围栏

地理围栏（Region Monitoring）允许在用户进入或离开某地理区域时收到回调，适用于签到、附近提醒等场景。使用 `CLCircularRegion` 定义圆形区域，设置 `notifyOnEntry` 和 `notifyOnExit`，调用 `startMonitoring(for:)` 开始监控。系统会管理区域监控，应用被终止后，进入/离开事件仍可能被系统唤醒应用处理。注意：区域数量有限制（约 20 个），且区域半径不能过小（建议不小于 100 米）。

```swift
let region = CLCircularRegion(center: CLLocationCoordinate2D(latitude: 39.9, longitude: 116.4), 
                             radius: 100, 
                             identifier: "office")
region.notifyOnEntry = true
region.notifyOnExit = true
locationManager.startMonitoring(for: region)
```

## 总结

Core Location 为 iOS 应用提供了强大的定位能力，使用时需注意权限说明、合理选择精度和更新策略，并考虑电量消耗。地理围栏适合签到、附近提醒等场景，需注意系统限制和审核对后台定位的要求。
