# Download Butler — Project Architecture

> **Definitive Structural Blueprint** for the Download Butler Android application.
>
> Version: 1.0.0
> Last Updated: 2026-07-31
> Status: Living Document

---

## Table of Contents

1. [Executive Overview](#1-executive-overview)
2. [Tech Stack Specification](#2-tech-stack-specification)
3. [Directory & Package Structure Map](#3-directory--package-structure-map)
4. [Database Schema Specification](#4-database-schema-specification)

---

## 1. Executive Overview

| Attribute     | Value                                                                                     |
| ------------- | ----------------------------------------------------------------------------------------- |
| **Project**   | Download Butler                                                                            |
| **Tagline**   | Privacy-First, Offline-First Automated Download Manager for Android                        |
| **Package**   | `com.example` (matches existing codebase structure)                                        |
| **Platform**  | Android (min SDK / target SDK as configured in the Gradle module)                          |
| **Core Goal** | Automatically monitor, organize, clean, and rename files in the Android **Downloads** folder with minimal user interaction and **zero data tracking**. |

### 1.1 Mission

Download Butler is an opinionated, background-first automation tool. It watches the Android
`Downloads` directory, applies intelligent naming rules to media files, detects and removes
duplicates, inspects and cleans orphaned APK artifacts, and organizes everything into a
user-defined folder structure — all without requiring the user to lift a finger.

### 1.2 Design Principles

- **Privacy-First** — Every operation happens **on-device**. No analytics, no telemetry, no
  third-party SDKs, and no network calls unless the user explicitly enables a **Bring-Your-Own-AI**
  endpoint.
- **Offline-First** — The entire rename, duplicate, and cleanup pipeline works fully offline.
  Network access exists solely as an *optional* enhancement for AI-assisted categorization.
- **Minimal Interaction** — Rules, thresholds, and folders are configured once; the app then runs
  autonomously via background services and scheduled jobs.
- **Deterministic & Auditable** — Every action is logged to the `processed_files_log` table so the
  user can always see exactly what was changed, why, and how much space was reclaimed.

### 1.3 Core Capabilities

1. **Smart Renaming** — Regex-driven engine that parses movie (`Name.2024.1080p.BluRay.x264`)
   and TV (`Show.S01E02.HDTV`) naming conventions into human-friendly titles.
2. **Duplicate Detection** — Streaming hash engine that identifies duplicate files by content,
   regardless of filename.
3. **Automated Cleaning** — APK inspector + temporary-file scanner that reclaims storage safely.
4. **Rule-Based Organization** — A matching pipeline that routes files into target folders based on
   user-defined automation rules.
5. **AI-Assisted Enrichment (Optional)** — BYO-AI provider support for advanced categorization
   (never required, never enabled by default).

---

## 2. Tech Stack Specification

| Concern                   | Technology                                              |
| ------------------------- | ------------------------------------------------------- |
| **Language**              | Kotlin (100% idiomatic, Kotlin-first APIs)              |
| **UI Framework**          | Jetpack Compose + Material 3 (Dynamic Color enabled)    |
| **Architecture**          | MVVM (Model-View-ViewModel) + Clean Architecture        |
| **Local Database**        | Room Database (SQLite abstraction layer)                |
| **Async & Concurrency**   | Kotlin Coroutines & Flow (`StateFlow` / `SharedFlow`)   |
| **Background Execution**  | Android WorkManager + MediaStore `ContentObserver` Service |
| **REST Client**           | OkHttp & Retrofit (BYO-AI endpoints only)               |
| **Testing**               | JUnit4, Robolectric, Roborazzi                          |

### 2.1 Why These Choices

- **Kotlin + Compose + Material 3** — Modern declarative UI with automatic Dynamic Color theming
  for a native, adaptive look with zero tracking.
- **MVVM + Clean Architecture** — Business logic is framework-free and fully unit-testable;
  `domain` never depends on Android, `data` never leaks into UI.
- **Room** — Type-safe SQLite with compile-time query verification and Flow integration.
- **Coroutines / Flow** — Structured concurrency for long-running file operations and reactive
  database observation.
- **WorkManager** — Guaranteed, battery-friendly background execution that survives process death.
- **ContentObserver Service** — Instant reactivity when the Downloads folder changes.
- **OkHttp / Retrofit** — Battle-tested HTTP stack, used *only* for user-configured AI providers.
- **Robolectric + Roborazzi** — Fast JVM-based Android UI/screenshot testing without an emulator.

### 2.2 Dependency Isolation Rules

- `domain` module → **zero** Android / networking / Room dependencies (pure Kotlin + Coroutines).
- `data` module → the only module allowed to touch Room, Retrofit, OkHttp, and file system APIs.
- `ui` module → Compose only; never performs I/O directly (always via ViewModels → use cases → repositories).

---

## 3. Directory & Package Structure Map

### 3.1 Visual Directory Tree

```
com.example/
├── data/
│   ├── model/                          # Plain data structures (DTOs)
│   │   ├── DownloadItem.kt
│   │   ├── SmartRenameResult.kt
│   │   ├── DuplicateGroup.kt
│   │   ├── ApkItemInfo.kt
│   │   ├── CleanUpSummary.kt
│   │   └── AutomationRule.kt
│   ├── database/                       # Room entities, DAOs, singleton
│   │   ├── AutomationRuleEntity.kt
│   │   ├── ProcessedFileLogEntity.kt
│   │   ├── AppStatisticsEntity.kt
│   │   ├── AiProviderConfigEntity.kt
│   │   ├── AutomationRuleDao.kt
│   │   ├── ProcessedFileLogDao.kt
│   │   ├── AppStatisticsDao.kt
│   │   ├── AiProviderConfigDao.kt
│   │   └── DownloadButlerDatabase.kt   # Room Database singleton
│   ├── repository/
│   │   ├── FileRepository.kt
│   │   ├── RulesRepository.kt
│   │   ├── StatsRepository.kt
│   │   └── AiRepository.kt
│   └── ai/                             # BYO-AI provider support
│       ├── AiService.kt
│       ├── GeminiProvider.kt           # Gemini REST (Google)
│       ├── OpenAiCompatibleProvider.kt # OpenAI-compatible APIs
│       ├── OpenRouterProvider.kt       # OpenRouter gateway
│       └── OllamaProvider.kt           # Local Ollama instances
│
├── domain/
│   ├── renamer/
│   │   ├── SmartRenamer.kt             # Regex engine
│   │   ├── MovieRenamePatterns.kt
│   │   └── TvShowRenamePatterns.kt
│   ├── duplicates/
│   │   └── DuplicateDetector.kt        # Streaming hash engine
│   ├── cleaner/
│   │   ├── EasyCleaner.kt
│   │   ├── ApkInspector.kt
│   │   └── TempFileScanner.kt
│   └── rules/
│       └── RulesEngine.kt              # Rule evaluation pipeline
│
├── service/
│   └── DownloadMonitorService.kt       # Background service
│
├── receiver/
│   └── PackageInstallReceiver.kt       # APK install detector
│
└── ui/
    ├── navigation/
    │   └── NavGraph.kt
    ├── screens/
    │   ├── DashboardScreen.kt
    │   ├── FilesScreen.kt
    │   ├── RulesScreen.kt
    │   ├── SettingsScreen.kt
    │   └── StatisticsScreen.kt
    ├── viewmodel/
    │   ├── DashboardViewModel.kt
    │   ├── FilesViewModel.kt
    │   ├── RulesViewModel.kt
    │   ├── SettingsViewModel.kt
    │   └── StatisticsViewModel.kt
    ├── components/
    │   ├── FileCard.kt
    │   ├── StatCard.kt
    │   ├── RuleEditor.kt
    │   └── EmptyStateView.kt
    └── theme/
        ├── Color.kt
        ├── Theme.kt
        └── Type.kt
```

### 3.2 Layer Responsibilities

#### `com.example.data.model` — Data Structures

Immutable, plain Kotlin data classes used to shuttle information between layers. These are the
**on-disk/logic models** — distinct from the Room `Entity` classes (mapping happens inside
repositories).

| Class               | Purpose                                                            |
| ------------------- | ------------------------------------------------------------------ |
| `DownloadItem`      | A single file observed in Downloads (path, name, size, mime, timestamps). |
| `SmartRenameResult` | Outcome of a rename operation (original → new name, matched pattern, status). |
| `DuplicateGroup`    | A group of files sharing identical content hash (candidates for dedup). |
| `ApkItemInfo`       | Metadata about a discovered APK (label, version, install state).    |
| `CleanUpSummary`    | Aggregated result of a cleanup pass (items removed, bytes freed).   |
| `AutomationRule`    | User-defined rule used by the `RulesEngine` for folder routing.     |

#### `com.example.data.database` — Room Persistence

- **Entities** — Four `@Entity` classes mirroring the schema in [Section 4](#4-database-schema-specification).
- **DAOs** — One per entity, exposing `Flow`-based reactive queries plus suspend CRUD.
- **Database Singleton** — `DownloadButlerDatabase` extends `RoomDatabase`, built once via a
  `@Provides`/DI container or application-level singleton, with schema versioning + migrations.

#### `com.example.data.repository` — Data Gateways

The single source of truth for the rest of the app. Repositories mediate between Room, the file
system, the AI layer, and the domain use cases.

| Repository        | Responsibility                                                              |
| ----------------- | --------------------------------------------------------------------------- |
| `FileRepository`  | MediaStore & Downloads folder enumeration, file I/O, rename/delete/move ops. |
| `RulesRepository` | CRUD for automation rules; exposes rules as `StateFlow<List<AutomationRule>>`. |
| `StatsRepository` | Read/write aggregated statistics; emits stats as a `StateFlow`.              |
| `AiRepository`    | Routes classification requests to the configured active AI provider.        |

#### `com.example.data.ai` — BYO-AI Layer

`AiService` is a thin interface; each provider implements it. All providers are **user-configured**
via the `ai_provider_config` table and are **opt-in only**. No provider is called unless:
1. The user has enabled it in Settings, **and**
2. A rule / user action explicitly requests AI-assisted classification.

| Provider                     | Compatibility                                   |
| ---------------------------- | ------------------------------------------------ |
| `GeminiProvider`             | Google Gemini REST (`generativelanguage.googleapis.com`) |
| `OpenAiCompatibleProvider`   | Any OpenAI-compatible `/v1/chat/completions` endpoint |
| `OpenRouterProvider`         | OpenRouter unified API gateway                   |
| `OllamaProvider`             | Local Ollama server (`http://localhost:11434`)   |

#### `com.example.domain.*` — Pure Business Logic

The **heart** of Download Butler. Zero Android dependencies; fully unit-testable on the JVM.

- **`renamer/SmartRenamer`** — A Regex engine exposing two pipelines:
  - `renameMovie(filename): SmartRenameResult` — parses patterns like
    `Movie.Name.2024.1080p.BluRay.x264-GROUP`.
  - `renameTvShow(filename): SmartRenameResult` — parses patterns like
    `Show.Name.S01E02.720p.HDTV.x264`.
- **`duplicates/DuplicateDetector`** — Computes streaming content hashes (chunked reads) to detect
  true duplicates regardless of filename; returns `DuplicateGroup` lists.
- **`cleaner/EasyCleaner`** — Orchestrates `ApkInspector` (reads APK manifest for label/version,
  cross-checks install state) and `TempFileScanner` (identifies `.tmp`, `.crdownload`, stale
  archives); produces `CleanUpSummary`.
- **`rules/RulesEngine`** — A deterministic matching pipeline that evaluates each `AutomationRule`
  against a `DownloadItem` (match type, match value, threshold) and returns the target folder when a
  rule fires.

#### `com.example.service` — Background Service

`DownloadMonitorService` is the foreground/background observer that:

1. Registers a `ContentObserver` on the MediaStore Downloads collection.
2. Reacts to create/modify events and hands new files to the pipeline (rename → dedupe → clean → route).
3. Coordinates with WorkManager for scheduled maintenance (deep dedupe scans, temp cleanup).

#### `com.example.receiver` — Broadcast Receiver

`PackageInstallReceiver` listens for `ACTION_PACKAGE_ADDED`. When an app is installed, it triggers
`EasyCleaner` to purge the corresponding downloaded APK from Downloads, since the install makes the
APK redundant.

#### `com.example.ui` — Presentation Layer

- **`navigation/NavGraph`** — Compose Navigation routes (Dashboard, Files, Rules, Settings, Statistics).
- **`screens/*`** — Material 3 composables; each screen observes a single ViewModel.
- **`viewmodel/*`** — MVVM `ViewModel`s exposing `StateFlow` UI state; collect via `collectAsStateWithLifecycle()`.
- **`components/*`** — Reusable UI building blocks (file cards, stat cards, rule editor, empty states).
- **`theme/*`** — Material 3 theming with **Dynamic Color** as the default and a static fallback palette.

### 3.3 Data Flow (One Cycle)

```
MediaStore ContentObserver / PackageInstallReceiver
        │
        ▼
DownloadMonitorService ──► FileRepository.listNewFiles()
        │
        ▼
domain/renamer/SmartRenamer ──► rename movie/TV files
domain/duplicates/DuplicateDetector ──► dedupe identical content
domain/cleaner/EasyCleaner ──► purge redundant APKs & temp files
        │
        ▼
domain/rules/RulesEngine ──► route files to target folders
        │
        ▼
data/repository/StatsRepository ──► update app_statistics
data/repository (logs) ──► insert processed_files_log
        │
        ▼
Room Database ──► Flow ──► ViewModel StateFlow ──► Compose UI
```

---

## 4. Database Schema Specification

Room database name: **`download_butler.db`** (version 1).

### 4.1 `automation_rules`

Stores user-defined routing rules evaluated by the `RulesEngine`.

| Column         | Type      | Constraints | Description                                    |
| -------------- | --------- | ----------- | ---------------------------------------------- |
| `id`           | `Long`    | PK, AUTO    | Surrogate primary key.                         |
| `name`         | `String`  | NOT NULL    | Human-readable rule name.                      |
| `matchType`    | `String`  | NOT NULL    | Match category: `EXTENSION`, `FILENAME`, `MIME`, `SIZE`, `SOURCE`. |
| `matchValue`   | `String`  | NOT NULL    | Value to compare against (e.g. `*.apk`, `[Tt]vSeries`). |
| `action`       | `String`  | NOT NULL    | Action on match: `MOVE`, `DELETE`, `RENAME`, `KEEP`. |
| `targetFolder` | `String`  | NULLABLE    | Destination folder (required when `action = MOVE`). |
| `daysThreshold`| `Int`     | DEFAULT 0   | Age threshold in days (e.g. delete files older than N days). |
| `isEnabled`    | `Boolean` | DEFAULT 1  | Whether the rule participates in evaluation.    |
| `priority`     | `Int`     | NOT NULL    | Evaluation order (lower = higher priority).     |

```sql
CREATE TABLE automation_rules (
    id            INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
    name          TEXT    NOT NULL,
    matchType     TEXT    NOT NULL,
    matchValue    TEXT    NOT NULL,
    action        TEXT    NOT NULL,
    targetFolder  TEXT,
    daysThreshold INTEGER NOT NULL DEFAULT 0,
    isEnabled     INTEGER NOT NULL DEFAULT 1,
    priority      INTEGER NOT NULL
);
```

### 4.2 `processed_files_log`

Immutable audit log of every file operation performed by the app.

| Column        | Type       | Constraints | Description                                   |
| ------------- | ---------- | ----------- | --------------------------------------------- |
| `id`          | `Long`     | PK, AUTO    | Surrogate primary key.                        |
| `originalPath`| `String`   | NOT NULL    | Full path before processing.                  |
| `newPath`     | `String`   | NULLABLE    | Full path after processing (null if deleted). |
| `fileName`    | `String`   | NOT NULL    | Final file name after the operation.          |
| `actionType`  | `String`   | NOT NULL    | `RENAMED`, `MOVED`, `DELETED`, `DEDUPLICATED`, `CLEANED`, `SKIPPED`. |
| `bytesSaved`  | `Long`     | DEFAULT 0   | Storage reclaimed by this action (bytes).     |
| `category`    | `String`   | NULLABLE    | Classification: `MOVIE`, `TV_SHOW`, `APK`, `TEMPORARY`, `OTHER`. |
| `timestamp`   | `Long`     | NOT NULL    | Epoch millis of the operation.                |

```sql
CREATE TABLE processed_files_log (
    id           INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
    originalPath TEXT    NOT NULL,
    newPath      TEXT,
    fileName     TEXT    NOT NULL,
    actionType   TEXT    NOT NULL,
    bytesSaved   INTEGER NOT NULL DEFAULT 0,
    category     TEXT,
    timestamp    INTEGER NOT NULL
);
```

### 4.3 `app_statistics`

A single-row (aggregate) table tracking lifetime counters.

| Column                    | Type   | Constraints | Description                              |
| ------------------------- | ------ | ----------- | ---------------------------------------- |
| `id`                      | `Long` | PK          | Fixed row id (always `1`) for the aggregate. |
| `totalFilesOrganized`     | `Long` | DEFAULT 0   | Files moved into target folders.         |
| `totalFilesRenamed`       | `Long` | DEFAULT 0   | Files successfully renamed.              |
| `totalDuplicatesRemoved`  | `Long` | DEFAULT 0   | Duplicate files deleted.                 |
| `totalApksCleaned`        | `Long` | DEFAULT 0   | APKs purged post-install.                |
| `totalBytesSaved`         | `Long` | DEFAULT 0   | Cumulative storage reclaimed (bytes).    |
| `totalDownloadsProcessed` | `Long` | DEFAULT 0   | Total files processed overall.           |

```sql
CREATE TABLE app_statistics (
    id                      INTEGER PRIMARY KEY NOT NULL,
    totalFilesOrganized     INTEGER NOT NULL DEFAULT 0,
    totalFilesRenamed       INTEGER NOT NULL DEFAULT 0,
    totalDuplicatesRemoved  INTEGER NOT NULL DEFAULT 0,
    totalApksCleaned        INTEGER NOT NULL DEFAULT 0,
    totalBytesSaved         INTEGER NOT NULL DEFAULT 0,
    totalDownloadsProcessed INTEGER NOT NULL DEFAULT 0
);
```

### 4.4 `ai_provider_config`

User-configured BYO-AI endpoint settings. Stored locally; API keys never leave the device.

| Column          | Type      | Constraints | Description                                 |
| --------------- | --------- | ----------- | ------------------------------------------- |
| `providerId`    | `String`  | PK          | Stable provider identifier (e.g. `gemini`, `openrouter`, `openai`, `ollama`). |
| `displayName`   | `String`  | NOT NULL    | User-facing provider label.                 |
| `baseUrl`       | `String`  | NOT NULL    | API base URL (endpoint root).               |
| `apiKey`        | `String`  | NULLABLE    | API key (empty/null for local providers like Ollama). |
| `defaultModel`  | `String`  | NULLABLE    | Model identifier used by default.           |
| `isEnabled`     | `Boolean` | DEFAULT 0  | Whether this provider is active.            |
| `isDefault`     | `Boolean` | DEFAULT 0  | The fallback provider when rules do not specify one. |

```sql
CREATE TABLE ai_provider_config (
    providerId    TEXT    PRIMARY KEY NOT NULL,
    displayName   TEXT    NOT NULL,
    baseUrl       TEXT    NOT NULL,
    apiKey        TEXT,
    defaultModel  TEXT,
    isEnabled     INTEGER NOT NULL DEFAULT 0,
    isDefault     INTEGER NOT NULL DEFAULT 0
);
```

### 4.5 Entity ↔ Model Mapping

| Room Entity            | In-Memory Model            | DAO                    |
| ---------------------- | -------------------------- | ---------------------- |
| `AutomationRuleEntity` | `AutomationRule`           | `AutomationRuleDao`    |
| `ProcessedFileLogEntity` | `ProcessedFileLog`       | `ProcessedFileLogDao`  |
| `AppStatisticsEntity`  | `AppStatistics`            | `AppStatisticsDao`     |
| `AiProviderConfigEntity` | `AiProviderConfig`       | `AiProviderConfigDao`  |

Mapping logic lives exclusively inside the repositories — entities never escape the `data.database`
layer.

---

## 5. Appendix

### 5.1 Glossary

- **BYO-AI** — Bring-Your-Own-AI: user-provided model API endpoints (Gemini, OpenAI-compatible, OpenRouter, Ollama).
- **DTO** — Data Transfer Object; a plain in-memory data class.
- **MediaStore** — Android's unified media content provider.
- **Streaming Hash** — Content hash computed in fixed-size chunks so memory usage stays flat even for very large files.

### 5.2 Change Log

| Date       | Version | Notes                                        |
| ---------- | ------- | -------------------------------------------- |
| 2026-07-31 | 1.0.0   | Initial blueprint; directory map & DB schema finalized. |

---

*This document is the authoritative source for the Download Butler architecture. Changes to the
structure, tech stack, or schema must be reflected here first.*
