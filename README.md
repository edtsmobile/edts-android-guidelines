# EDTS Android Guidelines

Mandatory development guidelines, folder structures, coding standards, and security requirements for EDTS Android projects. These rules ensure consistency, safety, and code quality across all target repositories.

---

## 🚀 Onboarding & Architecture Generations

EDTS Android projects are classified into two generations. Developers and AI agents MUST identify the project's generation before writing or refactoring any code.

| Layer                    | **Gen 1 (View-based)**    | **Gen 2 (Jetpack Compose)**                   |
| ------------------------ | ------------------------- | --------------------------------------------- |
| **UI Framework**         | XML Layouts + ViewBinding | Jetpack Compose                               |
| **Dependency Injection** | Koin                      | Hilt                                          |
| **Async & Reactive**     | LiveData + Callbacks      | Coroutines + StateFlow                        |
| **Image Loading**        | Glide                     | Coil                                          |
| **Navigation**           | `ModuleNavigator`         | Compose Navigation 3 (`androidx.navigation3`) |

---

## 📖 Guideline Reading Order

When boarding a new project, read the guideline rules in the following sequence to build appropriate context:

1. **Onboarding**:
   - Project type detection: read **[Project Detection](rules/1-project-detection.md)**.
   - Agent bootstrap configuration: read **[Agent Bootstrap Guide](rules/2-init-general-rule.md)**.
2. **Architectural Blueprints**:
   - For legacy/View-based apps: read **[View-Based Architecture](rules/3-view-based.md)**.
   - For multi-module Compose apps: read **[Compose Multi-Module Architecture](rules/4a-compose-multi-module.md)**.
   - For simpler/single-module Compose apps: read **[Compose Single-Module Architecture](rules/4b-compose-single-module.md)**.
3. **Layer Development**:
   - Networking & API layer: read **[Networking & API Layer Guidelines](rules/5-networking.md)**.
   - Data mapper conventions: read **[Data Mapping Guidelines](rules/6-mapping.md)**.
   - Local storage: read **[Local Storage Rules](rules/7-local-storage.md)**.
4. **Coding Standards**:
   - Conventions and style: read **[Code Quality](rules/8-code-quality.md)**.
   - Obfuscation and safety: read **[Security Guidelines](rules/9-security.md)**.
5. **Quality Gates**:
   - Unit testing: read **[Unit Testing Requirements](rules/10-unit-testing.md)**.
   - Code review checklist: read **[Code Review Rules](rules/11-code-review.md)**.
   - Git compliance & safety: read **[Git Safety Rules](rules/11-code-review.md#7-git-safety)**.

---

## 🛠️ Bootstrapping Target Projects

An AI agent working on a target repository must verify the presence of `CLAUDE.md` in the project root. To bootstrap or align standard instructions:

```bash
# Run detection and generate/merge the standard CLAUDE.md template rules
npx skills run edts-android init-general-rule
```

---

## 🔄 OpenSpec Development Workflow

This repository uses OpenSpec to model capability changes. When proposing updates, bug fixes, or new guidelines, follow this lifecycle:

1. **Propose a Change**: Create proposals, specs, design documents, and tasks.
   ```bash
   /opsx-propose <change-name>
   ```
2. **Apply the Change**: Implement code or documentation changes systematically.
   ```bash
   /opsx-apply
   ```
3. **Verify the Change**: Cross-reference the implementation with requirements and scenarios.
   ```bash
   /opsx-verify
   ```
4. **Archive & Sync**: Sync spec modifications to main spec definitions and archive the change proposal.
   ```bash
   /opsx-archive
   ```

---

## 📦 Guideline Installation & Management

Ensure Node.js is installed locally, then run the appropriate command:

```sh
# Add the guidelines skill
npx skills add edtsmobile/edts-android-guidelines

# Update the guidelines skill
npx skills update edts-android-guidelines

# Remove the guidelines skill
npx skills remove edts-android-guidelines
```

---

## 📦 How to Use

Add this to your AGENTS.md or CLAUDE.md in EDTS Android projects.


```sh
# EDTS Android Project
Use `edts-android-guidelines` skill.
```
