# UI Testing Rules

> [!IMPORTANT]
> **Prerequisite:** Do not add UI tests or create UI test targets (`androidTest/`) if the project does not already have an existing UI test configuration in `build.gradle.kts` / `libs.versions.toml`.

---

## Core Rules

1. **Test user journeys instead of implementation details** — verify critical user journeys (Happy Paths, conversion funnels, major error screens) end-to-end. Do not test visual padding or internal ViewModel states in UI tests (reserve that for Unit/Screenshot tests).
2. **Direct Navigation is Allowed (Case-by-Case)** — launching directly into the target screen using Intent parameters or Compose isolated screens is allowed to eliminate flakiness and reduce execution time, but is not mandatory. Choose between direct navigation and full multi-screen flows depending on the test objective (e.g. testing isolated screen states vs. end-to-end user journeys).
3. **Disable Tracker & Analytics (Mandatory)** — suppress or stub all analytics, tracking, and telemetry SDKs (`Tracker`, `Sentinel`, `FirebaseAnalytics`, `AppsFlyer`) during UI test runs to avoid polluting production/staging analytics data, eliminate network overhead, and prevent crashes from missing dependencies.
4. **Mock API responses & Remote Config via Hand-Rolled Fakes (Mandatory)** — Never rely on live backend or staging services. SANGAT DISARANKAN menggunakan **Hand-Rolled Fakes (`FakeXxxRepository : IXxxRepository`)** alih-alih dynamic proxy MockK (`mockk-android`) pada runtime Android (ART) untuk mencegah `KotlinReflectionInternalError: Unresolved class: String`. Gunakan Koin module overrides atau Hilt `@UninstallModules`.
5. **Keep tests isolated & independent (Zero Residual State)** — execute OS-level state wipes (`adb shell pm clear <pkg>` or Maestro `clearState`) or in-app storage reset (`UITestHelper.resetStorage()`) between test suites.
6. **Use Arrange, Act, Assert Pattern** — structure tests explicitly: Arrange (setup mocks, remote config, target screen/intent), Act (launch Activity/Composable & perform user actions), Assert (verify UI elements).
7. **Use compile-time safe accessibility IDs** — avoid raw string literals, localized text, or brittle view index hierarchies. Define nested `object` constants mirroring the agreed naming schema.
8. **Inherit from `BaseUITest` (Mandatory)** — centralize app initialization, system alert/permission handlers (`GrantPermissionRule`), automated shell-level device preconditioning (autofill kill-switch, animation zeroing), and resilient interaction helpers (`waitForView`, `tapWhenVisible`, `tapUntilSatisfied`, `typeIntoField` with `replaceText`, `checkButtonEnabled`).
9. **Never rely solely on `BuildConfig.DEBUG` — use `UITestHelper.isUITesting`** — `BuildConfig.DEBUG` is active during everyday manual development. Always require an explicit UI testing flag before activating test mocks or suppressing trackers.
10. **Prevent `App.kt` Clutter with `UITestHelper`** — encapsulate all argument parsing, storage resets, mock configs, and analytics suppression in a dedicated `UITestHelper` located in `app/src/debug/`.
11. **Production Code Immutability (Hands-Off Rule)** — Saat membuat UI test, DILARANG KERAS memodifikasi kode produksi (Activity, Fragment, ViewModel, UseCase, Layout XML) selain HANYA menambahkan Test Tag (`setTestTag`). Jika ditemukan bug di kode produksi atau dibutuhkan refactor agar view testable, WAJIB konfirmasi ke developer terlebih dahulu sebelum mengeksekusi dan pastikan tidak mengubah fungsionalitas yang ada.
12. **Exhaustive & Boundary Test Coverage (Sedetail Mungkin)** — Jangan hanya menguji 1 happy path. Uji skenario sedetail mungkin:
    - *Boundary values*: minimum & maximum panjang karakter, prefix format (misal 08 vs 62 vs non-digit).
    - *Validation error states*: field wajib kosong, password mismatch, format email/password invalid.
    - *Component state toggles*: tombol disabled saat input kosong, enabled saat valid, tombol switcher mode (misal tombol 'Ubah').
    - *Domain edge cases*: akun terhapus (`deleted = true`), akun belum terdaftar, response error code spesifik.
13. **Mandatory Device Verification (No Assumptions)** — UI test TIDAK BOLEH dianggap selesai hanya karena kompilasi sukses. Wajib dieksekusi langsung pada target device/emulator via `connectedAndroidTest` dan diverifikasi 100% GREEN.
14. **Use String Resource IDs (No Hardcoded Copy Text)** — Selalu gunakan `context.getString(R.string.xxx)` atau `withText(R.string.xxx)` dalam assertion. Jangan meng-hardcode string teks di dalam file test agar tahan terhadap perubahan copy teks produk.
15. **Anti-Arbitrary Sleep (No Raw `Thread.sleep`)** — Dilarang menggunakan `Thread.sleep()` sembarangan di badan test. Gunakan polling reaktif `waitForView()` atau `tapUntilSatisfied()`.
16. **Test Independence & Zero Sequential Coupling** — Setiap skenario uji `@Test` harus berdiri sendiri (hermetis). Dilarang membuat Test B yang berasumsi Test A sudah dieksekusi atau sudah menyiapkan state/data tertentu.

---

## Test Structure: Arrange, Act, Assert

### 1. View-based Architecture (XML + Koin + Quadrant)

```kotlin
@RunWith(AndroidJUnit4::class)
class CouponListUITests : BaseUITest() {

    private val fakeCouponRepo = FakeCouponRepository()

    @Before
    fun setUp() {
        // Arrange: override Koin repository dengan Hand-Rolled Fake (zero reflection, aman di ART)
        loadKoinModules(module {
            single<ICouponRepository> { fakeCouponRepo }
        })
    }

    @Test
    fun test_exchangeButton_displaysSuccess() {
        // Arrange: stub data contract langsung via property
        fakeCouponRepo.couponsResponse = flowOf(
            Result.Success(listOf(dummyCouponItem))
        )

        // Act: launch Activity directly with Intent params
        val intent = Intent(context, CouponListActivity::class.java).apply {
            putExtra("EXTRA_INITIAL_SCREEN", UITestScreen.COUPON_LIST)
        }
        ActivityScenario.launch<CouponListActivity>(intent)

        // Resilient tap helper dari BaseUITest
        tapWhenVisible(withContentDescription(PoinkuAccessibilityId.Coupon.CouponList.BUTTON_USE_COUPON))

        // Assert: verify expected UI state via polling
        waitForView(withContentDescription(PoinkuAccessibilityId.Coupon.CouponList.SUCCESS_BADGE))
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

## `BaseUITest` Superclass (Complete Reference)

Warisi seluruh UI test suite dari `BaseUITest`. Kelas ini menyediakan:
1. **Otomatisasi Prekondisi Device via Shell (`UiAutomation`)**: Mematikan autofill prompt (Google Password Manager) dan men-zero animasi sistem agar timing Espresso 100% deterministik.
2. **Auto-Dismiss Permission Dialogs (`GrantPermissionRule`)**: Menghindari system dialog memblokir UI flow.
3. **Target App Context**: Menyediakan `context` aplikasi sesungguhnya via `targetContext`.
4. **Resilient Interaction Helpers**: Menangani polling (`waitForView`), tap yang tertelan pada popup/slide-up window transition (`tapUntilSatisfied`), compound view input dengan `replaceText` (`typeIntoField`), dan verifikasi tombol (`checkButtonEnabled`).

```kotlin
open class BaseUITest {

    companion object {
        @BeforeClass
        @JvmStatic
        fun disableSystemOverlaysAndAnimations() {
            // 1. Kill autofill service agar Google Password Manager/Autofill tidak menutupi view/CTA
            runShellCommand("settings put secure autofill_service null")
            // 2. Zero-out animasi sistem agar timing interaksi deterministik dan tidak memicu flakiness
            runShellCommand("settings put global window_animation_scale 0")
            runShellCommand("settings put global transition_animation_scale 0")
            runShellCommand("settings put global animator_duration_scale 0")
        }

        private fun runShellCommand(command: String) {
            try {
                InstrumentationRegistry.getInstrumentation().uiAutomation
                    .executeShellCommand(command)
                    .close()
            } catch (_: Exception) {
            }
        }
    }

    @get:Rule(order = 0)
    val permissionRule: GrantPermissionRule = GrantPermissionRule.grant(
        android.Manifest.permission.ACCESS_FINE_LOCATION,
        android.Manifest.permission.POST_NOTIFICATIONS
    )

    /** Target app context (bukan instrumentation context). */
    protected val context: Context
        get() = InstrumentationRegistry.getInstrumentation().targetContext

    @Before
    @CallSuper
    open fun setUpBase() {
        // Pastikan kembali autofill mati sebelum tiap test suite berjalan
        runShellCommand("settings put secure autofill_service null")
        // Auto-disable seluruh tracking & telemetry (No-Op Gateway Pattern)
        AnalyticsBypassHelper.disableAllAnalytics(context)
    }

    // ------------------------------------------------------------------
    // Resilient Interaction Helpers
    // ------------------------------------------------------------------

    /** Polling until view displayed — menghindari flakiness akibat animasi/async API. */
    protected fun waitForView(matcher: Matcher<View>, timeoutMs: Long = 5_000): ViewInteraction {
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

    protected fun tapWhenVisible(matcher: Matcher<View>, timeoutMs: Long = 5_000) {
        waitForView(matcher, timeoutMs)
        onView(matcher).perform(click())
    }

    /**
     * Tap berulang sampai [verify] terpenuhi.
     * Mengatasi fenomena "Click Swallowing" pada transisi Activity slide-up (seperti
     * IdmPopupActivity / BottomSheet) atau saat soft keyboard baru ditutup, di mana
     * event klik pertama ditelan oleh proses transisi layout window.
     */
    protected fun tapUntilSatisfied(
        matcher: Matcher<View>,
        verify: Matcher<View>,
        maxTaps: Int = 5,
        retryDelayMs: Long = 600
    ) {
        waitForView(matcher)
        repeat(maxTaps) {
            onView(matcher).perform(click())
            try {
                waitForView(verify, timeoutMs = retryDelayMs)
                return
            } catch (e: Exception) {
                // Tap tertelan atau async state masih transition — coba lagi
            }
        }
        waitForView(verify)
    }

    /**
     * Mengetik ke dalam field ber-tag [tag].
     * 
     * 1. Target Child EditText: Tag biasanya dipasang pada container (TextFieldView / TextInputLayout),
     *    sehingga helper otomatis menargetkan EditText turunan di dalamnya via isDescendantOfA.
     * 2. NestedScrollView Safety: Membungkus scrollTo() dalam try-catch karena action bawaan
     *    Espresso crash jika container bukan ScrollView murni (misal NestedScrollView).
     * 3. replaceText vs typeText: Menggunakan replaceText() untuk mengeliminasi race condition
     *    "keystroke drop" (karakter pertama terpotong saat IME keyboard sedang membuka).
     */
    protected fun typeIntoField(tag: String, text: String) {
        val interaction = onView(
            allOf(
                isAssignableFrom(EditText::class.java),
                isDescendantOfA(withContentDescription(tag))
            )
        )
        try {
            interaction.perform(scrollTo())
        } catch (_: Exception) {
            // View may already be visible or inside a non-ScrollView container (e.g. NestedScrollView)
        }
        interaction.perform(click(), replaceText(text), closeSoftKeyboard())
    }

    /** Verifikasi status isEnabled tombol (misal tombol submit non-aktif saat form kosong). */
    protected fun checkButtonEnabled(tag: String, enabled: Boolean) {
        waitForView(withContentDescription(tag))
        onView(withContentDescription(tag)).check(
            matches(if (enabled) isEnabled() else not(isEnabled()))
        )
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

---

## Hand-Rolled Fakes vs. MockK on Android ART Runtime

### 1. Masalah Fatal MockK di Android Instrumentation (`androidTest`)
Pada unit test lokal (JVM di desktop), MockK bekerja sangat baik. Namun di Android Instrumentation (`androidTest`), runtime yang digunakan adalah **Android ART** (bukan standard HotSpot JVM).
- **Dexmaker Proxy Limitation**: Library `mockk-android` men-generate bytecode proxy dinamis via Dexmaker.
- **Reflection Crash pada Primitives/String**: Saat melakukan recording mock invocation (misal: `coEvery { mockRepo.isUserExist("08123") }`), MockK sering melempar:
  ```
  KotlinReflectionInternalError: Unresolved class: String
  ```
  atau crash bytecode verifier saat stubbing method coroutine Flow.
- **Koin 3+ DSL Incompatibility**: Sintaks `single<T>(override = true)` sudah deprecated/dihapus pada Koin 3+. Override modul Koin harus dilakukan secara clean via instance fake tanpa parameter `override = true`.

### 2. Standar Solusi: Hand-Rolled Fake (Test Double)
Implementasikan interface repository domain (`IXxxRepository`) menggunakan kelas Fake murni dalam folder `androidTest/.../fake/`:

```kotlin
/**
 * Hand-rolled Fake ICouponRepository untuk hermetic UI testing.
 * Zero reflection overhead, zero Dexmaker dependency, 100% aman di ART.
 */
class FakeCouponRepository : ICouponRepository {

    // Sediakan property Flow yang bisa diubah kapan saja per skenario uji
    var couponsResponse: Flow<Result<List<CouponItem>?>> = flowOf(
        Result(Result.Status.SUCCESS, emptyList(), "01", "OK")
    )

    override fun getCoupons(): Flow<Result<List<CouponItem>?>> = couponsResponse

    // Method mutasi cukup mengembalikan response default sukses
    override fun redeemCoupon(couponId: String): Flow<Result<Boolean?>> = 
        flowOf(Result(Result.Status.SUCCESS, true, "01", "OK"))
}
```

### 3. Matriks Perbandingan

| Dimensi | MockK Android (`mockk-android`) | Hand-Rolled Fake (`FakeRepository`) |
|---|---|---|
| **Stabilitas ART** | ❌ Rentan `KotlinReflectionInternalError` | ✅ **100% Native Bytecode (Bebas Crash)** |
| **Kecepatan Run** | ⚠️ Lambat (beban bytecode generation di device) | ⚡ **10x Lebih Cepat** (instansiasi POJO biasa) |
| **Keterbacaan Test** | ⚠️ Boilerplate `coEvery { ... } returns ...` berulang | ✅ Cukup re-assign property: `fakeRepo.response = ...` |
| **Refactoring Safety** | ❌ Runtime fail jika nama/tipe method berubah | ✅ Compile-time check via Kotlin compiler |

---

## Resilient Interaction Patterns & Common UI Automation Pitfalls

### 1. Autofill Service Interference & Kill-Switch
* **Gejala**: Pada Android 12 hingga 16, Google Password Manager atau autofill provider otomatis memunculkan pop-up / dropdown saat `EditText` mendapatkan fokus. Dialog ini memblokir sentuhan ke tombol submit atau navigasi di bawahnya.
* **Solusi**: Matikan autofill service via shell di `BaseUITest` (`settings put secure autofill_service null`).

### 2. Zeroing System Animations
* **Gejala**: Transisi window, fading dialog, dan interpolator animasi bawaan OS membuat timing Espresso tidak sinkron dan menyebabkan flaky click.
* **Solusi**: Zero-out seluruh skala animasi sistem via shell:
  ```bash
  settings put global window_animation_scale 0
  settings put global transition_animation_scale 0
  settings put global animator_duration_scale 0
  ```

### 3. `replaceText` vs `typeText` (Keystroke Dropping)
* **Gejala**: Memanggil `typeText("628123456789")` menghasilkan input `"28123456789"` karena karakter pertama `'6'` tertelan saat keyboard software (IME) sedang beranimasi membuka.
* **Solusi**: Gunakan `replaceText(text)`. `replaceText` men-set teks langsung pada view menggunakan `Editable.replace()`, memicu seluruh listener `TextWatcher` / `TextFieldDelegate` secara atomik tanpa mengirimkan virtual key event per milidetik.

### 4. Compound Component Targeting (`TextFieldView` / `TextInputLayout`)
* **Gejala**: Memasang tag pada container compound view (seperti EDTS DS `TextFieldView`) lalu memanggil `onView(withContentDescription(tag)).perform(...)` menyebabkan crash karena container bukan turunan `EditText`.
* **Solusi**: Targetkan child `EditText` di dalam container ber-tag:
  ```kotlin
  onView(
      allOf(
          isAssignableFrom(EditText::class.java),
          isDescendantOfA(withContentDescription(tag))
      )
  )
  ```

### 5. `NestedScrollView` vs `ScrollView` (`scrollTo()` PerformException)
* **Gejala**: Action bawaan Espresso `scrollTo()` melempar exception jika parent bukan `android.widget.ScrollView` murni (hampir semua layout modern EDTS menggunakan `NestedScrollView`).
* **Solusi**: Bungkus `interaction.perform(scrollTo())` dalam blok `try-catch` agar tidak crash jika view berada di dalam `NestedScrollView` atau sudah terlihat di viewport.

### 6. Click-Swallowing pada Popup / Slide-Up Transitions (`tapUntilSatisfied`)
* **Gejala**: Pada activity dengan transisi slide-up (seperti `IdmPopupActivity`), modal bottom sheet, atau saat soft keyboard baru tertutup, klik pertama sering kali tertelan oleh sistem windowing. Espresso mencatat klik berhasil, namun state halaman tidak berubah.
* **Solusi**: Gunakan helper `tapUntilSatisfied(matcher, verify, maxTaps = 5, retryDelayMs = 600)` yang memvalidasi perubahan state setelah tiap tap dan mengulang jika event tertelan.

---

## Build Configuration, Dependencies & Device Execution

### 1. Dekopling Firebase BoM di `libs.versions.toml`
Pada Android Gradle Plugin (AGP), dependensi Firebase BoM di `implementation` **tidak otomatis merambat** ke classpath `androidTest`. Jika pengujian mengakses `FirebaseAnalytics` via `AnalyticsBypassHelper`, pin modul Firebase `-ktx` secara eksplisit:

```toml
[libraries]
# Wajib dipin eksplisit agar androidTest classpath dapat mengompilasi AnalyticsBypassHelper
firebase-analytics = { group = "com.google.firebase", name = "firebase-analytics-ktx", version = "21.2.0" }
firebase-crashlytics = { group = "com.google.firebase", name = "firebase-crashlytics-ktx", version = "18.3.2" }
firebase-config = { group = "com.google.firebase", name = "firebase-config-ktx", version = "21.2.0" }
```

### 2. JDK Compatibility
Pastikan eksekusi test menggunakan **Java 17 (misal: Corretto 17)**. Versi Java 21+ atau Java 25 dapat memicu incompatibilities pada toolchain AGP/Kotlin:
```bash
export JAVA_HOME="/Users/<user>/Library/Java/JavaVirtualMachines/corretto-17.0.17/Contents/Home"
```

### 3. Cheat Sheet Perintah Eksekusi Gradle

```bash
# 1. Jalankan seluruh UI Test suite di device / emulator tertentu:
ANDROID_SERIAL=<device_serial> ./gradlew :app:connectedDevelopmentDebugAndroidTest

# 2. Jalankan spesifik satu test class:
ANDROID_SERIAL=<device_serial> ./gradlew :app:connectedDevelopmentDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=edts.klikidm.android.uitest.auth.LoginUITests

# 3. Jalankan spesifik satu method uji:
ANDROID_SERIAL=<device_serial> ./gradlew :app:connectedDevelopmentDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=edts.klikidm.android.uitest.auth.LoginUITests#loginExistingUserShowsPasswordStep

# 4. Bersihkan state aplikasi antar run (Zero Residual State):
adb -s <device_serial> shell pm clear edts.klikidm.dev.android
```

