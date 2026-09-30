# UI Testing Rules

> [!IMPORTANT]
> **Prerequisite:** Do not add UI tests or create UI test targets (`androidTest/`) if the project does not already have an existing UI test configuration in `build.gradle.kts` / `libs.versions.toml`.

---

## Core Rules

1. **Test user journeys instead of implementation details** — verify critical user journeys (Happy Paths, conversion funnels, major error screens) end-to-end. Do not test visual padding or internal ViewModel states in UI tests (reserve that for Unit/Screenshot tests).
2. **Direct Navigation is Allowed (Case-by-Case)** — launching directly into the target screen using Intent parameters or Compose isolated screens is allowed to eliminate flakiness and reduce execution time, but is not mandatory. Choose between direct navigation and full multi-screen flows depending on the test objective (e.g. testing isolated screen states vs. end-to-end user journeys).
3. **Disable Tracker & Analytics (Mandatory)** — suppress or stub all analytics, tracking, and telemetry SDKs (`Tracker`, `Sentinel`, `FirebaseAnalytics`, `AppsFlyer`) during UI test runs to avoid polluting production/staging analytics data, eliminate network overhead, and prevent crashes from missing dependencies.
4. **Mock API responses & Remote Config (Mandatory)** — never rely on live backend or staging services. Use Koin module overrides, Hilt `@UninstallModules`, or OkHttp `MockWebServer`.
5. **Keep tests isolated & independent (Zero Residual State)** — execute OS-level state wipes (`adb shell pm clear <pkg>` or Maestro `clearState`) or in-app storage reset (`UITestHelper.resetStorage()`) between test suites.
6. **Use Arrange, Act, Assert Pattern** — structure tests explicitly: Arrange (setup mocks, remote config, target screen/intent), Act (launch Activity/Composable & perform user actions), Assert (verify UI elements).
7. **Use compile-time safe accessibility IDs** — avoid raw string literals, localized text, or brittle view index hierarchies. Define nested `object` constants mirroring the agreed naming schema.
8. **Inherit from `BaseUITest`** — centralize app initialization, system alert/permission handlers (`GrantPermissionRule`), and resilient interaction helpers (`tapWhenVisible`, `waitForView`, `clearAndTypeText`).
9. **Never rely solely on `BuildConfig.DEBUG` — use `UITestHelper.isUITesting`** — `BuildConfig.DEBUG` is active during everyday manual development. Always require an explicit UI testing flag before activating test mocks or suppressing trackers.
10. **Prevent `App.kt` Clutter with `UITestHelper`** — encapsulate all argument parsing, storage resets, mock configs, and analytics suppression in a dedicated `UITestHelper` located in `app/src/debug/`.

---

## Test Structure: Arrange, Act, Assert

### 1. View-based Architecture (XML + Koin + Quadrant)

```kotlin
@RunWith(AndroidJUnit4::class)
class CouponListUITests : BaseUITest() {

    private val mockCouponRepo: ICouponRepository = mockk(relaxed = true)

    @Before
    fun setUp() {
        // Arrange: override Koin repository with test mock
        loadKoinModules(module {
            single<ICouponRepository>(override = true) { mockCouponRepo }
        })
    }

    @Test
    fun test_exchangeButton_displaysSuccess() {
        // Arrange: stub data contract
        coEvery { mockCouponRepo.getCoupons() } returns flowOf(
            Result.Success(listOf(dummyCouponItem))
        )

        // Act: launch Activity directly with Intent params
        val intent = Intent(context, CouponListActivity::class.java).apply {
            putExtra("EXTRA_INITIAL_SCREEN", UITestScreen.COUPON_LIST)
        }
        ActivityScenario.launch<CouponListActivity>(intent)

        tapWhenVisible(withContentDescription(PoinkuAccessibilityId.Coupon.CouponList.BUTTON_USE_COUPON))

        // Assert: verify expected UI state
        onView(withContentDescription(PoinkuAccessibilityId.Coupon.CouponList.SUCCESS_BADGE))
            .check(matches(isDisplayed()))
    }
}
```

### 2. Jetpack Compose Architecture (Hilt + Flows)

```kotlin
@UninstallModules(CouponDataModule::class)
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class CouponListComposeUITests : BaseUITest() {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @BindValue @JvmField
    val mockCouponRepo: ICouponRepository = mockk(relaxed = true)

    @Before
    fun setUp() {
        hiltRule.inject()
    }

    @Test
    fun test_exchangeButton_displaysSuccess() {
        // Arrange
        coEvery { mockCouponRepo.getCoupons() } returns flowOf(
            Result.Success(listOf(dummyCouponItem))
        )

        // Act: isolated Composable mounting
        composeTestRule.setContent {
            CouponListScreen(viewModel = hiltViewModel())
        }

        composeTestRule.onNodeWithTag(PoinkuAccessibilityId.Coupon.CouponList.BUTTON_USE_COUPON)
            .performClick()

        // Assert
        composeTestRule.onNodeWithTag(PoinkuAccessibilityId.Coupon.CouponList.SUCCESS_BADGE)
            .assertIsDisplayed()
    }
}
```

---

## Direct Navigation (Optional / Case-by-Case)

Direct navigation is **not mandatory**, but **allowed** based on specific test needs:

- **Use Direct Navigation when:** Testing deep or isolated screens, verifying specific screen states/error cases, or when you want to minimize execution time and avoid flakiness caused by preceding screens in a long flow.
- **Use Multi-Screen Flows when:** Testing end-to-end user journeys, critical conversion funnels, screen transitions, or multi-step flows where navigation itself is under test. *Note: Espresso `ActivityScenario` automatically supports multi-activity transitions across the same process.*

### Launch Arguments & Enums
Define compile-time constants for testing parameters:

```kotlin
object UITestArgument {
    const val UI_TESTING = "--ui-testing"
    const val RESET_STATE = "--reset-state"
    const val EXTRA_INITIAL_SCREEN = "EXTRA_INITIAL_SCREEN"
    const val EXTRA_SCENARIO = "EXTRA_SCENARIO"
}

enum class UITestScreen {
    COUPON_LIST,
    PRODUCT_DETAIL,
    CART
}

enum class UITestScenario {
    COUPON_EXCHANGE_SUCCESS,
    COUPON_EXCHANGE_FAILURE
}
```

---

## `UITestHelper` (`app/src/debug/`)

Encapsulate test handling inside `UITestHelper` in the **`debug` source set** (`app/src/debug/java/.../UITestHelper.kt`) so `App.kt` stays completely clean:

```kotlin
object UITestHelper {
    val isUITesting: Boolean by lazy {
        runCatching {
            Class.forName("androidx.test.platform.app.InstrumentationRegistry")
            true
        }.getOrDefault(false) || System.getProperty("ui.testing") == "true"
    }

    fun configure(application: Application) {
        if (!isUITesting) return

        // 1. Silent Analytics Bypass
        AnalyticsBypassHelper.disableAllAnalytics(application)

        // 2. Storage / Cache Reset if requested
        if (System.getProperty("reset.state") == "true") {
            resetStorage(application)
        }
    }

    private fun resetStorage(context: Context) {
        // Clear Room DB tables, SharedPreferences, and Glide/HTTP caches
    }
}
```

> [!WARNING]
> Never activate test mocks or bypasses based solely on `BuildConfig.DEBUG` — always verify `UITestHelper.isUITesting`.

In `App.kt`:
```kotlin
if (BuildConfig.DEBUG) {
    UITestHelper.configure(this)
}
```

---

## `BaseUITest` Superclass

Inherit all UI test suites from `BaseUITest` to eliminate boilerplate, auto-dismiss system permission dialogs, and use robust interaction helpers:

```kotlin
open class BaseUITest {

    @get:Rule(order = 0)
    val permissionRule: GrantPermissionRule = GrantPermissionRule.grant(
        android.Manifest.permission.ACCESS_FINE_LOCATION,
        android.Manifest.permission.POST_NOTIFICATIONS
    )

    protected val context: Context
        get() = ApplicationProvider.getApplicationContext()

    @Before
    @CallSuper
    open fun setUpBase() {
        // Auto-disable all analytics & telemetry
        AnalyticsBypassHelper.disableAllAnalytics(context)
    }

    // MARK: - Resilient Interaction Helpers
    fun tapWhenVisible(matcher: Matcher<View>, timeoutMs: Long = 5000) {
        waitForView(matcher, timeoutMs)
        onView(matcher).perform(click())
    }

    fun clearAndTypeText(matcher: Matcher<View>, text: String) {
        onView(matcher).perform(clearText(), typeText(text), closeSoftKeyboard())
    }

    fun waitForView(matcher: Matcher<View>, timeoutMs: Long = 5000): ViewInteraction {
        val endTime = System.currentTimeMillis() + timeoutMs
        do {
            try {
                return onView(matcher).check(matches(isDisplayed()))
            } catch (e: Exception) {
                Thread.sleep(100)
            }
        } while (System.currentTimeMillis() < endTime)
        return onView(matcher).check(matches(isDisplayed()))
    }
}
```

---

## Disable Tracker & Analytics (The No-Op Gateway Pattern)

Never use conditional `if (isTesting)` checks inside `App.kt` or `IdmActivity`. When third-party tracking libraries (`Tracker`, `Sentinel`, `FirebaseAnalytics`) are called by base classes, skipping `.init()` causes crashes (`NoBeanDefFoundException`).

Instead, let production initialize normally, then **override the transport repositories in test setup**:

```kotlin
object AnalyticsBypassHelper {
    fun disableAllAnalytics(context: Context) {
        // 1. Firebase Analytics Official SDK Kill-Switch
        FirebaseAnalytics.getInstance(context).setAnalyticsCollectionEnabled(false)

        // 2. EDTS Tracker & Sentinel Repository Overrides (Koin)
        loadKoinModules(module {
            single<ITrackerRepository>(override = true) { NoOpTrackerRepository() }
            single<ISentinelRepository>(override = true) { NoOpSentinelRepository() }
        })
    }
}

// In-memory No-Op Implementations
class NoOpTrackerRepository : ITrackerRepository {
    override fun createSession(): Flow<String> = emptyFlow()
    override fun setUserId(userId: Long): Flow<Long> = flowOf(userId)
    override fun trackPage(pageName: String, pageId: String, path: String): Flow<TrackerResponse> = 
        flowOf(TrackerResponse(status = true))
    override fun trackClick(name: String, cat: String, url: String?, details: Any?): Flow<TrackerResponse> = 
        flowOf(TrackerResponse(status = true))
    // All other methods return flowOf(TrackerResponse(true))
}

class NoOpSentinelRepository : ISentinelRepository {
    override fun createSession(): Flow<String?> = flowOf(null)
    override fun track(group: String, name: String, details: Any?, userDetails: Any?): Flow<Any?> = flowOf(null)
    override fun submit(): Flow<Any?> = flowOf(null)
}
```

---

## Accessibility Identifiers

### 1. Compile-Time Safe Identifiers
Define nested Kotlin `object`s to prevent typo bugs and raw string literals:

```kotlin
object PoinkuAccessibilityId {
    object Coupon {
        object CouponList {
            const val BUTTON_USE_COUPON = "poinku_coupon_couponList_buttonUseCoupon"
            const val SUCCESS_BADGE = "poinku_coupon_couponList_successBadge"
            fun couponCard(id: Any) = "poinku_coupon_couponList_couponCard-$id"
        }
    }
}
```

### 2. Usage

#### View-based (XML ViewBinding)
```kotlin
// In Fragment / Activity
binding.btnUseCoupon.setTestTag(PoinkuAccessibilityId.Coupon.CouponList.BUTTON_USE_COUPON)
```

#### Jetpack Compose
```kotlin
Button(
    onClick = { ... },
    modifier = Modifier.semantics {
        testTag = PoinkuAccessibilityId.Coupon.CouponList.BUTTON_USE_COUPON
    }
) { ... }
```

### 3. Naming Convention

- **Pattern**: `<app>_<module - optional>_<pageName>_<component>-<id - optional>`
- **Example**: `poinku_coupon_couponList_couponCard-123`

| Category | Pattern | Example |
|---|---|---|
| **Static Element (with Module)** | `<app>_<module>_<pageName>_<component>` | `poinku_coupon_couponList_buttonUseCoupon` |
| **Static Element (without Module)** | `<app>_<pageName>_<component>` | `poinku_home_buttonPoint` |
| **Dynamic Collection Item** | `<app>_<module - optional>_<pageName>_<component>-<id>` | `poinku_coupon_couponList_couponCard-123` |

### 4. Dynamic Extension & Auto-Resolving `pageName` via Tracker

Pada project EDTS yang mengadopsi library `Tracker` (`id.co.edtslib.tracker`), property `Tracker.currentPageName` secara otomatis diperbarui oleh base classes setiap kali layar baru dibuka (`onResume()`).

Gunakan `Tracker.currentPageName` sebagai **default value parameter `page`** di extension helper:

```kotlin
/**
 * Extension untuk memasang Test Tag terstandarisasi.
 * Parameter [page] secara default otomatis mengambil nama halaman aktif dari Tracker.
 */
fun View.setTestTag(
    component: String,
    page: String = Tracker.currentPageName.toTestTagPageName(),
    module: String? = null,
    id: Any? = null,
    app: String = "klik"
) {
    val modPart = module?.let { "_$it" }.orEmpty()
    val idPart = id?.let { "-$it" }.orEmpty()
    this.contentDescription = "${app}${modPart}_${page}_${component}${idPart}"
}

fun View.setTestTag(tag: String) {
    this.contentDescription = tag
}

/**
 * Sanitasi string judul tracker menjadi camelCase (contoh: "Detail Produk" -> "detailProduk").
 */
fun String?.toTestTagPageName(): String {
    return this?.trim()
        ?.split(Regex("\\s+"))
        ?.mapIndexed { index, s ->
            if (index == 0) s.lowercase() else s.replaceFirstChar { it.uppercase() }
        }
        ?.joinToString("")
        ?.ifEmpty { "defaultPage" }
        ?: "defaultPage"
}
```

### 5. Reusable ViewHolder / Component Rule (Zero-Boilerplate)

Karena `page` secara default membaca `Tracker.currentPageName`, reusable component (seperti `ProductViewHolder`, `ProductCard`) **tidak perlu dipasangi parameter `pageContext` secara berantai dari Fragment/Adapter**:

```kotlin
// Reusable ViewHolder: CUKUP isi component & id!
class ProductViewHolder(
    private val binding: ItemProductBinding
) : RecyclerView.ViewHolder(binding.root) {

    fun bind(item: Product, module: String = "food") {
        // Otomatis menghasilkan:
        // - klik_food_productDetail_productItem-123 (jika sedang di PDP)
        // - klik_food_cartPage_productItem-123 (jika sedang di Cart)
        binding.root.setTestTag(
            module = module,
            component = "productItem",
            id = item.id
        )

        binding.btnAddToCart.setTestTag(
            module = module,
            component = "btnAddToCart",
            id = item.id
        )
    }
}
```

> **Catatan Override:** Jika suatu saat diperlukan nama layar yang berbeda dari nama tracking, developer tetap dapat meng-override parameter secara manual:
> `binding.root.setTestTag(page = "customScreen", component = "productItem", id = item.id)`
