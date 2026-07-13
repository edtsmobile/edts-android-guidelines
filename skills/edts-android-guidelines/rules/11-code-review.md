# Code Review Checklist Rules

Use this checklist whenever reviewing a pull request, patch, or local change set in an EDTS Android project.

## 1. Review Order

1. **Detect project type first** using [rules/1-project-detection.md](1-project-detection.md) to identify whether the project is View-based vs. Compose and Single vs. Multi-Module.
2. **Apply the matching architecture rules** ([rules/3-view-based.md](rules/3-view-based.md) or [rules/4a-compose-multi-module.md](rules/4a-compose-multi-module.md) / [rules/4b-compose-single-module.md](rules/4b-compose-single-module.md)).
3. **Apply code quality, unit testing, and security rules** ([rules/8-code-quality.md](rules/8-code-quality.md), [rules/10-unit-testing.md](rules/10-unit-testing.md), [rules/9-security.md](rules/9-security.md)).
4. **Report findings** before summaries, ordered by severity.
5. **Include file and line references** for every actionable issue.

---

## 2. Architecture & Boundary Checks

- **Verify boundaries**: Reject any business, database, or network logic written in Activities, Fragments, or Composable functions.
- **Data flow**: Verify that repositories only return data models (entities) defined in `core-domain` (or the domain layer), never raw network responses or raw DB models directly to presentation.
- **Koin completeness**: Verify Koin registration for all new ViewModels, UseCases, Repositories, and DataSources. Verify the correct scopes are used (`viewModel`, `factory`, `single`).
- **MapStruct correctness**: Verify response-to-model mapping is handled using MapStruct `@Mapper` interfaces. No manual loop/mapping code.
- **ViewModel thread safety**: Verify that UI updates via `StateFlow` or `LiveData` occur on the Main thread.
- **No direct feature coupling**: In multi-module projects, ensure feature modules navigate using navigation contracts (`ModuleNavigator`) instead of direct imports.

---

## 3. correctness & UI Checklists

### Compose Review
- Ensure UI state is defined as an immutable `data class` updated only with `.copy()`.
- Ensure ViewModels expose state via `StateFlow` and backing state via `MutableStateFlow`.
- Verify Composable screens use `hiltViewModel()` (from `androidx.hilt.navigation.compose`).
- Check that remote data loading screens show a shimmer loading state (`ShimmerComp` or shimmer modifier).

### View-Based Review
- Verify that Activities extend `BaseActivity<VB>` (or project base class) and do not call `findViewById`.
- Verify ViewModels use LiveData/MutableLiveData for outputs observed in Activities/Fragments.
- Verify LiveData observations are restricted to the `setupObserver()` method.

---

## 4. Test Review

- Require JUnit4 + MockK + Google Truth for all new unit tests.
- Verify tests are located in the same relative package path in the `test` source set.
- Check that new/modified public methods with branching logic have at least one test case per branch.
- Verify the SUT is named `sut`.
- Confirm tests follow the backtick descriptive naming convention and use the AAA structure.
- Verify that `@Before` setup resets all mocks.
- Ensure that Compose functions or Android system framework components are not being unit tested.

---

## 5. Security Review

- **Reject hardcoded secrets**: Flag any hardcoded API keys, base URLs, client secrets, or credentials. They must reside in `gradle.properties` (gitignored) and be referenced via `BuildConfig`.
- **Cleartext check**: Reject any addition of `android:usesCleartextTraffic="true"` in production manifest configurations.
- **Token exposure**: Reject logging of tokens, passwords, cookies, or raw PII in logs or diagnostic outputs.
- **Storage check**: Ensure all session tokens, user credentials, and PII are stored via `EncryptedSharedPreferences` or `HttpHeaderLocalSource`.

---

## 6. Review Output Format

- **Format**: Every actionable finding must cite the exact file path and line number in the following format:
  ```
  [SEVERITY] file/path:LineNumber — description and what should change
  ```
- **Severity labels**:
  - `CRITICAL`: Security issues (leaked keys, plain storage for credentials), cleartext traffic.
  - `HIGH`: Architectural boundary violations (networking in views, direct coupling between features), crash risks, incorrect async threading.
  - `MEDIUM`: Quality issues, missing unit tests for public branching logic, incorrect naming, redundant files.
  - `LOW`: Stylistic deviations, minor code organization comments, formatting.
- Lead with findings; omit generic praise.

---

## 7. Git Safety

- **Never run `git push --force` (or `-f`)**: Under no circumstances should raw force pushes be executed on any branch.
- **Force Override Prevention (`--force-with-lease` / `--force-if-includes`)**: If a history rewrite is absolutely necessary on a non-shared branch, developers MUST use `--force-with-lease` and `--force-if-includes` together:
  ```bash
  git push origin <branch-name> --force-with-lease --force-if-includes
  ```
- **Resolving Diverged Remotes**: If a push is rejected because the remote has diverged:
  1. Fetch the latest changes: `git fetch origin`
  2. Rebase or merge local commits: `git rebase origin/<branch-name>`
  3. Resolve conflicts locally and test the build before pushing again.
