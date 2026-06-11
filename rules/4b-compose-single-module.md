# Jetpack Compose — Single-Module Architecture

> [!WARNING]
> The single-module approach (monolithic structure) is less battle-tested than the multi-module approach for large EDTS applications. Use this approach only for smaller applications, utilities, or proof-of-concept projects. Do not mix it with multi-module structures.

---

## Folder Structure

In a single-module project, all files live within the main application module (typically `:app`), structured into `data/`, `domain/`, and `ui/` layers under the root package namespace.

```
app/src/main/java/[package]/
├── App.kt                           # Application class initializing Hilt (@HiltAndroidApp)
├── di/                              # Dependency Injection modules
│   └── AppModule.kt                 # Central Hilt Module
├── data/                            # Data Layer
│   ├── source/
│   │   ├── local/                   # Room / EncryptedSharedPreferences
│   │   └── remote/                  # Retrofit API Services
│   ├── repository/                  # Repository implementations conforming to domain contracts
│   └── mapper/                      # MapStruct mappers
├── domain/                          # Domain Layer (Pure business logic)
│   ├── model/                       # Domain entities / models
│   ├── repository/                  # Repository interfaces (contracts)
│   └── use_case/                    # UseCase interfaces & Interactor implementations
└── ui/                              # Presentation Layer
    ├── base/                        # BaseComposeActivity, BaseViewModel
    ├── theme/                       # Colors, Type, Shapes, Theme
    ├── component/                   # Reusable components named <Name>Comp
    └── feature/                     # Feature screens, organized by feature sub-folders
        └── [feature_name]/
            ├── [Feature]Screen.kt   # Screen Composable
            └── [Feature]ViewModel.kt# ViewModel using StateFlow
```

---

## 1. Application class & Hilt Initialization

DI configuration is set up using Hilt. Annotate the application class with `@HiltAndroidApp` and declare binding/provider configurations inside `@Module` classes.

### App.kt

```kotlin
package com.edts.mobile

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class App : Application()
```

### di/AppModule.kt

```kotlin
package com.edts.mobile.di

import dagger.Binds
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton
import com.edts.mobile.data.repository.ProductRepositoryImpl
import com.edts.mobile.domain.repository.ProductRepository

@Module
@InstallIn(SingletonComponent::class)
abstract class AppModule {

    @Binds
    @Singleton
    abstract fun bindProductRepository(
        impl: ProductRepositoryImpl
    ): ProductRepository
    
    companion object {
        @Provides
        @Singleton
        fun provideInfoMapper(): InfoMapper = InfoMapper.INSTANCE
    }
}
```

> [!NOTE]
> ViewModels and UseCases do not require manual registration in Hilt modules; they are resolved automatically using `@Inject constructor` and `@HiltViewModel`.

---

## 2. ViewModel & StateFlow (MVI Pattern)

ViewModels in single-module projects must follow the exact same MVI `StateFlow<ScreenState>` patterns as multi-module projects, using a single immutable state class and user intents processed via a `processIntent()` function.

```kotlin
package com.edts.mobile.ui.feature.home

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import com.edts.mobile.domain.use_case.GetProductsUseCase
import com.edts.mobile.core.util.Resource

// 1. UI State
data class HomeScreenState(
    val isLoading: Boolean = false,
    val items: List<Product> = emptyList(),
    val errorMessage: String? = null
)

// 2. User Intent
sealed interface HomeIntent {
    data object LoadProducts : HomeIntent
    data object Refresh : HomeIntent
}

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
                            items = resource.data ?: emptyList()
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

## 3. Composable Screen UI & hiltViewModel()

Inject ViewModels using `hiltViewModel()` from Hilt in screen-level composables. Reusable layout pieces must be extracted to the `ui/component/` directory and use the `Comp` suffix.

```kotlin
package com.edts.mobile.ui.feature.home

import androidx.compose.runtime.*
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.hilt.navigation.compose.hiltViewModel
import com.edts.mobile.ui.component.ProductItemComp

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.processIntent(HomeIntent.LoadProducts)
    }

    // Pass state down to a stateless layout or component
}
```

---

## 4. Compose Navigation (Navigation 3)

In single-module applications, manage screen navigation reactively using Navigation 3 by defining `@Serializable` routes and rendering destinations via `NavDisplay`.

```kotlin
package com.edts.mobile.ui.navigation

import androidx.compose.runtime.Composable
import androidx.navigation3.NavDisplay
import androidx.navigation3.entry
import androidx.navigation3.entryProvider
import androidx.navigation3.rememberNavBackStack
import kotlinx.serialization.Serializable
import com.edts.mobile.ui.feature.home.HomeScreen
import com.edts.mobile.ui.feature.detail.DetailScreen

@Serializable
object HomeRoute

@Serializable
data class DetailRoute(val id: String)

@Composable
fun AppNavigation() {
    val backStack = rememberNavBackStack(initialRoute = HomeRoute)

    NavDisplay(
        backStack = backStack,
        entryProvider = entryProvider {
            entry<HomeRoute> {
                HomeScreen(onNavigateToDetail = { id ->
                    backStack.push(DetailRoute(id))
                })
            }
            entry<DetailRoute> { backStackEntry ->
                val route = backStackEntry.route<DetailRoute>()
                DetailScreen(
                    id = route?.id.orEmpty(),
                    onBack = { backStack.pop() }
                )
            }
        }
    )
}
```

---

## Graduating to Multi-Module

While a single-module structure is simpler to bootstrap, it is a starting architecture. As the application grows, compile-time performance, merge-conflict frequency, and clean architecture enforcement will degrade.

### Graduation Triggers

Teams MUST graduate to a multi-module architecture (conforming to [Jetpack Compose — Multi-Module Architecture](file:///Users/ridhanfadhilah/Public/Fadhil/mobile/android/edts/edts-mobile-guidelines/edts-android-guidelines/rules/4a-compose-multi-module.md)) when any of the following triggers are met:

1. **Feature Scope**: The project exceeds **5 distinct business domains or features** (e.g., Auth, Profile, Home, plus 3 or more functional domains).
2. **Team Size**: More than **3 Android developers** actively commit to the repository concurrently.
3. **Build Performance**: Clean build execution time exceeds **3 minutes** on standard local development machines.
4. **Code Coupling**: Code boundaries leak (e.g., presentation components implicitly import models across domain limits without structural protection).

### Graduation Steps (Out-In Extraction Sequence)

To transition a single-module app to multi-module systematically, follow this extraction sequence:

1. **Core Utilities & Base Models**: Create `:core:model` and `:core:utils`. Move base domain models and pure Kotlin helper classes first.
2. **Design System & UI Resources**: Move `ui/theme/` and shared layout widgets in `ui/component/` to `:core:design-system`.
3. **Data Layer**: Extract local Room databases, Retrofit API definitions, mappers, and repository implementations to `:core:data`.
4. **Navigation Contracts**: Establish the typed route serialization definitions and navigation contract interfaces in `:core:navigation`.
5. **Feature Modules Extraction**: For each subdirectory under `ui/feature/`, extract it into its own `:feature:xxx` module. Each feature module depends on `:core:model`, `:core:data`, and `:core:design-system`.
6. **DI & Dependency Refactoring**: Shift Hilt `@InstallIn` modules to the appropriate new Gradle modules.
7. **Monolithic App Cleanup**: Re-target `:app` to act as the thin compositor shell. `:app` depends on all feature modules and only contains the Application initialization (`App.kt`), the entry Activity, and the root navigation `AppNavigation` coordinator.

---

## Rules

1. **Layer Directories**: Do not mix files outside of `data/`, `domain/`, and `ui/` folder packages.
2. **Hilt Setup**: ViewModels must be annotated with `@HiltViewModel` and constructor-injected with `@Inject constructor`. Declare abstract module interfaces with `@Binds` for interface implementations.
3. **Component Naming**: All reusable widgets must live in `ui/component/` and end with `Comp.kt`.
4. **State Management**: Modify UI state strictly via `_state.update { it.copy(...) }`. Direct mutable property bindings inside ViewModels are forbidden.
5. **No direct ViewModel instantiation**: Screen Composables must resolve ViewModels only through `hiltViewModel()`.
6. **Navigation 3**: Screen navigation must be state-driven using Jetpack Compose Navigation 3 (`androidx.navigation3`). Managing navigation via older event-driven `NavController` properties is prohibited.
7. **Resource check**: Before adding a new repository, database, or API service file, check if a matching implementation already exists. If yes, ask: *"I found an existing `<FileName>` — should I add to that file or create a new one?"* Wait for developer confirmation.
