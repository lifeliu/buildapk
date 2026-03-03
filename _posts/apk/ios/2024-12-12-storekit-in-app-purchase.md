---
layout: post
title: iOS StoreKit 应用内购买完全指南
categories: ios
tags: [StoreKit, 应用内购买, IAP, 订阅, iOS]
date: 2024/12/12 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_058.jpg)

## 引言

StoreKit 是 Apple 提供的应用内购买框架，支持消耗型（如游戏货币）、非消耗型（如永久解锁功能）、自动续期订阅和非续期订阅等产品类型。StoreKit 2 在 iOS 15 引入，提供基于 async/await 的现代 API，简化了购买流程和交易验证。实现应用内购买需要配置 App Store Connect 中的产品、处理购买流程、验证收据或交易、实现恢复购买，并符合 App Store 审核指南。本文将介绍 StoreKit 2 的基础用法、购买流程、恢复购买以及最佳实践。

## 产品类型说明

- **消耗型**：使用后耗尽，可多次购买，如虚拟货币、道具。
- **非消耗型**：一次购买永久有效，如去广告、解锁功能。
- **自动续期订阅**：按周期自动续费，如会员月付/年付，需处理续期、取消、宽限期等。
- **非续期订阅**：固定时长，到期后不自动续费。

在 App Store Connect 中创建产品时需选择类型，并配置价格、本地化信息等。产品 ID 需与代码中使用的字符串一致。

## StoreKit 2 基础

StoreKit 2 使用 `Product` 表示可购买产品，通过 `Product.products(for:)` 异步获取产品信息。购买时调用 `product.purchase()`，返回 `Product.PurchaseResult`：`.success` 包含验证后的 `Transaction`，`.userCancelled` 表示用户取消，`.pending` 表示等待家长批准等。成功购买后应调用 `transaction.finish()` 完成交易，否则该交易会重复出现在 `Transaction.currentEntitlements` 中。`checkVerified` 用于验证交易签名，确保交易来自 Apple 且未被篡改。

```swift
import StoreKit

// 获取产品
let products = try await Product.products(for: ["com.app.premium_monthly"])

// 购买
guard let product = products.first else { return }
let result = try await product.purchase()

switch result {
case .success(let verification):
    let transaction = try checkVerified(verification)
    await transaction.finish()
case .userCancelled:
    break
case .pending:
    break
@unknown default:
    break
}
```

## 恢复购买

用户换设备或重装应用后，需要恢复已购买内容。`Transaction.currentEntitlements` 是一个异步序列，包含当前用户有权使用的所有交易。遍历并验证交易，根据 `productID` 解锁对应功能。对于订阅，还需检查 `expirationDate` 判断是否在有效期内。应在应用内提供明显的"恢复购买"入口，满足审核要求。

```swift
for await result in Transaction.currentEntitlements {
    guard case .verified(let transaction) = result else { continue }
    // 处理已购买内容
}
```

## 审核与最佳实践

1. **提供恢复购买**：审核要求必须提供恢复购买功能，且入口清晰。
2. **价格展示**：使用 `product.displayPrice` 展示本地化价格，勿硬编码。
3. **网络与错误处理**：购买依赖网络，需处理超时、失败和用户取消，给予友好提示。
4. **沙盒测试**：在 App Store Connect 创建沙盒测试账号，在设备上登录后测试购买流程，不会产生真实扣款。

## 总结

正确实现应用内购买需要处理购买流程、交易验证和恢复购买等环节，确保符合 App Store 审核要求。StoreKit 2 的 async/await API 显著简化了实现，建议新项目优先使用 StoreKit 2。
