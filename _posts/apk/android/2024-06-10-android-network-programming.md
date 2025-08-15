---
layout: post
title: Android 网络编程深入指南
categories: android
tags: [网络编程, Android, HTTP, OkHttp, Retrofit]
date: 2024/6/10 08:50:00
---

![title](https://image.sideproject.cn/titlex/titlex_network.jpg)

## 引言

网络编程是现代 Android 应用开发中不可或缺的重要组成部分。随着移动互联网的快速发展，几乎所有的应用都需要与服务器进行数据交互，获取实时信息、同步用户数据或提供在线服务。Android 提供了丰富的网络编程 API 和第三方库，使开发者能够高效地实现各种网络功能。本文将深入探讨 Android 网络编程的核心概念、最佳实践和高级技巧。

## 网络编程基础

### HTTP 协议基础

HTTP（HyperText Transfer Protocol）是现代网络应用的基础协议。在 Android 开发中，我们主要使用 HTTP/HTTPS 协议进行网络通信。

```kotlin
// HTTP 请求方法
enum class HttpMethod {
    GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
}

// HTTP 状态码
object HttpStatusCode {
    const val OK = 200
    const val CREATED = 201
    const val NO_CONTENT = 204
    const val BAD_REQUEST = 400
    const val UNAUTHORIZED = 401
    const val FORBIDDEN = 403
    const val NOT_FOUND = 404
    const val INTERNAL_SERVER_ERROR = 500
    const val SERVICE_UNAVAILABLE = 503
}

// 基础网络请求数据类
data class NetworkRequest(
    val url: String,
    val method: HttpMethod = HttpMethod.GET,
    val headers: Map<String, String> = emptyMap(),
    val body: String? = null,
    val timeout: Long = 30000L
)

data class NetworkResponse(
    val statusCode: Int,
    val headers: Map<String, List<String>>,
    val body: String?,
    val isSuccessful: Boolean = statusCode in 200..299
)
```

### 原生 HttpURLConnection

```kotlin
class HttpURLConnectionClient {
    
    suspend fun executeRequest(request: NetworkRequest): Result<NetworkResponse> {
        return withContext(Dispatchers.IO) {
            try {
                val url = URL(request.url)
                val connection = url.openConnection() as HttpURLConnection
                
                // 配置连接
                connection.apply {
                    requestMethod = request.method.name
                    connectTimeout = request.timeout.toInt()
                    readTimeout = request.timeout.toInt()
                    doInput = true
                    
                    // 设置请求头
                    request.headers.forEach { (key, value) ->
                        setRequestProperty(key, value)
                    }
                    
                    // 处理请求体
                    if (request.body != null && request.method in listOf(HttpMethod.POST, HttpMethod.PUT, HttpMethod.PATCH)) {
                        doOutput = true
                        setRequestProperty("Content-Type", "application/json; charset=UTF-8")
                        
                        outputStream.use { output ->
                            output.write(request.body.toByteArray(Charsets.UTF_8))
                            output.flush()
                        }
                    }
                }
                
                // 执行请求并处理响应
                val statusCode = connection.responseCode
                val headers = connection.headerFields
                
                val responseBody = if (statusCode in 200..299) {
                    connection.inputStream.use { input ->
                        input.bufferedReader().use { reader ->
                            reader.readText()
                        }
                    }
                } else {
                    connection.errorStream?.use { error ->
                        error.bufferedReader().use { reader ->
                            reader.readText()
                        }
                    }
                }
                
                connection.disconnect()
                
                Result.success(
                    NetworkResponse(
                        statusCode = statusCode,
                        headers = headers,
                        body = responseBody
                    )
                )
                
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
    
    // GET 请求
    suspend fun get(
        url: String,
        headers: Map<String, String> = emptyMap()
    ): Result<NetworkResponse> {
        return executeRequest(
            NetworkRequest(
                url = url,
                method = HttpMethod.GET,
                headers = headers
            )
        )
    }
    
    // POST 请求
    suspend fun post(
        url: String,
        body: String,
        headers: Map<String, String> = emptyMap()
    ): Result<NetworkResponse> {
        return executeRequest(
            NetworkRequest(
                url = url,
                method = HttpMethod.POST,
                headers = headers,
                body = body
            )
        )
    }
    
    // 文件下载
    suspend fun downloadFile(
        url: String,
        outputFile: File,
        progressCallback: ((bytesRead: Long, totalBytes: Long) -> Unit)? = null
    ): Result<File> {
        return withContext(Dispatchers.IO) {
            try {
                val connection = URL(url).openConnection() as HttpURLConnection
                connection.connectTimeout = 30000
                connection.readTimeout = 30000
                
                val totalBytes = connection.contentLengthLong
                var bytesRead = 0L
                
                connection.inputStream.use { input ->
                    outputFile.outputStream().use { output ->
                        val buffer = ByteArray(8192)
                        var bytes: Int
                        
                        while (input.read(buffer).also { bytes = it } != -1) {
                            output.write(buffer, 0, bytes)
                            bytesRead += bytes
                            progressCallback?.invoke(bytesRead, totalBytes)
                        }
                    }
                }
                
                connection.disconnect()
                Result.success(outputFile)
                
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
}
```

## OkHttp 网络库

### OkHttp 基础使用

```kotlin
class OkHttpNetworkClient {
    
    private val client = OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .addInterceptor(LoggingInterceptor())
        .addInterceptor(AuthInterceptor())
        .addNetworkInterceptor(CacheInterceptor())
        .cache(createCache())
        .build()
    
    private fun createCache(): Cache {
        val cacheDir = File(context.cacheDir, "http_cache")
        val cacheSize = 10 * 1024 * 1024L // 10MB
        return Cache(cacheDir, cacheSize)
    }
    
    // 同步请求
    fun executeSync(request: Request): Response {
        return client.newCall(request).execute()
    }
    
    // 异步请求
    fun executeAsync(request: Request, callback: Callback) {
        client.newCall(request).enqueue(callback)
    }
    
    // 协程支持
    suspend fun execute(request: Request): Result<Response> {
        return withContext(Dispatchers.IO) {
            try {
                val response = client.newCall(request).execute()
                Result.success(response)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
    
    // GET 请求
    suspend fun get(
        url: String,
        headers: Map<String, String> = emptyMap()
    ): Result<String> {
        val requestBuilder = Request.Builder().url(url)
        headers.forEach { (key, value) ->
            requestBuilder.addHeader(key, value)
        }
        
        return execute(requestBuilder.build()).mapCatching { response ->
            response.body?.string() ?: ""
        }
    }
    
    // POST JSON 请求
    suspend fun postJson(
        url: String,
        json: String,
        headers: Map<String, String> = emptyMap()
    ): Result<String> {
        val mediaType = "application/json; charset=utf-8".toMediaType()
        val requestBody = json.toRequestBody(mediaType)
        
        val requestBuilder = Request.Builder()
            .url(url)
            .post(requestBody)
        
        headers.forEach { (key, value) ->
            requestBuilder.addHeader(key, value)
        }
        
        return execute(requestBuilder.build()).mapCatching { response ->
            response.body?.string() ?: ""
        }
    }
    
    // 文件上传
    suspend fun uploadFile(
        url: String,
        file: File,
        fieldName: String = "file",
        additionalFields: Map<String, String> = emptyMap(),
        progressCallback: ((bytesWritten: Long, totalBytes: Long) -> Unit)? = null
    ): Result<String> {
        val requestBody = MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .apply {
                // 添加文件
                val fileBody = file.asRequestBody("application/octet-stream".toMediaType())
                val progressRequestBody = ProgressRequestBody(fileBody, progressCallback)
                addFormDataPart(fieldName, file.name, progressRequestBody)
                
                // 添加其他字段
                additionalFields.forEach { (key, value) ->
                    addFormDataPart(key, value)
                }
            }
            .build()
        
        val request = Request.Builder()
            .url(url)
            .post(requestBody)
            .build()
        
        return execute(request).mapCatching { response ->
            response.body?.string() ?: ""
        }
    }
    
    // 文件下载
    suspend fun downloadFile(
        url: String,
        outputFile: File,
        progressCallback: ((bytesRead: Long, totalBytes: Long) -> Unit)? = null
    ): Result<File> {
        return withContext(Dispatchers.IO) {
            try {
                val request = Request.Builder().url(url).build()
                val response = client.newCall(request).execute()
                
                if (!response.isSuccessful) {
                    throw IOException("Download failed: ${response.code}")
                }
                
                val body = response.body ?: throw IOException("Response body is null")
                val totalBytes = body.contentLength()
                var bytesRead = 0L
                
                body.byteStream().use { input ->
                    outputFile.outputStream().use { output ->
                        val buffer = ByteArray(8192)
                        var bytes: Int
                        
                        while (input.read(buffer).also { bytes = it } != -1) {
                            output.write(buffer, 0, bytes)
                            bytesRead += bytes
                            progressCallback?.invoke(bytesRead, totalBytes)
                        }
                    }
                }
                
                Result.success(outputFile)
                
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
}

// 进度监听的 RequestBody
class ProgressRequestBody(
    private val delegate: RequestBody,
    private val progressCallback: ((bytesWritten: Long, totalBytes: Long) -> Unit)?
) : RequestBody() {
    
    override fun contentType(): MediaType? = delegate.contentType()
    
    override fun contentLength(): Long = delegate.contentLength()
    
    override fun writeTo(sink: BufferedSink) {
        val totalBytes = contentLength()
        val progressSink = ProgressSink(sink, totalBytes, progressCallback)
        val bufferedSink = progressSink.buffer()
        
        delegate.writeTo(bufferedSink)
        bufferedSink.flush()
    }
}

class ProgressSink(
    private val delegate: Sink,
    private val totalBytes: Long,
    private val progressCallback: ((bytesWritten: Long, totalBytes: Long) -> Unit)?
) : ForwardingSink(delegate) {
    
    private var bytesWritten = 0L
    
    override fun write(source: Buffer, byteCount: Long) {
        super.write(source, byteCount)
        bytesWritten += byteCount
        progressCallback?.invoke(bytesWritten, totalBytes)
    }
}
```

### 拦截器实现

```kotlin
// 日志拦截器
class LoggingInterceptor : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val startTime = System.nanoTime()
        
        Log.d("HTTP", "Sending request ${request.url} on ${chain.connection()}")
        Log.d("HTTP", "Headers: ${request.headers}")
        
        val response = chain.proceed(request)
        val endTime = System.nanoTime()
        
        Log.d("HTTP", "Received response for ${response.request.url} in ${(endTime - startTime) / 1e6}ms")
        Log.d("HTTP", "Response code: ${response.code}")
        Log.d("HTTP", "Response headers: ${response.headers}")
        
        return response
    }
}

// 认证拦截器
class AuthInterceptor : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        
        // 添加认证头
        val token = getAuthToken()
        val authenticatedRequest = if (token != null) {
            originalRequest.newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        } else {
            originalRequest
        }
        
        val response = chain.proceed(authenticatedRequest)
        
        // 处理 401 未授权响应
        if (response.code == 401) {
            response.close()
            
            // 尝试刷新 token
            val newToken = refreshAuthToken()
            if (newToken != null) {
                val newRequest = originalRequest.newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
                return chain.proceed(newRequest)
            }
        }
        
        return response
    }
    
    private fun getAuthToken(): String? {
        // 从 SharedPreferences 或其他存储中获取 token
        return null
    }
    
    private fun refreshAuthToken(): String? {
        // 刷新 token 的逻辑
        return null
    }
}

// 缓存拦截器
class CacheInterceptor : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        // 检查网络状态
        val isNetworkAvailable = isNetworkAvailable()
        
        val modifiedRequest = if (!isNetworkAvailable) {
            // 无网络时强制使用缓存
            request.newBuilder()
                .cacheControl(
                    CacheControl.Builder()
                        .onlyIfCached()
                        .maxStale(7, TimeUnit.DAYS)
                        .build()
                )
                .build()
        } else {
            request
        }
        
        val response = chain.proceed(modifiedRequest)
        
        return if (isNetworkAvailable) {
            // 有网络时设置缓存策略
            response.newBuilder()
                .header("Cache-Control", "public, max-age=300") // 5分钟缓存
                .removeHeader("Pragma")
                .build()
        } else {
            response
        }
    }
    
    private fun isNetworkAvailable(): Boolean {
        // 检查网络连接状态
        return true
    }
}

// 重试拦截器
class RetryInterceptor(private val maxRetries: Int = 3) : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        var request = chain.request()
        var response: Response? = null
        var exception: IOException? = null
        
        for (i in 0..maxRetries) {
            try {
                response?.close()
                response = chain.proceed(request)
                
                if (response.isSuccessful || !shouldRetry(response.code)) {
                    return response
                }
                
            } catch (e: IOException) {
                exception = e
                if (i == maxRetries) {
                    throw e
                }
            }
            
            // 指数退避
            if (i < maxRetries) {
                val delay = (1000 * Math.pow(2.0, i.toDouble())).toLong()
                Thread.sleep(delay)
            }
        }
        
        return response ?: throw (exception ?: IOException("Unknown error"))
    }
    
    private fun shouldRetry(code: Int): Boolean {
        return code in listOf(408, 429, 500, 502, 503, 504)
    }
}
```

## Retrofit 网络框架

### Retrofit 基础配置

```kotlin
// API 接口定义
interface ApiService {
    
    @GET("users/{id}")
    suspend fun getUser(@Path("id") userId: String): Response<User>
    
    @GET("users")
    suspend fun getUsers(
        @Query("page") page: Int,
        @Query("size") size: Int,
        @Query("sort") sort: String? = null
    ): Response<PagedResponse<User>>
    
    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): Response<User>
    
    @PUT("users/{id}")
    suspend fun updateUser(
        @Path("id") userId: String,
        @Body user: UpdateUserRequest
    ): Response<User>
    
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") userId: String): Response<Unit>
    
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part file: MultipartBody.Part,
        @Part("description") description: RequestBody
    ): Response<UploadResponse>
    
    @Streaming
    @GET("download/{fileId}")
    suspend fun downloadFile(@Path("fileId") fileId: String): Response<ResponseBody>
    
    @FormUrlEncoded
    @POST("auth/login")
    suspend fun login(
        @Field("username") username: String,
        @Field("password") password: String
    ): Response<AuthResponse>
    
    @Headers("Cache-Control: max-age=3600")
    @GET("config")
    suspend fun getConfig(): Response<AppConfig>
}

// 数据模型
data class User(
    val id: String,
    val username: String,
    val email: String,
    val fullName: String,
    val avatarUrl: String?,
    val createdAt: String,
    val updatedAt: String
)

data class CreateUserRequest(
    val username: String,
    val email: String,
    val password: String,
    val fullName: String
)

data class UpdateUserRequest(
    val email: String?,
    val fullName: String?
)

data class PagedResponse<T>(
    val data: List<T>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int
)

data class AuthResponse(
    val accessToken: String,
    val refreshToken: String,
    val expiresIn: Long,
    val user: User
)

data class UploadResponse(
    val fileId: String,
    val fileName: String,
    val fileSize: Long,
    val downloadUrl: String
)

data class AppConfig(
    val apiVersion: String,
    val features: Map<String, Boolean>,
    val settings: Map<String, Any>
)
```

### Retrofit 网络层实现

```kotlin
class NetworkModule {
    
    companion object {
        private const val BASE_URL = "https://api.example.com/v1/"
        private const val CONNECT_TIMEOUT = 30L
        private const val READ_TIMEOUT = 30L
        private const val WRITE_TIMEOUT = 30L
    }
    
    @Provides
    @Singleton
    fun provideOkHttpClient(
        loggingInterceptor: HttpLoggingInterceptor,
        authInterceptor: AuthInterceptor,
        cacheInterceptor: CacheInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(CONNECT_TIMEOUT, TimeUnit.SECONDS)
            .readTimeout(READ_TIMEOUT, TimeUnit.SECONDS)
            .writeTimeout(WRITE_TIMEOUT, TimeUnit.SECONDS)
            .addInterceptor(authInterceptor)
            .addInterceptor(cacheInterceptor)
            .addInterceptor(loggingInterceptor)
            .cache(createCache())
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(
        okHttpClient: OkHttpClient,
        gsonConverterFactory: GsonConverterFactory
    ): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(gsonConverterFactory)
            .addCallAdapterFactory(CoroutineCallAdapterFactory())
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
    
    @Provides
    @Singleton
    fun provideGsonConverterFactory(): GsonConverterFactory {
        val gson = GsonBuilder()
            .setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
            .setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES)
            .create()
        
        return GsonConverterFactory.create(gson)
    }
    
    @Provides
    @Singleton
    fun provideHttpLoggingInterceptor(): HttpLoggingInterceptor {
        return HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) {
                HttpLoggingInterceptor.Level.BODY
            } else {
                HttpLoggingInterceptor.Level.NONE
            }
        }
    }
    
    private fun createCache(): Cache {
        val cacheDir = File(context.cacheDir, "http_cache")
        val cacheSize = 10 * 1024 * 1024L // 10MB
        return Cache(cacheDir, cacheSize)
    }
}

// Repository 层实现
class UserRepository(
    private val apiService: ApiService,
    private val userDao: UserDao
) {
    
    suspend fun getUser(userId: String, forceRefresh: Boolean = false): Result<User> {
        return try {
            if (!forceRefresh) {
                // 先尝试从本地数据库获取
                val localUser = userDao.getUserById(userId)
                if (localUser != null && !isDataStale(localUser.updatedAt)) {
                    return Result.success(localUser)
                }
            }
            
            // 从网络获取
            val response = apiService.getUser(userId)
            if (response.isSuccessful) {
                val user = response.body()!!
                userDao.insertUser(user)
                Result.success(user)
            } else {
                Result.failure(HttpException(response))
            }
            
        } catch (e: Exception) {
            // 网络错误时尝试从本地获取
            val localUser = userDao.getUserById(userId)
            if (localUser != null) {
                Result.success(localUser)
            } else {
                Result.failure(e)
            }
        }
    }
    
    suspend fun getUsers(
        page: Int,
        size: Int,
        sort: String? = null
    ): Result<PagedResponse<User>> {
        return try {
            val response = apiService.getUsers(page, size, sort)
            if (response.isSuccessful) {
                val pagedResponse = response.body()!!
                // 缓存用户数据
                userDao.insertUsers(pagedResponse.data)
                Result.success(pagedResponse)
            } else {
                Result.failure(HttpException(response))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun createUser(request: CreateUserRequest): Result<User> {
        return try {
            val response = apiService.createUser(request)
            if (response.isSuccessful) {
                val user = response.body()!!
                userDao.insertUser(user)
                Result.success(user)
            } else {
                Result.failure(HttpException(response))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun updateUser(userId: String, request: UpdateUserRequest): Result<User> {
        return try {
            val response = apiService.updateUser(userId, request)
            if (response.isSuccessful) {
                val user = response.body()!!
                userDao.updateUser(user)
                Result.success(user)
            } else {
                Result.failure(HttpException(response))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun deleteUser(userId: String): Result<Unit> {
        return try {
            val response = apiService.deleteUser(userId)
            if (response.isSuccessful) {
                userDao.deleteUser(userId)
                Result.success(Unit)
            } else {
                Result.failure(HttpException(response))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun uploadAvatar(userId: String, imageFile: File): Result<String> {
        return try {
            val requestFile = imageFile.asRequestBody("image/*".toMediaType())
            val body = MultipartBody.Part.createFormData("avatar", imageFile.name, requestFile)
            val description = "User avatar".toRequestBody("text/plain".toMediaType())
            
            val response = apiService.uploadFile(body, description)
            if (response.isSuccessful) {
                val uploadResponse = response.body()!!
                Result.success(uploadResponse.downloadUrl)
            } else {
                Result.failure(HttpException(response))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    private fun isDataStale(updatedAt: String): Boolean {
        // 检查数据是否过期（例如：超过1小时）
        val updateTime = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", Locale.getDefault())
            .parse(updatedAt)?.time ?: 0
        val currentTime = System.currentTimeMillis()
        return (currentTime - updateTime) > TimeUnit.HOURS.toMillis(1)
    }
}

class HttpException(private val response: Response<*>) : Exception() {
    val code: Int = response.code()
    override val message: String = "HTTP ${response.code()} ${response.message()}"
}
```

## WebSocket 实时通信

### WebSocket 客户端实现

```kotlin
class WebSocketClient(
    private val url: String,
    private val okHttpClient: OkHttpClient
) {
    
    private var webSocket: WebSocket? = null
    private var isConnected = false
    private var reconnectAttempts = 0
    private val maxReconnectAttempts = 5
    private val reconnectDelay = 5000L
    
    private val listeners = mutableSetOf<WebSocketListener>()
    
    interface WebSocketListener {
        fun onConnected()
        fun onDisconnected(code: Int, reason: String)
        fun onMessage(message: String)
        fun onError(throwable: Throwable)
    }
    
    fun connect() {
        if (isConnected) return
        
        val request = Request.Builder()
            .url(url)
            .build()
        
        webSocket = okHttpClient.newWebSocket(request, object : okhttp3.WebSocketListener() {
            override fun onOpen(webSocket: WebSocket, response: Response) {
                isConnected = true
                reconnectAttempts = 0
                notifyConnected()
            }
            
            override fun onMessage(webSocket: WebSocket, text: String) {
                notifyMessage(text)
            }
            
            override fun onMessage(webSocket: WebSocket, bytes: ByteString) {
                notifyMessage(bytes.utf8())
            }
            
            override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
                webSocket.close(1000, null)
            }
            
            override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
                isConnected = false
                notifyDisconnected(code, reason)
                
                // 自动重连
                if (reconnectAttempts < maxReconnectAttempts) {
                    scheduleReconnect()
                }
            }
            
            override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
                isConnected = false
                notifyError(t)
                
                // 自动重连
                if (reconnectAttempts < maxReconnectAttempts) {
                    scheduleReconnect()
                }
            }
        })
    }
    
    fun disconnect() {
        webSocket?.close(1000, "Client disconnect")
        webSocket = null
        isConnected = false
    }
    
    fun sendMessage(message: String): Boolean {
        return webSocket?.send(message) ?: false
    }
    
    fun sendMessage(bytes: ByteString): Boolean {
        return webSocket?.send(bytes) ?: false
    }
    
    fun addListener(listener: WebSocketListener) {
        listeners.add(listener)
    }
    
    fun removeListener(listener: WebSocketListener) {
        listeners.remove(listener)
    }
    
    private fun scheduleReconnect() {
        reconnectAttempts++
        
        Handler(Looper.getMainLooper()).postDelayed({
            if (!isConnected) {
                connect()
            }
        }, reconnectDelay * reconnectAttempts)
    }
    
    private fun notifyConnected() {
        listeners.forEach { it.onConnected() }
    }
    
    private fun notifyDisconnected(code: Int, reason: String) {
        listeners.forEach { it.onDisconnected(code, reason) }
    }
    
    private fun notifyMessage(message: String) {
        listeners.forEach { it.onMessage(message) }
    }
    
    private fun notifyError(throwable: Throwable) {
        listeners.forEach { it.onError(throwable) }
    }
}

// WebSocket 消息处理
class ChatWebSocketManager(
    private val webSocketClient: WebSocketClient,
    private val gson: Gson
) : WebSocketClient.WebSocketListener {
    
    private val messageFlow = MutableSharedFlow<ChatMessage>()
    private val connectionStateFlow = MutableStateFlow(ConnectionState.DISCONNECTED)
    
    enum class ConnectionState {
        CONNECTED, DISCONNECTED, CONNECTING, ERROR
    }
    
    data class ChatMessage(
        val id: String,
        val type: MessageType,
        val content: String,
        val senderId: String,
        val timestamp: Long
    )
    
    enum class MessageType {
        TEXT, IMAGE, FILE, SYSTEM
    }
    
    init {
        webSocketClient.addListener(this)
    }
    
    fun connect() {
        connectionStateFlow.value = ConnectionState.CONNECTING
        webSocketClient.connect()
    }
    
    fun disconnect() {
        webSocketClient.disconnect()
        connectionStateFlow.value = ConnectionState.DISCONNECTED
    }
    
    fun sendMessage(content: String, type: MessageType = MessageType.TEXT) {
        val message = ChatMessage(
            id = UUID.randomUUID().toString(),
            type = type,
            content = content,
            senderId = getCurrentUserId(),
            timestamp = System.currentTimeMillis()
        )
        
        val json = gson.toJson(message)
        webSocketClient.sendMessage(json)
    }
    
    fun observeMessages(): Flow<ChatMessage> = messageFlow.asSharedFlow()
    
    fun observeConnectionState(): Flow<ConnectionState> = connectionStateFlow.asStateFlow()
    
    override fun onConnected() {
        connectionStateFlow.value = ConnectionState.CONNECTED
        
        // 发送认证消息
        sendAuthMessage()
    }
    
    override fun onDisconnected(code: Int, reason: String) {
        connectionStateFlow.value = ConnectionState.DISCONNECTED
    }
    
    override fun onMessage(message: String) {
        try {
            val chatMessage = gson.fromJson(message, ChatMessage::class.java)
            messageFlow.tryEmit(chatMessage)
        } catch (e: Exception) {
            Log.e("WebSocket", "Failed to parse message: $message", e)
        }
    }
    
    override fun onError(throwable: Throwable) {
        connectionStateFlow.value = ConnectionState.ERROR
        Log.e("WebSocket", "WebSocket error", throwable)
    }
    
    private fun sendAuthMessage() {
        val authToken = getAuthToken()
        val authMessage = mapOf(
            "type" to "auth",
            "token" to authToken
        )
        
        val json = gson.toJson(authMessage)
        webSocketClient.sendMessage(json)
    }
    
    private fun getCurrentUserId(): String {
        // 获取当前用户ID
        return "current_user_id"
    }
    
    private fun getAuthToken(): String {
        // 获取认证token
        return "auth_token"
    }
}
```

## 网络安全

### HTTPS 和证书固定

```kotlin
class SecurityConfig {
    
    companion object {
        
        fun createSecureOkHttpClient(context: Context): OkHttpClient {
            return OkHttpClient.Builder()
                .certificatePinner(createCertificatePinner())
                .hostnameVerifier(createHostnameVerifier())
                .sslSocketFactory(
                    createSSLSocketFactory(context),
                    createTrustManager(context)
                )
                .build()
        }
        
        private fun createCertificatePinner(): CertificatePinner {
            return CertificatePinner.Builder()
                .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
                .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
                .build()
        }
        
        private fun createHostnameVerifier(): HostnameVerifier {
            return HostnameVerifier { hostname, session ->
                // 自定义主机名验证逻辑
                HttpsURLConnection.getDefaultHostnameVerifier().verify(hostname, session)
            }
        }
        
        private fun createSSLSocketFactory(context: Context): SSLSocketFactory {
            val trustManagerFactory = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm()
            )
            
            val keyStore = KeyStore.getInstance("BKS")
            context.assets.open("certificates.bks").use { inputStream ->
                keyStore.load(inputStream, "password".toCharArray())
            }
            
            trustManagerFactory.init(keyStore)
            
            val sslContext = SSLContext.getInstance("TLS")
            sslContext.init(null, trustManagerFactory.trustManagers, null)
            
            return sslContext.socketFactory
        }
        
        private fun createTrustManager(context: Context): X509TrustManager {
            val trustManagerFactory = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm()
            )
            
            val keyStore = KeyStore.getInstance("BKS")
            context.assets.open("certificates.bks").use { inputStream ->
                keyStore.load(inputStream, "password".toCharArray())
            }
            
            trustManagerFactory.init(keyStore)
            
            return trustManagerFactory.trustManagers[0] as X509TrustManager
        }
    }
}

// 请求签名
class RequestSigner {
    
    private val secretKey = "your_secret_key"
    
    fun signRequest(request: NetworkRequest): NetworkRequest {
        val timestamp = System.currentTimeMillis().toString()
        val nonce = UUID.randomUUID().toString()
        
        val signatureData = buildString {
            append(request.method.name)
            append(request.url)
            append(request.body ?: "")
            append(timestamp)
            append(nonce)
        }
        
        val signature = generateHMAC(signatureData, secretKey)
        
        val headers = request.headers.toMutableMap().apply {
            put("X-Timestamp", timestamp)
            put("X-Nonce", nonce)
            put("X-Signature", signature)
        }
        
        return request.copy(headers = headers)
    }
    
    private fun generateHMAC(data: String, key: String): String {
        val mac = Mac.getInstance("HmacSHA256")
        val secretKeySpec = SecretKeySpec(key.toByteArray(), "HmacSHA256")
        mac.init(secretKeySpec)
        
        val hashBytes = mac.doFinal(data.toByteArray())
        return Base64.encodeToString(hashBytes, Base64.NO_WRAP)
    }
}
```

## 性能优化

### 网络请求优化

```kotlin
class NetworkOptimizer {
    
    // 请求合并
    class RequestBatcher<T>(
        private val batchSize: Int = 10,
        private val batchTimeout: Long = 100L,
        private val executor: suspend (List<T>) -> List<Any>
    ) {
        
        private val pendingRequests = mutableListOf<BatchRequest<T>>()
        private var batchJob: Job? = null
        
        data class BatchRequest<T>(
            val data: T,
            val deferred: CompletableDeferred<Any>
        )
        
        suspend fun execute(data: T): Any {
            val deferred = CompletableDeferred<Any>()
            val request = BatchRequest(data, deferred)
            
            synchronized(pendingRequests) {
                pendingRequests.add(request)
                
                if (pendingRequests.size >= batchSize) {
                    processBatch()
                } else if (batchJob == null) {
                    batchJob = CoroutineScope(Dispatchers.IO).launch {
                        delay(batchTimeout)
                        synchronized(pendingRequests) {
                            if (pendingRequests.isNotEmpty()) {
                                processBatch()
                            }
                        }
                    }
                }
            }
            
            return deferred.await()
        }
        
        private suspend fun processBatch() {
            val batch = pendingRequests.toList()
            pendingRequests.clear()
            batchJob?.cancel()
            batchJob = null
            
            try {
                val results = executor(batch.map { it.data })
                batch.forEachIndexed { index, request ->
                    request.deferred.complete(results[index])
                }
            } catch (e: Exception) {
                batch.forEach { request ->
                    request.deferred.completeExceptionally(e)
                }
            }
        }
    }
    
    // 连接池优化
    fun createOptimizedOkHttpClient(): OkHttpClient {
        val connectionPool = ConnectionPool(
            maxIdleConnections = 10,
            keepAliveDuration = 5,
            timeUnit = TimeUnit.MINUTES
        )
        
        return OkHttpClient.Builder()
            .connectionPool(connectionPool)
            .protocols(listOf(Protocol.HTTP_2, Protocol.HTTP_1_1))
            .build()
    }
    
    // 请求去重
    class RequestDeduplicator {
        
        private val ongoingRequests = mutableMapOf<String, Deferred<Any>>()
        
        suspend fun <T> execute(key: String, request: suspend () -> T): T {
            val existingRequest = ongoingRequests[key]
            
            return if (existingRequest != null) {
                @Suppress("UNCHECKED_CAST")
                existingRequest.await() as T
            } else {
                val deferred = CoroutineScope(Dispatchers.IO).async {
                    try {
                        request()
                    } finally {
                        ongoingRequests.remove(key)
                    }
                }
                
                ongoingRequests[key] = deferred
                deferred.await()
            }
        }
    }
}
```

## 总结

Android 网络编程是移动应用开发的核心技能之一。通过本文的深入探讨，我们学习了：

1. **基础概念**：HTTP 协议、原生 HttpURLConnection 的使用
2. **OkHttp 框架**：强大的网络库及其拦截器机制
3. **Retrofit 框架**：类型安全的 REST API 客户端
4. **WebSocket**：实时双向通信的实现
5. **网络安全**：HTTPS、证书固定、请求签名等安全措施
6. **性能优化**：请求合并、连接池、去重等优化技巧

掌握这些技能后，开发者可以构建出高效、安全、可靠的网络层，为用户提供流畅的网络体验。在实际开发中，应该根据具体需求选择合适的网络方案，并始终关注性能、安全性和用户体验。