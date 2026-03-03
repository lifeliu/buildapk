---
layout: post
title: Android Espresso UI 自动化测试指南
categories: android
tags: [Espresso, UI 测试, 自动化测试, Android, 仪器测试]
date: 2024/8/22 11:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_037.jpg)

## 引言

Espresso 是 Android 推荐的 UI 测试框架，用于编写与 UI 交互的仪器测试（Instrumentation Test）。与单元测试不同，Espresso 测试运行在设备或模拟器上，启动 Activity/Fragment，模拟用户操作（点击、输入、滑动等），并验证界面行为。它适合验证关键用户流程，如登录、下单、设置等端到端场景。本文将介绍 Espresso 的核心用法、元素查找、异步等待以及最佳实践。

## Espresso 与单元测试的定位

单元测试针对业务逻辑，快速、隔离、不依赖 UI。UI 测试验证用户可见的行为，较慢、较脆弱（UI 变化可能导致失败），但能发现集成问题。两者互补：单元测试保证逻辑正确，UI 测试保证关键流程可用。UI 测试应聚焦核心路径，避免过度覆盖，以控制维护成本。

## 添加依赖

```kotlin
dependencies {
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test:rules:1.5.0")
}
```

## 基础 UI 交互

`onView(withId(R.id.xxx))` 查找视图，`perform(click())`、`perform(typeText("xxx"))` 执行操作，`check(matches(withText("xxx")))` 验证视图状态。Espresso 会自动等待主线程空闲和异步任务完成后再执行操作，但若存在自定义的异步逻辑（如网络请求），可能需要 IdlingResource 或更现代的替代方案。`ActivityScenarioRule` 在测试前启动指定 Activity，测试结束后自动关闭。

```kotlin
@RunWith(AndroidJUnit4::class)
class LoginActivityTest {
    
    @get:Rule
    val activityRule = ActivityScenarioRule(LoginActivity::class.java)
    
    @Test
    fun loginWithValidCredentials() {
        onView(withId(R.id.edit_username))
            .perform(typeText("user@example.com"))
        
        onView(withId(R.id.edit_password))
            .perform(typeText("password123"))
        
        onView(withId(R.id.btn_login))
            .perform(click())
        
        onView(withId(R.id.welcome_text))
            .check(matches(withText("Welcome!")))
    }
}
```

## 列表测试

RecyclerView 的项需通过 `RecyclerViewActions` 操作。`actionOnItemAtPosition` 对指定位置的项执行操作。若列表需要滚动才能找到目标项，Espresso 会自动滚动查找；若使用 `onView(withText("xxx"))` 查找列表中的文本，也会自动滚动。为提升稳定性，建议为关键视图设置 `android:contentDescription` 或 `testTag`，使用 `withContentDescription` 或自定义 Matcher 查找。

```kotlin
@Test
fun `click on list item should open detail`() {
    onView(withId(R.id.recycler_view))
        .perform(RecyclerViewActions.actionOnItemAtPosition<ViewHolder>(0, click()))
    
    onView(withId(R.id.detail_title))
        .check(matches(isDisplayed()))
}
```

## IdlingResource 异步等待

当界面依赖网络请求、数据库查询等异步操作时，Espresso 可能在这些操作完成前就执行断言，导致失败。IdlingResource 允许注册"空闲"条件，Espresso 在条件满足前会等待。实现 IdlingResource 接口，在异步操作开始时标记为 busy，完成时标记为 idle。注意：测试结束后务必 `unregister`，避免影响其他测试。对于 Kotlin 协程，可考虑使用 `IdlingRegistry` 与 `CoroutineDispatcher` 的集成，或使用 `ComposeTestRule` 的同步机制（若使用 Compose）。

```kotlin
@Test
fun testWithAsyncOperation() {
    val idlingResource = SimpleIdlingResource()
    IdlingRegistry.getInstance().register(idlingResource)
    
    onView(withId(R.id.btn_load)).perform(click())
    
    onView(withId(R.id.content)).check(matches(isDisplayed()))
    
    IdlingRegistry.getInstance().unregister(idlingResource)
}
```

## 总结

Espresso 提供了强大的 UI 测试能力，是自动化测试 Android 应用的关键工具。合理使用元素查找、等待机制和测试规则，可以编写稳定、可维护的 UI 测试，覆盖关键用户路径，与单元测试一起构建完整的质量保障体系。
