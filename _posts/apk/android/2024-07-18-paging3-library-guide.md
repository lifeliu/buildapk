---
layout: post
title: Android Paging 3 分页加载完全指南
categories: android
tags: [Paging3, 分页, RecyclerView, Android, Jetpack]
date: 2024/7/18 14:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_035.jpg)

## 引言

Paging 3 是 Android Jetpack 的分页库，用于高效加载和显示分页数据。与手动实现分页逻辑相比，Paging 3 内置加载状态（加载中、错误、无更多数据）、重试机制、预加载和边界回调，大幅减少样板代码。它支持从网络、数据库或两者结合（RemoteMediator）加载数据，与 RecyclerView、Flow、LiveData 无缝集成。本文将深入探讨 Paging 3 的 PagingSource、Pager 配置、LoadState 处理以及 RemoteMediator 的用法。

## 为什么使用 Paging 3

手动分页需要管理页码、加载状态、重试、列表与加载状态的 UI 同步等，代码冗长且易出错。Paging 3 将"何时加载"、"加载多少"、"如何展示状态"抽象为可配置的组件，开发者只需实现数据源（PagingSource 或 RemoteMediator）和 UI 绑定。Paging 3 基于 Kotlin 协程和 Flow 设计，与 ViewModel、Repository 的现代架构自然契合。内置的 LoadState 可方便地展示加载中、错误、重试等状态，提升用户体验。

## 添加依赖

```kotlin
dependencies {
    implementation("androidx.paging:paging-runtime-ktx:3.2.1")
}
```

## PagingSource 实现

PagingSource 是数据源抽象，根据 `LoadParams`（包含 key、loadSize 等）加载一页数据，返回 `LoadResult.Page` 或 `LoadResult.Error`。`key` 是上一页/下一页的标识，通常为页码；`prevKey` 和 `nextKey` 用于告诉 Paging 如何加载上一页和下一页，为 null 表示没有更多数据。网络分页时，根据 API 的页码或游标约定实现；数据库分页可使用 `PagingSource` 的 Room 集成（Room 可自动生成 PagingSource）。

```kotlin
class ArticlePagingSource(
    private val api: ApiService
) : PagingSource<Int, Article>() {
    
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        return try {
            val page = params.key ?: 1
            val response = api.getArticles(page, params.loadSize)
            
            LoadResult.Page(
                data = response.articles,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.hasMore) page + 1 else null
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```

## 创建 Pager

Pager 是 Paging 3 的入口，接收 `PagingConfig`（pageSize、enablePlaceholders、initialLoadSize 等）和 `pagingSourceFactory`。`flow` 属性返回 `Flow<PagingData>`，可在 ViewModel 中暴露给 UI 层。`enablePlaceholders` 为 true 时，未加载的项会占位，适合固定尺寸的列表；为 false 时列表长度随加载动态变化，更常见。

```kotlin
val articles: Flow<PagingData<Article>> = Pager(
    config = PagingConfig(
        pageSize = 20,
        enablePlaceholders = false
    ),
    pagingSourceFactory = { ArticlePagingSource(api) }
).flow
```

## 在 RecyclerView 中使用

使用 `PagingDataAdapter` 替代普通 Adapter，通过 `submitData` 提交 PagingData。`withLoadStateFooter` 可在列表底部添加加载状态（加载中、错误、重试）。收集 Flow 时使用 `repeatOnLifecycle(Lifecycle.State.STARTED)` 确保仅在界面可见时收集，避免泄漏和后台执行。`adapter.loadStateFlow` 可观察整体加载状态，用于显示全局加载指示器或错误提示。

```kotlin
@AndroidEntryPoint
class ArticleFragment : Fragment() {
    
    private val viewModel: ArticleViewModel by viewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        val adapter = ArticleAdapter()
        binding.recyclerView.adapter = adapter.withLoadStateFooter(
            footer = ArticleLoadStateAdapter { adapter.retry() }
        )
        
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.articles.collect { pagingData ->
                    adapter.submitData(pagingData)
                }
            }
        }
    }
}
```

## RemoteMediator 网络+数据库

RemoteMediator 用于"网络 + 本地数据库"的分页：本地数据库作为单一数据源，RemoteMediator 在需要时从网络加载并写入数据库，Paging 从数据库读取。适用于离线优先、缓存优先的场景。`load` 根据 `LoadType`（REFRESH、PREPEND、APPEND）决定加载策略，返回 `MediatorResult.Success` 或 `MediatorResult.Error`。需配合 Room 的 `RemoteMediator` 和 `PagingSource` 使用，配置稍复杂，但能实现完整的分页缓存方案。

```kotlin
@ExperimentalPagingApi
class ArticleRemoteMediator(
    private val database: AppDatabase,
    private val api: ApiService
) : RemoteMediator<Int, Article>() {
    
    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, Article>
    ): MediatorResult {
        return try {
            val page = when (loadType) {
                LoadType.REFRESH -> 1
                LoadType.PREPEND -> return MediatorResult.Success(true)
                LoadType.APPEND -> state.lastItemOrNull()?.nextKey ?: return MediatorResult.Success(false)
            }
            
            val response = api.getArticles(page, state.config.pageSize)
            database.articleDao().insertAll(response.articles)
            
            MediatorResult.Success(endOfPaginationReached = !response.hasMore)
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

## 总结

Paging 3 简化了分页数据的加载和展示，是处理列表类数据的最佳选择。掌握 PagingSource、LoadState 和 RemoteMediator，可以构建高效、用户友好的分页列表，并支持网络与本地缓存的组合策略。
