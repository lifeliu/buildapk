---
layout: post
title: Android Lifecycle 生命周期感知组件详解
categories: android
tags: [Lifecycle, 生命周期, ViewModel, LiveData, Android]
date: 2024/5/15 09:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_032.jpg)

## 引言

Android Lifecycle 是 Jetpack 架构组件的核心，它允许其他组件感知 Activity 和 Fragment 的生命周期状态变化。传统方式中，开发者需要在 Activity 的各个生命周期方法中手动调用第三方库的初始化、暂停、销毁等，容易遗漏或顺序错误，导致内存泄漏或非法状态访问。Lifecycle 通过 `LifecycleOwner` 和 `LifecycleObserver` 的观察者模式，将生命周期事件标准化并分发给观察者，使组件能够自管理其行为。ViewModel、LiveData、LifecycleScope 等都基于 Lifecycle 实现。本文将深入探讨 Lifecycle 的状态与事件、LifecycleObserver 的用法、repeatOnLifecycle 以及自定义 LifecycleOwner。

## 为什么需要 Lifecycle

在 Activity 中直接使用回调或异步任务时，若在 Activity 已销毁后仍执行回调，可能导致崩溃或内存泄漏。Lifecycle 让组件能够感知"当前是否处于活跃状态"，从而在合适的时机启动或停止订阅、更新、资源占用等。例如，仅在 STARTED 以上状态时收集 Flow 更新 UI，在 STOPPED 时自动停止，可避免在不可见时仍执行不必要的操作，同时防止泄漏。

## Lifecycle 状态与事件

Lifecycle 使用状态（State）和事件（Event）两个维度描述生命周期。状态是离散的节点，事件是状态之间的转换。Observer 收到的是事件，内部状态机根据事件更新当前状态。理解状态与事件的对应关系有助于正确实现 Observer：例如 ON_START 事件后进入 STARTED 状态，ON_STOP 后进入 CREATED 状态。

```kotlin
// 生命周期状态
enum class State {
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED,
    DESTROYED
}

// 生命周期事件
enum class Event {
    ON_CREATE,
    ON_START,
    ON_RESUME,
    ON_PAUSE,
    ON_STOP,
    ON_DESTROY
}
```

## 使用 LifecycleObserver

`DefaultLifecycleObserver` 是接口，提供各生命周期方法的默认空实现，可按需重写。将 Observer 通过 `lifecycle.addObserver()` 注册后，LifecycleOwner 在状态变化时会回调对应方法。在 `onCreate` 中初始化、`onStart` 中注册监听、`onStop` 中注销、`onDestroy` 中释放，可以确保资源与生命周期正确绑定。注意：Observer 会持有 LifecycleOwner 的引用，若 Observer 是匿名内部类且持有 Activity 引用，需注意避免循环引用；通常将 Observer 设为静态或弱引用。

```kotlin
class MyLifecycleObserver : DefaultLifecycleObserver {
    
    override fun onCreate(owner: LifecycleOwner) {
        // 初始化资源
    }
    
    override fun onStart(owner: LifecycleOwner) {
        // 注册监听器
    }
    
    override fun onResume(owner: LifecycleOwner) {
        // 恢复更新
    }
    
    override fun onPause(owner: LifecycleOwner) {
        // 暂停更新
    }
    
    override fun onStop(owner: LifecycleOwner) {
        // 注销监听器
    }
    
    override fun onDestroy(owner: LifecycleOwner) {
        // 释放资源
    }
}

// 在 Activity 中注册
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(MyLifecycleObserver())
    }
}
```

## 自定义 LifecycleOwner

除 Activity 和 Fragment 外，自定义 View 或 Presenter 也可实现 `LifecycleOwner`，通过 `LifecycleRegistry` 手动分发状态。在合适的时机（如 View 的 onAttachedToWindow、onDetachedFromWindow）更新 `currentState`，观察者即可感知。这样可将生命周期能力扩展到非 Activity/Fragment 的组件，便于在自定义 View 中集成需要生命周期的逻辑。

```kotlin
class CustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs), LifecycleOwner {
    
    private val lifecycleRegistry = LifecycleRegistry(this)
    
    override fun getLifecycle(): Lifecycle = lifecycleRegistry
    
    fun onAttached() {
        lifecycleRegistry.currentState = Lifecycle.State.STARTED
    }
    
    fun onDetached() {
        lifecycleRegistry.currentState = Lifecycle.State.DESTROYED
    }
}
```

## 在特定状态下执行

`repeatOnLifecycle` 是 Kotlin 扩展，在 `lifecycleScope` 中启动一个协程，当生命周期进入指定状态（如 STARTED）时执行块内的代码，离开该状态时取消协程。这非常适合 Flow 的收集：仅在界面可见时收集，避免后台执行和泄漏。这是替代 `lifecycleScope.launch` 直接收集 Flow 的推荐方式，因为后者在 STOPPED 时不会自动取消。

```kotlin
lifecycleScope.launch {
    lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        // 仅在 STARTED 或 RESUMED 时执行
        collectFlow()
    }
}
```

## 总结

Lifecycle 是构建健壮 Android 应用的基础，合理使用可以避免常见的内存泄漏和状态管理问题。掌握 LifecycleObserver、repeatOnLifecycle 和自定义 LifecycleOwner，可以编写生命周期感知的组件，与 ViewModel、LiveData、Flow 等配合构建现代化的 Android 架构。
