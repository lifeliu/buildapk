---
layout: post
title: Android LiveData 与数据转换详解
categories: android
tags: [LiveData, 响应式, 数据观察, Android, ViewModel]
date: 2024/5/28 16:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_033.jpg)

## 引言

LiveData 是一种可观察的数据持有者类，具有生命周期感知能力。它确保只有处于活跃生命周期状态（STARTED 或 RESUMED）的观察者会收到数据更新，非活跃时自动停止通知，从而避免内存泄漏和"在已销毁的 Activity 上更新 UI"导致的崩溃。LiveData 是 MVVM 架构中连接 ViewModel 与 View 的常用桥梁，与 ViewModel 配合可实现配置变更（如旋转屏幕）时的数据保留。本文将深入探讨 LiveData 的使用、Transformations、MediatorLiveData 以及 LiveData 与协程、Flow 的配合方式。

## 为什么使用 LiveData

相比直接使用回调或 RxJava，LiveData 与 Lifecycle 集成，观察者无需手动在 onDestroy 中取消订阅，系统会自动管理。LiveData 保证在主线程分发更新，适合直接更新 UI。此外，LiveData 支持"粘性"更新：新观察者会立即收到最后一次的值，避免界面初始化时的空状态。需要注意的是，LiveData 是主线程同步分发的，不适合复杂的数据流转换；对于复杂场景，可考虑 StateFlow 或 SharedFlow。

## LiveData 基础

ViewModel 中通常使用 `MutableLiveData` 存储可变数据，对外暴露只读的 `LiveData`，防止外部修改。`postValue` 用于在后台线程更新，会 post 到主线程再分发；`value` 必须在主线程调用。观察时使用 `observe(lifecycleOwner, observer)`，传入 LifecycleOwner 后，LiveData 会仅在 owner 处于活跃状态时通知观察者，并在 DESTROYED 时自动移除观察，避免泄漏。

```kotlin
class UserViewModel : ViewModel() {
    private val _user = MutableLiveData<User>()
    val user: LiveData<User> = _user
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            val user = repository.getUser(id)
            _user.postValue(user)  // 后台线程使用 postValue
            // _user.value = user  // 主线程使用 value
        }
    }
}

// 观察
viewModel.user.observe(this) { user ->
    updateUI(user)
}
```

## Transformations

`Transformations.map` 将 LiveData 的值转换为另一种类型，生成新的 LiveData。当源 LiveData 更新时，转换后的 LiveData 也会更新。`Transformations.switchMap` 根据源 LiveData 的值切换数据源，适用于"根据 ID 加载详情"等场景：当 userId 变化时，取消对旧 UserDetail 的订阅，订阅新的 UserDetail LiveData。两者都返回新的 LiveData，观察者观察转换后的 LiveData 即可。

```kotlin
// map 转换
val userName: LiveData<String> = Transformations.map(user) { user ->
    user.name
}

// switchMap 切换数据源
val userDetail: LiveData<UserDetail> = Transformations.switchMap(userId) { id ->
    repository.getUserDetail(id)
}
```

## MediatorLiveData

MediatorLiveData 可观察多个 LiveData 源，当任一源更新时重新计算并设置自己的值。通过 `addSource` 添加源，在回调中根据各源的当前值计算合并结果。适用于合并多个数据源、实现"组合 LiveData"的场景。注意：`addSource` 会持有源的引用，若 MediatorLiveData 生命周期较长，需在适当时机 `removeSource` 避免泄漏。

```kotlin
val combined = MediatorLiveData<Pair<User, List<Order>>>()

combined.addSource(userLiveData) { user ->
    combined.value = user to (orderLiveData.value ?: emptyList())
}

combined.addSource(orderLiveData) { orders ->
    combined.value = (userLiveData.value ?: return@addSource) to orders
}
```

## LiveData 与协程

Room、协程等可通过 `asLiveData()` 扩展将 Flow 或 suspend 函数的结果转为 LiveData。在 Compose 中，可使用 `observeAsState()` 将 LiveData 转为 State，实现响应式 UI。对于新项目，若全面使用 Compose 和 Flow，可考虑 StateFlow 替代 LiveData；但 LiveData 在传统 View 系统与 ViewModel 配合中仍是成熟稳定的选择。

```kotlin
// liveData 构建器
val user: LiveData<User> = liveData {
    emit(repository.getUser(id))
}
```

## 总结

LiveData 通过生命周期感知和响应式更新，简化了 UI 与数据的同步，是 MVVM 架构中的重要组件。掌握 Transformations、MediatorLiveData 和与协程的互操作，可以灵活处理各种数据流场景。根据项目技术栈选择 LiveData 或 StateFlow，两者各有适用场景。
