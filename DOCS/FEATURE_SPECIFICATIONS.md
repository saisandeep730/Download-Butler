# Download Butler — Feature Specifications

> **Detailed feature documentation** for Download Butler: expected behavior, edge-case handling,
> and business-logic requirements for every built-in capability.
>
> Version: 1.0.0
> Last Updated: 2026-07-31
> Companion Document: [`PROJECT_ARCHITECTURE.md`](./PROJECT_ARCHITECTURE.md)

---

## Table of Contents

1. [Download Monitoring & Content Observer](#1-download-monitoring--content-observer)
2. [Offline Automatic File Organizer](#2-offline-automatic-file-organizer)
3. [Smart Movie & TV Series Renaming Engine](#3-smart-movie--tv-series-renaming-engine)
4. [Bring-Your-Own AI (BYO-AI) Integration](#4-bring-your-own-ai-byo-ai-integration)
5. [Streaming SHA-256 Duplicate Detector](#5-streaming-sha-256-duplicate-detector)
6. [Easy Clean & Installed APK Cleaner](#6-easy-clean--installed-apk-cleaner)
7. [Automation Rules Engine](#7-automation-rules-engine)
8. [Privacy & Data Transparency](#8-privacy--data-transparency)

---

## 1. Download Monitoring & Content Observer

### 1.1 Objective

Continuously watch the Android **Downloads** collection and react to file lifecycle events
(new, modified, or completed) without any user interaction.

### 1.2 Expected Behavior

| # | Behavior | Detail |
| - | -------- | ------ |
| 1 | **Continuous watch** | `DownloadMonitorService` registers a `ContentObserver` on `MediaStore.Downloads` (external volume) at startup and on every device boot. |
| 2 | **Reactive triggers** | On `INSERT`, `UPDATE`, and `DELETE` notifications, the service re-queries MediaStore for files in `STATE` completed, unchanged for ≥ N seconds (debounce). |
| 3 | **Background evaluation** | New/changed files are handed to the full processing pipeline (organize → rename → dedupe → route) only while the service is **active** (user-enabled toggle, default **ON**). |
| 4 | **Manual trigger** | The Dashboard exposes an instant **"Auto Organize"** button that runs the same pipeline immediately, bypassing the observer. |
| 5 | **Idle efficiency** | When no changes are detected, the service idles with no CPU, battery, or network activity. |

### 1.3 Edge-Case Handling

- **Duplicate notifications** — MediaStore may emit multiple events for one file write; a debounce
  window (e.g. 5 seconds since last size change) prevents re-processing.
- **File still being written** — Files with `MediaStore.Downloads.STATUS != STATUS_SUCCESS` or
  whose size changes between polls are skipped until stable.
- **Permissions revoked** — On `SecurityException`, the service degrades gracefully, stops watching,
  and surfaces a non-blocking "grant storage access" prompt in the UI.
- **Process death** — `START_STICKY` foreground service with a low-priority notification re-registers
  the observer after restart; WorkManager's `PERIODIC` job (e.g. every 6h) acts as a safety-net sweep.
- **Service disabled** — Observer deregistration is guaranteed on `onDestroy`; no orphaned listeners.

### 1.4 Business-Logic Requirements

- Only process files physically located under the Downloads folder tree.
- Never block the main thread: all MediaStore queries run on a background `Dispatchers.IO` scope.
- Every trigger must be idempotent — re-processing an already-logged file yields no side effects
  (guarded by the `processed_files_log` audit check).

---

## 2. Offline Automatic File Organizer

### 2.1 Objective

Automatically move newly observed files into category-based destination folders, entirely **offline**.

### 2.2 Extension → Category Mapping

| Category    | Extensions                                                              |
| ----------- | ---------------------------------------------------------------------- |
| **DOCUMENTS** | `pdf`, `doc`, `docx`, `txt`, `epub`, `xlsx`, `pptx`, `csv`, `odt`, `rtf` |
| **MOVIES**    | `mp4`, `mkv`, `avi`, `mov`, `wmv`, `webm`, `flv`, `m4v`                 |
| **IMAGES**    | `jpg`, `jpeg`, `png`, `gif`, `webp`, `svg`, `bmp`, `heic`               |
| **AUDIO**     | `mp3`, `flac`, `wav`, `aac`, `m4a`, `ogg`, `opus`, `wma`                |
| **ARCHIVES**  | `zip`, `rar`, `7z`, `tar`, `gz`, `bz2`, `xz`                            |
| **APKS**      | `apk`, `xapk`, `apkm`, `apks`                                           |

> Extensions are matched case-insensitively. Unknown extensions fall through to a configurable
> **OTHER / MISC** folder (or are left in place, per user preference).

### 2.3 Folder Mapping & Customization

- Default destinations live under the primary Downloads root:
  - `/Internal Storage/Download/Documents`
  - `/Internal Storage/Download/Movies`
  - `/Internal Storage/Download/Images`
  - `/Internal Storage/Download/Audio`
  - `/Internal Storage/Download/Archives`
  - `/Internal Storage/Download/APKs`
- **Per-category sub-mapping** — the user may override any category's destination (e.g.
  `Movies → /Internal Storage/Download/Media/Films`). Paths are user-picked via SAF (Storage Access
  Framework) to remain scoped-storage compliant.
- **Recursive source scan** — the organizer may also scan user-selected subfolders of Downloads, not
  only the root.

### 2.4 Expected Behavior

1. File arrives → extension resolved → category resolved → destination resolved.
2. Destination folder is created on demand (never fails silently; `mkdirs()` failure is logged).
3. File is **moved** (same volume, atomic rename) — never copied unless the user selects "copy" mode.
4. Success is recorded in `processed_files_log` with `actionType = MOVED`.

### 2.5 Edge-Case Handling

- **Name collision at destination** — if a file with the same name exists and is byte-identical
  (see [Duplicate Detector](#5-streaming-sha-256-duplicate-detector)), the new arrival is treated as
  a duplicate; otherwise a unique suffix is appended (`file (1).ext`).
- **File in use** — move fails with `IOException`; the item is requeued once and then skipped with a
  log entry.
- **Extensionless files** — categorized via MIME-type sniff; if undeterminable, routed to MISC.
- **Cross-volume destination** — when the user picks a folder on external/OTG storage, the move
  degrades to a streaming copy + verified delete.

### 2.6 Business-Logic Requirements

- `actionType`, `category`, and `bytesSaved` must always be written to the audit log.
- `totalFilesOrganized` and `totalBytesSaved` counters must be updated atomically in the same
  transaction as the log row.

---

## 3. Smart Movie & TV Series Renaming Engine

### 3.1 Objective

Convert raw, tag-heavy release filenames into clean, human-readable titles using **pure regex** —
no network, no lookup services.

### 3.2 Movie Regex Engine

**Parses:**
- **Title** — leading name tokens (spaces/`.`/`_` normalized to spaces, `Title Case`).
- **Release year** — 4-digit `19xx`/`20xx`; if present, the file routes to a `Movies/{Year}/` folder.
- **Quality / resolution** — `1080p`, `720p`, `4K`, `2160p`, `480p`, `SD`.

**Strips release tags** (removed from the final name):
`BluRay`, `Bluray`, `WEB-DL`, `WEBRip`, `HDRip`, `DVDRip`, `BDRip`, `REMUX`, `x264`, `x265`,
`h264`, `h265`, `HEVC`, `HDR`, `HDR10`, `DTS`, `DTS-HD`, `Dolby`, `TrueHD`, `AC3`, `AAC`,
`GREENLITE`, `YIFY`, `RARBG`, `800MB`, `1.4GB`, and any `-GROUP` tag at the tail.

**Output example:**

```
Input : Movie.Name.2024.1080p.BluRay.x264-GROUP.mkv
Output: Movies/2024/Movie Name (2024) 1080p.mkv
```

### 3.3 TV Show Regex Engine

**Identifies episode markers:**
- `S01E02`, `S1E2`, `s01e02` (case-insensitive) — modern marker.
- `1x01` — legacy marker.
- `Season 1 / Episode 2` or `1 Episode 02` — spaced markers.
- `Show.Name.S01E02.720p.HDTV.x264` style tag tails (stripped like movies).

**Formats into subfolders:**

```
TV Shows/{Show Title}/Season {XX}/S{XX}E{XX} - {Episode Title}.ext
```

**Output example:**

```
Input : Show.Name.S01E02.720p.HDTV.x264-GROUP.mkv
Output: TV Shows/Show Name/Season 01/S01E02 - Episode Name.mkv
```

### 3.4 Expected Behavior

- Non-matching files are returned as **unchanged** with a `SKIPPED` status (never mangled).
- Renames are **previewed** in the UI (old → new) before any write to disk.
- Episode titles, when present, are extracted from the source name; otherwise the generic
  `S{XX}E{XX}` form is used.
- All results are captured in `SmartRenameResult` and logged with `actionType = RENAMED`.

### 3.5 Edge-Case Handling

- **Ambiguous title vs. year** — digits embedded in a title (e.g. `The 13th Floor`) are only treated
  as a year when a full 4-digit 19xx/20xx match exists at a token boundary.
- **Multiple seasons in one file** — always treats the first valid `SXXEXX` as the primary marker.
- **Unicode / non-ASCII names** — preserved verbatim; only ASCII whitespace and separators are
  normalized.
- **Illegal characters** for Android filenames (`/ \ : * ? " < > |`) are stripped from the output.
- **Conflicting target name** — if the target already exists, append ` (1)` before the extension.

### 3.6 Business-Logic Requirements

- The engine MUST be pure (no I/O); file-system renames live in `FileRepository`.
- Pattern tables are data-driven (`MovieRenamePatterns.kt`, `TvShowRenamePatterns.kt`) so new tags
  can be added without touching the parser core.

---

## 4. Bring-Your-Own AI (BYO-AI) Integration

### 4.1 Objective

Optionally enrich filename classification with an AI model. **Completely optional, opt-in, and
network-isolated from the rest of the app.**

### 4.2 Privacy Guarantee

- **Zero telemetry** — the app sends no analytics, no device fingerprints, nothing but the request
  the user explicitly initiates.
- **File contents are NEVER uploaded.** Only **filename string metadata** (the basename + optional
  category hint) is sent to the AI endpoint.
- API keys are stored locally in Room (`ai_provider_config`) and are never transmitted anywhere
  except to the user's configured endpoint.

### 4.3 Supported Providers

| Provider                    | Endpoint Profile                                       | Model Example          |
| --------------------------- | ------------------------------------------------------ | ---------------------- |
| **Google Gemini API**       | Gemini REST (`generativelanguage.googleapis.com`)      | `gemini-1.5-pro`       |
| **OpenAI REST**             | OpenAI-compatible `/v1/chat/completions`               | `gpt-4o-mini`          |
| **OpenRouter**              | OpenRouter unified gateway                             | any routed model       |
| **Ollama (local)**          | Local server (`http://localhost:11434`)                | `llama3`               |

Provider is selected via `isEnabled` / `isDefault` flags in `ai_provider_config`.

### 4.4 Expected Behavior

1. User enables a provider and (for remote providers) enters an API key in Settings.
2. A classification request (filename only) is sent via `AiRepository` → `AiService`.
3. The AI response is parsed into a **recommended** file name / category.
4. **Human-in-the-loop is mandatory:** the recommendation is shown in the UI as a pending proposal.
   No rename ever occurs without explicit user approval.
5. Approved proposals are executed; rejected proposals are discarded with no disk change.

### 4.5 Edge-Case Handling

- **Network failure / timeout** — request aborts with a graceful error; pipeline continues offline.
- **Malformed AI response** — response fails schema validation → discarded, user sees an error chip.
- **No provider enabled** — AI feature is hidden/disabled; app remains fully functional offline.
- **Key invalid (HTTP 401)** — surfaced once; provider auto-disabled until reconfigured.
- **Sensitive-looking filenames** — the UI always shows exactly what string will be sent, so the
  user can decline before the request fires.

### 4.6 Business-Logic Requirements

- Every AI call must time out (default 15s) and must be cancellable when the pipeline is stopped.
- AI classification is **never** a precondition for any other feature.

---

## 5. Streaming SHA-256 Duplicate Detector

### 5.1 Objective

Find and reclaim duplicate files by **content identity**, immune to filename differences.

### 5.2 Two-Stage Detection

**Stage 1 — Size grouping (cheap pre-filter):**
Files are grouped by identical byte length. Files with unique sizes cannot be duplicates and are
excluded immediately, avoiding any content reads.

**Stage 2 — Streaming SHA-256 (content verification):**
Each candidate group is read in **64 KB buffers** through a rolling SHA-256 `MessageDigest`.
Memory use stays flat regardless of file size → **no `OutOfMemoryError` on large files**.

```
Group by size ──► stream 64KB chunks ──► SHA-256 ──► cluster identical hashes ──► DuplicateGroup[]
```

### 5.3 Expected Behavior

- Returns `DuplicateGroup` objects: one "keeper" + one or more "redundant" file paths.
- Provides **reclaimable storage metrics** per group (sum of redundant bytes) and an app-wide total.
- **Keeper selection** — auto-selects the **oldest/original** file (earliest creation timestamp)
  to preserve provenance; tie-breaks prefer the original path casing / non-mangled name.
- Deletion of redundant files is always a **user-confirmed action** (list + confirm → delete).

### 5.4 Edge-Case Handling

- **File changes during scan** — hash is re-verified immediately before deletion; if the file
  changed, it is skipped.
- **File locked / unreadable** — streamed read fails → candidate excluded from the group.
- **Very large files (>2GB)** — no whole-file buffering; constant ~64KB heap footprint.
- **Scoped storage** — reads flow through `FileDescriptor`/SAF URIs so large content never needs a
  full path copy.
- **Hash collision paranoia** — duplicates at the same size+hash are additionally byte-compared in
  one final pass before any delete is offered.

### 5.5 Business-Logic Requirements

- Dedup actions log `actionType = DEDUPLICATED`, `bytesSaved = redundant size`.
- Increment `totalDuplicatesRemoved` and `totalBytesSaved` transactionally.
- Never delete the sole surviving copy; never offer the keeper in the delete list.

---

## 6. Easy Clean & Installed APK Cleaner

### 6.1 Objective

Reclaim storage safely: purge redundant APKs and disposable temporary files in one tap.

### 6.2 Installed APK Cleaner

- Inspects each APK in Downloads via **Android `PackageManager`**:
  - Reads manifest metadata (`label`, `versionName`) from the APK (see `ApkItemInfo`).
  - Checks `PackageManager.getPackageArchiveInfo(...)` / installed package list to determine whether
    the app is **already installed** on the device.
- APKs whose package is installed are flagged as **redundant** (the stored APK is no longer needed).
- APKs for **not-installed** apps are preserved and offered separately (useful for manual install).

### 6.3 Temporary & Empty File Detection

Scans Downloads for:
- **Temporary artifacts** — `.tmp`, `.crdownload`, `.part` files (stale partial downloads).
- **Zero-byte files** — empty files that carry no data and no value.
- Optional: stale archives / orphaned `.apks`-bundle splits left after install.

### 6.4 Expected Behavior

1. User taps **"Clean Now"** (or the periodic WorkManager sweep runs).
2. `EasyCleaner` orchestrates `ApkInspector` + `TempFileScanner`.
3. Candidate list is presented with item counts and **total reclaimable bytes**.
4. On confirm, items are deleted; results roll up into a `CleanUpSummary`.
5. `totalApksCleaned`, `totalBytesSaved` and audit logs are updated.

### 6.5 Edge-Case Handling

- **App uninstalled between scan and delete** — the APK becomes valid again; deletion is skipped
  and the item is re-flagged as "not installed".
- **System apps** — never considered (not actionable from a Downloads file anyway).
- **Split/bundle APKs (`apks`, `apkm`)** — grouped so all parts are cleaned or none are.
- **User choice preserved** — nothing is deleted without an explicit confirmation screen.
- **PackageManager unavailable** (rare / OEM) — falls back to name-based heuristics and a warning.

### 6.6 Business-Logic Requirements

- APK inspection must be fast: read manifest + query installed state, never install the APK.
- Cleanup must be idempotent and must handle `FileNotFoundException` gracefully.

---

## 7. Automation Rules Engine

### 7.1 Objective

A user-configurable, priority-ordered pipeline that decides what happens to each file.

### 7.2 Match Types (evaluated in priority order)

| Priority | Match Type        | Matcher Semantics                                         |
| -------- | ----------------- | --------------------------------------------------------- |
| 1        | `EXTENSION`       | File extension equals `matchValue` (e.g. `apk`, `torrent`). |
| 2        | `REGEX`           | Filename matches `matchValue` regex (e.g. `^S\d+E\d+`).    |
| 3        | `SIZE_GREATER_MB` | File size > `matchValue` (integer MB).                     |
| 4        | `CATEGORY`        | File category equals `matchValue` (e.g. `DOCUMENTS`, `APKS`). |
| 5        | `ALL_DOWNLOADS`   | Catch-all; matches every file in Downloads.                |

> Evaluation is **first-match-wins** in `priority` order (`isEnabled = true` rules only). Lower
> priority rules never fire once a higher-priority rule has matched.

### 7.3 Rule Actions

| Action              | Semantics                                                            |
| ------------------- | ------------------------------------------------------------------- |
| `MOVE_TO_CATEGORY`  | Move file into `targetFolder` (resolved per category mapping).       |
| `RENAME_SMART`      | Run the Smart Renaming engine; keep the file in place.               |
| `DELETE_AFTER_DAYS` | Delete the file once it is older than `daysThreshold` days.          |
| `DELETE_IMMEDIATELY`| Delete the file at once (destructive; always confirm-wrapped).       |

### 7.4 Expected Behavior

1. Each file passes through the prioritized rule list.
2. The **first enabled matching rule** determines the action.
3. Actions execute through the shared pipeline and are fully audited.
4. If no rule matches, the file takes the default route (Organizer category mapping).

### 7.5 Edge-Case Handling

- **Rule with empty `targetFolder` + `MOVE_TO_CATEGORY`** — rejected at save time with validation
  feedback (cannot move to nowhere).
- **`DELETE_IMMEDIATELY` safety** — always wrapped in a confirmation dialog listing the files.
- **Circular/conflicting rules** — no recursion possible by design (single pass, first-match-wins).
- **Disabled rules** — skipped entirely; toggling re-enables without reordering.
- **Priority ties** — resolved by stable insertion order (deterministic).

### 7.6 Business-Logic Requirements

- Rule CRUD must be reactive: the pipeline observes `automation_rules` via `StateFlow`; edits apply
  to the next file without service restart.
- Destructive actions must always log `actionType = DELETED` with `bytesSaved`.

---

## 8. Privacy & Data Transparency

### 8.1 Objective

Guarantee the app is **private by construction**: offline-first, no tracking, no ads, no analytics.

### 8.2 Guarantees

| Guarantee             | Detail                                                                  |
| --------------------- | ----------------------------------------------------------------------- |
| **Offline-first**     | All core features (organize, rename, dedupe, clean, rules) run fully offline. |
| **Zero ads**          | No ad SDKs, no ad placements, anywhere.                                  |
| **Zero tracking**     | No device identifier, no IP logging, no fingerprinting, no telemetry.    |
| **Zero analytics**    | No external analytics SDKs (Firebase Analytics, Mixpanel, etc.).         |
| **No background network** | No network activity unless the user explicitly uses BYO-AI.           |
| **Transparent data map** | A Settings section documents exactly which data is stored, where, and who sees it. |
| **Local-first storage** | All data (rules, logs, stats, AI config) lives in local Room on-device. |

### 8.3 Data Residency

| Data                          | Location           | Leaves Device?     |
| ----------------------------- | ------------------ | ------------------ |
| Automation rules              | Room               | Never              |
| Processing audit log          | Room               | Never              |
| Lifetime statistics           | Room               | Never              |
| AI provider config + API keys | Room               | Only to user's configured endpoint, only on user-initiated requests |
| Filename metadata for AI      | —                  | Only if user enables AI + approves the request |

### 8.4 Business-Logic Requirements

- The dependency tree must contain **no** tracking/analytics SDKs (enforced via a dependency
  allow-list review).
- BYO-AI must remain disabled until the user explicitly opts in; no background AI calls ever.
- Any future change to data collection requires a spec revision here before shipping.

---

## Appendix

### Change Log

| Date       | Version | Notes                                                      |
| ---------- | ------- | ---------------------------------------------------------- |
| 2026-07-31 | 1.0.0   | Initial feature specification for all eight feature areas. |

*This document is the authoritative functional specification. Behavioral changes must be reflected
here before implementation.*
