# Download Butler — Coding Rules & Guidelines

> **Mandatory developer standards** for every code modification, new file creation, and refactoring
> step in the Download Butler Android codebase. These rules apply to OpenCode and all contributors.
>
> Version: 1.0.0
> Last Updated: 2026-07-31
> Companion Documents: [`PROJECT_ARCHITECTURE.md`](./PROJECT_ARCHITECTURE.md),
> [`FEATURE_SPECIFICATIONS.md`](./FEATURE_SPECIFICATIONS.md)

---

## Table of Contents

1. [Architectural & Coding Rules](#1-architectural--coding-rules)
2. [Privacy & Safety Rules](#2-privacy--safety-rules)
3. [Error Handling & Resilience](#3-error-handling--resilience)
4. [UI & Design Standards](#4-ui--design-standards)
5. [Verification & Testing Requirements](#5-verification--testing-requirements)
6. [Git & Commit Standards](#6-git--commit-standards)
7. [Definition of Done](#7-definition-of-done)

---

## 1. Architectural & Coding Rules

### 1.1 Strict MVVM Layering

The app is strictly layered **UI → ViewModel → Use Case → Repository → Data source**. Cross-layer
violations are rejected in review.

| Layer       | Allowed                                              | Forbidden                                                       |
| ----------- | ---------------------------------------------------- | --------------------------------------------------------------- |
| **View** (Compose) | Observing `StateFlow`; emitting UI events to the ViewModel | Business logic, database queries, file I/O, HTTP calls, direct repository access |
| **ViewModel** | Orchestrating use cases/repositories; exposing `StateFlow`; translating events | Android framework calls (Context-free by design), storage of business state outside `StateFlow` |
| **Domain**   | Pure business logic (rename, dedupe, rules, clean)    | Android SDK imports, Room, Retrofit, file-system APIs           |
| **Data**     | Room, MediaStore, file I/O, Retrofit, AI providers    | UI imports, Compose references                                   |

**Mandatory rules:**

- Compose composables **never** call `Room`, `ContentResolver`, `File`, or `Retrofit` directly.
- All actions flow through ViewModels: `uiEvent → viewModel.handle(action) → state update`.
- Views must not hold long-lived references to repositories; everything is funneled through the
  ViewModel's public API.

### 1.2 Asynchronous IO — Never Block the Main Thread

- All file system operations, database transactions, hashing, and HTTP calls **MUST** use
  Coroutines with `Dispatchers.IO` (or a bounded `Dispatchers.Default` for CPU-bound pure logic).
- The Main thread performs **only** composition and event dispatch.
- Correct patterns:

```kotlin
// Data layer: use withContext(Dispatchers.IO)
suspend fun hashFile(path: String): String =
    withContext(Dispatchers.IO) {
        // streaming hash
    }
```

- **Forbidden:** `runBlocking`, `GlobalScope`, `.blockingGet()`, `Thread.sleep()` on the main thread,
  synchronous Retrofit calls, and any `Blocking*` API on the UI path.
- Prefer `viewModelScope.launch { }` for UI-scoped work; cancel scopes on clear.

### 1.3 Immutable State with StateFlow

- ViewModels expose state exclusively through **`StateFlow`** (single source of truth).
- UI state is collected with `collectAsStateWithLifecycle()` — never `.collectAsState()` alone.
- State classes are **immutable** data classes; updates use `.copy()`/`update {}`.

```kotlin
data class DashboardUiState(
    val isScanning: Boolean = false,
    val stats: AppStatistics? = null,
    val items: List<DownloadItem> = emptyList()
)

val uiState: StateFlow<DashboardUiState> = _uiState.asStateFlow()
```

- One-shot events use `SharedFlow` (or a sealed `UiEvent` Channel) — never mutable state exposed
  for one-shot consumption.
- **Forbidden:** exposing `MutableStateFlow` publicly, mutable `var` state in ViewModels, or
  `LiveData`-style imperative updates on the UI thread.

### 1.4 String Externalization

- **Hardcoded user-facing text in UI screens is forbidden.**
- Every user-facing string MUST live in `res/values/strings.xml` and be referenced via
  `stringResource(R.string.xxx)`.
- Dynamic content composes with `%1$s` / `%2$d` placeholders and `stringResource(...)` args.
- Log messages and exception messages (developer-facing) stay in code; anything the user can see
  must be a resource.

```kotlin
// Correct
Text(stringResource(R.string.dashboard_empty_title))

// Forbidden
Text("No files to organize yet")
```

---

## 2. Privacy & Safety Rules

### 2.1 Network Access — Opt-In Only

- **Never initiate a network call unless explicitly triggered by the user for an AI feature request.**
- The app performs zero background network activity by default.
- AI requests must be user-initiated, user-visible (filename metadata previewed), and cancellable.
- Add a defensive check in `AiRepository`: abort if the provider is disabled or the user has not
  approved the request.

### 2.2 Secure Key Management

- **Never hardcode API keys** in source, resources, or commit history.
- API keys are stored in **encrypted preferences** (e.g. `EncryptedSharedPreferences` /
  `androidx.security-crypto`) or injected via `BuildConfig` environment variables for debug builds.
- The Room `ai_provider_config.apiKey` field is the user's local storage of record for runtime use;
  encryption at rest is required. Keys never appear in logs.
- Review checklist before any commit: no `"AIza..."`, `"sk-..."`, or provider tokens in diff.

### 2.3 Safe Deletion

- Before executing any file deletion:
  1. **Verify file existence** (`File.exists()` / SAF URI check).
  2. Confirm the file is still the expected duplicate candidate (re-hash for dedup deletes).
  3. Delete, then **log the deleted byte count** to the local `StatsRepository`
     (increment `totalBytesSaved` + the relevant counter) and write an audit row.
- Deletion is always user-confirmed at the UI layer (dialog) except where an enabled rule explicitly
  authorizes it.

```kotlin
if (file.exists()) {
    val bytes = file.length()
    val deleted = file.delete()           // file.delete() must return true
    if (deleted) statsRepository.recordDeletion(bytes, actionType)
}
```

---

## 3. Error Handling & Resilience

### 3.1 Try-Catch with Graceful Fallbacks

- Wrap **all** file I/O, network calls, and package parsing in `try-catch` blocks with graceful
  fallbacks. No unhandled exceptions may escape the data layer.
- Prefer `Result<T>` / sealed `Outcome<T>` returns over swallowed errors:

```kotlin
sealed interface RenameOutcome {
    data class Success(val result: SmartRenameResult) : RenameOutcome
    data class Failure(val fileName: String, val reason: Throwable) : RenameOutcome
}
```

- Distinguish **recoverable** errors (retry/fallback) from **fatal** errors (propagate as a
  controlled state). Never `catch` and ignore silently — log and surface a state update.
- Log via a crash-safe logger; never log file contents, API keys, or full paths of sensitive files.

### 3.2 SmartRenamer AI Fallback

- If an AI network request **fails or times out**, fall back **seamlessly** to the offline Regex
  parser result — the app must **never crash** and must never leave a file un-renamed because the
  network was down.
- Order of operations:
  1. Attempt offline regex result **first** (instant, deterministic).
  2. If the user enabled AI, request the AI proposal in parallel.
  3. If AI succeeds → present proposal for human approval. If AI fails → the regex result stands.
- Timeouts: AI calls default to **15s**, cancellable, and must never block the organize pipeline.

### 3.3 Defensive Memory Management

- **Always use buffered chunk streams** for file reading and hashing.
- Standard buffer: **64 KB `ByteArray`** (see Feature Spec §5). Never load whole large files into
  memory.
- Verify bounded allocations in review: no `readBytes()` on anything but tiny config files, no
  `readText()` on arbitrary downloads, no unbounded list materialization of directory trees.

```kotlin
val buffer = ByteArray(BUFFER_SIZE_64KB) // 64 * 1024
while (input.read(buffer) != -1) { digest.update(buffer) }
```

---

## 4. UI & Design Standards

### 4.1 Material 3 & Dynamic Color

- Adhere to **Material 3** design principles; use `MaterialTheme` tokens (colors, typography, shapes)
  — no raw `Color(0xFF...)` literals scattered through screens.
- **Dynamic Color** is enabled by default: `dynamicColorScheme(context)` on Android 12+; fall back
  to the static palette below.
- System components (buttons, switches, checkboxes, cards, top bars) come from Material 3; custom
  components must follow the same elevation, tonal, and motion guidance.
- Support both light and dark themes; test contrast in both.

### 4.2 Test Tags on Every Interactive Element

- Every interactive UI element — **Buttons, Switches, Checkboxes, Navigation items, dialogs,
  slider/threshold controls** — MUST include an explicit `Modifier.testTag("...")`.
- Tag naming convention: `<Feature>_<Element>[_<Variant>]`, e.g.:

```kotlin
Button(
    onClick = viewModel::autoOrganize,
    modifier = Modifier.testTag("dashboard_auto_organize_button")
)
Switch(
    checked = rule.isEnabled,
    onCheckedChange = { onToggle(rule) },
    modifier = Modifier.testTag("rule_toggle_${rule.id}")
)
```

- Tags are stable identifiers: never generated from user text, never reordered casually, and added
  for both the element and its confirmation/delete variants.
- Screenshots for Roborazzi tests use the same tags for deterministic capture.

---

## 5. Verification & Testing Requirements

### 5.1 Mandatory Unit Tests

- **Every core logic modification in `domain/` or `data/` must include corresponding unit tests.**
- Tests live in `src/test/java/com/example/...` mirroring the production package:
  - `domain/renamer/SmartRenamerTest.kt`
  - `domain/duplicates/DuplicateDetectorTest.kt`
  - `domain/cleaner/EasyCleanerTest.kt`
  - `domain/rules/RulesEngineTest.kt`
  - `data/repository/*Test.kt`
- Coverage expectations per change:
  - New public function in `domain/` → happy path + primary edge cases.
  - New query/transaction in `data/` → success + failure + empty-input cases.
  - Regex engine changes → table-driven tests for each pattern class (`19xx/20xx`, tags, `SXXEXX`,
    `1x01`, season strings, unicode, collisions).
- UI tests (Robolectric + Roborazzi) are required for new screens or new interactive components.

### 5.2 Test Execution Gate

- **All tests must pass cleanly when executing `./gradlew test`.**
- The full command for a contribution: `./gradlew test` (JVM unit tests + Robolectric/Roborazzi).
- A change is **not complete** until the suite is green. New tests must prove the fix/feature:
  commit with a red suite is rejected; a skipped/disabled test requires explicit justification.
- Keep tests deterministic: fixed data, no network, no real MediaStore in unit tests (fake/DAO
  doubles only).

---

## 6. Git & Commit Standards

- Commit messages are concise and imperative (e.g. `Add streaming hash to DuplicateDetector`).
- Each commit is a single logical change; no mixed refactor + feature commits.
- Never commit API keys, build outputs, `local.properties`, or `.env` files.
- PRs must reference the affected spec sections from
  `DOCS/FEATURE_SPECIFICATIONS.md` / `DOCS/PROJECT_ARCHITECTURE.md`.

---

## 7. Definition of Done

A change is **done** only when all of the following hold:

- [ ] Follows strict MVVM layering (§1.1); no cross-layer leaks.
- [ ] No main-thread blocking IO (§1.2).
- [ ] ViewModel state is `StateFlow`, collected via `collectAsStateWithLifecycle()` (§1.3).
- [ ] All user-facing strings externalized to `strings.xml` (§1.4).
- [ ] No new network call unless user-triggered; no keys hardcoded or logged (§2.1–§2.2).
- [ ] Deletions verify existence, log byte counts to `StatsRepository` (§2.3).
- [ ] File I/O, network, and package parsing wrapped with graceful fallbacks; AI falls back to regex
      (§3.1–§3.2).
- [ ] Buffered 64 KB chunked reads used for file hashing/reading (§3.3).
- [ ] Material 3 + dynamic color followed; every interactive element has a `Modifier.testTag` (§4).
- [ ] Unit tests added for every `domain/`/`data/` change; `./gradlew test` passes cleanly (§5).

---

## Appendix

### Change Log

| Date       | Version | Notes                                            |
| ---------- | ------- | ------------------------------------------------ |
| 2026-07-31 | 1.0.0   | Initial developer standards for Download Butler. |

*These rules are binding. Deviations require a documented, reviewed exception. When in doubt, prefer
the stricter interpretation.*
