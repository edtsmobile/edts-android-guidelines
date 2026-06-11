# Jetpack Compose — Multi-Module Architecture

> Use this file when the project is **multi-module** and uses **Jetpack Compose**. For single-module Compose projects, use [3b-compose-single-module.md](3b-compose-single-module.md) instead.

---

## Workspace Layout & Dependencies

```
                     ┌───────────┐
                     │   :app    │ (Thin launcher)
                     └─────┬─────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
      ┌─────────────┐┌─────────────┐┌─────────────┐
      │:feature-home││:feature-cart││  :core-nav  │
      └──────┬──────┘└──────┬──────┘└─────────────┘
             │              │
             └──────┬───────┘
                    ▼
             ┌─────────────┐
             │ :core-data  │ (Repository implementations)
             └──────┬──────┘
                    ▼
             ┌─────────────┐
             │:core-domain │ (Use cases & Domain entities)
             └─────────────┘
```

- **`:app`**: Root module, initializes Koin and hosts the top-level `NavHost`.
- **`:feature-xxx`**: Specific feature modules, containing Presentation code (`@Composable`, `ViewModel`, Koin feature modules).
- **`:core-nav`**: Interface module for cross-feature navigation.
- **`:core-data`**: Network sources, local databases, MapStruct mappers, and repository implementations.
- **`:core-domain`**: Pure Kotlin module with domain models, repository interfaces, and use case interactor classes.

---

## 1. BaseComposeActivity Template

Activities in Compose modules act as thin hosts that set up the content view and delegate navigation to Compose `NavHost`. They must extend `BaseComposeActivity`.

```kotlin
package com.edts.mobile.core.ui.base

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.runtime.Composable

abstract class BaseComposeActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Content()
            }
        }
    }

    @Composable
    abstract fun Content()
}
```

---

## 2. ScreenState Immutable Data Class Template

Screen state must be modeled as a single, immutable Kotlin `data class`.

```kotlin
package com.edts.mobile.feature_home.presentation.state

import com.edts.mobile.core_domain.model.ProductModel

data class HomeScreenState(
    val isLoading: Boolean = false,
    val products: List<ProductModel> = emptyList(),
    val errorMessage: String? = null
)
```

---

## 3. ViewModel StateFlow Template

ViewModels in Compose projects must expose state via `StateFlow` and trigger asynchronous actions inside `viewModelScope.launch`.

```kotlin
package com.edts.mobile.feature_home.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import com.edts.mobile.core_domain.use_case.home.GetProductsUseCase
import com.edts.mobile.feature_home.presentation.state.HomeScreenState
import com.edts.mobile.core.util.Resource

class HomeViewModel(
    private val getProductsUseCase: GetProductsUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(HomeScreenState())
    val state: StateFlow<HomeScreenState> = _state.asStateFlow()

    fun loadProducts() {
        viewModelScope.launch {
            getProductsUseCase.execute().collect { resource ->
                _state.update { currentState ->
                    when (resource) {
                        is Resource.Loading -> currentState.copy(isLoading = true)
                        is Resource.Success -> currentState.copy(
                            isLoading = false,
                            products = resource.data ?: emptyList()
                        )
                        is Resource.Error -> currentState.copy(
                            isLoading = false,
                            errorMessage = resource.message
                        )
                    }
                }
            }
        }
    }
}
```

---

## 4. @Composable Screen Template (with koinViewModel())

Composable screens must inject their ViewModels using Koin's `koinViewModel()`. For preview and testability, separate the stateful screen from the stateless content layout.

```kotlin
package com.edts.mobile.feature_home.presentation.screen

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import org.koin.androidx.compose.koinViewModel
import com.edts.mobile.feature_home.presentation.viewmodel.HomeViewModel
import com.edts.mobile.feature_home.presentation.state.HomeScreenState

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = koinViewModel(),
    onNavigateToDetail: (productId: String) -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.loadProducts()
    }

    HomeScreenContent(
        state = state,
        onProductClick = onNavigateToDetail
    )
}

@Composable
private fun HomeScreenContent(
    state: HomeScreenState,
    onProductClick: (String) -> Unit
) {
    // Stateless Compose UI Layout
}
```

---

## 5. Reusable Components & Naming (`Comp` suffix)

All reusable compose components must use the suffix `Comp` (e.g. `ProductCardComp.kt`, `PrimaryButtonComp.kt`).
- If feature-specific: Place in `feature-[name]/src/main/java/[package]/[feature]/component/`.
- If shared: Place in `core-resource/src/main/java/[package]/component/`.

---

## 6. Shimmer Loading States

Every page displaying remote data **must** implement a shimmer loading state (never a simple spinner or empty blank page). Use the project's standard `ShimmerComp` or shimmer Modifier:

```kotlin
@Composable
fun ProductsLoadingComp() {
    Column {
        repeat(5) {
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(80.dp)
                    .shimmerModifier() // Project-wide shimmer modifier
            )
        }
    }
}
```

---

## 7. DI configuration (Koin 4.1.0 DSL)

Define repository and usecase modules in `core-domain` / `core-data` structures, and register ViewModels inside feature modules.

```kotlin
// In :feature-home Koin module
val featureHomeModule = module {
    viewModel { HomeViewModel(get()) }
}

// In :core-data Koin module
val coreDataModule = module {
    single { InfoMapper.INSTANCE }
    singleOf(::ProductRepository) bind IProductRepository::class
}

// In :core-domain Koin module
val coreDomainModule = module {
    factoryOf(::GetProductsInteractor) bind GetProductsUseCase::class
}
```

---

## 8. MapStruct @Mapper Usage

MapStruct interfaces belong to `:core-data` in the `mapper` package.

```kotlin
package com.edts.mobile.core_data.mapper

import org.mapstruct.Mapper
import org.mapstruct.factory.Mappers
import com.edts.mobile.core_data.model.ProductResponse
import com.edts.mobile.core_domain.model.ProductModel

@Mapper
interface ProductMapper {
    companion object {
        val INSTANCE: ProductMapper = Mappers.getMapper(ProductMapper::class.java)
    }

    fun mapResponseToModel(response: ProductResponse): ProductModel
}
```

---

## 9. ModuleNavigator for Cross-Feature Navigation

Compose modules must not direct-link feature code. Define the navigation contract in `:core-nav`:

```kotlin
package com.edts.mobile.navigation

import androidx.navigation.NavGraphBuilder

interface ModuleNavigator {
    fun registerGraph(navGraphBuilder: NavGraphBuilder)
}
```

---

## 10. Compose Previews

When writing `@Preview` blocks, pass state directly or construct mock/fake parameters. **Never** attempt to call Koin inject or make active network calls inside previews. Use fake UseCase implementations if necessary:

```kotlin
class FakeGetProductsUseCase : GetProductsUseCase {
    override fun execute() = flowOf(Resource.Success(listOf(ProductModel("1", "Fake Item"))))
}
```

---

## Rules

1. **No direct XML**: Do not inflate XML layouts in Compose code. UI must be 100% Compose.
2. **Immutable State**: State must be defined as an immutable `data class` updated only with `.copy()`.
3. **StateFlow for state**: ViewModels must expose state via `StateFlow` and backing mutable state via `MutableStateFlow`.
4. **Koin injection**: Use `koinViewModel()` inside Screen composables, not standard `getViewModel()` or manually instantiated ViewModels.
5. **No direct feature imports**: Navigate only via navigation modules/contracts, never import other feature modules directly.
6. **Previews**: Ensure previews run locally by avoiding Koin dependency lookup in preview code blocks.
7. **Resource check**: Before writing any new networking or repository class, inspect existing files. If matching services or repositories exist, ask the developer: *"I found an existing `<FileName>` — should I add to that file or create a new one?"* Wait for developer confirmation.
