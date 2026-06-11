# Android Networking & API Layer Guidelines

This guide defines the standardized configurations, architecture patterns, error mapping, and security requirements for the networking and API layer in EDTS Android projects.

---

## 1. OkHttpClient & Retrofit Configuration

All networking requests MUST utilize a single centralized OkHttpClient and Retrofit client provided by the `EdtsKu` dependency graph. Do not construct custom standalone Retrofit builders unless explicitly approved.

### Timeouts and Logging

- **Timeouts**: Connect, read, and write timeouts MUST be configured to **30 seconds** (`30L`).
- **HTTP Logging**: The `HttpLoggingInterceptor` MUST be configured only in **debug builds** to prevent logging sensitive user tokens or API response content in production environment logs.

```kotlin
val okHttpClient = OkHttpClient.Builder()
    .connectTimeout(30L, TimeUnit.SECONDS)
    .readTimeout(30L, TimeUnit.SECONDS)
    .writeTimeout(30L, TimeUnit.SECONDS)
    .apply {
        if (BuildConfig.DEBUG) {
            addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
        }
    }
    .build()
```

---

## 2. BaseDataSource & `getResult {}` Exception Wrapper

All remote data source classes MUST extend `BaseDataSource` (`id.co.edtslib.data`). Every API call execution must be wrapped using the `getResult` helper function to capture network errors and format them safely.

```kotlin
package id.co.edts.mobile.data.source.remote.datasource

import id.co.edtslib.data.BaseDataSource
import javax.inject.Inject
import id.co.edts.mobile.data.source.remote.api.AuthApiService

class AuthRemoteDataSource @Inject constructor(
    private val apiService: AuthApiService
) : BaseDataSource() {

    suspend fun login(request: LoginRequest) = getResult { 
        apiService.login(request) 
    }
}
```

### Exception Mapping

The `getResult {}` wrapper intercepts generic exceptions (e.g., `SocketTimeoutException`, `UnknownHostException`, or Retrofit `HttpException`) and maps them into an error `Result` wrapper status to prevent crashes and ensure predictable error delivery down to the domain/presentation layer.

---

## 3. Custom Result Status Wrapper

EDTS Android apps MUST utilize the custom `id.co.edtslib.data.Result` wrapper to model API states rather than using Kotlin's standard `Result` class. 

### Result Class Structure

```kotlin
package id.co.edtslib.data

data class Result<out T>(
    val status: Status,
    val data: T?,
    val code: String?,
    val message: String?
) {
    enum class Status { SUCCESS, ERROR, LOADING, UNAUTHORIZED }
}
```

- **`SUCCESS`**: The request succeeded, and data is populated.
- **`ERROR`**: The request failed due to an API error, network exception, or mapping error.
- **`LOADING`**: An asynchronous request is in progress.
- **`UNAUTHORIZED`**: The authentication session has expired or is invalid.

---

## 4. API Response Models

All raw backend payloads MUST be modeled using the standard `ApiResponse` or `ApiContentResponse` structures to maintain consistency.

### Standard Response Classes

| Class | Type of Content | `isSuccess()` Condition |
|---|---|---|
| `ApiResponse<T>` | Single item or object | `status == "00"` or `status == "01"` |
| `ApiContentResponse<T>` | Paginated content lists | status is successful AND `data?.content != null` |

```kotlin
// Example structure for standard response
data class ApiResponse<T>(
    val status: String,
    val message: String,
    val data: T?,
    val timestamp: String
) {
    fun isSuccess(): Boolean = status == "00" || status == "01"
}
```

---

## 5. Security & SSL Certificate Pinning

### Plaintext Prevention

API base URLs and SSL certificate fingerprint pins MUST NOT be committed to Git repositories in plaintext strings. 

- **Hex Obscuration**: Base URLs and SSL certificate pins must be converted to hex values and decoded at runtime during initialization.
- **`BuildConstants`**: Utilize configuration plugins to map and access keys dynamically.

```kotlin
// Example decoding inside Application class setup
EdtsKu.init(this, BuildConfig.BASE_URL) {
    setDebugging(BuildConfig.DEBUG)
    setSslDomain(CommonUtil.hexToAscii(BuildConfig.SSL_DOMAIN_HEX))
    setSslPinner(CommonUtil.hexToAscii(BuildConfig.SSL_PINNER_HEX))
}
```
