# Guidelines Backlog

> Issues, flaws, and missing guidelines discovered during project review.
> Each item is a candidate for a future `/opsx-propose` change.

---

## 🔴 Bugs (Fix Soon)

### BUG-1: `koinViewModel()` check in Compose code review section
- **File**: `rules/7-code-review.md` — Section 3, Compose Review, line 31
- **Problem**: The Compose review checklist says *"Verify Composable screens use `koinViewModel()`"*, but Gen 2 (Compose) uses **Hilt** (`hiltViewModel()`), not Koin. This would cause a reviewer to incorrectly flag valid Gen 2 Hilt code as a violation.
- **Fix**: Replace `koinViewModel()` with `hiltViewModel()` in the Compose review checklist.
- **Severity**: High — actively misleading for code reviewers.

---

## 🟡 Incomplete / Partial Coverage

### GAP-1: `CLAUDE.md` bootstrap template is slimmer than the reference
- **File**: `rules/9-init-android-claude.md` — Section 2 (template)
- **Problem**: The reference `CLAUDE.md` is 975 lines and includes a detailed **Module Structure** section, full **EDTSKU Library** integration notes, and project-specific networking setup. The template in rule 9 only covers ~90 lines and uses vague placeholders (`[UI_FRAMEWORK]`, `[DI_MODULE_CONVENTION]`). A bootstrapped `CLAUDE.md` in a target project will be significantly less useful than the reference.
- **Fix**: Expand the template with a richer Module Structure section; clarify which sections are static vs. project-specific; add EDTSKU initialization notes.
- **Severity**: Medium — affects quality of generated `CLAUDE.md` files.

### GAP-2: No migration path from Single-Module (3b) to Multi-Module (3a)
- **File**: `rules/3b-compose-single-module.md`
- **Problem**: Rule 3b explicitly warns that single-module is "less battle-tested" for large apps, but provides no guidance on *when* or *how* to migrate to multi-module (3a). Teams may stay on single-module too long or start mixing patterns.
- **Fix**: Add a brief "When to Graduate to Multi-Module" section to rule 3b with clear trigger signals (e.g., team size, feature count, build time thresholds) and a link to 3a.
- **Severity**: Medium — affects long-term project health decisions.

---

## 🔵 Missing Guidelines

### MISSING-1: No dedicated networking / API layer guideline
- **Files**: Mentioned in `rules/2-view-based.md`, `rules/3a-compose-multi-module.md`, `CLAUDE.md` — but no `rules/10-networking.md` exists.
- **Problem**: The networking layer (`BaseDataSource`, `getResult {}`, `ApiResponse` wrapper, OkHttp interceptors, error handling, certificate pinning setup) is referenced across multiple rules and the CLAUDE.md reference, but there is no single canonical guideline. This is one of the most common sources of bugs in Android apps.
- **Suggested file**: `rules/10-networking.md`
- **Suggested content**: Retrofit setup, `BaseDataSource` usage, `getResult {}` pattern, `ApiResponse`/`Result` wrapper, error handling strategy, interceptors, certificate pinning via `BuildConfig`.
- **Severity**: High — highest practical impact on day-to-day development.

### MISSING-2: No canonical MapStruct guideline
- **Files**: `rules/2-view-based.md`, `rules/3a-compose-multi-module.md`, `rules/5-unit-testing.md`
- **Problem**: MapStruct usage (`@Mapper`, `@Mapping`, `Mappers.getMapper(...)`) appears across three rule files without a dedicated spec. If the team decides to switch strategies (e.g., to manual extension functions or Kotlin-specific mapping), there is no single file to update.
- **Suggested fix**: Extract a `rules/11-mapping.md` (or add a section to `rules/4-code-quality.md`) as the single source of truth for mapping conventions.
- **Severity**: Low — duplication risk, not yet a real problem.

### MISSING-3: Thin `README.md` — poor onboarding experience
- **File**: `README.md` (currently 625 bytes)
- **Problem**: Anyone landing on this repo without prior context cannot understand the two-generation standard, how rules chain together, or which file to open first. New team members or external contributors have no starting point.
- **Suggested fix**: Expand `README.md` to include: overview of Gen 1 vs Gen 2, the rules reading order, how the OpenSpec workflow is used, and how to run `init-android-claude`.
- **Severity**: Low — affects discoverability, not correctness.

---

## 📋 Status

| ID | Title | Priority | Status |
|---|---|---|---|
| BUG-1 | `koinViewModel()` in Compose review | 🔴 High | ✅ Resolved |
| GAP-1 | CLAUDE.md template completeness | 🟡 Medium | Open |
| GAP-2 | Single-module → multi-module migration path | 🟡 Medium | Open |
| MISSING-1 | Networking / API layer guideline | 🔴 High | Open |
| MISSING-2 | Canonical MapStruct guideline | 🔵 Low | Open |
| MISSING-3 | README.md onboarding | 🔵 Low | Open |
