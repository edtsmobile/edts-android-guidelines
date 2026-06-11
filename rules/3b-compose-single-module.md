# Jetpack Compose — Single-Module Architecture

> [!WARNING]
> The single-module approach (monolithic structure) is less battle-tested than the multi-module approach for large EDTS applications. Use this approach only for smaller applications, utilities, or proof-of-concept projects. Do not mix it with multi-module structures.

---

## Folder Structure

In a single-module project, all files live within the main application module (typically `:app`), structured into `data/`, `domain/`, and `ui/` layers under the root package namespace.

```
app/src/main/java/[package]/
├── App.kt                           # Application class initializing Koin
├── AppModule.kt                     # Central Koin DI module registry
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

## 1. Application class & Koin Initialization

DI configuration is declared in a central `AppModule.kt` file and initialized in the custom `Application` class.

### App.kt

```kotlin
package com.edts.mobile

import android.app.Application
import org.koin.android.ext.koin.androidContext
import org.koin.android.ext.koin.androidLogger
import org.koin.core.context.startKoin

class App : Application() {
    override fun onCreate() {
        super.onCreate()
        
        startKoin {
            androidLogger()
            androidContext(this@App)
            modules(appModule)
        }
    }
}
```

### AppModule.kt

```kotlin
package com.edts.mobile

import org.koin.core.module.dsl.viewModelOf
import org.koin.core.module.dsl.factoryOf
import org.koin.core.module.dsl.singleOf
import org.koin.dsl.bind
import org.koin.dsl.module
import com.edts.mobile.data.repository.ProductRepository
import com.edts.mobile.domain.repository.IProductRepository
import com.edts.mobile.domain.use_case.GetProductsUseCase
import com.edts.mobile.domain.use_case.GetProductsInteractor
import com.edts.mobile.ui.feature.home.HomeViewModel

val appModule = module {
    // Data Sources & Mappers
    single { InfoMapper.INSTANCE }
    
    // Repositories
    singleOf(::ProductRepository) bind IProductRepository::class
    
    // Use Cases
    factoryOf(::GetProductsInteractor) bind GetProductsUseCase::class
    
    // ViewModels
    viewModelOf(::HomeViewModel)
}
```

---

## 2. ViewModel & StateFlow Pattern

ViewModels in single-module projects must follow the exact same `StateFlow<ScreenState>` patterns as multi-module projects, using a single immutable state class updated via state copying.

```kotlin
package com.edts.mobile.ui.feature.home

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import com.edts.mobile.domain.use_case.GetProductsUseCase
import com.edts.mobile.core.util.Resource

data class HomeScreenState(
    val isLoading: Boolean = false,
    val items: List<Product> = emptyList()
)

class HomeViewModel(
    private val getProductsUseCase: GetProductsUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(HomeScreenState())
    val state: StateFlow<HomeScreenState> = _state.asStateFlow()

    fun load() {
        viewModelScope.launch {
            getProductsUseCase.execute().collect { resource ->
                _state.update { currentState ->
                    when (resource) {
                        is Resource.Loading -> currentState.copy(isLoading = true)
                        is Resource.Success -> currentState.copy(isLoading = false, items = resource.data ?: emptyList())
                        is Resource.Error -> currentState.copy(isLoading = false)
                    }
                }
            }
        }
    }
}
```

---

## 3. Composable Screen UI & koinViewModel()

Inject ViewModels using `koinViewModel()` in screen-level composables. Reusable layout pieces must be extracted to the `ui/component/` directory and use the `Comp` suffix.

```kotlin
package com.edts.mobile.ui.feature.home

import androidx.compose.runtime.*
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import org.koin.androidx.compose.koinViewModel
import com.edts.mobile.ui.component.ProductItemComp

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = koinViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.load()
    }

    // Pass state down to a stateless layout or component
}
```

---

## Rules

1. **Layer Directories**: Do not mix files outside of `data/`, `domain/`, and `ui/` folder packages.
2. **Koin Declarations**: Register all classes in `AppModule.kt`. Do not use manual instantiation in code components.
3. **Component Naming**: All reusable widgets must live in `ui/component/` and end with `Comp.kt`.
4. **State Management**: Modify UI state strictly via `_state.update { it.copy(...) }`. Direct mutable property bindings inside ViewModels are forbidden.
5. **No direct ViewModel instantiation**: Screen Composables must resolve ViewModels only through `koinViewModel()`.
6. **Resource check**: Before adding a new repository, database, or API service file, check if a matching implementation already exists. If yes, ask: *"I found an existing `<FileName>` — should I add to that file or create a new one?"* Wait for developer confirmation.
