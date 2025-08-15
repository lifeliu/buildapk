---
layout: post
title: Android 安全开发深入指南
categories: android
tags: [安全开发, Android, 加密, 权限管理, 代码混淆]
date: 2024/12/15 17:30:00
---

![title](https://image.sideproject.cn/titlex/titlex_security.jpg)

## 引言

随着移动应用的普及和用户对隐私保护意识的增强，Android 应用的安全性变得越来越重要。恶意攻击、数据泄露、逆向工程等安全威胁层出不穷，开发者必须在应用设计和开发的各个阶段都考虑安全因素。Android 平台提供了多层次的安全机制，从系统级的权限控制到应用级的数据加密，从网络通信安全到代码混淆保护。本文将深入探讨 Android 安全开发的核心技术和最佳实践，帮助开发者构建更加安全可靠的移动应用，保护用户数据和应用资产不受恶意攻击。

## 数据加密与存储安全

### 对称加密实现

```kotlin
class SymmetricEncryption {
    
    companion object {
        private const val TRANSFORMATION = "AES/GCM/NoPadding"
        private const val ANDROID_KEYSTORE = "AndroidKeyStore"
        private const val KEY_ALIAS = "MySecretKey"
        private const val IV_LENGTH = 12
    }
    
    private val keyStore: KeyStore = KeyStore.getInstance(ANDROID_KEYSTORE).apply {
        load(null)
    }
    
    // 生成密钥
    @RequiresApi(Build.VERSION_CODES.M)
    fun generateSecretKey(keyAlias: String = KEY_ALIAS) {
        val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, ANDROID_KEYSTORE)
        
        val keyGenParameterSpec = KeyGenParameterSpec.Builder(
            keyAlias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setUserAuthenticationRequired(false)
            .setRandomizedEncryptionRequired(true)
            .build()
        
        keyGenerator.init(keyGenParameterSpec)
        keyGenerator.generateKey()
    }
    
    // 获取密钥
    private fun getSecretKey(keyAlias: String = KEY_ALIAS): SecretKey {
        return keyStore.getKey(keyAlias, null) as SecretKey
    }
    
    // 加密数据
    fun encrypt(data: String, keyAlias: String = KEY_ALIAS): EncryptedData? {
        return try {
            val secretKey = getSecretKey(keyAlias)
            val cipher = Cipher.getInstance(TRANSFORMATION)
            cipher.init(Cipher.ENCRYPT_MODE, secretKey)
            
            val iv = cipher.iv
            val encryptedBytes = cipher.doFinal(data.toByteArray(Charsets.UTF_8))
            
            EncryptedData(
                encryptedData = Base64.encodeToString(encryptedBytes, Base64.DEFAULT),
                iv = Base64.encodeToString(iv, Base64.DEFAULT)
            )
        } catch (e: Exception) {
            Log.e("SymmetricEncryption", "Encryption failed", e)
            null
        }
    }
    
    // 解密数据
    fun decrypt(encryptedData: EncryptedData, keyAlias: String = KEY_ALIAS): String? {
        return try {
            val secretKey = getSecretKey(keyAlias)
            val cipher = Cipher.getInstance(TRANSFORMATION)
            
            val iv = Base64.decode(encryptedData.iv, Base64.DEFAULT)
            val gcmSpec = GCMParameterSpec(128, iv)
            
            cipher.init(Cipher.DECRYPT_MODE, secretKey, gcmSpec)
            
            val encryptedBytes = Base64.decode(encryptedData.encryptedData, Base64.DEFAULT)
            val decryptedBytes = cipher.doFinal(encryptedBytes)
            
            String(decryptedBytes, Charsets.UTF_8)
        } catch (e: Exception) {
            Log.e("SymmetricEncryption", "Decryption failed", e)
            null
        }
    }
    
    // 删除密钥
    fun deleteKey(keyAlias: String = KEY_ALIAS) {
        try {
            keyStore.deleteEntry(keyAlias)
        } catch (e: Exception) {
            Log.e("SymmetricEncryption", "Failed to delete key", e)
        }
    }
    
    // 检查密钥是否存在
    fun keyExists(keyAlias: String = KEY_ALIAS): Boolean {
        return try {
            keyStore.containsAlias(keyAlias)
        } catch (e: Exception) {
            false
        }
    }
    
    data class EncryptedData(
        val encryptedData: String,
        val iv: String
    )
}

// 安全的 SharedPreferences 实现
class SecureSharedPreferences(private val context: Context) {
    
    private val encryption = SymmetricEncryption()
    private val sharedPreferences = context.getSharedPreferences("secure_prefs", Context.MODE_PRIVATE)
    
    init {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && !encryption.keyExists()) {
            encryption.generateSecretKey()
        }
    }
    
    // 存储加密字符串
    fun putString(key: String, value: String) {
        val encryptedData = encryption.encrypt(value)
        if (encryptedData != null) {
            sharedPreferences.edit()
                .putString("${key}_data", encryptedData.encryptedData)
                .putString("${key}_iv", encryptedData.iv)
                .apply()
        }
    }
    
    // 获取解密字符串
    fun getString(key: String, defaultValue: String? = null): String? {
        val encryptedData = sharedPreferences.getString("${key}_data", null)
        val iv = sharedPreferences.getString("${key}_iv", null)
        
        return if (encryptedData != null && iv != null) {
            encryption.decrypt(SymmetricEncryption.EncryptedData(encryptedData, iv)) ?: defaultValue
        } else {
            defaultValue
        }
    }
    
    // 存储加密整数
    fun putInt(key: String, value: Int) {
        putString(key, value.toString())
    }
    
    // 获取解密整数
    fun getInt(key: String, defaultValue: Int = 0): Int {
        return getString(key)?.toIntOrNull() ?: defaultValue
    }
    
    // 存储加密布尔值
    fun putBoolean(key: String, value: Boolean) {
        putString(key, value.toString())
    }
    
    // 获取解密布尔值
    fun getBoolean(key: String, defaultValue: Boolean = false): Boolean {
        return getString(key)?.toBooleanStrictOrNull() ?: defaultValue
    }
    
    // 删除数据
    fun remove(key: String) {
        sharedPreferences.edit()
            .remove("${key}_data")
            .remove("${key}_iv")
            .apply()
    }
    
    // 清空所有数据
    fun clear() {
        sharedPreferences.edit().clear().apply()
    }
    
    // 检查键是否存在
    fun contains(key: String): Boolean {
        return sharedPreferences.contains("${key}_data") && sharedPreferences.contains("${key}_iv")
    }
}
```

### 文件加密存储

```kotlin
class SecureFileManager(private val context: Context) {
    
    private val encryption = SymmetricEncryption()
    
    init {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && !encryption.keyExists()) {
            encryption.generateSecretKey()
        }
    }
    
    // 加密并保存文件
    fun saveEncryptedFile(fileName: String, data: String): Boolean {
        return try {
            val encryptedData = encryption.encrypt(data)
            if (encryptedData != null) {
                val file = File(context.filesDir, fileName)
                val fileData = JSONObject().apply {
                    put("data", encryptedData.encryptedData)
                    put("iv", encryptedData.iv)
                    put("timestamp", System.currentTimeMillis())
                }
                
                FileOutputStream(file).use { fos ->
                    fos.write(fileData.toString().toByteArray())
                }
                true
            } else {
                false
            }
        } catch (e: Exception) {
            Log.e("SecureFileManager", "Failed to save encrypted file", e)
            false
        }
    }
    
    // 读取并解密文件
    fun readEncryptedFile(fileName: String): String? {
        return try {
            val file = File(context.filesDir, fileName)
            if (!file.exists()) {
                return null
            }
            
            val fileContent = file.readText()
            val jsonObject = JSONObject(fileContent)
            
            val encryptedData = SymmetricEncryption.EncryptedData(
                encryptedData = jsonObject.getString("data"),
                iv = jsonObject.getString("iv")
            )
            
            encryption.decrypt(encryptedData)
        } catch (e: Exception) {
            Log.e("SecureFileManager", "Failed to read encrypted file", e)
            null
        }
    }
    
    // 加密并保存二进制文件
    fun saveEncryptedBinaryFile(fileName: String, data: ByteArray): Boolean {
        return try {
            val base64Data = Base64.encodeToString(data, Base64.DEFAULT)
            saveEncryptedFile(fileName, base64Data)
        } catch (e: Exception) {
            Log.e("SecureFileManager", "Failed to save encrypted binary file", e)
            false
        }
    }
    
    // 读取并解密二进制文件
    fun readEncryptedBinaryFile(fileName: String): ByteArray? {
        return try {
            val base64Data = readEncryptedFile(fileName)
            if (base64Data != null) {
                Base64.decode(base64Data, Base64.DEFAULT)
            } else {
                null
            }
        } catch (e: Exception) {
            Log.e("SecureFileManager", "Failed to read encrypted binary file", e)
            null
        }
    }
    
    // 删除加密文件
    fun deleteEncryptedFile(fileName: String): Boolean {
        return try {
            val file = File(context.filesDir, fileName)
            file.delete()
        } catch (e: Exception) {
            Log.e("SecureFileManager", "Failed to delete encrypted file", e)
            false
        }
    }
    
    // 列出所有加密文件
    fun listEncryptedFiles(): List<String> {
        return try {
            context.filesDir.listFiles()?.map { it.name } ?: emptyList()
        } catch (e: Exception) {
            Log.e("SecureFileManager", "Failed to list encrypted files", e)
            emptyList()
        }
    }
    
    // 获取文件信息
    fun getFileInfo(fileName: String): FileInfo? {
        return try {
            val file = File(context.filesDir, fileName)
            if (!file.exists()) {
                return null
            }
            
            val fileContent = file.readText()
            val jsonObject = JSONObject(fileContent)
            
            FileInfo(
                fileName = fileName,
                size = file.length(),
                lastModified = file.lastModified(),
                timestamp = jsonObject.optLong("timestamp", 0)
            )
        } catch (e: Exception) {
            Log.e("SecureFileManager", "Failed to get file info", e)
            null
        }
    }
    
    data class FileInfo(
        val fileName: String,
        val size: Long,
        val lastModified: Long,
        val timestamp: Long
    )
}
```

## 网络安全

### HTTPS 和证书固定

```kotlin
class SecureNetworkManager {
    
    companion object {
        private const val CONNECT_TIMEOUT = 30L
        private const val READ_TIMEOUT = 30L
        private const val WRITE_TIMEOUT = 30L
    }
    
    // 创建安全的 OkHttpClient
    fun createSecureOkHttpClient(
        certificatePins: List<CertificatePin> = emptyList(),
        enableLogging: Boolean = false
    ): OkHttpClient {
        val builder = OkHttpClient.Builder()
            .connectTimeout(CONNECT_TIMEOUT, TimeUnit.SECONDS)
            .readTimeout(READ_TIMEOUT, TimeUnit.SECONDS)
            .writeTimeout(WRITE_TIMEOUT, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
        
        // 添加证书固定
        if (certificatePins.isNotEmpty()) {
            val certificatePinner = CertificatePinner.Builder().apply {
                certificatePins.forEach { pin ->
                    add(pin.hostname, pin.pin)
                }
            }.build()
            
            builder.certificatePinner(certificatePinner)
        }
        
        // 添加安全拦截器
        builder.addInterceptor(SecurityInterceptor())
        
        // 添加请求签名拦截器
        builder.addInterceptor(RequestSignatureInterceptor())
        
        // 添加日志拦截器（仅在调试模式下）
        if (enableLogging && BuildConfig.DEBUG) {
            val loggingInterceptor = HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
                redactHeader("Authorization")
                redactHeader("Cookie")
            }
            builder.addInterceptor(loggingInterceptor)
        }
        
        return builder.build()
    }
    
    data class CertificatePin(
        val hostname: String,
        val pin: String
    )
    
    // 安全拦截器
    private class SecurityInterceptor : Interceptor {
        override fun intercept(chain: Interceptor.Chain): Response {
            val request = chain.request()
            
            // 检查是否使用 HTTPS
            if (request.url.scheme != "https") {
                throw SecurityException("Only HTTPS connections are allowed")
            }
            
            // 添加安全头
            val secureRequest = request.newBuilder()
                .addHeader("X-Requested-With", "XMLHttpRequest")
                .addHeader("Cache-Control", "no-cache")
                .addHeader("Pragma", "no-cache")
                .build()
            
            val response = chain.proceed(secureRequest)
            
            // 验证响应头
            validateSecurityHeaders(response)
            
            return response
        }
        
        private fun validateSecurityHeaders(response: Response) {
            // 检查 Content-Type
            val contentType = response.header("Content-Type")
            if (contentType?.contains("application/json") == true ||
                contentType?.contains("application/xml") == true
            ) {
                // 验证内容类型
            }
            
            // 检查安全头
            val securityHeaders = listOf(
                "Strict-Transport-Security",
                "X-Content-Type-Options",
                "X-Frame-Options",
                "X-XSS-Protection"
            )
            
            securityHeaders.forEach { header ->
                if (response.header(header) == null) {
                    Log.w("SecurityInterceptor", "Missing security header: $header")
                }
            }
        }
    }
    
    // 请求签名拦截器
    private class RequestSignatureInterceptor : Interceptor {
        
        private val apiKey = "your_api_key"
        private val secretKey = "your_secret_key"
        
        override fun intercept(chain: Interceptor.Chain): Response {
            val originalRequest = chain.request()
            
            // 生成时间戳和随机数
            val timestamp = System.currentTimeMillis().toString()
            val nonce = UUID.randomUUID().toString()
            
            // 生成签名
            val signature = generateSignature(originalRequest, timestamp, nonce)
            
            // 添加认证头
            val signedRequest = originalRequest.newBuilder()
                .addHeader("X-API-Key", apiKey)
                .addHeader("X-Timestamp", timestamp)
                .addHeader("X-Nonce", nonce)
                .addHeader("X-Signature", signature)
                .build()
            
            return chain.proceed(signedRequest)
        }
        
        private fun generateSignature(request: Request, timestamp: String, nonce: String): String {
            val method = request.method
            val url = request.url.toString()
            val body = request.body?.let { requestBody ->
                val buffer = Buffer()
                requestBody.writeTo(buffer)
                buffer.readUtf8()
            } ?: ""
            
            val signatureData = "$method\n$url\n$body\n$timestamp\n$nonce\n$secretKey"
            
            return try {
                val mac = Mac.getInstance("HmacSHA256")
                val secretKeySpec = SecretKeySpec(secretKey.toByteArray(), "HmacSHA256")
                mac.init(secretKeySpec)
                val hash = mac.doFinal(signatureData.toByteArray())
                Base64.encodeToString(hash, Base64.NO_WRAP)
            } catch (e: Exception) {
                Log.e("RequestSignatureInterceptor", "Failed to generate signature", e)
                ""
            }
        }
    }
}

// 安全的 API 客户端
class SecureApiClient {
    
    private val networkManager = SecureNetworkManager()
    private val gson = Gson()
    
    // 证书固定配置
    private val certificatePins = listOf(
        SecureNetworkManager.CertificatePin(
            hostname = "api.example.com",
            pin = "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
        )
    )
    
    private val okHttpClient = networkManager.createSecureOkHttpClient(
        certificatePins = certificatePins,
        enableLogging = BuildConfig.DEBUG
    )
    
    private val retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create(gson))
        .build()
    
    // API 接口
    interface ApiService {
        @GET("users/{id}")
        suspend fun getUser(@Path("id") userId: String): Response<UserResponse>
        
        @POST("users")
        suspend fun createUser(@Body user: CreateUserRequest): Response<UserResponse>
        
        @PUT("users/{id}")
        suspend fun updateUser(
            @Path("id") userId: String,
            @Body user: UpdateUserRequest
        ): Response<UserResponse>
        
        @DELETE("users/{id}")
        suspend fun deleteUser(@Path("id") userId: String): Response<Unit>
    }
    
    private val apiService = retrofit.create(ApiService::class.java)
    
    // 安全的 API 调用包装器
    suspend fun <T> safeApiCall(apiCall: suspend () -> Response<T>): ApiResult<T> {
        return try {
            val response = apiCall()
            if (response.isSuccessful) {
                val body = response.body()
                if (body != null) {
                    ApiResult.Success(body)
                } else {
                    ApiResult.Error("Empty response body")
                }
            } else {
                val errorBody = response.errorBody()?.string()
                ApiResult.Error("HTTP ${response.code()}: $errorBody")
            }
        } catch (e: Exception) {
            Log.e("SecureApiClient", "API call failed", e)
            when (e) {
                is IOException -> ApiResult.Error("Network error: ${e.message}")
                is HttpException -> ApiResult.Error("HTTP error: ${e.message}")
                else -> ApiResult.Error("Unknown error: ${e.message}")
            }
        }
    }
    
    // API 方法
    suspend fun getUser(userId: String): ApiResult<UserResponse> {
        return safeApiCall { apiService.getUser(userId) }
    }
    
    suspend fun createUser(user: CreateUserRequest): ApiResult<UserResponse> {
        return safeApiCall { apiService.createUser(user) }
    }
    
    suspend fun updateUser(userId: String, user: UpdateUserRequest): ApiResult<UserResponse> {
        return safeApiCall { apiService.updateUser(userId, user) }
    }
    
    suspend fun deleteUser(userId: String): ApiResult<Unit> {
        return safeApiCall { apiService.deleteUser(userId) }
    }
    
    // API 结果封装
    sealed class ApiResult<out T> {
        data class Success<out T>(val data: T) : ApiResult<T>()
        data class Error(val message: String) : ApiResult<Nothing>()
    }
    
    // 数据类
    data class UserResponse(
        val id: String,
        val name: String,
        val email: String,
        val createdAt: String
    )
    
    data class CreateUserRequest(
        val name: String,
        val email: String,
        val password: String
    )
    
    data class UpdateUserRequest(
        val name: String?,
        val email: String?
    )
}
```

## 身份验证与授权

### JWT Token 管理

```kotlin
class JwtTokenManager(private val context: Context) {
    
    private val securePrefs = SecureSharedPreferences(context)
    
    companion object {
        private const val ACCESS_TOKEN_KEY = "access_token"
        private const val REFRESH_TOKEN_KEY = "refresh_token"
        private const val TOKEN_EXPIRY_KEY = "token_expiry"
        private const val USER_ID_KEY = "user_id"
    }
    
    // 保存 Token
    fun saveTokens(accessToken: String, refreshToken: String, expiresIn: Long) {
        val expiryTime = System.currentTimeMillis() + (expiresIn * 1000)
        
        securePrefs.putString(ACCESS_TOKEN_KEY, accessToken)
        securePrefs.putString(REFRESH_TOKEN_KEY, refreshToken)
        securePrefs.putString(TOKEN_EXPIRY_KEY, expiryTime.toString())
        
        // 解析并保存用户信息
        val userInfo = parseJwtToken(accessToken)
        userInfo?.let {
            securePrefs.putString(USER_ID_KEY, it.userId)
        }
    }
    
    // 获取访问令牌
    fun getAccessToken(): String? {
        return securePrefs.getString(ACCESS_TOKEN_KEY)
    }
    
    // 获取刷新令牌
    fun getRefreshToken(): String? {
        return securePrefs.getString(REFRESH_TOKEN_KEY)
    }
    
    // 检查 Token 是否有效
    fun isTokenValid(): Boolean {
        val accessToken = getAccessToken()
        val expiryTimeStr = securePrefs.getString(TOKEN_EXPIRY_KEY)
        
        if (accessToken.isNullOrEmpty() || expiryTimeStr.isNullOrEmpty()) {
            return false
        }
        
        val expiryTime = expiryTimeStr.toLongOrNull() ?: 0
        val currentTime = System.currentTimeMillis()
        
        // 提前 5 分钟判断过期
        return currentTime < (expiryTime - 5 * 60 * 1000)
    }
    
    // 检查是否需要刷新 Token
    fun shouldRefreshToken(): Boolean {
        val expiryTimeStr = securePrefs.getString(TOKEN_EXPIRY_KEY)
        if (expiryTimeStr.isNullOrEmpty()) {
            return false
        }
        
        val expiryTime = expiryTimeStr.toLongOrNull() ?: 0
        val currentTime = System.currentTimeMillis()
        
        // 在过期前 10 分钟开始刷新
        return currentTime > (expiryTime - 10 * 60 * 1000)
    }
    
    // 清除 Token
    fun clearTokens() {
        securePrefs.remove(ACCESS_TOKEN_KEY)
        securePrefs.remove(REFRESH_TOKEN_KEY)
        securePrefs.remove(TOKEN_EXPIRY_KEY)
        securePrefs.remove(USER_ID_KEY)
    }
    
    // 获取用户 ID
    fun getUserId(): String? {
        return securePrefs.getString(USER_ID_KEY)
    }
    
    // 解析 JWT Token
    private fun parseJwtToken(token: String): JwtPayload? {
        return try {
            val parts = token.split(".")
            if (parts.size != 3) {
                return null
            }
            
            val payload = parts[1]
            val decodedBytes = Base64.decode(payload, Base64.URL_SAFE)
            val decodedString = String(decodedBytes, Charsets.UTF_8)
            
            val jsonObject = JSONObject(decodedString)
            
            JwtPayload(
                userId = jsonObject.optString("sub"),
                email = jsonObject.optString("email"),
                exp = jsonObject.optLong("exp"),
                iat = jsonObject.optLong("iat")
            )
        } catch (e: Exception) {
            Log.e("JwtTokenManager", "Failed to parse JWT token", e)
            null
        }
    }
    
    data class JwtPayload(
        val userId: String,
        val email: String,
        val exp: Long,
        val iat: Long
    )
}

// 生物识别认证管理器
class BiometricAuthManager(private val context: Context) {
    
    interface BiometricAuthCallback {
        fun onAuthenticationSucceeded()
        fun onAuthenticationFailed()
        fun onAuthenticationError(errorCode: Int, errorMessage: String)
    }
    
    // 检查生物识别是否可用
    fun isBiometricAvailable(): Boolean {
        return when (BiometricManager.from(context).canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_WEAK)) {
            BiometricManager.BIOMETRIC_SUCCESS -> true
            BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> false
            BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> false
            BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> false
            else -> false
        }
    }
    
    // 启动生物识别认证
    fun authenticate(
        activity: FragmentActivity,
        title: String,
        subtitle: String,
        description: String,
        callback: BiometricAuthCallback
    ) {
        if (!isBiometricAvailable()) {
            callback.onAuthenticationError(-1, "Biometric authentication not available")
            return
        }
        
        val biometricPrompt = BiometricPrompt(
            activity,
            ContextCompat.getMainExecutor(context),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    super.onAuthenticationSucceeded(result)
                    callback.onAuthenticationSucceeded()
                }
                
                override fun onAuthenticationFailed() {
                    super.onAuthenticationFailed()
                    callback.onAuthenticationFailed()
                }
                
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    super.onAuthenticationError(errorCode, errString)
                    callback.onAuthenticationError(errorCode, errString.toString())
                }
            }
        )
        
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle(title)
            .setSubtitle(subtitle)
            .setDescription(description)
            .setNegativeButtonText("Cancel")
            .build()
        
        biometricPrompt.authenticate(promptInfo)
    }
    
    // 使用加密的生物识别认证
    fun authenticateWithCrypto(
        activity: FragmentActivity,
        title: String,
        subtitle: String,
        description: String,
        cryptoObject: BiometricPrompt.CryptoObject,
        callback: BiometricAuthCallback
    ) {
        if (!isBiometricAvailable()) {
            callback.onAuthenticationError(-1, "Biometric authentication not available")
            return
        }
        
        val biometricPrompt = BiometricPrompt(
            activity,
            ContextCompat.getMainExecutor(context),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    super.onAuthenticationSucceeded(result)
                    // 可以使用 result.cryptoObject 进行加密操作
                    callback.onAuthenticationSucceeded()
                }
                
                override fun onAuthenticationFailed() {
                    super.onAuthenticationFailed()
                    callback.onAuthenticationFailed()
                }
                
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    super.onAuthenticationError(errorCode, errString)
                    callback.onAuthenticationError(errorCode, errString.toString())
                }
            }
        )
        
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle(title)
            .setSubtitle(subtitle)
            .setDescription(description)
            .setNegativeButtonText("Cancel")
            .build()
        
        biometricPrompt.authenticate(promptInfo, cryptoObject)
    }
}
```

## 代码混淆与反调试

### ProGuard 配置

```proguard
# 基本混淆配置
-dontusemixedcaseclassnames
-dontskipnonpubliclibraryclasses
-verbose

# 优化配置
-optimizations !code/simplification/arithmetic,!code/simplification/cast,!field/*,!class/merging/*
-optimizationpasses 5
-allowaccessmodification

# 保留注解
-keepattributes *Annotation*
-keepattributes Signature
-keepattributes InnerClasses
-keepattributes EnclosingMethod

# 保留行号信息（用于调试）
-keepattributes SourceFile,LineNumberTable

# 保留 Android 组件
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.app.backup.BackupAgentHelper
-keep public class * extends android.preference.Preference

# 保留 View 构造函数
-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet);
}
-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet, int);
}

# 保留枚举
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# 保留 Parcelable
-keepclassmembers class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator CREATOR;
}

# 保留 Serializable
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}

# 保留 Native 方法
-keepclasseswithmembernames class * {
    native <methods>;
}

# 保留反射使用的类和方法
-keep class com.yourpackage.model.** { *; }
-keep class com.yourpackage.api.** { *; }

# Retrofit 配置
-keepattributes Signature, InnerClasses, EnclosingMethod
-keepattributes RuntimeVisibleAnnotations, RuntimeVisibleParameterAnnotations
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}
-dontwarn org.codehaus.mojo.animal_sniffer.IgnoreJRERequirement
-dontwarn javax.annotation.**
-dontwarn kotlin.Unit
-dontwarn retrofit2.KotlinExtensions
-dontwarn retrofit2.KotlinExtensions$*

# OkHttp 配置
-dontwarn okhttp3.**
-dontwarn okio.**
-dontwarn javax.annotation.**
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase

# Gson 配置
-keepattributes Signature
-keepattributes *Annotation*
-dontwarn sun.misc.**
-keep class com.google.gson.** { *; }
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer

# 自定义混淆规则
-keep class com.yourpackage.security.** { *; }
-keep class com.yourpackage.crypto.** { *; }

# 字符串加密
-adaptclassstrings
-adaptresourcefilenames **.properties,**.xml,**.json
-adaptresourcefilecontents **.properties,META-INF/MANIFEST.MF
```

### 反调试检测

```kotlin
class AntiDebuggingManager {
    
    companion object {
        private const val TAG = "AntiDebuggingManager"
        
        // 加载 native 库
        init {
            try {
                System.loadLibrary("antidebug")
            } catch (e: UnsatisfiedLinkError) {
                Log.w(TAG, "Native anti-debug library not found")
            }
        }
    }
    
    // 检测调试器
    fun detectDebugger(): Boolean {
        return isDebuggerConnected() || 
               isBeingDebugged() || 
               checkDebuggerNative() ||
               detectEmulator() ||
               checkRootAccess()
    }
    
    // 检测调试器连接
    private fun isDebuggerConnected(): Boolean {
        return Debug.isDebuggerConnected()
    }
    
    // 检测应用是否被调试
    private fun isBeingDebugged(): Boolean {
        return try {
            val context = getApplicationContext()
            val applicationInfo = context.applicationInfo
            (applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE) != 0
        } catch (e: Exception) {
            false
        }
    }
    
    // Native 层调试检测
    private external fun checkDebuggerNative(): Boolean
    
    // 检测模拟器
    private fun detectEmulator(): Boolean {
        return checkEmulatorFiles() ||
               checkEmulatorProperties() ||
               checkEmulatorFeatures()
    }
    
    // 检测模拟器文件
    private fun checkEmulatorFiles(): Boolean {
        val emulatorFiles = arrayOf(
            "/system/lib/libc_malloc_debug_qemu.so",
            "/sys/qemu_trace",
            "/system/bin/qemu-props",
            "/dev/socket/qemud",
            "/dev/qemu_pipe",
            "/dev/socket/baseband_genyd",
            "/dev/socket/genyd"
        )
        
        return emulatorFiles.any { File(it).exists() }
    }
    
    // 检测模拟器属性
    private fun checkEmulatorProperties(): Boolean {
        val emulatorProps = mapOf(
            "ro.product.model" to listOf("sdk", "emulator", "Android SDK built for x86"),
            "ro.product.manufacturer" to listOf("Genymotion", "unknown"),
            "ro.product.device" to listOf("generic", "generic_x86", "vbox86p"),
            "ro.product.brand" to listOf("generic", "generic_x86"),
            "ro.board.platform" to listOf("android-x86")
        )
        
        return emulatorProps.any { (key, values) ->
            val prop = getSystemProperty(key)
            values.any { prop.contains(it, ignoreCase = true) }
        }
    }
    
    // 检测模拟器特征
    private fun checkEmulatorFeatures(): Boolean {
        // 检测电话功能
        val context = getApplicationContext()
        val telephonyManager = context.getSystemService(Context.TELEPHONY_SERVICE) as TelephonyManager
        
        val deviceId = try {
            telephonyManager.deviceId
        } catch (e: SecurityException) {
            null
        }
        
        // 模拟器通常返回固定的设备 ID
        val emulatorDeviceIds = listOf(
            "000000000000000",
            "e21833235b6eef10",
            "012345678912345"
        )
        
        return deviceId in emulatorDeviceIds
    }
    
    // 检测 Root 权限
    private fun checkRootAccess(): Boolean {
        return checkRootBinary() || checkRootPackages() || checkSuCommand()
    }
    
    // 检测 Root 二进制文件
    private fun checkRootBinary(): Boolean {
        val rootPaths = arrayOf(
            "/system/app/Superuser.apk",
            "/sbin/su",
            "/system/bin/su",
            "/system/xbin/su",
            "/data/local/xbin/su",
            "/data/local/bin/su",
            "/system/sd/xbin/su",
            "/system/bin/failsafe/su",
            "/data/local/su",
            "/su/bin/su"
        )
        
        return rootPaths.any { File(it).exists() }
    }
    
    // 检测 Root 应用包
    private fun checkRootPackages(): Boolean {
        val context = getApplicationContext()
        val packageManager = context.packageManager
        
        val rootPackages = arrayOf(
            "com.noshufou.android.su",
            "com.noshufou.android.su.elite",
            "eu.chainfire.supersu",
            "com.koushikdutta.superuser",
            "com.thirdparty.superuser",
            "com.yellowes.su",
            "com.topjohnwu.magisk",
            "com.kingroot.kinguser",
            "com.kingo.root",
            "com.smedialink.oneclickroot",
            "com.zhiqupk.root.global",
            "com.alephzain.framaroot"
        )
        
        return rootPackages.any { packageName ->
            try {
                packageManager.getPackageInfo(packageName, 0)
                true
            } catch (e: PackageManager.NameNotFoundException) {
                false
            }
        }
    }
    
    // 检测 su 命令
    private fun checkSuCommand(): Boolean {
        return try {
            val process = Runtime.getRuntime().exec(arrayOf("which", "su"))
            val reader = BufferedReader(InputStreamReader(process.inputStream))
            val result = reader.readLine()
            reader.close()
            process.waitFor()
            !result.isNullOrEmpty()
        } catch (e: Exception) {
            false
        }
    }
    
    // 获取系统属性
    private fun getSystemProperty(key: String): String {
        return try {
            val systemProperties = Class.forName("android.os.SystemProperties")
            val getMethod = systemProperties.getMethod("get", String::class.java)
            getMethod.invoke(null, key) as String
        } catch (e: Exception) {
            ""
        }
    }
    
    // 获取应用上下文（需要在实际使用时提供）
    private fun getApplicationContext(): Context {
        // 这里需要传入实际的 Context
        throw NotImplementedError("Context must be provided")
    }
    
    // 反调试响应
    fun handleDebuggingDetected() {
        // 可以采取以下措施：
        // 1. 退出应用
        // 2. 清除敏感数据
        // 3. 发送警告到服务器
        // 4. 显示警告信息
        
        Log.w(TAG, "Debugging or tampering detected!")
        
        // 清除敏感数据
        clearSensitiveData()
        
        // 退出应用
        exitProcess(0)
    }
    
    // 清除敏感数据
    private fun clearSensitiveData() {
        try {
            // 清除 SharedPreferences
            val context = getApplicationContext()
            val sharedPrefs = context.getSharedPreferences("secure_prefs", Context.MODE_PRIVATE)
            sharedPrefs.edit().clear().apply()
            
            // 清除缓存文件
            context.cacheDir.deleteRecursively()
            
            // 清除内部存储文件
            context.filesDir.listFiles()?.forEach { file ->
                if (file.name.contains("sensitive")) {
                    file.delete()
                }
            }
        } catch (e: Exception) {
            Log.e(TAG, "Failed to clear sensitive data", e)
        }
    }
}
```

## 权限管理

### 动态权限请求

```kotlin
class PermissionManager(private val activity: Activity) {
    
    interface PermissionCallback {
        fun onPermissionGranted(permissions: List<String>)
        fun onPermissionDenied(permissions: List<String>)
        fun onPermissionPermanentlyDenied(permissions: List<String>)
    }
    
    private var permissionCallback: PermissionCallback? = null
    private val requestCode = 1001
    
    // 检查权限
    fun hasPermissions(permissions: Array<String>): Boolean {
        return permissions.all { permission ->
            ContextCompat.checkSelfPermission(activity, permission) == PackageManager.PERMISSION_GRANTED
        }
    }
    
    // 请求权限
    fun requestPermissions(
        permissions: Array<String>,
        callback: PermissionCallback
    ) {
        this.permissionCallback = callback
        
        val deniedPermissions = permissions.filter { permission ->
            ContextCompat.checkSelfPermission(activity, permission) != PackageManager.PERMISSION_GRANTED
        }
        
        if (deniedPermissions.isEmpty()) {
            callback.onPermissionGranted(permissions.toList())
            return
        }
        
        // 检查是否需要显示权限说明
        val shouldShowRationale = deniedPermissions.any { permission ->
            ActivityCompat.shouldShowRequestPermissionRationale(activity, permission)
        }
        
        if (shouldShowRationale) {
            showPermissionRationale(deniedPermissions.toTypedArray()) {
                ActivityCompat.requestPermissions(activity, deniedPermissions.toTypedArray(), requestCode)
            }
        } else {
            ActivityCompat.requestPermissions(activity, deniedPermissions.toTypedArray(), requestCode)
        }
    }
    
    // 处理权限请求结果
    fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        if (requestCode != this.requestCode) {
            return
        }
        
        val grantedPermissions = mutableListOf<String>()
        val deniedPermissions = mutableListOf<String>()
        val permanentlyDeniedPermissions = mutableListOf<String>()
        
        for (i in permissions.indices) {
            val permission = permissions[i]
            val grantResult = grantResults[i]
            
            if (grantResult == PackageManager.PERMISSION_GRANTED) {
                grantedPermissions.add(permission)
            } else {
                if (ActivityCompat.shouldShowRequestPermissionRationale(activity, permission)) {
                    deniedPermissions.add(permission)
                } else {
                    permanentlyDeniedPermissions.add(permission)
                }
            }
        }
        
        when {
            grantedPermissions.isNotEmpty() && deniedPermissions.isEmpty() && permanentlyDeniedPermissions.isEmpty() -> {
                permissionCallback?.onPermissionGranted(grantedPermissions)
            }
            permanentlyDeniedPermissions.isNotEmpty() -> {
                permissionCallback?.onPermissionPermanentlyDenied(permanentlyDeniedPermissions)
            }
            else -> {
                permissionCallback?.onPermissionDenied(deniedPermissions)
            }
        }
    }
    
    // 显示权限说明
    private fun showPermissionRationale(
        permissions: Array<String>,
        onPositive: () -> Unit
    ) {
        val permissionNames = permissions.map { getPermissionName(it) }
        val message = "This app needs the following permissions to function properly:\n\n" +
                permissionNames.joinToString("\n") { "• $it" }
        
        AlertDialog.Builder(activity)
            .setTitle("Permissions Required")
            .setMessage(message)
            .setPositiveButton("Grant") { _, _ -> onPositive() }
            .setNegativeButton("Cancel") { dialog, _ ->
                dialog.dismiss()
                permissionCallback?.onPermissionDenied(permissions.toList())
            }
            .setCancelable(false)
            .show()
    }
    
    // 显示设置页面
    fun showAppSettings() {
        AlertDialog.Builder(activity)
            .setTitle("Permission Required")
            .setMessage("Some permissions have been permanently denied. Please enable them in app settings.")
            .setPositiveButton("Settings") { _, _ ->
                val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
                val uri = Uri.fromParts("package", activity.packageName, null)
                intent.data = uri
                activity.startActivity(intent)
            }
            .setNegativeButton("Cancel") { dialog, _ -> dialog.dismiss() }
            .show()
    }
    
    // 获取权限名称
    private fun getPermissionName(permission: String): String {
        return when (permission) {
            Manifest.permission.CAMERA -> "Camera"
            Manifest.permission.RECORD_AUDIO -> "Microphone"
            Manifest.permission.READ_EXTERNAL_STORAGE -> "Storage (Read)"
            Manifest.permission.WRITE_EXTERNAL_STORAGE -> "Storage (Write)"
            Manifest.permission.ACCESS_FINE_LOCATION -> "Location (Precise)"
            Manifest.permission.ACCESS_COARSE_LOCATION -> "Location (Approximate)"
            Manifest.permission.READ_CONTACTS -> "Contacts (Read)"
            Manifest.permission.WRITE_CONTACTS -> "Contacts (Write)"
            Manifest.permission.READ_PHONE_STATE -> "Phone State"
            Manifest.permission.CALL_PHONE -> "Phone Calls"
            Manifest.permission.SEND_SMS -> "SMS (Send)"
            Manifest.permission.READ_SMS -> "SMS (Read)"
            else -> permission.substringAfterLast(".")
        }
    }
    
    // 常用权限组合
    companion object {
        val CAMERA_PERMISSIONS = arrayOf(
            Manifest.permission.CAMERA
        )
        
        val STORAGE_PERMISSIONS = arrayOf(
            Manifest.permission.READ_EXTERNAL_STORAGE,
            Manifest.permission.WRITE_EXTERNAL_STORAGE
        )
        
        val LOCATION_PERMISSIONS = arrayOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        )
        
        val AUDIO_PERMISSIONS = arrayOf(
            Manifest.permission.RECORD_AUDIO
        )
        
        val CONTACTS_PERMISSIONS = arrayOf(
            Manifest.permission.READ_CONTACTS,
            Manifest.permission.WRITE_CONTACTS
        )
    }
}

// 权限管理扩展函数
fun Activity.requestPermissions(
    permissions: Array<String>,
    callback: PermissionManager.PermissionCallback
) {
    val permissionManager = PermissionManager(this)
    permissionManager.requestPermissions(permissions, callback)
}

fun Activity.hasPermissions(permissions: Array<String>): Boolean {
    val permissionManager = PermissionManager(this)
    return permissionManager.hasPermissions(permissions)
}
```

## 总结

Android 安全开发是一个复杂而重要的主题，涉及多个层面的安全防护。通过本文的深入探讨，我们学习了：

1. **数据加密与存储安全**：对称加密、安全的 SharedPreferences、文件加密存储
2. **网络安全**：HTTPS 配置、证书固定、请求签名、安全的 API 客户端
3. **身份验证与授权**：JWT Token 管理、生物识别认证
4. **代码混淆与反调试**：ProGuard 配置、反调试检测、防篡改措施
5. **权限管理**：动态权限请求、权限说明、用户体验优化

在实际开发中，安全防护需要从设计阶段就开始考虑，采用多层防护策略：

**数据保护**：敏感数据加密存储，传输过程使用 HTTPS，避免在日志中输出敏感信息。

**身份验证**：实现强身份验证机制，使用安全的 Token 管理，支持多因素认证。

**代码保护**：使用代码混淆，实现反调试检测，防止逆向工程和动态分析。

**权限控制**：遵循最小权限原则，合理请求和使用权限，保护用户隐私。

**安全测试**：定期进行安全测试，包括静态代码分析、动态安全测试、渗透测试等。

通过系统学习和实践这些安全开发技术，开发者能够构建更加安全可靠的 Android 应用，有效保护用户数据和应用资产，提升用户信任度和应用竞争力。安全是一个持续的过程，需要随着技术发展和威胁变化不断更新和完善安全防护措施。