# Code Quality Rules

## General Rules

- **No force non-null assertion** (`!!`): Avoid using the `!!` operator. Always use safe calls (`?.`), Elvis operator (`?: return`, `?: error(...)`), or `let {}`. The only exception is during DI parameter resolution where a missing dependency is a critical program setup error, which must be clearly commented.
- **Timber for all logging**: Debug and diagnostic logging must use `Timber.d()`, `Timber.e()`, etc. `println()`, `Log.d()`, and other `android.util.Log.*` calls are strictly forbidden.
- **No magic numbers or strings**: Extract all numeric or string literals with domain meaning to named constants.
- **Single responsibility per file**: Each Kotlin file must contain exactly one primary class, object, or interface. Unrelated classes or utilities must be split out.
- **Do not modify build files without permission**: Never modify `build.gradle`, `build.gradle.kts`, `settings.gradle`, or `gradle.properties` unless the developer explicitly requests it. Ask before adding any new dependencies.
- **State updates via copy()**: Update screen state objects in Compose only using `state.value.copy(...)` or `_state.update { it.copy(...) }`. Direct state property mutations are forbidden.
- **Keep Composables small and focused**: Avoid large, monolithic Composable functions. Extract components and layouts into separate files (using the `Comp` suffix) to ensure readability and previewability.
- **Keep ViewModels clean**: ViewModels must not import `android.*` or `androidx.*` classes beyond `ViewModel`, `viewModelScope`, and standard Jetpack LiveData/StateFlow wrappers. They must not contain UI or context dependencies.
- **Android Studio Default Formatting Settings**: All Kotlin and XML files must be formatted using Android Studio's default formatting settings. Run the formatter (`Cmd + Option + L` on macOS, `Ctrl + Alt + L` on Windows/Linux) on all files before committing changes.

---

## Naming Conventions

| Element | Convention | Example |
| --- | --- | --- |
| Files | `PascalCase` matching primary type exactly | `HomeViewModel.kt` |
| Classes / Objects / Interfaces | `PascalCase` | `ProductRepository`, `HomeViewModel` |
| Functions | `camelCase` | `loadProducts()` |
| Variables / Properties | `camelCase` | `productList` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Composable Functions | `PascalCase` | `HomeScreen` |
| Reusable Composable Widgets | `PascalCase` with `Comp` suffix | `ProductCardComp` |
| UI State Classes | `PascalCase + ScreenState` or `PascalCase + UiState` | `HomeScreenState`, `HomeUiState` |
| Koin Modules (Gen 1) | `camelCase + Module` | `homeFeatureModule` |
| Hilt Modules (Gen 2) | `PascalCase + Module` | `NetworkModule`, `RepositoryModule` |
| Mappers | `PascalCase + Mapper` | `ProductMapper` |
| Repository Interfaces (Gen 1) | `I + PascalCase + Repository` | `IGetInfoRepository` |
| Repository Implementations (Gen 1) | `PascalCase + Repository` | `GetInfoRepository` |
| Repository Interfaces (Gen 2) | `PascalCase + Repository` | `ProductRepository` |
| Repository Implementations (Gen 2) | `PascalCase + RepositoryImpl` | `ProductRepositoryImpl` |
| UseCase Interfaces (Gen 1) | `PascalCase + UseCase` | `GetInfoUseCase` |
| UseCase Interactors / Impl (Gen 1) | `PascalCase + Interactor` | `GetInfoInteractor` |
| UseCase Class (Gen 2) | `Verb + noun + UseCase` | `GetProductsUseCase` |
| Room Entity (Gen 2) | `Noun + Entity` | `ProductEntity` |
| DTO / Response (Gen 2) | `Noun + Response` | `ProductResponse` |
| Domain Model (Gen 2) | `Noun only` | `Product` |

