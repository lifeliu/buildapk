---
layout: post
title: Android Navigation Component 导航组件完全指南
categories: android
tags: [Navigation, 导航, Fragment, Android, Jetpack]
date: 2024/4/25 11:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_031.jpg)

## 引言

Navigation Component 是 Android Jetpack 的一部分，用于简化应用内导航的实现。它提供了统一的导航 API，支持 Fragment、Activity 以及自定义导航目标，同时自动处理返回栈、过渡动画、深层链接和类型安全的参数传递（Safe Args）。传统方式中，Fragment 的添加、替换、返回栈管理需要大量样板代码，且容易出错。Navigation Component 通过声明式导航图和集中化的 NavController，大幅简化了这些逻辑。本文将深入探讨 Navigation Component 的核心用法、Safe Args、返回栈管理以及最佳实践。

## 为什么使用 Navigation Component

传统 Fragment 管理依赖 `FragmentManager` 和 `FragmentTransaction`，手动处理 add、replace、addToBackStack 等，代码冗长且易遗漏返回栈逻辑。Navigation Component 将导航结构声明在 XML 导航图中，通过 `NavController` 执行导航，自动处理返回栈和过渡动画。Safe Args 插件可生成类型安全的导航参数类，避免手写 Bundle 时的键名错误和类型不匹配。此外，Navigation 与 Toolbar、BottomNavigationView 等 UI 组件集成，可自动同步标题和选中状态。

## 添加依赖

```kotlin
dependencies {
    implementation("androidx.navigation:navigation-fragment-ktx:2.7.6")
    implementation("androidx.navigation:navigation-ui-ktx:2.7.6")
}
```

若使用 Safe Args，需在项目级 build.gradle 添加 `navigation-safe-args-gradle-plugin`，并在应用级应用该插件。

## 创建导航图

导航图是 XML 文件，定义所有导航目标和它们之间的 action。`startDestination` 指定默认显示的目标。每个 fragment 可定义多个 action，指向不同的 destination。`argument` 声明目标所需的参数及类型，Safe Args 会根据此生成 `*Args` 和 `*Directions` 类。`enterAnim`、`exitAnim` 等可配置过渡动画。

```xml
<!-- res/navigation/nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment"
        android:label="Home">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment"
        android:label="Detail">
        <argument
            android:name="itemId"
            app:argType="string" />
    </fragment>
</navigation>
```

## 在 Activity 中配置

`NavHostFragment` 是导航目标的容器，通过 `navGraph` 指定导航图。`defaultNavHost="true"` 表示该 NavHost 会拦截系统返回键，用于弹出返回栈而非退出 Activity。Activity 的布局中只需一个 `FragmentContainerView`，无需为每个 Fragment 单独声明。

```xml
<androidx.constraintlayout.widget.ConstraintLayout>
    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        app:navGraph="@navigation/nav_graph"
        app:defaultNavHost="true" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

## 执行导航

使用 Safe Args 时，通过生成的 `*Directions` 类创建 action，参数类型安全。也可使用 `navigate(resId, bundle)` 直接传递 Bundle，但需确保键名和类型与目标声明的 argument 一致。`findNavController()` 可在 Fragment 或 View 中获取 NavController，需确保调用时 Fragment 已添加到 NavHost。

```kotlin
// 使用 Safe Args 传递参数
val action = HomeFragmentDirections.actionHomeToDetail(itemId = "123")
findNavController().navigate(action)

// 使用 Bundle
findNavController().navigate(
    R.id.detailFragment,
    bundleOf("itemId" to "123")
)
```

## 接收参数

使用 Safe Args 时，通过 `by navArgs()` 委托获取参数，类型安全且无需手动解析 Bundle。`navArgs()` 会从 `arguments` 中读取并构造 `DetailFragmentArgs` 实例。

```kotlin
class DetailFragment : Fragment() {
    private val args: DetailFragmentArgs by navArgs()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        val itemId = args.itemId
        loadDetail(itemId)
    }
}
```

## 返回栈与 Pop 行为

`popBackStack()` 弹出栈顶；`popBackStack(R.id.homeFragment, false)` 弹出直到指定目标（inclusive 参数控制是否包含该目标）。`NavOptions` 可配置 `popUpTo`、动画等，实现"跳转到详情并清空中间页"等复杂导航逻辑。例如登录成功后 `popUpTo` 登录页并 inclusive，可清除登录页及之上的所有页面。

```kotlin
// 弹出到指定目标
findNavController().popBackStack(R.id.homeFragment, false)

// 使用 NavOptions 配置
val navOptions = NavOptions.Builder()
    .setPopUpTo(R.id.homeFragment, true)
    .setEnterAnim(R.anim.slide_in_right)
    .build()
findNavController().navigate(R.id.detailFragment, null, navOptions)
```

## 总结

Navigation Component 提供了声明式的导航方式，简化了 Fragment 管理和页面跳转逻辑。掌握导航图、Safe Args、返回栈和 NavOptions，可以构建清晰、可维护的导航结构，并支持深层链接和过渡动画等高级特性。
