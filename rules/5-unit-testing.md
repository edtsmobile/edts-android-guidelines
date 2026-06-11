# Unit Testing Rules

## 1. Technology Stack

You must exclusively use the following testing stack for Android unit tests:
- **Runner**: JUnit4 (standard Android test runner). *JUnit5 is strictly forbidden.*
- **Mocking**: MockK (for mocking final classes, objects, and coroutines). *Mockito is strictly forbidden.*
- **Assertions**: Google Truth (`assertThat(actual).isEqualTo(expected)`). *JUnit assertEquals is forbidden.*
- **Async/Coroutines**: Kotlinx Coroutines Test (`runTest`).
- **Reactive/Streams**: Turbine (Optional, for Flow / StateFlow stream asserting).

---

## 2. Folder Structure

The test folder directory **must** mirror the production directory structure exactly under the `test` source set.

```
app/
├── src/
│   ├── main/java/com/edts/mobile/
│   │   └── feature/home/
│   │       ├── HomeViewModel.kt
│   │       └── HomeScreen.kt
│   └── test/java/com/edts/mobile/
│       └── feature/home/
│           └── HomeViewModelTest.kt     # Mirror of HomeViewModel.kt
```

- **One test file per class rule**: Each test class must test exactly one class and be named `<ClassName>Test.kt`.
- **Exclusion**: Do **not** test `@Composable` functions, `Activity`, `Fragment`, or third-party libraries (e.g. Retrofit, Room). Focus only on testing ViewModels, UseCases/Interactors, Repositories, Mappers, and pure utility classes.

---

## 3. MainDispatcherRule Helper

Since JVM unit tests lack a main Looper/UI thread, you **must** inject a custom `TestDispatcher` using the JUnit4 `TestWatcher` rule.

```kotlin
package com.edts.mobile.test

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.rules.TestWatcher
import org.junit.runner.Description

@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    val testDispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

---

## 4. Test Structure & Naming Conventions

- **Name SUT as `sut`**: The System Under Test must be declared as a variable named `sut`.
- **Test method naming**: Test functions must use backtick syntax naming in the exact format:
  ```kotlin
  `methodName returns expectedResult when condition`
  ```
  *(Example: `` `fetchData returns error when network fails` ``)*
- **AAA Pattern**: Every test body must contain three clearly commented sections:
  ```kotlin
  // Arrange -> Setup mocks, data, and preconditions
  // Act     -> Invoke the target method on the SUT
  // Assert  -> Verify outcomes and verify interactions using Google Truth
  ```

---

## 5. ViewModel Test Template (StateFlow / Compose)

```kotlin
package com.edts.mobile.ui.feature.home

import com.google.common.truth.Truth.assertThat
import io.mockk.*
import io.mockk.impl.annotations.MockK
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.test.runTest
import org.junit.Before
import org.junit.Rule
import org.junit.Test
import org.junit.runner.RunWith
import org.junit.runners.JUnit4
import com.edts.mobile.core.util.Resource
import com.edts.mobile.domain.use_case.GetProductsUseCase
import com.edts.mobile.test.MainDispatcherRule

@RunWith(JUnit4::class)
@OptIn(ExperimentalCoroutinesApi::class)
class HomeViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @MockK
    lateinit var getProductsUseCase: GetProductsUseCase
    
    // SUT
    private lateinit var sut: HomeViewModel

    @Before
    fun setUp() {
        MockKAnnotations.init(this)
        clearAllMocks()
        sut = HomeViewModel(getProductsUseCase)
    }

    @Test
    fun `loadProducts updates state to success when use case emits success`() = runTest {
        // Arrange
        val mockProducts = listOf(Product("1", "Product A"))
        coEvery { getProductsUseCase.execute() } returns flowOf(Resource.Success(mockProducts))

        // Act
        sut.loadProducts()

        // Assert
        assertThat(sut.state.value.isLoading).isFalse()
        assertThat(sut.state.value.products).isEqualTo(mockProducts)
        coVerify(exactly = 1) { getProductsUseCase.execute() }
    }
}
```

---

## 6. Repository Test Template

```kotlin
package com.edts.mobile.data.repository

import com.google.common.truth.Truth.assertThat
import io.mockk.*
import io.mockk.impl.annotations.MockK
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.test.runTest
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith
import org.junit.runners.JUnit4
import com.edts.mobile.data.source.remote.InfoRemoteDataSource
import com.edts.mobile.data.mapper.InfoMapper
import com.edts.mobile.core.util.Resource

@RunWith(JUnit4::class)
@OptIn(ExperimentalCoroutinesApi::class)
class GetInfoRepositoryTest {

    @MockK
    lateinit var remoteDataSource: InfoRemoteDataSource
    
    // Mappers are simple stateless helpers, use the real instance
    private val mapper: InfoMapper = InfoMapper.INSTANCE
    
    private lateinit var sut: GetInfoRepository

    @Before
    fun setUp() {
        MockKAnnotations.init(this)
        clearAllMocks()
        sut = GetInfoRepository(remoteDataSource, mapper)
    }

    @Test
    fun `getInfo returns success when remote call succeeds`() = runTest {
        // Arrange
        val response = InfoResponse(apiField = "Value")
        coEvery { remoteDataSource.fetchInfo(any()) } returns response

        // Act
        val result = sut.getInfo("id").first()

        // Assert
        assertThat(result).isInstanceOf(Resource.Success::class.java)
        assertThat((result as Resource.Success).data?.domainField).isEqualTo("Value")
        coVerify(exactly = 1) { remoteDataSource.fetchInfo("id") }
    }
}
```

---

## 7. Rules & Anti-Patterns

1. **No `runBlocking`**: Wrap async test bodies only in `runTest {}` to support time-skipping features.
2. **Suspend functions**: Use `coEvery` to stub and `coVerify` to assert against suspend functions.
3. **No Thread.sleep**: Never sleep threads. For time delay testing, use `advanceTimeBy(ms)` or `advanceUntilIdle()` inside `runTest`.
4. **Mock reset discipline**: Reset mocks before each test via `clearAllMocks()` in the `@Before` setup block.
5. **Real instances for helpers**: Do not mock simple stateless helper classes (such as MapStruct mappers or math util classes); instantiate and use real objects.
6. **No logic leaks**: Never write conditionals (`if`/`else`) or loops (`for`/`while`) in unit tests. Keep execution strictly linear.
7. **Definition of Done (DoD)**:
   - Branch test coverage **must** be > 80% for the class.
   - You must test at least three distinct path scenarios: Happy path, Error/Exception path, and Empty/Loading state.
8. **100% Kotlin**: All test classes must be written in Kotlin. Java tests are forbidden.
