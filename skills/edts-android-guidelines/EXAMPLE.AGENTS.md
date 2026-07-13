# Android Project Guide

> This file provides guidance to AI agents when working with code in this repository. Keep project-specific setup here and link detailed engineering standards to the EDTS Android rule files.

---

## Project Identity

```text
Application Name    : [PROJECT_NAME]
Package Name        : [PACKAGE_NAME]
Min SDK             : [MIN_SDK]
Target SDK          : [TARGET_SDK]
Compile SDK         : [COMPILE_SDK]
Version Name        : 1.0.0
Version Code        : 1
```

---

## Build Flavors

| Flavor | Application ID | Environment |
| --- | --- | --- |
| `development` | `edts.[PROJECT_NAME].dev` | Development |
| `uat` | `edts.[PROJECT_NAME].uat` | UAT |
| `production` | `edts.[PROJECT_NAME].android` | Production |

Update this table to match the actual target project's Gradle configuration.

---

## Technology Stack

| Layer | Technology | Rule |
| --- | --- | --- |
| Project detection | View-based vs Compose, single vs multi-module | `rules/1-project-detection.md` |
| View-based UI | XML + ViewBinding, Koin, LiveData, Glide | `rules/3-view-based.md` |
| Compose UI | Jetpack Compose, Hilt, StateFlow, Coil, Navigation 3 | `rules/4a-compose-multi-module.md` or `rules/4b-compose-single-module.md` |
| Networking | Retrofit + OkHttp + EDTSKU `BaseDataSource` | `rules/5-networking.md` |
| Mapping | MapStruct and simple Kotlin extension mappers | `rules/6-mapping.md` |
| Local storage | Room, KSP, EDTSKU local sources | `rules/7-local-storage.md` |
| Code quality | Kotlin conventions, Gradle, resources, coroutine rules | `rules/8-code-quality.md` |
| Security | Secrets, token storage, TLS, sensitive files | `rules/9-security.md` |
| Testing | JUnit4, MockK, Truth, Coroutines Test, Turbine | `rules/10-unit-testing.md` |
| Review | Code review checklist and Git safety | `rules/11-code-review.md` |

---

## Mandatory Reading Order

1. Read `rules/1-project-detection.md`.
2. Read the matching architecture rule:
   - View-based: `rules/3-view-based.md`
   - Compose multi-module: `rules/4a-compose-multi-module.md`
   - Compose single-module: `rules/4b-compose-single-module.md`
3. Read the layer rule for the code being changed: networking, mapping, local storage, code quality, security, or testing.
4. For reviews, read `rules/11-code-review.md` before writing findings.

---

## Project Commands

Adapt these commands to the target project's real flavors and modules:

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

---

## Agent Workflow

- Log every AI-made change to `ai.log` in the project root using `[YYYY-MM-DD HH:MM:SS] <brief description>`.
- Ask before changing ambiguous architecture, dependency, SDK, flavor, or module decisions.
- Do not run `git commit`, `git push`, or destructive file operations unless explicitly requested.
- Before creating a service, datasource, repository, database, mapper, UseCase, or shared component, check whether a matching implementation already exists.
- Preserve project-specific setup notes, environment variables, build flavors, and credential instructions.

---

## Important Local References

```text
gradle/libs.versions.toml
build-logic/
app/src/main/AndroidManifest.xml
app/build.gradle.kts
settings.gradle.kts
```
