---
name: edts-android-guidelines
description: Mandatory architecture, folder-structure, naming, and code-quality rules for EDTS Android projects. Use this whenever reading, writing, reviewing, or refactoring Kotlin/XML code in an EDTS Android app or module.
---

# EDTS Android Development Guidelines

You are working on an **EDTS Android project**. These guidelines are mandatory — follow them without being reminded.

**First, always detect the project type** (View-based vs. Jetpack Compose, Single-Module vs. Multi-Module) using [rules/1-project-detection.md](rules/1-project-detection.md), then apply the matching rules below.

| File | Topic |
| --- | --- |
| [rules/1-project-detection.md](rules/1-project-detection.md) | Detecting View-based vs. Compose and Single vs. Multi-Module |
| [rules/2-view-based.md](rules/2-view-based.md) | View-based Architecture — structure, base classes, DI, mapping |
| [rules/3a-compose-multi-module.md](rules/3a-compose-multi-module.md) | Jetpack Compose Multi-Module Architecture |
| [rules/3b-compose-single-module.md](rules/3b-compose-single-module.md) | Jetpack Compose Single-Module Architecture |
| [rules/4-code-quality.md](rules/4-code-quality.md) | Naming conventions and Kotlin code quality standards |
| [rules/5-unit-testing.md](rules/5-unit-testing.md) | Unit testing rules and patterns (JUnit4 + MockK + Google Truth) |
| [rules/6-security.md](rules/6-security.md) | Security rules (tokens, storage, networking) |
| [rules/7-code-review.md](rules/7-code-review.md) | Code review checklist and reporting rules |

## AI Workflow Rules

These rules govern how you operate when assisting in an EDTS Android project.

- **Log every AI-made change** to `ai.log` in the project root — one line per change point, format: `[YYYY-MM-DD HH:MM:SS] <brief description>`. Create the file if it does not exist; never overwrite existing entries.
- **Ask the user about any unclear decision** before proceeding. Do not assume ambiguous requirements, architecture choices, or scope. A clarifying question is always cheaper than a wrong implementation.
