---
layout: post
title: Android 动画开发完全指南
categories: android
tags: [动画, Property Animation, 过渡动画, Lottie, Android]
date: 2024/12/22 14:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_045.jpg)

## 引言

Android 提供了多种动画实现方式：视图动画（View Animation，已过时）、属性动画（Property Animation）、过渡动画（Transition）以及第三方库如 Lottie。合理的动画能够提升用户体验，使界面更加生动，引导用户注意力。本文介绍各类动画的实现方式、适用场景以及最佳实践。Compose 的动画 API 与 View 系统不同，本文主要聚焦 View 体系。

## 动画类型概览

- **视图动画**：仅能动画 View 的透明度、缩放、平移、旋转，且不改变实际布局属性，已不推荐。
- **属性动画**：可动画任意对象的任意属性，通过反射或 TypeEvaluator 实现，是最灵活的方案。
- **过渡动画**：用于布局变化时的过渡（如 visibility、位置变化），TransitionManager 自动计算起始和结束状态并插值。
- **Lottie**：播放 After Effects 导出的 JSON 动画，适合复杂矢量动画、加载动效等。

## 属性动画 (ObjectAnimator)

ObjectAnimator 通过反射调用目标对象的 setter，持续更新属性值。支持 alpha、translationX/Y、scaleX/Y、rotation 等 View 常用属性。可设置 duration、interpolator、repeatCount 等。`start()` 启动动画，`cancel()` 取消。注意：目标属性必须有对应的 getter/setter，或通过 `Property` 自定义。

```kotlin
val animator = ObjectAnimator.ofFloat(view, "alpha", 0f, 1f)
animator.duration = 300
animator.interpolator = DecelerateInterpolator()
animator.start()
```

## ValueAnimator

ValueAnimator 不直接操作对象，仅产生插值后的数值，通过 `addUpdateListener` 将数值应用到目标。适用于需要同时控制多个属性、或动画非标准属性的场景。`ofFloat`、`ofInt`、`ofObject` 支持不同类型，`ofArgb` 可动画颜色。

```kotlin
ValueAnimator.ofFloat(0f, 1f).apply {
    duration = 500
    addUpdateListener { animator ->
        val value = animator.animatedValue as Float
        view.translationY = value * 100
    }
    start()
}
```

## 过渡动画 (Transition)

Transition 用于布局变化时的平滑过渡。`beginDelayedTransition` 记录当前状态，随后对 View 的修改（如 visibility、位置）会被捕获为结束状态，Transition 自动计算并播放动画。支持 Fade、Slide、Explode、ChangeBounds 等，可组合使用。适用于 Fragment 切换、展开/收起、列表项增删等场景。

```kotlin
val transition = Fade()
transition.duration = 300
TransitionManager.beginDelayedTransition(container, transition)
targetView.visibility = View.VISIBLE
```

## Lottie 动画

Lottie 解析 JSON 格式的动画文件，在 Android 上原生渲染。设计师在 After Effects 中制作动画，通过 Bodymovin 插件导出 JSON，即可在应用中播放。适合加载动画、空状态插画、引导动画等。支持循环、速度控制、颜色替换等。注意 JSON 文件体积，复杂动画可能较大，可按需加载或使用网络资源。

```kotlin
dependencies {
    implementation("com.airbnb.android:lottie:6.2.0")
}
```

```xml
<com.airbnb.lottie.LottieAnimationView
    android:id="@+id/animation_view"
    android:layout_width="200dp"
    android:layout_height="200dp"
    app:lottie_rawRes="@raw/loading_animation"
    app:lottie_loop="true" />
```

## 总结

根据场景选择合适的动画方式：属性动画适合自定义属性变化和简单动效；过渡动画适合布局变化；Lottie 适合复杂矢量动画。注意动画时长不宜过长（通常 200–500ms），避免过度动画影响性能和体验。Compose 项目可使用 `animate*AsState`、`AnimatedVisibility` 等 Compose 动画 API。
