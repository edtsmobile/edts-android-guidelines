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

## Coroutines & Flow

- Use `Dispatchers.IO` for blocking I/O or heavyweight database work.
- Keep Flow transformations in repositories or domain use cases, not in ViewModels, when the transformation is data/business logic.
- Use `StateFlow` for durable UI state and `SharedFlow` or channels for one-off events.
- Compose screens must collect state using `collectAsStateWithLifecycle()`, not `collectAsState()`.

```kotlin
fun getProducts(): Flow<List<Product>> = dao.getAll()
    .map { entities -> entities.map { it.toDomain() } }
    .flowOn(Dispatchers.IO)

private val _events = MutableSharedFlow<HomeEvent>()
val events: SharedFlow<HomeEvent> = _events.asSharedFlow()
```

---

## Gradle & Dependencies

- Store all dependency versions in `gradle/libs.versions.toml`; never hardcode versions directly in module `build.gradle.kts` files.
- Use Gradle Kotlin DSL (`.gradle.kts`) and existing convention plugins from `build-logic/` whenever the target project already provides them.
- Do not change `compileSdk`, `targetSdk`, or `minSdk` without explicit developer confirmation.
- Use KSP for supported processors such as Room and MapStruct. KAPT is forbidden for new setup.

```kotlin
plugins {
    alias(libs.plugins.project.android.feature)
    alias(libs.plugins.project.android.hilt)
}
```

---

## Asset & Resource Policy

Before creating new resources, inspect existing project resources and reuse matching names, colors, drawables, strings, typography, and components.

When migrating from a legacy app:

- Copy API endpoints, request/response field names, business rules, validation rules, navigation flows, and user-facing copy.
- Copy all drawables, icons, launcher assets, raw assets, fonts, animations, and exact brand color values unless product/design explicitly approves a change.
- Modernize implementation details such as XML to Compose, Koin to Hilt, Glide/Picasso to Coil for Compose, LiveData to StateFlow/SharedFlow, and Fragment navigation to typed Compose Navigation 3.
- Do not replace product-specific assets with generic Material icons.
- Do not change user-facing strings without product approval.
- Do not hardcode user-facing strings in Kotlin code; use resources or the project's established localization mechanism.

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
