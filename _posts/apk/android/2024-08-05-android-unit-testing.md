---
layout: post
title: Android 单元测试完全指南
categories: android
tags: [单元测试, JUnit, Mockito, Android, 测试]
date: 2024/8/5 09:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_036.jpg)

## 引言

单元测试是保证 Android 应用质量的重要手段。Android 单元测试运行在 JVM 上，无需模拟器或真机，执行速度快，适合在每次提交时自动运行。通过针对 ViewModel、Repository、UseCase 等业务逻辑的隔离测试，可以快速发现回归、支撑重构，并作为活文档描述预期行为。本文介绍如何使用 JUnit、Mockito 和 Kotlin 协程测试工具编写 Android 单元测试，以及依赖注入与 Mock 策略。

## 为什么需要单元测试

单元测试在修改代码后能快速反馈是否正确，减少手动回归的工作量。当业务逻辑复杂或多人协作时，测试用例可以明确"正确行为"的定义。良好的测试覆盖率也为重构提供信心。编写可测试的代码往往促使更好的设计：依赖注入、职责单一等原则会自然浮现。Android 单元测试不依赖 Android 框架，可测试纯 Kotlin/Java 逻辑，是性价比最高的测试层级。

## 添加依赖

```kotlin
dependencies {
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.mockito:mockito-core:5.7.0")
    testImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
}
```

Mockito 用于创建 Mock 对象，mockito-kotlin 提供 Kotlin 友好的 API（如 `whenever`）。kotlinx-coroutines-test 提供 `runTest`、`StandardTestDispatcher` 等，用于测试协程代码。

## 基础 JUnit 测试

测试类中的测试方法以 `@Test` 标记，使用 `assertEquals`、`assertTrue`、`assertThrows` 等断言验证结果。`@Before` 在每个测试前执行，用于初始化；`@After` 用于清理。使用 `@testable import` 可访问模块内的 internal 成员。测试方法名应描述场景和预期，如 `addition_shouldReturnCorrectResult` 或使用 Kotlin 反引号 `` `addition should return correct result` ``。

```kotlin
class CalculatorTest {
    
    @Test
    fun `addition should return correct result`() {
        val calculator = Calculator()
        assertEquals(5, calculator.add(2, 3))
    }
    
    @Test
    fun `division by zero should throw exception`() {
        val calculator = Calculator()
        assertThrows(ArithmeticException::class.java) {
            calculator.divide(10, 0)
        }
    }
}
```

## Mockito 模拟依赖

通过 `@Mock` 或 `mock()` 创建 Mock 对象，使用 `whenever(mock.method()).thenReturn(value)` 定义行为。测试时注入 Mock 到被测对象，验证其与依赖的交互是否正确。`verify(mock).method()` 可断言方法被调用的次数和参数。ViewModel 测试时，注入 Mock Repository，设置返回值，调用 ViewModel 方法，断言 LiveData 或 StateFlow 的值是否符合预期。

```kotlin
class UserViewModelTest {
    
    @Mock
    lateinit var repository: UserRepository
    
    private lateinit var viewModel: UserViewModel
    
    @Before
    fun setup() {
        MockitoAnnotations.openMocks(this)
        viewModel = UserViewModel(repository)
    }
    
    @Test
    fun `loadUsers should update LiveData`() = runTest {
        val users = listOf(User("1", "Alice"), User("2", "Bob"))
        whenever(repository.getUsers()).thenReturn(users)
        
        viewModel.loadUsers()
        
        assertEquals(users, viewModel.users.value)
    }
}
```

## 协程测试

测试中使用 `runTest` 创建测试协程作用域，`StandardTestDispatcher` 控制虚拟时间。`Dispatchers.setMain(testDispatcher)` 将 Main 调度器替换为测试调度器，使 ViewModel 中的 `viewModelScope.launch` 在测试中同步执行。`advanceUntilIdle()` 推进虚拟时间直到所有协程完成。测试结束后需 `Dispatchers.resetMain()` 恢复，避免影响其他测试。

```kotlin
@Test
fun `loadData should handle async correctly`() = runTest {
    val testDispatcher = StandardTestDispatcher(testScheduler)
    Dispatchers.setMain(testDispatcher)
    
    val viewModel = ViewModel(repository)
    viewModel.loadData()
    
    advanceUntilIdle()
    
    assertNotNull(viewModel.data.value)
    
    Dispatchers.resetMain()
}
```

## 总结

良好的单元测试能够提高代码质量和可维护性，是 Android 开发中不可或缺的实践。从 ViewModel、Repository 等核心业务逻辑开始编写测试，结合依赖注入和 Mock，可以构建快速、稳定的测试套件，支撑持续集成和重构。
