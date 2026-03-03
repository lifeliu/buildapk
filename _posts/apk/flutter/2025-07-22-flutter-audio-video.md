---
layout: post
title: Flutter 音视频开发完全指南
categories: flutter
tags: [Flutter, 音视频]
date: 2025/7/22 09:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_026.jpg)

## 引言

Flutter 通过 video_player、audioplayers、just_audio 等插件实现音视频播放。video_player 支持网络和本地视频，提供播放、暂停、进度控制、全屏等能力。audioplayers 和 just_audio 支持多种音频格式，just_audio 功能更丰富，支持播放列表、后台播放等。音视频播放涉及平台原生能力，需注意资源释放（dispose）、后台播放权限和平台差异。本文将介绍 video_player 和 audioplayers 的基本用法、控制与常见问题处理。

## video_player

VideoPlayerController 需先 initialize 再使用，initialize 是异步的。controller.value 包含 duration、position、isPlaying 等，可监听 value 变化更新 UI。播放结束后可监听 position 与 duration 判断，或使用 addListener。dispose 必须在 Widget 或 Controller 不再使用时调用，通常放在 State 的 dispose 中。网络视频需确保 URL 可访问，支持 HLS、DASH 等流媒体格式取决于平台。

```dart
final controller = VideoPlayerController.network(url);
await controller.initialize();
controller.play();

// 在 Widget 中
VideoPlayer(controller);
Slider(
  value: controller.value.position.inMilliseconds.toDouble(),
  max: controller.value.duration.inMilliseconds.toDouble(),
  onChanged: (v) => controller.seekTo(Duration(milliseconds: v.toInt())),
);

// 释放
@override
void dispose() {
  controller.dispose();
  super.dispose();
}
```

## audioplayers 与 just_audio

audioplayers 的 AudioPlayer 支持 play、pause、stop、seek。可设置 releaseMode 控制播放完成后的行为。just_audio 提供 AudioSource、ConcatenatingAudioSource 等，支持更复杂的播放场景。后台播放需在 Android 的 AndroidManifest 和 iOS 的 Info.plist 中配置相应权限和能力。

## 总结

音视频播放需注意资源释放和平台兼容性。video_player 适合简单视频场景，复杂需求可考虑 better_player、media_kit 等。音频播放根据需求选择 audioplayers 或 just_audio，注意后台播放的配置。
