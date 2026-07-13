# Agent Bootstrap Guide

> This file defines only the bootstrap rules for `CLAUDE.md` and `AGENTS.md` in EDTS Android target projects. Architecture, networking, storage, testing, security, and code quality details live in their dedicated rule files.

---

## 1. Project Detection

Before bootstrapping a target repository, run the detection steps in [1-project-detection.md](1-project-detection.md) to identify:

- **Generation**: View-based (Gen 1) vs Jetpack Compose (Gen 2).
- **Structure**: Single-module vs multi-module.

Use the detected project type to select the relevant architecture rules:

- View-based: [3-view-based.md](3-view-based.md)
- Compose multi-module: [4a-compose-multi-module.md](4a-compose-multi-module.md)
- Compose single-module: [4b-compose-single-module.md](4b-compose-single-module.md)

If the target project is empty or mixed enough that detection is ambiguous, ask the developer before generating or changing project instructions.

---

## 2. Metadata Extraction

Parse the nearest reliable Gradle files, usually `settings.gradle.kts`, `app/build.gradle.kts`, and `gradle/libs.versions.toml`, to resolve:

- `[PROJECT_NAME]`: repository or application name.
- `[PACKAGE_NAME]`: application namespace or package.
- `[MIN_SDK]`, `[TARGET_SDK]`, `[COMPILE_SDK]`: SDK target values.
- Active flavors and application IDs, if already declared.

Preserve existing project-specific build flavors, credentials setup notes, local commands, and custom development setup parameters.

---

## 3. Bootstrap Files

The generated `CLAUDE.md` file must contain exactly:

```markdown
@AGENTS.md
```

The generated or merged `AGENTS.md` file should follow [../EXAMPLE.AGENTS.md](../EXAMPLE.AGENTS.md), then link to the rules below instead of duplicating their content inline:

| Topic | Rule |
| --- | --- |
| Project detection | [1-project-detection.md](1-project-detection.md) |
| View-based architecture | [3-view-based.md](3-view-based.md) |
| Compose multi-module architecture | [4a-compose-multi-module.md](4a-compose-multi-module.md) |
| Compose single-module architecture | [4b-compose-single-module.md](4b-compose-single-module.md) |
| Networking and API layer | [5-networking.md](5-networking.md) |
| Mapping | [6-mapping.md](6-mapping.md) |
| Local storage | [7-local-storage.md](7-local-storage.md) |
| Code quality, Gradle, resources | [8-code-quality.md](8-code-quality.md) |
| Security | [9-security.md](9-security.md) |
| Unit testing | [10-unit-testing.md](10-unit-testing.md) |
| Code review and Git safety | [11-code-review.md](11-code-review.md) |

---

## 4. Merge Behavior

When bootstrapping a target project:

1. If `CLAUDE.md` does not exist:
   - Create `CLAUDE.md` with only `@AGENTS.md`.
   - Create `AGENTS.md` from [../EXAMPLE.AGENTS.md](../EXAMPLE.AGENTS.md), replacing placeholders with detected metadata.
2. If `CLAUDE.md` exists and does not contain `@AGENTS.md`:
   - Move existing `CLAUDE.md` content into `AGENTS.md`.
   - Replace `CLAUDE.md` with only `@AGENTS.md`.
   - Non-destructively merge missing standard sections into `AGENTS.md`.
3. If `CLAUDE.md` already contains `@AGENTS.md`:
   - Non-destructively merge missing standard sections into `AGENTS.md`.

Never overwrite project-specific notes without developer confirmation.

---

## 5. Target Project Commands

When adding standard commands to `AGENTS.md`, adapt these to the target project's real flavors and modules:

```bash
./gradlew assembleDevelopmentDebug
./gradlew assembleProductionRelease
./gradlew bundleUat
./gradlew testDevelopmentDebugUnitTest
./gradlew test
./gradlew connectedDevelopmentDebugAndroidTest
./gradlew lint
./gradlew clean
```
