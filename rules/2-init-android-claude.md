# Jetpack Claude Bootstrap Guide (init-android-claude)

> This file defines the mandatory standard bootstrap template for `CLAUDE.md` files within EDTS Android projects.

---

## 1. Project Detection & Bootstrapping

AI agents and developers working on an EDTS Android project MUST ensure that a standard `CLAUDE.md` exists at the root of the repository.

1. **Detection**: Run the detection steps defined in `rules/1-project-detection.md` to identify:
   - **Generation**: View-based (Gen 1) vs Jetpack Compose (Gen 2).
   - **Structure**: Single-Module vs Multi-Module.
2. **Metadata Extraction**: Parse target files (e.g. `app/build.gradle.kts`, `settings.gradle.kts`) to resolve the following variables:
   - `[PROJECT_NAME]`: The name of the project repository (e.g., `membercard`, `wallet`).
   - `[PACKAGE_NAME]`: The application or module namespace (e.g., `edts.membercard.android`).
   - `[MIN_SDK]`, `[TARGET_SDK]`, `[COMPILE_SDK]`: SDK target numbers.
3. **Write/Merge**:
   - If `CLAUDE.md` does not exist, write the standard template below.
   - If `CLAUDE.md` already exists, non-destructively merge the rules and references sections while preserving custom project-specific build flavors, credentials, or custom setup parameters.

---

## 2. Standard CLAUDE.md Template

The generated `CLAUDE.md` file MUST strictly conform to the following layout:

```markdown
# Android Project Guide

> This file provides guidance to AI Agent when working with code in this repository.

---

## Project Identity

```
Application Name    : [PROJECT_NAME]
Package Name        : edts.[PROJECT_NAME].android
Min SDK             : [MIN_SDK]
Target SDK          : [TARGET_SDK]
Compile SDK         : [COMPILE_SDK]
Version Name        : 1.0.0
Version Code        : 1
```

---

## Build Flavors

| Flavor       | Application ID              | Environment |
|--------------|-----------------------------|-------------|
| `staging`    | `edts.[PROJECT_NAME].dev`   | Staging     |
| `aws`        | `edts.[PROJECT_NAME].aws`   | UAT         |
| `production` | `edts.[PROJECT_NAME].android` | Production |

---

## Technology Stack

| Layer        | Technology                                                      | Notes |
|--------------|-----------------------------------------------------------------|-------|
| Language     | **Kotlin**                                                      | Java only for legacy code; do not write new Kotlin-Java hybrids |
| UI           | [UI_FRAMEWORK] (Jetpack Compose / XML ViewBinding)               | Gen 2 (Compose) vs Gen 1 (View-based) |
| Architecture | [ARCH] (MVI / MVVM)                                             | Follow architectural generation standard |
| DI           | [DI_FRAMEWORK] (Hilt / Koin)                                    | Switch depends on UI framework |
| Network      | **Retrofit + OkHttp + Gson**                                    | |
| Database     | **Room**                                                        | Use KSP, not KAPT |
| Async        | **Coroutines + StateFlow**                                      | Use Flow, NOT LiveData for new Compose layers |
| Image        | [IMAGE_LOADER] (Coil / Glide)                                   | Coil for Compose, Glide for View-based |
| Navigation   | [NAV_LIB] (Navigation 3 / ModuleNavigator)                       | Follow cross-feature navigation guideline |
| Build        | **Gradle Kotlin DSL**                                           | `.gradle.kts`, use Version Catalog |
| Testing      | **JUnit4 + MockK + Google Truth + Kotlinx Coroutines Test + Turbine** | Required for ViewModels and UseCases |

---

## Code Conventions (Naming)

| Type | Convention | Example |
|---|---|---|
| Composable | PascalCase, screen name + "Screen" | `HomeScreen` |
| ViewModel | PascalCase + "ViewModel" | `HomeViewModel` |
| UseCase | [USE_CASE_CONVENTION] | Gen 2: `GetProductsUseCase` |
| Repository interface | [REP_INTERFACE_CONVENTION] | Gen 2: `ProductRepository` |
| Repository impl | [REP_IMPL_CONVENTION] | Gen 2: `ProductRepositoryImpl` |
| Room Entity | Noun + "Entity" | `ProductEntity` |
| DTO/Response | Noun + "Response" | `ProductResponse` |
| Domain model | Noun only | `Product` |
| Hilt Module / Koin Module | [DI_MODULE_CONVENTION] | `NetworkModule` / `homeFeatureModule` |

---

## Architecture Rules — MUST FOLLOW

### Clean Architecture Boundaries
```
Presentation  -->  Domain  <--  Data
(ViewModel)       (UseCase)     (Repository Impl)
```
- **FORBIDDEN**: Presentation layer directly imports classes from the Data layer.
- **FORBIDDEN**: UseCases import Android framework classes (`android.*`).
- **FORBIDDEN**: Room Entity is used directly in a ViewModel; always map to a Domain model.

### Formatting
Always format Kotlin and XML files using Android Studio's default settings. Use the default shortcut (`Cmd + Option + L` on macOS, `Ctrl + Alt + L` on Windows/Linux) to format files before committing.

---

## EDTSKU Library — Complete Reference
Standard wrapper methods and setup details for initialization:
- Initialize `EdtsKu` once in `Application.onCreate()`.
- Remote Data Sources MUST extend `BaseDataSource` and use `getResult {}` to wrap API calls.
- Results MUST use `id.co.edtslib.data.Result` wrapper status: `SUCCESS`, `ERROR`, `LOADING`, `UNAUTHORIZED`.
```
