---
layout: post
title: Android NDK 与 JNI 原生开发指南
categories: android
tags: [NDK, JNI, 原生开发, C++, Android]
date: 2024/10/8 14:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_039.jpg)

## 引言

Android NDK（Native Development Kit）允许开发者使用 C/C++ 编写应用的部分代码，通过 JNI（Java Native Interface）与 Kotlin/Java 层交互。NDK 适用于高性能计算（如图像处理、音视频编解码）、复用现有 C/C++ 库、或访问底层硬件能力。本文介绍 NDK 和 JNI 的配置、Native 方法声明、C++ 实现、CMake 构建以及常见注意事项。

## 为什么使用 NDK

纯 Kotlin/Java 在某些计算密集型场景下性能不足，C/C++ 可提供更高效的计算能力。许多成熟的库（如 FFmpeg、OpenCV）仅提供 C/C++ API，通过 NDK 可集成到 Android 应用。此外，对性能敏感的算法（如加密、压缩）用 C/C++ 实现可显著提升速度。但 NDK 开发复杂度较高，调试困难，且增加 APK 体积和兼容性负担，应仅在确有需要时使用。

## 配置 NDK

在 `android.defaultConfig` 中指定 `ndkVersion`，确保团队使用相同 NDK 版本。`abiFilters` 限制生成的 native 库架构，通常包含 `arm64-v8a`（主流 64 位设备）和 `x86_64`（模拟器）。减少 abi 可减小 APK 体积，但会失去对部分设备的支持。使用 `externalNativeBuild` 和 `cmake` 配置 CMake 路径，或使用 `ndk-build` 与 Android.mk。

```kotlin
// build.gradle.kts
android {
    ndkVersion = "25.2.9519653"
    
    defaultConfig {
        ndk {
            abiFilters += listOf("armeabi-v7a", "arm64-v8a", "x86_64")
        }
    }
}
```

## 声明 Native 方法

在 Kotlin/Java 类中声明 `external` 方法，方法名、参数和返回值会映射到 C/C++ 的 JNI 函数。`System.loadLibrary("native-lib")` 加载编译生成的 `libnative-lib.so`，需在调用 native 方法前执行，通常在静态块中。JNI 函数名由包名、类名和方法名组成，格式为 `Java_包名_类名_方法名`，包名中的点替换为下划线。

```kotlin
class NativeLib {
    companion object {
        init {
            System.loadLibrary("native-lib")
        }
    }
    
    external fun stringFromJNI(): String
    external fun processData(input: ByteArray): ByteArray
}
```

## C++ 实现

JNI 函数签名需与 Java 层一致，使用 `JNIEXPORT` 和 `JNICALL` 宏。`JNIEnv*` 提供访问 Java 对象、调用 Java 方法、异常处理等能力。`jobject` 是调用该方法的 Java 对象（静态方法时为 `jclass`）。处理 `jbyteArray` 等引用类型时，需通过 `GetByteArrayElements` 获取指针，使用完毕后 `ReleaseByteArrayElements` 释放，避免内存泄漏。注意线程：JNIEnv 与线程绑定，在子线程中需通过 `AttachCurrentThread` 获取 Env。

```cpp
// native-lib.cpp
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_NativeLib_stringFromJNI(JNIEnv* env, jobject) {
    return env->NewStringUTF("Hello from C++");
}

extern "C" JNIEXPORT jbyteArray JNICALL
Java_com_example_NativeLib_processData(JNIEnv* env, jobject, jbyteArray input) {
    jsize length = env->GetArrayLength(input);
    jbyte* bytes = env->GetByteArrayElements(input, nullptr);
    
    // 处理数据...
    
    env->ReleaseByteArrayElements(input, bytes, 0);
    return env->NewByteArray(length);
}
```

## CMakeLists.txt

CMake 定义 native 库的构建规则。`add_library` 指定源文件和输出库名，`find_library` 查找系统库（如 log），`target_link_libraries` 链接依赖。Android Studio 创建的 NDK 项目通常已包含 CMake 配置，按需修改即可。

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("native-lib")

add_library(native-lib SHARED native-lib.cpp)
find_library(log-lib log)
target_link_libraries(native-lib ${log-lib})
```

## 总结

NDK 和 JNI 为 Android 开发提供了原生性能扩展能力，适用于计算密集型或需要复用 C/C++ 代码的场景。掌握 JNI 函数命名、引用类型管理和线程安全，可以正确实现 Java 与 Native 的互操作。注意 APK 体积、多 ABI 支持和崩溃调试的复杂性，合理评估使用 NDK 的必要性。
