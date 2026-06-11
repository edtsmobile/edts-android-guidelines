# Jetpack Compose — Multi-Module Architecture

> Use this file when the project is **multi-module** and uses **Jetpack Compose**. For single-module Compose projects, use [4b-compose-single-module.md](4b-compose-single-module.md) instead.

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

- **`:app`**: Root module, initializes Hilt and hosts the top-level `NavDisplay` back stack.
- **`:feature-xxx`**: Specific feature modules, containing Presentation code (`@Composable`, `ViewModel`, Hilt feature modules).
- **`:core-nav`**: Module declaring the shared `@Serializable` route contracts.
- **`:core-data`**: Network sources, local databases, MapStruct mappers, and repository implementations.
- **`:core-domain`**: Pure Kotlin module with domain models, repository interfaces, and use case classes.

---

## 1. BaseComposeActivity Template

Activities in Compose modules act as thin hosts that set up the content view and delegate navigation to Compose Navigation 3 (`NavDisplay`). They must extend `BaseComposeActivity` and be annotated with Hilt's `@AndroidEntryPoint`.

```kotlin
package com.edts.mobile.core.ui.base

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.runtime.Composable
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
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

## 2. ScreenState & Intent Templates (MVI Pattern)

Screen state must be modeled as a single, immutable Kotlin `data class`, and user actions modeled as `sealed interface` intents.

```kotlin
package com.edts.mobile.feature_home.presentation.state

import com.edts.mobile.core_domain.model.ProductModel

// Screen State
data class HomeScreenState(
    val isLoading: Boolean = false,
    val products: List<ProductModel> = emptyList(),
    val errorMessage: String? = null
)

// UI Intents
sealed interface HomeIntent {
    data object LoadProducts : HomeIntent
    data object Refresh : HomeIntent
}
```

---

## 3. ViewModel StateFlow Template (Hilt + MVI)

ViewModels in Compose projects must be annotated with `@HiltViewModel`, expose state via `StateFlow`, and process intents using `processIntent()`.

```kotlin
package com.edts.mobile.feature_home.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import com.edts.mobile.core_domain.use_case.home.GetProductsUseCase
import com.edts.mobile.feature_home.presentation.state.HomeScreenState
import com.edts.mobile.feature_home.presentation.intent.HomeIntent
import com.edts.mobile.core.util.Resource

@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getProductsUseCase: GetProductsUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(HomeScreenState())
    val state: StateFlow<HomeScreenState> = _state.asStateFlow()

    fun processIntent(intent: HomeIntent) {
        when (intent) {
            is HomeIntent.LoadProducts -> loadProducts()
            is HomeIntent.Refresh -> loadProducts()
        }
    }

    private fun loadProducts() {
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

## 4. @Composable Screen Template (with hiltViewModel())

Composable screens must inject their ViewModels using Hilt's `hiltViewModel()`. For preview and testability, separate the stateful screen from the stateless content layout.

```kotlin
package com.edts.mobile.feature_home.presentation.screen

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.hilt.navigation.compose.hiltViewModel
import com.edts.mobile.feature_home.presentation.viewmodel.HomeViewModel
import com.edts.mobile.feature_home.presentation.state.HomeScreenState
import com.edts.mobile.feature_home.presentation.intent.HomeIntent

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel(),
    onNavigateToDetail: (productId: String) -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.processIntent(HomeIntent.LoadProducts)
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

## 7. DI Configuration (Hilt Modules)

Register bindings and providers in Hilt modules using `@Module` and `@InstallIn`. ViewModels do not need manual registration.

```kotlin
package com.edts.mobile.core_data.di

import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton
import com.edts.mobile.core_data.repository.ProductRepositoryImpl
import com.edts.mobile.core_domain.repository.ProductRepository

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindProductRepository(
        impl: ProductRepositoryImpl
    ): ProductRepository
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

## 9. Compose Navigation (Navigation 3)

Define feature screens and flow graph entries using Jetpack Compose Navigation 3 with `@Serializable` typed routes and `entry` builders:

```kotlin
package com.edts.mobile.feature_home.navigation

import androidx.navigation3.entry
import kotlinx.serialization.Serializable
import com.edts.mobile.feature_home.presentation.screen.HomeScreen

@Serializable
object HomeRoute

// Feature modules define their screens as Nav3 Navigation Entries
fun homeNavigationEntry(
    onNavigateToDetail: (productId: String) -> Unit
) = entry<HomeRoute> {
    HomeScreen(onNavigateToDetail = onNavigateToDetail)
}
```

---

## 9a. Host App NavDisplay Configuration

The central `:app` module aggregates feature screen entries and renders them using `NavDisplay` observing a `rememberNavBackStack()` back stack:

```kotlin
package com.edts.mobile.navigation

import androidx.compose.runtime.Composable
import androidx.navigation3.NavDisplay
import androidx.navigation3.entryProvider
import androidx.navigation3.rememberNavBackStack
import com.edts.mobile.feature_home.navigation.HomeRoute
import com.edts.mobile.feature_home.navigation.homeNavigationEntry
import com.edts.mobile.feature_detail.navigation.DetailRoute
import com.edts.mobile.feature_detail.navigation.detailNavigationEntry

@Composable
fun AppNavigation() {
    val backStack = rememberNavBackStack(initialRoute = HomeRoute)

    NavDisplay(
        backStack = backStack,
        entryProvider = entryProvider {
            +homeNavigationEntry(onNavigateToDetail = { productId ->
                backStack.push(DetailRoute(productId))
            })
            +detailNavigationEntry(onBack = {
                backStack.pop()
            })
        }
    )
}
```

---

## 10. Image Loading (Coil)

Jetpack Compose screens must use Coil (`AsyncImage`) for loading remote network images.

```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.res.painterResource
import coil3.compose.AsyncImage
import com.edts.mobile.feature_home.R

@Composable
fun ProductImageComp(
    imageUrl: String,
    contentDescription: String?,
    modifier: Modifier = Modifier
) {
    AsyncImage(
        model = imageUrl,
        contentDescription = contentDescription,
        placeholder = painterResource(R.drawable.placeholder_image),
        error = painterResource(R.drawable.error_image),
        contentScale = ContentScale.Crop,
        modifier = modifier
    )
}
```

---

## 11. Compose Previews

When writing `@Preview` blocks, pass state directly or construct mock/fake parameters. **Never** attempt to trigger dependency lookups or make network calls inside previews. If a Composable requires a ViewModel in the preview block, inject a mock ViewModel or use an interface providing preview-friendly mock data.

---

## Rules

1. **No direct XML**: Do not inflate XML layouts in Compose code. UI must be 100% Compose.
2. **Immutable State**: State must be defined as an immutable `data class` updated only with `.copy()`.
3. **StateFlow for state**: ViewModels must expose state via `StateFlow` and backing mutable state via `MutableStateFlow`.
4. **Hilt injection**: Use `hiltViewModel()` inside Screen composables, not standard ViewModels or Koin inject methods.
5. **No direct feature imports**: Navigate only via Navigation 3, pushing serializable routes onto the back stack instead of importing other feature modules directly.
6. **Previews**: Ensure previews run locally by avoiding DI dependency lookup in preview code blocks. Use mock ViewModels or interfaces supplying preview-friendly mock data for preview rendering.
7. **Image Loading**: Always load images using Coil (`AsyncImage`) in `@Composable` contexts. Using Glide or other traditional View-based libraries in Compose layouts is prohibited.
8. **Resource check**: Before writing any new networking or repository class, inspect existing files. If matching services or repositories exist, ask the developer: *"I found an existing `<FileName>` — should I add to that file or create a new one?"* Wait for developer confirmation.
