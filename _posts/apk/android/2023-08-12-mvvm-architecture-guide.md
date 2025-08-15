---
layout: post
title: Android MVVM 架构模式深入解析
categories: android
tags: [MVVM, 架构模式, Android, ViewModel, DataBinding]
date: 2023/8/12 16:45:00
---

![title](https://image.sideproject.cn/titlex/titlex_mvvm.jpg)

## 引言

MVVM（Model-View-ViewModel）是 Android 开发中广泛采用的架构模式，它通过清晰的职责分离和数据绑定机制，帮助开发者构建可维护、可测试的应用程序。随着 Android Architecture Components 的推出，MVVM 模式在 Android 开发中变得更加重要和实用。本文将深入探讨 MVVM 架构模式的核心概念、实现方式以及最佳实践。

## MVVM 架构概述

### 架构组成

MVVM 架构由三个主要组件组成：

1. **Model（模型）**：负责数据管理，包括网络请求、数据库操作、业务逻辑等
2. **View（视图）**：负责 UI 展示，包括 Activity、Fragment、Compose 等
3. **ViewModel（视图模型）**：连接 Model 和 View，管理 UI 相关数据和状态

```kotlin
// Model 层
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatar: String
)

interface UserRepository {
    suspend fun getUsers(): List<User>
    suspend fun getUserById(id: String): User?
    suspend fun updateUser(user: User): Boolean
}

// ViewModel 层
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val _users = MutableLiveData<List<User>>()
    val users: LiveData<List<User>> = _users
    
    private val _isLoading = MutableLiveData<Boolean>()
    val isLoading: LiveData<Boolean> = _isLoading
    
    fun loadUsers() {
        viewModelScope.launch {
            _isLoading.value = true
            try {
                val userList = repository.getUsers()
                _users.value = userList
            } catch (e: Exception) {
                // 处理错误
            } finally {
                _isLoading.value = false
            }
        }
    }
}

// View 层
class UserActivity : AppCompatActivity() {
    private lateinit var viewModel: UserViewModel
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        viewModel = ViewModelProvider(this)[UserViewModel::class.java]
        
        observeViewModel()
        viewModel.loadUsers()
    }
    
    private fun observeViewModel() {
        viewModel.users.observe(this) { users ->
            updateUserList(users)
        }
        
        viewModel.isLoading.observe(this) { isLoading ->
            showLoading(isLoading)
        }
    }
}
```

## 核心组件详解

### 1. ViewModel

ViewModel 是 MVVM 架构的核心，它具有以下特点：

- 在配置更改时保持数据
- 管理 UI 相关的数据
- 不持有 View 的引用
- 具有生命周期感知能力

```kotlin
class ProductViewModel(
    private val productRepository: ProductRepository,
    private val cartRepository: CartRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()
    
    private val _events = Channel<ProductEvent>()
    val events = _events.receiveAsFlow()
    
    fun loadProducts(category: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            try {
                val products = productRepository.getProductsByCategory(category)
                _uiState.update { 
                    it.copy(
                        products = products,
                        isLoading = false,
                        error = null
                    )
                }
            } catch (e: Exception) {
                _uiState.update { 
                    it.copy(
                        isLoading = false,
                        error = e.message
                    )
                }
            }
        }
    }
    
    fun addToCart(product: Product) {
        viewModelScope.launch {
            try {
                cartRepository.addProduct(product)
                _events.send(ProductEvent.ProductAddedToCart(product.name))
            } catch (e: Exception) {
                _events.send(ProductEvent.Error(e.message ?: "Unknown error"))
            }
        }
    }
    
    fun searchProducts(query: String) {
        viewModelScope.launch {
            if (query.isBlank()) {
                loadProducts("all")
                return@launch
            }
            
            _uiState.update { it.copy(isLoading = true) }
            
            try {
                val products = productRepository.searchProducts(query)
                _uiState.update { 
                    it.copy(
                        products = products,
                        isLoading = false
                    )
                }
            } catch (e: Exception) {
                _uiState.update { 
                    it.copy(
                        isLoading = false,
                        error = e.message
                    )
                }
            }
        }
    }
}

data class ProductUiState(
    val products: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

sealed class ProductEvent {
    data class ProductAddedToCart(val productName: String) : ProductEvent()
    data class Error(val message: String) : ProductEvent()
}
```

### 2. Repository 模式

Repository 模式是 MVVM 架构中 Model 层的重要组成部分：

```kotlin
class UserRepositoryImpl(
    private val apiService: ApiService,
    private val userDao: UserDao,
    private val networkManager: NetworkManager
) : UserRepository {
    
    override suspend fun getUsers(): List<User> {
        return if (networkManager.isConnected()) {
            try {
                val users = apiService.getUsers()
                userDao.insertUsers(users)
                users
            } catch (e: Exception) {
                // 网络失败时从本地获取
                userDao.getAllUsers()
            }
        } else {
            userDao.getAllUsers()
        }
    }
    
    override suspend fun getUserById(id: String): User? {
        return userDao.getUserById(id) ?: run {
            if (networkManager.isConnected()) {
                try {
                    val user = apiService.getUserById(id)
                    userDao.insertUser(user)
                    user
                } catch (e: Exception) {
                    null
                }
            } else {
                null
            }
        }
    }
    
    override suspend fun updateUser(user: User): Boolean {
        return try {
            if (networkManager.isConnected()) {
                apiService.updateUser(user)
                userDao.updateUser(user)
                true
            } else {
                // 离线时标记为待同步
                userDao.markForSync(user)
                false
            }
        } catch (e: Exception) {
            false
        }
    }
    
    fun observeUsers(): Flow<List<User>> {
        return userDao.observeUsers()
    }
}
```

### 3. 数据绑定

使用 Data Binding 或 View Binding 简化 View 层代码：

```kotlin
// 使用 View Binding
class UserFragment : Fragment() {
    
    private var _binding: FragmentUserBinding? = null
    private val binding get() = _binding!!
    
    private val viewModel: UserViewModel by viewModels()
    private lateinit var userAdapter: UserAdapter
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentUserBinding.inflate(inflater, container, false)
        return binding.root
    }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        setupRecyclerView()
        observeViewModel()
        setupSwipeRefresh()
        
        viewModel.loadUsers()
    }
    
    private fun setupRecyclerView() {
        userAdapter = UserAdapter { user ->
            viewModel.selectUser(user)
        }
        
        binding.recyclerViewUsers.apply {
            adapter = userAdapter
            layoutManager = LinearLayoutManager(requireContext())
        }
    }
    
    private fun observeViewModel() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                updateUI(state)
            }
        }
        
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.events.collect { event ->
                handleEvent(event)
            }
        }
    }
    
    private fun updateUI(state: UserUiState) {
        binding.apply {
            progressBar.isVisible = state.isLoading
            recyclerViewUsers.isVisible = !state.isLoading && state.error == null
            textViewError.isVisible = state.error != null
            
            state.error?.let {
                textViewError.text = it
            }
            
            userAdapter.submitList(state.users)
        }
    }
    
    private fun setupSwipeRefresh() {
        binding.swipeRefreshLayout.setOnRefreshListener {
            viewModel.refreshUsers()
        }
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

## 状态管理

### 使用 StateFlow 和 SharedFlow

```kotlin
class NewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {
    
    // UI 状态
    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()
    
    // 一次性事件
    private val _events = MutableSharedFlow<NewsEvent>()
    val events: SharedFlow<NewsEvent> = _events.asSharedFlow()
    
    // 用户操作
    private val _userActions = MutableSharedFlow<UserAction>()
    
    init {
        handleUserActions()
        loadNews()
    }
    
    private fun handleUserActions() {
        viewModelScope.launch {
            _userActions.collect { action ->
                when (action) {
                    is UserAction.LoadNews -> loadNews(action.category)
                    is UserAction.RefreshNews -> refreshNews()
                    is UserAction.BookmarkNews -> bookmarkNews(action.newsId)
                    is UserAction.ShareNews -> shareNews(action.news)
                }
            }
        }
    }
    
    fun submitAction(action: UserAction) {
        viewModelScope.launch {
            _userActions.emit(action)
        }
    }
    
    private fun loadNews(category: String = "general") {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            
            newsRepository.getNews(category)
                .catch { exception ->
                    _uiState.update { 
                        it.copy(
                            isLoading = false,
                            error = exception.message
                        )
                    }
                }
                .collect { news ->
                    _uiState.update { 
                        it.copy(
                            news = news,
                            isLoading = false,
                            error = null
                        )
                    }
                }
        }
    }
    
    private fun bookmarkNews(newsId: String) {
        viewModelScope.launch {
            try {
                newsRepository.bookmarkNews(newsId)
                _events.emit(NewsEvent.NewsBookmarked)
            } catch (e: Exception) {
                _events.emit(NewsEvent.Error(e.message ?: "Bookmark failed"))
            }
        }
    }
}

data class NewsUiState(
    val news: List<News> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

sealed class UserAction {
    data class LoadNews(val category: String) : UserAction()
    object RefreshNews : UserAction()
    data class BookmarkNews(val newsId: String) : UserAction()
    data class ShareNews(val news: News) : UserAction()
}

sealed class NewsEvent {
    object NewsBookmarked : NewsEvent()
    data class Error(val message: String) : NewsEvent()
}
```

## 依赖注入

### 使用 Hilt 进行依赖注入

```kotlin
// Application 类
@HiltAndroidApp
class MyApplication : Application()

// Module
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}

@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    
    @Provides
    @Singleton
    fun provideUserRepository(
        apiService: ApiService,
        userDao: UserDao
    ): UserRepository {
        return UserRepositoryImpl(apiService, userDao)
    }
}

// ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    // ViewModel 实现
}

// Activity/Fragment
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Activity 实现
    }
}
```

## 测试策略

### ViewModel 单元测试

```kotlin
class UserViewModelTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    private val mockRepository = mockk<UserRepository>()
    private lateinit var viewModel: UserViewModel
    
    @Before
    fun setup() {
        viewModel = UserViewModel(mockRepository)
    }
    
    @Test
    fun `loadUsers should update uiState with users when repository returns data`() = runTest {
        // Given
        val expectedUsers = listOf(
            User("1", "John Doe", "john@example.com"),
            User("2", "Jane Smith", "jane@example.com")
        )
        coEvery { mockRepository.getUsers() } returns expectedUsers
        
        // When
        viewModel.loadUsers()
        
        // Then
        val uiState = viewModel.uiState.value
        assertEquals(expectedUsers, uiState.users)
        assertFalse(uiState.isLoading)
        assertNull(uiState.error)
    }
    
    @Test
    fun `loadUsers should update uiState with error when repository throws exception`() = runTest {
        // Given
        val errorMessage = "Network error"
        coEvery { mockRepository.getUsers() } throws RuntimeException(errorMessage)
        
        // When
        viewModel.loadUsers()
        
        // Then
        val uiState = viewModel.uiState.value
        assertTrue(uiState.users.isEmpty())
        assertFalse(uiState.isLoading)
        assertEquals(errorMessage, uiState.error)
    }
    
    @Test
    fun `searchUsers should filter users based on query`() = runTest {
        // Given
        val allUsers = listOf(
            User("1", "John Doe", "john@example.com"),
            User("2", "Jane Smith", "jane@example.com"),
            User("3", "Bob Johnson", "bob@example.com")
        )
        val query = "John"
        val expectedUsers = listOf(allUsers[0], allUsers[2])
        
        coEvery { mockRepository.searchUsers(query) } returns expectedUsers
        
        // When
        viewModel.searchUsers(query)
        
        // Then
        val uiState = viewModel.uiState.value
        assertEquals(expectedUsers, uiState.users)
    }
}
```

### Repository 测试

```kotlin
class UserRepositoryTest {
    
    private val mockApiService = mockk<ApiService>()
    private val mockUserDao = mockk<UserDao>()
    private val mockNetworkManager = mockk<NetworkManager>()
    
    private lateinit var repository: UserRepository
    
    @Before
    fun setup() {
        repository = UserRepositoryImpl(mockApiService, mockUserDao, mockNetworkManager)
    }
    
    @Test
    fun `getUsers should return data from API when network is available`() = runTest {
        // Given
        val apiUsers = listOf(User("1", "John", "john@example.com"))
        every { mockNetworkManager.isConnected() } returns true
        coEvery { mockApiService.getUsers() } returns apiUsers
        coEvery { mockUserDao.insertUsers(any()) } just Runs
        
        // When
        val result = repository.getUsers()
        
        // Then
        assertEquals(apiUsers, result)
        coVerify { mockUserDao.insertUsers(apiUsers) }
    }
    
    @Test
    fun `getUsers should return cached data when network is unavailable`() = runTest {
        // Given
        val cachedUsers = listOf(User("1", "John", "john@example.com"))
        every { mockNetworkManager.isConnected() } returns false
        coEvery { mockUserDao.getAllUsers() } returns cachedUsers
        
        // When
        val result = repository.getUsers()
        
        // Then
        assertEquals(cachedUsers, result)
        coVerify(exactly = 0) { mockApiService.getUsers() }
    }
}
```

## 最佳实践

### 1. 单一职责原则

```kotlin
// 好的做法：职责分离
class UserProfileViewModel(
    private val userRepository: UserRepository,
    private val imageRepository: ImageRepository,
    private val analyticsService: AnalyticsService
) : ViewModel() {
    
    fun updateProfile(user: User) {
        viewModelScope.launch {
            try {
                userRepository.updateUser(user)
                analyticsService.trackEvent("profile_updated")
            } catch (e: Exception) {
                // 处理错误
            }
        }
    }
    
    fun uploadAvatar(imageUri: Uri) {
        viewModelScope.launch {
            try {
                val imageUrl = imageRepository.uploadImage(imageUri)
                userRepository.updateUserAvatar(imageUrl)
            } catch (e: Exception) {
                // 处理错误
            }
        }
    }
}
```

### 2. 错误处理

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

class RobustViewModel(
    private val repository: Repository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<Result<List<Data>>>(Result.Loading)
    val uiState: StateFlow<Result<List<Data>>> = _uiState.asStateFlow()
    
    fun loadData() {
        viewModelScope.launch {
            _uiState.value = Result.Loading
            
            try {
                val data = repository.getData()
                _uiState.value = Result.Success(data)
            } catch (e: Exception) {
                _uiState.value = Result.Error(e)
            }
        }
    }
}
```

### 3. 内存泄漏预防

```kotlin
// 避免在 ViewModel 中持有 Context 引用
class BadViewModel(private val context: Context) : ViewModel() {
    // 不好的做法：可能导致内存泄漏
}

// 好的做法：使用 Application Context 或依赖注入
@HiltViewModel
class GoodViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val repository: Repository
) : ViewModel() {
    // 安全的做法
}
```

## 总结

MVVM 架构模式为 Android 开发提供了清晰的代码组织结构和职责分离。通过合理使用 ViewModel、Repository、数据绑定等组件，开发者可以构建出可维护、可测试、可扩展的应用程序。

关键要点包括：

1. **清晰的职责分离**：Model 负责数据，View 负责展示，ViewModel 负责逻辑
2. **生命周期感知**：使用 ViewModel 和 LiveData/StateFlow 处理配置更改
3. **单向数据流**：数据从 Model 流向 View，用户操作从 View 流向 ViewModel
4. **依赖注入**：使用 Hilt 等工具管理依赖关系
5. **全面测试**：为每个层级编写单元测试

掌握 MVVM 架构模式是现代 Android 开发的必备技能，它不仅能提高代码质量，还能显著提升开发效率和团队协作能力。