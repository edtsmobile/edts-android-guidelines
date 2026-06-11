# Android Project Guide

> This file provides guidance to AI Agent when working with code in this repository.

---

## Project Identity

```
Application Name    : [PROJECT_NAME]
Package Name        : edts.[PROJECT_NAME].android
Min SDK             : 24
Target SDK          : 36
Compile SDK         : 36
Version Name        : 1.0.0
Version Code        : 1
```

---

## Build Flavors

| Flavor        | Application ID              | Environment |
|---------------|-----------------------------|-------------|
| `development` | `edts.[PROJECT_NAME].dev`   | Development |
| `uat`         | `edts.[PROJECT_NAME].uat`   | UAT         |
| `production`  | `edts.[PROJECT_NAME].android` | Production  |

---

## Technology Stack

| Layer        | Technology                                  | Notes                                                 |
|--------------|---------------------------------------------|-------------------------------------------------------|
| Language     | **Kotlin**                                  | Java only for legacy code; do not write new Java code |
| UI           | **Jetpack Compose** (Compose) / **XML** (View-based) | Gen 2 (Compose) vs Gen 1 (View-based)                 |
| Architecture | **MVI** (Compose) / **MVVM** (View-based)   | Follow architectural generation standard              |
| DI           | **Hilt** (Compose) / **Koin** (View-based)  | Switch depends on UI framework                        |
| Network      | **Retrofit + OkHttp + Gson**                |                                                       |
| Database     | **Room**                                    | Use KSP, not KAPT                                     |
| Async        | **Coroutines + StateFlow**                  | Use Flow, NOT LiveData                                |
| Image        | **Coil** (Compose) / **Glide** (View-based) | For Compose: `AsyncImage`                             |
| Navigation   | **Navigation 3** (Compose) / **ModuleNavigator** (View-based) | Follow cross-feature navigation guideline |
| Build        | **Gradle Kotlin DSL**                       | `.gradle.kts`, use Version Catalog                    |
| Testing      | **JUnit4 + MockK + Google Truth + Kotlinx Coroutines Test + Turbine**                | Required for ViewModels and UseCases                  |

---

## Module Structure

```
edts-[PROJECT_NAME]-android/
├── app/                        # Entry point, DI setup, MainActivity
├── core/
│   ├── data/                   # Base repository, network client, interceptor
│   │   ├── di/                 # Hilt modules: NetworkModule, AuthModule, HomeModule, etc.
│   │   ├── mapper/             # Response → Domain mappers
│   │   │   └── remote/
│   │   │       ├── auth/       # e.g. AuthMapper.kt
│   │   │       └── home/       # e.g. HomeMapper.kt
│   │   └── source/
│   │       ├── firebase/       # RemoteConfig, FirebaseHelper
│   │       ├── local/          # LocalDataSource implementations
│   │       └── remote/
│   │           ├── api/        # Retrofit service interfaces (per feature)
│   │           ├── datasource/ # Remote data sources extending BaseDataSource
│   │           ├── repository/ # Repository implementations
│   │           ├── request/    # Request DTOs
│   │           └── response/   # Response DTOs + base/ (ApiResponse, etc.)
│   ├── domain/                 # Base UseCase, Result wrapper
│   │   ├── error/              # AppError sealed class
│   │   ├── model/              # Domain models (per feature sub-package)
│   │   ├── navigator/          # Navigation interfaces
│   │   ├── repository/         # Repository interfaces (per feature sub-package)
│   │   └── usecase/            # Use cases (per feature sub-package)
│   ├── navigator/              # NavigationHost, AppNavigator implementation
│   ├── resource/               # Shared Compose resources, constants, composables
│   ├── tracker/                # Tracker library wrapper (id.co.edtslib:tracker)
│   └── utils/                  # Generic utility functions, extensions
├── presentation/               # Shared Compose UI, qualifiers, base activity classes
├── design-system/              # Reusable Compose components, theme, typography
├── feature/
│   ├── splash/                 # splash feature
│   ├── onboarding/             # onboarding feature
│   ├── auth/                   # auth feature
│   ├── home/                   # home feature
│   ├── profile/                # profile feature
│   ├── membercard/             # member card feature
│   ├── point/                  # point feature
│   ├── coupon/                 # coupon feature
│   ├── mission/                # mission feature
│   ├── wallet/                 # wallet feature
│   ├── payment/                # payment feature
│   ├── notification/           # notification feature
│   ├── referral/               # referral feature
│   ├── review/                 # review feature
│   ├── webview/                # webview feature
│   └── appupdate/              # appupdate feature
└── build-logic/                # Gradle convention plugins
```

---

## Architecture Rules — MUST FOLLOW

### Clean Architecture Boundaries

Multi-Module + MVVM + Clean Architecture + Navigation.

```
Presentation  -->  Domain  <--  Data
(ViewModel)       (UseCase)     (Repository Impl)
```

- **FORBIDDEN**: Presentation layer directly imports classes from the Data layer
- **FORBIDDEN**: UseCases import Android framework classes (`android.*`)
- **FORBIDDEN**: Room Entity is used directly in a ViewModel; always map to a Domain model
- **FORBIDDEN**: Business logic inside a Composable

### ViewModel (MVI Pattern for Compose)

ViewModels in Compose projects must expose state via `StateFlow` and process user actions via intents.

```kotlin
// CORRECT
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getProductsUseCase: GetProductsUseCase
) : ViewModel() {
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    fun processIntent(intent: HomeIntent) {
        when (intent) {
            is HomeIntent.LoadProducts -> loadProducts()
            is HomeIntent.Refresh -> loadProducts()
        }
    }

    private fun loadProducts() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            // invoke UseCase and update state
        }
    }
}

// WRONG — do not inject Repository directly into ViewModel
class HomeViewModel @Inject constructor(
    private val repository: ProductRepository  // ❌
)
```

### UI State & Intents

```kotlin
// Use this pattern for every screen state
data class HomeUiState(
    val isLoading: Boolean = false,
    val products: List<Product> = emptyList(),
    val errorMessage: String? = null
)

// Define all user actions as Intents
sealed interface HomeIntent {
    data object LoadProducts : HomeIntent
    data object Refresh : HomeIntent
}
```

### UseCase

```kotlin
// One UseCase = one responsibility
// Naming: verb + noun, examples: GetProducts, CreateOrder, DeleteUser
class GetProductsUseCase @Inject constructor(
    private val repository: ProductRepository
) {
    suspend operator fun invoke(): Result<List<Product>> =
        repository.getProducts()
}
```

---

## Code Conventions

### Naming

| Type                 | Convention                         | Example                             |
|----------------------|------------------------------------|-------------------------------------|
| Composable           | PascalCase, screen name + "Screen" | `HomeScreen`, `ProfileScreen`       |
| ViewModel            | PascalCase + "ViewModel"           | `HomeViewModel`                     |
| UseCase              | Verb + noun + "UseCase"            | `GetProductsUseCase`                |
| Repository interface | Noun + "Repository"                | `ProductRepository`                 |
| Repository impl      | Noun + "RepositoryImpl"            | `ProductRepositoryImpl`             |
| Room Entity          | Noun + "Entity"                    | `ProductEntity`                     |
| DTO/Response         | Noun + "Response"                  | `ProductResponse`                   |
| Domain model         | Noun only                          | `Product`                           |
| Hilt Module          | Noun + "Module"                    | `NetworkModule`, `RepositoryModule` |

### Compose

```kotlin
// Composables should be stateless whenever possible
// Hoist state to the ViewModel, not inside the Composable

// CORRECT — stateless composable
@Composable
fun ProductCard(
    product: Product,
    onAddToCart: (Product) -> Unit,
    modifier: Modifier = Modifier  // modifier is always the last parameter
) { ... }

// Use collectAsStateWithLifecycle(), NOT collectAsState()
val uiState by viewModel.uiState.collectAsStateWithLifecycle()

// Every Composable must have a Preview
@Preview(showBackground = true)
@Composable
private fun ProductCardPreview() {
    AppTheme { ProductCard(product = fakeProduct, onAddToCart = {}) }
}

// Previews with ViewModels
// Screen previews that require ViewModel data must use mock ViewModels or interfaces
// supplying hardcoded mock data to isolate preview rendering.
```

### Coroutines & Flow

```kotlin
// Use the appropriate Dispatcher
viewModelScope.launch {
    withContext(Dispatchers.IO) { /* I/O operation */ }
}

// Flow transformations belong in the repository, not the ViewModel
fun getProducts(): Flow<List<Product>> = dao.getAll()
    .map { entities -> entities.map { it.toDomain() } }
    .flowOn(Dispatchers.IO)

// Use StateFlow for UI state, SharedFlow for events
private val _events = MutableSharedFlow<HomeEvent>()
val events: SharedFlow<HomeEvent> = _events.asSharedFlow()
```

### Formatting

Always format Kotlin and XML files using Android Studio's default settings. Use the default shortcut (`Cmd + Option + L` on macOS, `Ctrl + Alt + L` on Windows/Linux) to format files before committing.

---

## Gradle & Dependencies

### Version Catalog

All library versions live in `gradle/libs.versions.toml`. **DO NOT** hardcode versions in `build.gradle.kts`.

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.1.0"
compose-bom = "2025.01.00"
hilt = "2.54"
room = "2.7.0"

[libraries]
# references go here, use them in build.gradle.kts via libs.xxx

[plugins]
# references go here, use them in build.gradle.kts via libs.plugin.xxx
```

### Convention Plugins

Use convention plugins from `build-logic/` for consistent Gradle configuration:

```kotlin
// build.gradle.kts in each feature module
plugins {
    alias(libs.plugins.myapp.android.feature)
    alias(libs.plugins.myapp.android.hilt)
}
```

### Adding a New Dependency

1. Add the version in `libs.versions.toml`
2. Add the library entry in `libs.versions.toml`
3. Reference it in the `build.gradle.kts` of the module that needs it
4. **DO NOT** duplicate dependencies that already exist in the `core` module

---

## Testing

### Testing Rules

- Every **ViewModel** must have a unit test
- Every **UseCase** must have a unit test
- **Repository** must have an integration test with in-memory Room
- UI tests (Compose) are optional; prioritize critical flows

### Test Setup

```kotlin
// build.gradle.kts
testImplementation(libs.junit4)
testImplementation(libs.mockk)
testImplementation(libs.kotlinx.coroutines.test)
testImplementation(libs.turbine)          // for Flow tests

// In the test class
@RunWith(JUnit4::class)
class HomeViewModelTest {
    @MockK lateinit var getProductsUseCase: GetProductsUseCase

    private val testDispatcher = UnconfinedTestDispatcher()

    @Before
    fun setup() {
        MockKAnnotations.init(this)
        Dispatchers.setMain(testDispatcher)
    }

    @Test
    fun `when loadProducts success, uiState should be Success`() = runTest {
        coEvery { getProductsUseCase() } returns Result.success(fakeProducts)
        val viewModel = HomeViewModel(getProductsUseCase)

        viewModel.uiState.test {
            // Assert loading and then success states using Google Truth
            assertThat(awaitItem()).isEqualTo(HomeUiState(isLoading = true))
        }
    }
}
```

### Fake vs Mock

- Use **Fake** (manual implementation) for Repositories in ViewModel tests
- Use **Mock** (MockK) for external dependencies such as API services

---

## API & Networking

```
Base URL: **hex-encoded** in BuildConstants.kt. Decoded at build time via BuildConfig fields.
Timeout : Connect 30s, Read 30s, Write 30s
Auth    : Bearer token
```

### Error Handling

```kotlin
// Use Result<T> for all operations that can fail
// Do not let exceptions reach the ViewModel

sealed class AppError : Exception() {
    data object NetworkError : AppError()
    data object UnauthorizedError : AppError()
    data class ServerError(val code: Int) : AppError()
    data class UnknownError(val cause: Throwable) : AppError()
}

// In repository
suspend fun getProducts(): Result<List<Product>> = runCatching {
    val response = api.getProducts()
    response.body()?.map { it.toDomain() } ?: emptyList()
}.mapFailure { throwable -> throwable.toAppError() }
```

## Local Storage (Room Database)

Use Room with Kotlin KSP for local caching. All local entities and DAOs must reside strictly in the `:core-data` layer.

### Entity Setup
Always separate local database models from domain models. Local database models must be annotated with `@Entity`.
```kotlin
@Entity(tableName = "products")
data class ProductEntity(
    @PrimaryKey val id: String,
    val name: String,
    val price: Double
) {
    fun toDomain() = Product(id = id, name = name, price = price)
}
```

### DAO Setup
DAO interfaces must specify local queries. DAOs must return a Kotlin `Flow` for all read operations to reactively update the UI.
```kotlin
@Dao
interface ProductDao {
    @Query("SELECT * FROM products")
    fun getAllProducts(): Flow<List<ProductEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertProducts(products: List<ProductEntity>)
}
```

### Database Setup
```kotlin
@Database(entities = [ProductEntity::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao
}
```

---

## Key External Libraries
- **EDTSKU** (2.9.4) — Base and core utilities (internal EDTS library)
- **EDTSDS** (1.3.32) — EDTS Design System (internal EDTS library)
- **EdtsUiKit** (0.17.0) — EDTS Design System (internal EDTS library)
- **EDTS Tracker** (2.3.21) — Tracker user activity (internal EDTS library)

---

## EDTSKU Library — Complete Reference

> Source: [github.com/Dhanfinix/edtsku (branch: edtsku-modern)](https://github.com/Dhanfinix/edtsku/tree/edtsku-modern)
> Package: `id.co.edtslib`

EDTSKU is the **foundational library** for all EDTS Android apps. It provides base classes for Activity, Fragment, ViewModel, networking (OkHttp/Retrofit), local storage, result handling, and session management. **Do not reimplement what EDTSKU already provides.**

---

### 1. Initialization — Application Class

Initialize `EdtsKu` once in `Application.onCreate()` using the DSL form:

```kotlin
@HiltAndroidApp
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        EdtsKu.init(this, BuildConfig.BASE_URL) {
            setDebugging(BuildConfig.DEBUG)
            setPackageName(packageName)
            setVersionName(BuildConfig.VERSION_NAME)
            setSslDomain(CommonUtil.hexToAscii("..."))   // hex-encoded domain
            setSslPinner(CommonUtil.hexToAscii("..."))   // hex-encoded pin
            setRefreshTokenUrlPath("/auth/refresh-token")
            setTimeout(30L)
            setTrackerConfig(TrackerConfig(...))
        }
    }
}
```

**`EdtsKuConfig` key fields:**

| Field                 | Type             | Purpose                                      |
|-----------------------|------------------|----------------------------------------------|
| `baseUrlApi`          | `String`         | Base URL for all API calls                   |
| `sslDomain`           | `String`         | Domain for SSL pinning                       |
| `sslPinner`           | `String`         | Certificate pin (SHA-256)                    |
| `refreshTokenUrlPath` | `String`         | Endpoint path for token refresh              |
| `debugging`           | `Boolean`        | Enables HTTP logging; uses plain SharedPrefs |
| `enableSignature`     | `Boolean`        | Enables request signature injection          |
| `timeout`             | `Long?`          | Connect/read/write timeout in seconds        |
| `trackerConfig`       | `TrackerConfig?` | Initializes EDTS Tracker                     |

**Access from anywhere:**
```kotlin
EdtsKu.getDependencies()  // EdtsKuDependencies — Retrofit, SharedPrefs, etc.
EdtsKu.getConfig()        // EdtsKuConfig — current configuration
```

---

### 2. Base UI Classes

#### `BaseComposeActivity` (`id.co.edtslib.uibase_compose`)

The required base class for all Compose-based screens. Extends `CommonBaseActivity`.

```kotlin
@AndroidEntryPoint
class HomeActivity : BaseComposeActivity() {

    @Composable
    override fun ContentUI() {
        // Main screen content goes here
        HomeScreen()
    }

    @Composable
    override fun ProjectTheme(content: @Composable () -> Unit) {
        AppTheme { content() }  // wrap with your design system theme
    }

    // Tracker page name (optional)
    override fun getTrackerPageName() = "home"

    // Allow back navigation
    override fun canBack() = true
}
```

Built-in capabilities from `BaseComposeActivity`:
- `showSnackBar(message, actionLabel, duration, paddingValues, onDismiss, onAction, snackbarComp)` — Snackbar from Activity
- `getBaseState()` — collect `BaseActivityState` (Snackbar host state)

Capabilities inherited from `CommonBaseActivity`:
- `enableEdgeToEdge()` called automatically
- `getTrackerPageName()` / `enablePageTracker()` — auto page tracking
- `isHomeActivity()` — double-tap back to quit
- `canBack()` — enable/disable back navigation
- `clonerAllowed()`, `emulatorAllowed()`, `rootAllowed()` — security guards
- Firebase Remote Config auto-setup if `@xml/remote_config` exists

#### `BaseComposeViewModel` (`id.co.edtslib.uibase_compose`)

Base ViewModel injected automatically by `BaseComposeActivity`. **Do not extend this** for feature ViewModels — use `ViewModel()` directly and call `showSnackBar()` via the Activity.

```kotlin
// BaseComposeActivity exposes:
protected val baseViewModel: BaseComposeViewModel by viewModels()
// Snackbar is managed via Activity, not feature ViewModel
```

#### `BaseComposeFragment` (`id.co.edtslib.uibase_compose`)

**Deprecated for pure Compose projects.** Only use for legacy interop:

```kotlin
// Only if Fragment is strictly required (e.g., navigation interop)
class SomeFragment : BaseComposeFragment() {
    @Composable
    override fun ContentUI() { /* content */ }
}
```

---

### 3. Network & Data Layer

#### Retrofit Instance — Hilt Module

**Always get Retrofit from `EdtsKu`, never build your own:**

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit = EdtsKu.getDependencies().retrofit

    @Provides
    @Singleton
    fun provideMyApiService(retrofit: Retrofit): MyApiService =
        retrofit.create(MyApiService::class.java)
}
```

The Retrofit instance from EDTSKU already includes:
- `AuthInterceptor` — injects Authorization header, app identifier, SHA1, and request signature
- `SafeOkHttpClient` — SSL pinning (when `sslDomain`/`sslPinner` set) + HTTP logging
- `GsonConverterFactory`

#### `BaseDataSource` (`id.co.edtslib.data`)

Extend this for all Remote Data Sources. Use `getResult {}` to wrap API calls — it handles all HTTP error codes (400, 401, 500, 503) and network exceptions:

```kotlin
class ProductRemoteDataSource @Inject constructor(
    private val apiService: ProductApiService
) : BaseDataSource() {

    suspend fun getProducts() = getResult { apiService.getProducts() }
    suspend fun createOrder(req: OrderRequest) = getResult { apiService.createOrder(req) }
}
```

`getResult{}` automatically:
- Returns `Result.success(body)` on 2xx + `isSuccess()` = true
- Returns `Result.unauthorized(message)` on 401
- Returns `Result.error(code, message)` on 400/500/503
- Returns `Result.error("ConnectionError", ...)` on network errors
- Calls `Tracker.trackSubmissionFailed()` on all error cases

---

### 4. Result Wrapper

**`Result<T>`** (`id.co.edtslib.data`) — use this everywhere, not Kotlin's `kotlin.Result`:

```kotlin
data class Result<out T>(
    val status: Status,
    val data: T?,
    val code: String?,
    val message: String?
) {
    enum class Status { SUCCESS, ERROR, LOADING, UNAUTHORIZED }
}
```

Create results:
```kotlin
Result.success(data)
Result.error(code, message, data)
Result.unauthorized(message)
Result.loading()
```

---

### 5. Repository Patterns

#### `NetworkBoundGetResource` — for GET with local cache

Use when data should be cached locally (Room/DataStore) and fetched from network only when stale:

```kotlin
class ProductRepository @Inject constructor(
    private val remoteDataSource: ProductRemoteDataSource,
    private val dao: ProductDao
) {
    fun getProducts(): Flow<Result<List<Product>>> =
        object : NetworkBoundGetResource<List<Product>, ApiResponse<List<ProductResponse>>>() {
            override fun getCached() = dao.getAll().map { it.map { e -> e.toDomain() } }
            override fun shouldFetch(data: List<Product>?) = data.isNullOrEmpty()
            override suspend fun createCall() = remoteDataSource.getProducts()
            override suspend fun saveCallResult(data: ApiResponse<List<ProductResponse>>) {
                dao.insertAll(data.data?.map { it.toEntity() } ?: emptyList())
            }
        }.asFlow()
}
```

`NetworkBoundGetResource` automatically handles 401 → token refresh → retry.

#### `NetworkBoundProcessResource` — for POST/PUT/DELETE

Use for mutating operations (no local cache):

```kotlin
fun createOrder(req: OrderRequest): Flow<Result<Order?>> =
    object : NetworkBoundProcessResource<Order?, ApiResponse<OrderResponse>>() {
        override suspend fun createCall() = remoteDataSource.createOrder(req)
        override suspend fun callBackResult(data: ApiResponse<OrderResponse>): Order? =
            data.data?.toDomain()
    }.asFlow()
```

`NetworkBoundProcessResource` also handles 401 → token refresh → retry automatically.

---

### 6. API Response Models

| Class | Fields | `isSuccess()` condition |
|---|---|---|
| `ApiResponse<T>` | `data: T?`, `message`, `status`, `timestamp` | `status == "00"` or `"01"` |
| `ApiContentResponse<T>` | `data: ContentResponse<T>?`, `message`, `status`, `timestamp` | status "00"/"01" AND `data?.content != null` |
| `ApiPagingResponse` | Paged list wrapper | — |

```kotlin
// Typical API service definition
interface ProductApiService {
    @GET("products")
    suspend fun getProducts(): Response<ApiResponse<List<ProductResponse>>>

    @POST("orders")
    suspend fun createOrder(@Body req: OrderRequest): Response<ApiResponse<OrderResponse>>
}
```

---

### 7. Local Storage

#### `HttpHeaderLocalSource` — Session/Token management

Manages HTTP headers stored in encrypted SharedPreferences:

```kotlin
val headerSource = EdtsKu.getDependencies().getHttpHeaderLocalSource

// Set auth token after login
headerSource.setBearerToken(token)
headerSource.setHeader("refresh-token", refreshToken)

// Check login state
headerSource.isLogged()

// Clear auth on logout
headerSource.logout()
```

#### `LocalDataSource<T>` — General SharedPreferences storage

**Prefer DataStore (Krypta's `BaseDataStore`) over this.** Use `LocalDataSource` only for synchronous reads required inside `NetworkBoundGetResource.shouldFetch()`:

```kotlin
class UserSessionLocalSource : LocalDataSource<UserSession>() {
    override fun getKeyName() = "user_session"
    override fun getValue(json: String) = Gson().fromJson(json, UserSession::class.java)
    override fun expiredInterval() = 3600 // seconds; 0 = never expires
}

// Usage
val session: UserSession? = userSessionLocalSource.getCached()  // synchronous
userSessionLocalSource.save(session)
userSessionLocalSource.clear()
```

---

### 8. Authentication Flow

EDTSKU handles token refresh automatically inside `NetworkBoundGetResource` and `NetworkBoundProcessResource`. Manual handling is only needed for the initial login:

```kotlin
// After successful login, store tokens:
EdtsKu.getDependencies().getHttpHeaderLocalSource.apply {
    setBearerToken(response.token)
    setHeader("refresh-token", response.refreshToken)
}

// On logout:
EdtsKu.getDependencies().getHttpHeaderLocalSource.logout()
```

When a 401 is received, the library:
1. Takes the stored `refresh-token`
2. Calls `SessionRemoteDataSource.refreshToken()`
3. On success: updates `Authorization` and `refresh-token` headers, retries original call
4. On failure: calls `localDataSource.logout()` and emits `Result.unauthorized`

---

### 9. ProcessResult — Delegate Pattern (Legacy)

`ProcessResult` and `ProcessResultDelegate` are the older pattern for observing results. Prefer direct `Result` handling in the ViewModel:

```kotlin
// Legacy — avoid in new code
ProcessResult(result, object : ProcessResultDelegate<Product> {
    override fun success(data: Product?) { /* ... */ }
    override fun error(code: String?, message: String?, data: Product?) { /* ... */ }
    override fun errorConnection(data: Product?) { /* ... */ }
    override fun errorSystem(data: Product?) { /* ... */ }
    override fun unAuthorize(message: String?) { /* ... */ }
    override fun loading() { /* ... */ }
})

// Preferred — handle Result directly in ViewModel
when (result.status) {
    Result.Status.SUCCESS -> _uiState.value = UiState.Success(result.data)
    Result.Status.ERROR -> _uiState.value = UiState.Error(result.message)
    Result.Status.UNAUTHORIZED -> _events.emit(Event.Logout)
    Result.Status.LOADING -> _uiState.value = UiState.Loading
}
```

---

### 10. Complete Integration Checklist

When adding a new feature that uses EDTSKU:

```
✅ Application: EdtsKu.init() called with correct baseUrl, SSL, timeout
✅ Activity: extends BaseComposeActivity, override ContentUI() and ProjectTheme()
✅ Hilt Network Module: Retrofit from EdtsKu.getDependencies().retrofit
✅ RemoteDataSource: extends BaseDataSource, uses getResult {}
✅ Repository GET: uses NetworkBoundGetResource (cache-first) 
✅ Repository POST/PUT/DELETE: uses NetworkBoundProcessResource
✅ API responses: typed as ApiResponse<T> or ApiContentResponse<T>
✅ Token storage: HttpHeaderLocalSource.setBearerToken() / setHeader("refresh-token", ...)
✅ Result handling: use id.co.edtslib.data.Result, NOT kotlin.Result
✅ Local storage sync: LocalDataSource<T> only for synchronous needs
✅ Local storage async: Krypta BaseDataStore for all other preferences
```

---

## Security — REQUIRED

```
❌ DO NOT store API keys, tokens, or secrets in:
   - Source code
   - strings.xml
   - build.gradle.kts (except via local.properties, which is in .gitignore)
   - Files committed to Git

✅ USE:
   - local.properties  → for development (already in .gitignore)
   - BuildConfig       → for values injected at build time
   - EncryptedSharedPreferences → for sensitive data on device
   - Android Keystore  → for cryptographic keys
```

### Files Claude Must Not Touch

```
local.properties          # contains development API keys
google-services.json      # Firebase config
keystore.jks / *.keystore # signing key
.env*                     # environment variables
secrets.gradle.kts        # secret values
```

---

## Build Commands

```bash
# Build
./gradlew assembleDevelopmentDebug        # Debug build
./gradlew assembleProductionRelease       # Production release build
./gradlew bundleUat                       # UAT app bundle

# Test
./gradlew testDevelopmentDebugUnitTest    # Unit tests for development debug variant
./gradlew test                            # All unit tests
./gradlew connectedDevelopmentDebugAndroidTest  # Instrumented tests

# Single test class
./gradlew :feature:notification:testDevelopmentDebugUnitTest --tests "*.NotificationViewModelTest"

# Lint
./gradlew lint                            # Lint (uses lint-baseline.xml for suppressions)

# Clean
./gradlew clean
```

> Before committing, always run `./gradlew test lint` first.

---

## Navigation Between Screens (Compose Navigation 3)

```kotlin
// 1. Use typed routes with Kotlin Serialization
@Serializable
object HomeRoute

@Serializable
data class DetailRoute(val productId: Int)

// 2. Define screen entries
val homeEntry = entry<HomeRoute> {
    HomeScreen(onNavigateToDetail = { id -> backStack.push(DetailRoute(id)) })
}
val detailEntry = entry<DetailRoute> { backStackEntry ->
    val route = backStackEntry.route<DetailRoute>()
    DetailScreen(productId = route?.productId ?: 0)
}

// 3. Render via NavDisplay using a rememberNavBackStack
val backStack = rememberNavBackStack(initialRoute = HomeRoute)
NavDisplay(
    backStack = backStack,
    entryProvider = entryProvider {
        +homeEntry
        +detailEntry
    }
)
```

---

### What to Copy vs Rewrite

```
✅ COPY (port directly): API endpoints, request/response field names, business logic rules,
   validation logic, navigation flows, UI content/text
✅ REWRITE (modernize): UI code (XML → Compose), DI (Koin → Hilt), 
   image loading (Glide → Coil), async handling (callbacks → Flow/coroutines)
❌ DO NOT copy: Koin modules, XML layouts, ViewBinding code, KAPT annotations
```

### Asset & Resource Policy

Ensure all resources (drawables, layout XMLs, strings, colors, animations) are systematically named and organized. Reuse existing resources from core modules whenever possible to prevent duplication. Do not invent new resource keys if generic matching keys are already available.

#### Rules for Assets in This Project

```
✅ COPY all drawables/icons from the legacy app — pixel-for-pixel match required
✅ COPY all strings from the legacy app — same wording, same copy
✅ COPY bottom navigation icons, tab icons, launcher icons from legacy
✅ COPY colors (exact hex values) — match the legacy brand palette
✅ COPY raw/ and assets/ directories if used in the legacy app
✅ COPY animations, Lottie files, fonts if used in legacy

❌ DO NOT invent new icons or use generic Material icons as substitutes
❌ DO NOT change string copy or wording without explicit product approval
❌ DO NOT skip importing an asset just because it seems "branding-specific"
```

#### What Changes vs What Stays the Same

```
STAYS THE SAME (copy from legacy):
  ✅ All icons & drawables
  ✅ All user-facing strings
  ✅ Brand colors & typography
  ✅ Bottom navigation icons & labels
  ✅ UI layout, spacing, visual hierarchy

CHANGES (modernized):
  ✅ XML layouts → Jetpack Compose
  ✅ Koin → Hilt
  ✅ Glide/Picasso → Coil 3 (AsyncImage)
  ✅ LiveData → StateFlow / SharedFlow
  ✅ Fragment navigation → Compose Navigation (typed routes)
  ✅ Single module → Multi-module
```

---

## Things Claude Must NOT Do

```
❌ Change the root project build.gradle.kts without explicit confirmation
❌ Change compileSdk / targetSdk / minSdk versions
❌ Replace core libraries (Hilt → Koin, Retrofit → Ktor, etc.)
❌ Delete or rename existing modules
❌ Put business logic inside Composables
❌ Import Android classes (android.*) in the domain layer
❌ Use LiveData for new code
❌ Use KAPT (use KSP)
❌ Hardcode user-facing strings — put them in strings.xml
❌ Commit files that contain secrets or credentials
❌ Run git commit without explicit instruction from the user
❌ Run git push to any remote without explicit instruction from the user
```

---

## Things Claude Must ALWAYS Do

```
✅ Follow the existing package structure before creating new files
✅ Check whether a similar UseCase already exists before creating a new one
✅ Mapping: Entity → Domain model, DTO → Domain model (do not leak upward)
✅ Add @Preview for every new Composable
✅ Write unit tests for every new ViewModel and UseCase
✅ Use modifier: Modifier = Modifier as the last Composable parameter
✅ Use collectAsStateWithLifecycle(), not collectAsState()
✅ Add copyright/license header if creating a new file in a public module
```

---

## Important File References

```
gradle/libs.versions.toml              → all dependency versions
build-logic/convention/                → convention plugins
design-system/src/.../theme/           → colors, typography, shape
design-system/src/.../component/       → shared UI components
core/core-data/src/.../network/        → base network setup
app/src/main/.../di/                   → root DI modules
app/src/main/AndroidManifest.xml       → permissions & entry points
```

---

*Updated: 11 June 2026 — Added Reference App section: path, package mapping table, technology notes, copy-vs-rewrite guide*
*Updated: 30 May 2026 — Fixed Module Structure to match actual implementation (core/data source tree, removed non-existent core/testing)*
*CLAUDE.md version: 1.0*
