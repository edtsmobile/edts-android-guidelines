# Project Type Detection

Before writing code, detect the UI architecture and project structure from the nearest existing files.

## 1. Use **Jetpack Compose** when nearby files contain:

- `@Composable`
- `setContent {`
- `BaseComposeActivity`
- Composable UI libraries in `build.gradle` / `libs.versions.toml`

## 2. Use **View-based** (XML + ViewBinding) when nearby files contain:

- `ViewBinding` imports or binding variables (e.g., `inflate(inflater)`)
- `.xml` layout files under `res/layout/`
- Classes extending `BaseActivity<VB>` (where `VB` is a ViewBinding class)

## 3. Detect the **structure** (Single-Module vs. Multi-Module)

After identifying the UI library, determine the module layout:

- **Multi-Module**: Check if `settings.gradle` or `settings.gradle.kts` includes multiple module declarations, specifically naming:
  - Core modules: `:core-domain`, `:core-data`, `:core-network`, etc.
  - Feature modules: `:feature-home`, `:feature-login`, etc.
  - → For Compose: follow [3a-compose-multi-module.md](3a-compose-multi-module.md).
  - → For View-based: follow [2-view-based.md](2-view-based.md).
- **Single-Module**: Check if the project has only a single `app` module (monolithic structure) with all source code under one package namespace (typically divided by subpackages like `data/`, `domain/`, `ui/`).
  - → For Compose: follow [3b-compose-single-module.md](3b-compose-single-module.md).

If the layout is genuinely ambiguous (e.g., a new empty repo or mixed signals), **ask the developer** which architecture to use before writing any code. Do not mix single-module and multi-module conventions.
