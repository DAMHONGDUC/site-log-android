# 00 — Overview (Android)

<dl><dd>

- Product, data model, architecture, data flows, infrastructure, conventions and decisions for the Android app.
- Same product and feature specs as iOS; spec numbers `01`–`19` are stable and used by [ROADMAP.md](../ROADMAP.md).
- This is the approved target design for `site-log-android`; nothing here is implemented yet.
- **Reading order:** this overview → the feature spec → its guide in [feature-guide/](../feature-guide/README.md).
- iOS original: [docs/feature-docs/00-overview.md](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/feature-docs/00-overview.md).

</dd></dl>

## Contents

<dl><dd>

1. [Product](#1-product) — positioning, core rule, stack, non-goals
2. [Market](#2-market)
3. [Data model](#3-data-model)
4. [Architecture](#4-architecture) — layout, modules, MVVM, Hilt, startup, errors
5. [Data flows](#5-data-flows)
6. [Infrastructure](#6-infrastructure)
7. [Conventions](#7-conventions)
8. [Feature specs](#8-feature-specs)
9. [Decisions](#9-decisions) — A01–A22
10. [Beginner glossary](#beginner-glossary)

</dd></dl>

## Beginner glossary

<dl><dd>

| Term | Meaning | Short example |
|---|---|---|
| Composable | A Kotlin function annotated `@Composable` that describes UI | `Text(stringResource(R.string.projects_list_title))` |
| ViewModel | An `androidx.lifecycle.ViewModel` that survives rotation, holds screen state and handles actions | `ProjectListViewModel` loads rows |
| `StateFlow` | A stream that always has a current value; the screen collects it | `val state: StateFlow<ProjectListState>` |
| Coroutine | A function that can pause without blocking a thread | `viewModelScope.launch { load() }` |
| Gradle module | A separately compiled part of the app with its own dependencies | `:domain`, `:data` |
| Hilt | Google's DI library on Dagger; generates the wiring at compile time | `@Provides fun projectRepository(...)` |
| Room | Google's SQLite library: `@Entity` tables, `@Dao` queries, migrations | `ProjectModel`, `ProjectDao` |
| WorkManager | Runs deferrable work that survives app kill and reboot | The upload worker |
| Use case | One named action the app can perform | `CreateProjectUseCase` |
| Repository | A boundary for reading and writing app data | `ProjectRepository` |
| Entity | A business concept with no storage or UI types | `Project` |
| DTO | A data-transfer object at a system boundary | JSON from the server |
| String resource | A key in `strings.xml` whose text is translated | `R.string.access_sign_in_title` → “Sign in” |

</dd></dl>

## 1. Product

<dl><dd>

| | |
|---|---|
| What | Offline-first Android app for recording construction site conditions |
| Deliverable | Signed PDF inspection report |
| Users | Site supervisors, subcontractors, apartment handover teams |

```
Create project → start session → walk floors/rooms, capture + log issues
→ reach WiFi, background upload → export signed PDF report
```

### 1.1 Positioning and constraints

<dl><dd>

- Competitors lose evidence three ways: stalled or incomplete uploads, photos vanishing on reload, sync failing with no reason.
- SiteLog assumes the network is bad.

| Constraint | Consequence |
|---|---|
| Sites, basements, alleys have weak or no signal | Every write path completes offline |
| 150–250 files and several GB per session | Upload survives process death, Doze, network loss and reboot |
| User swipes the app away and keeps walking | No state lives only in RAM |
| Many mid-range Android phones on site (3–4 GB RAM) | Downsampled bitmaps, streamed hashing, one bitmap per PDF page |
| The PDF is the paid deliverable | Never truncate, never silently drop images |

</dd></dl>

### 1.2 Core rule

<dl><dd>

- `Capture` is immutable after creation; only descriptive metadata is editable.
- Each carries a `sha256` computed after the file closes, before the DB write.
- Annotations are a separate vector layer, so markup never invalidates a hash.

</dd></dl>

### 1.3 Stack

<dl><dd>

| Area | Choice |
|---|---|
| UI | Jetpack Compose (BOM 2026.09), Material 3; `minSdk` 29, `targetSdk` 36, `compileSdk` 37 ([A22](#a22--minimum-sdk-29)) |
| Language | Kotlin 2.4, code targets Java 21; coroutines + Flow; AGP 9 with built-in Kotlin, Gradle 9.6 (its daemon runs on JDK 25) |
| Architecture | Clean Architecture (Presentation · Domain · Data), MVVM; `:domain`, `:data`, `:platformkit`, `:core`, `:designsystem` are Gradle modules, Presentation lives in `:app` ([A09](#a09--layers-as-gradle-modules)) |
| Dependency injection | [Hilt](https://dagger.dev/hilt/) ([A01](#a01--hilt-for-dependency-injection)) |
| Navigation | [Navigation 3](https://developer.android.com/guide/navigation/navigation-3) (`androidx.navigation3`) ([A17](#a17--navigation-3)) |
| Local store | Room with KSP, exported schemas ([A10](#a10--room-for-the-local-store)) |
| Settings | DataStore Preferences |
| Secrets | Android Keystore (AES-GCM key, HMAC key) |
| Media | CameraX (`camera-core`, `camera-video`, `camera-view`), `MediaRecorder` for audio, Media3 Transformer to join pre-record segments |
| Plans | `PdfRenderer` + `BitmapRegionDecoder`, Compose tiles |
| Crypto | `MessageDigest`, `Mac`, `SecretKeyFactory` (PBKDF2), Keystore |
| External devices | `android.bluetooth.le` (BLE central; GATT server for the mock) |
| Transport | Hand-written WorkManager + OkHttp engine ([A02](#a02--hand-written-upload-engine)) |
| Object storage | Amazon S3, multipart + presigned URLs (same backend) |
| Realtime | OkHttp `WebSocket` ([A04](#a04--websocket-realtime-channel)) |
| Remote images | [Coil](https://coil-kt.github.io/coil/) 3 ([A05](#a05--coil-for-remote-images)) |
| Auth / crash / push | Firebase Android BoM: Auth, Crashlytics, Cloud Messaging — **no Firestore** ([A07](#a07--firebase-crashlytics)) |
| Sign-in | Credential Manager (Google), Firebase `OAuthProvider("apple.com")` (Apple), email |
| Location | Fused Location Provider (`play-services-location`) |
| JSON | kotlinx.serialization |
| Tests | JUnit 4, kotlinx-coroutines-test, Turbine, Truth, Robolectric, Room `MigrationTestHelper`, Compose UI test; Roborazzi optional, last (F18) ([A06](#a06--test-libraries)) |
| Tooling | Gradle Kotlin DSL, version catalog, convention plugins, detekt 1.23, ktlint 1.8 (`android_studio` style) via Spotless, Android Lint, Fastlane `supply`, GitHub Actions |
| Dev only | Compose Live Edit, `@Preview`, LeakCanary (debug) |
| Backend | Shared with iOS: [site-log-be](https://github.com/DAMHONGDUC/site-log-be) — Node 24 + Fastify, PostgreSQL 16 on AWS ([03 API](03-backend.md)) |

- A dependency is part of the app only after it appears in `gradle/libs.versions.toml`.

</dd></dl>

### 1.4 Non-goals

<dl><dd>

| Excluded | Reason |
|---|---|
| Firebase Storage, third-party upload libraries | The transfer engine is the core of the product |
| Photo picker / `ACTION_IMAGE_CAPTURE` intents | A hash only means something for a file the app produced |
| Firebase touching file bytes or data | Identity, crash reporting and push only |
| Burning annotations into original pixels | Destroys the hash |
| RFIs, scheduling, budgets, CAD, real-time co-editing, cloud AI defect detection | Different product, or indefensible in a dispute |
| Tablets-first layouts, Wear OS, Android Auto | One phone layout first; large screens only scale |

</dd></dl>

</dd></dl>

## 2. Market

<dl><dd>

- Same research as iOS (September 2026): [iOS §2 Market](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/feature-docs/00-overview.md#2-market).
- Android adds one fact: site crews often carry mid-range Android phones (Samsung A-series, Xiaomi, Oppo), so memory and battery budgets are set on a 4 GB device ([05 §19](05-platform.md#19-performance-profiling)).
- OEM battery savers (Xiaomi, Oppo, Samsung) kill background work harder than stock Android; Diagnostics shows the fix per maker ([09](05-platform.md#09-diagnostics-and-observability)).

</dd></dl>

## 3. Data model

<dl><dd>

### 3.1 Hierarchy

<dl><dd>

```
Project
 ├ PlanSheet                                → 11
 ├ ChecklistTemplate                        → 13
 └ Session (survey run)
    └ Location (floor / unit / room)
       ├ ChecklistRun → ChecklistResult     → 13
       └ Capture (photo | video | audio)
          ├ Annotation (vector overlay)     → 12
          └ Issue → PlanPin                 → 03, 11
```

- `Location` belongs to `Project`, not `Session`: cross-session before/after pairing needs it.

</dd></dl>

### 3.2 Entities

<dl><dd>

| Entity | Fields | Rules |
|---|---|---|
| `Project` | `id`, `name`, `address`, `clientName`, `createdAt` | Cascade-deletes sessions, locations, plans |
| `Session` | `id`, `startedAt`, `endedAt?`, `surveyorName`, `note`, `state` | `DRAFT → ACTIVE → CLOSED → EXPORTED`; `CLOSED` blocks captures, `EXPORTED` is read-only |
| `Location` | `id` (stable across renames), `code` (`"A-12.05"`), `kind` (`FLOOR/UNIT/ROOM`), `parentId?`, `sortIndex` | Max depth 3 |
| `Capture` | `id`, `kind`, `filePath` (**relative**), `sha256`, `byteSize`, `capturedAt`, `trustedTime`, `verificationCode?`, `latitude?`/`longitude?`/`horizontalAccuracy?`, `deviceModel`, `durationMs?`, `deviceReading?`, `uploadState`, `remoteKey?`, `isExcludedFromReport` | GPS nullable — basements have none |
| `Issue` | `id`, `title`, `detail`, `severity` (`CRITICAL/MAJOR/MINOR`), `status` (`OPEN → IN_PROGRESS → RESOLVED → VERIFIED`), `dueDate?`, `locationId`, `assigneeName?`, `trade?`, `planPin?`, `sourceChecklistResultId?`, `captureIds`, `resolvedByIssueId?` | Anchored to `Location`; `severity` drives upload priority |

- Entities are Kotlin `data class`es with `val` fields; ids are `java.util.UUID`, times `java.time.Instant`.
- **Frozen after creation:** `id`, `filePath`, `sha256`, `capturedAt`, `trustedTime`, GPS, `deviceModel`, `byteSize`, `kind`.
- No API deletes a `Capture` from an exported session; hiding sets `isExcludedFromReport` + audit entry.
- Store paths relative to `filesDir`: absolute paths break after a restore to a new device.

</dd></dl>

### 3.3 UploadState

<dl><dd>

- `PENDING → UPLOADING → SYNCED`; `FAILED` reachable from `UPLOADING`.
- `FAILED` is not terminal and never deletes local files.
- Full job state machine: [04](02-upload-sync.md#04-upload-engine).

</dd></dl>

### 3.4 Schema versioning

<dl><dd>

- Room `@Database(version = n, exportSchema = true)`; schemas exported to `data/schemas/` and committed.
- Unique indices mirror the Postgres rules: `@Index(value = ["projectId", "code"], unique = true)` on locations, `["projectId", "sha256"]` on captures.
- Per change: bump `version`, add an `@AutoMigration` or a hand-written `Migration(n-1, n)`; never `fallbackToDestructiveMigration()`.
- Test migrations with `MigrationTestHelper` against an old-version fixture database, never an empty one.

</dd></dl>

</dd></dl>

## 4. Architecture

<dl><dd>

- Clean Architecture: dependencies point inward to `:domain`.
- Layers are Gradle modules; `:app` holds the composition root (`di/`, `startup/`) and `presentation/`.
- `:app` is the only module that wires layers together.
- Data sources (Room, DataStore, Keystore-backed storage, files, backend calls, Firebase Auth) live in `:data` under `datasources/local/` and `remote/`: interface + `Impl` only.
- System services (permissions, camera, Bluetooth, biometrics, speech, rendering, device, push, crash reporting) and standalone engines on system APIs (`HttpClient`, `uploadkit`, `realtime`) live in `:platformkit`, which knows nothing about Domain.
- Presentation uses MVVM.

```mermaid
flowchart LR
    App[":app composition root<br/>Hilt modules"] --> P["Presentation (:app)<br/>Screen + ViewModel"]
    App --> Data[":data<br/>repository impls, data sources (local · remote), models, mappers"]
    App --> I
    P --> D[":domain (pure JVM)<br/>entities, use cases, repository interfaces"]
    Data --> D
    Data --> I[":platformkit (system services)<br/>permissions · camera · bluetooth · biometrics · speech<br/>rendering · device · push · crashreporting<br/>networking · uploadkit · realtime"]
    P --> C[":core<br/>logger, clock, crypto"]
    P --> DS[":designsystem<br/>tokens, components"]
    Data --> C
    I --> C
```

### 4.1 Repo layout

<dl><dd>

```
site-log-android/
├── app/
│   └── src/main/java/com/dd/sitelog/
│       ├── SiteLogApplication.kt          # @HiltAndroidApp; Crashlytics, logger
│       ├── MainActivity.kt                # @AndroidEntryPoint; edge-to-edge, setContent
│       ├── di/                            # CoreModule, PlatformModule, DataSourcesModule,
│       │                                  # RepositoriesModule, UseCasesModule, SyncTablesModule
│       ├── startup/                       # StartupSequence, StartupStep, steps/
│       ├── fakes/                         # FakeData.kt: skeleton sample data
│       └── presentation/
│           ├── navigation/                # NavigationCoordinator, CoordinatedStack, appEntries()
│           ├── root/                      # RootScreen + RootViewModel: startup → access → main tabs
│           ├── access/                    # splash/ signin/ biometriclock/ passcodeunlock/ privacy/
│           ├── maintab/
│           └── <feature>/                 # <Feature>Route.kt + one folder per screen
│   └── src/main/res/                      # values/strings.xml, values-vi/strings.xml, drawable/, xml/
│   └── src/debug/java/…/devmenu/          # dev menu, DevDataModule (debug builds only)
├── domain/src/main/kotlin/com/dd/sitelog/domain/
│   ├── entities/<feature>/                # data classes, enums, <Feature>Error
│   ├── repositories/                      # every boundary interface, flat
│   ├── usecases/<feature>/                # one use case per file; devdata/
│   └── Outcome.kt
├── data/src/main/kotlin/com/dd/sitelog/data/
│   ├── repositories/                      # <Noun>RepositoryImpl
│   ├── datasources/local/                 # persistence/ (Room DAOs) securestore/ filestorage/ — interface + Impl
│   ├── datasources/remote/                # <Noun>RemoteDataSource + Impl over platformkit engines; auth/
│   ├── models/local/                      # Room @Entity <Noun>Model, schema/ (SiteLogDatabase, migrations)
│   ├── models/remote/                     # @Serializable <Noun>DTO
│   └── mappers/
├── data/schemas/                          # Room exported JSON schemas (committed)
├── platformkit/src/main/kotlin/com/dd/sitelog/platformkit/
│   ├── permissions/ camera/ bluetooth/ biometrics/ speech/ securestore/
│   ├── rendering/ (pdf, csv, tiles, stamps) device/ push/ crashreporting/
│   └── networking/ (HttpClient) uploadkit/ realtime/
├── core/src/main/kotlin/com/dd/sitelog/core/  # logger/ clock/ crypto/
├── designsystem/src/main/kotlin/com/dd/sitelog/designsystem/  # tokens/ components/ theme/
├── build-logic/convention/                # sitelog.android.library, sitelog.jvm.library, sitelog.compose …
├── gradle/libs.versions.toml
├── config/detekt/detekt.yml  .editorconfig
├── tools/ci/                              # check-no-println.sh, check-imports.sh
├── fastlane/  .github/workflows/  Gemfile
└── docs/
```

- Kotlin source folders follow the package: `com.dd.sitelog.<module>.<folder>`.
- Why modules: [A09](#a09--layers-as-gradle-modules); data sources vs PlatformKit: [A11](#a11--data-sources-in-data-platformkit-for-system-services).

</dd></dl>

### 4.2 Layers and dependency rules

<dl><dd>

| Layer | Where | May use |
|---|---|---|
| Presentation | `:app` `presentation/` | `:domain`, `:core`, `:designsystem`, Compose, Hilt (`hiltViewModel()`); never `:data` or `:platformkit` |
| Domain | `:domain` (Kotlin JVM module) | Kotlin stdlib, `java.time`, `java.util.UUID`, `kotlinx-coroutines-core` (for `Flow`) only |
| Data | `:data` | `:domain`, `:platformkit`, `:core`, Room, DataStore, kotlinx.serialization |
| PlatformKit | `:platformkit` | `:core`, Android SDK, Jetpack, Firebase, OkHttp; never `:domain`, `:data` or `:app` |
| DesignSystem | `:designsystem` | Compose, Material 3; nothing from the app |
| Core | `:core` (Android library) | Android SDK only; used by every layer except `:domain` and `:designsystem` |
| Composition root | `:app` `di/`, `startup/`, `fakes/` | Everything |

| Rule | Enforcement |
|---|---|
| `:domain` has no Android, no Room, no Hilt | Compiler: it is a `kotlin("jvm")` module with no Android plugin |
| `:domain` never uses `:data`, `:platformkit`, `:core`; `:platformkit` uses only `:core` | Compiler (module dependencies) |
| Presentation never imports `com.dd.sitelog.data` or `.platformkit` (`:app` depends on them for `di/` only) | CI `check-imports.sh` |
| Presentation never touches Room, DataStore, OkHttp, `WorkManager` | CI `check-imports.sh` |
| View models call use cases only — never a repository | Review |
| Domain folders: `entities/<feature>/`, `usecases/<feature>/`; `repositories/` flat | Review |
| DTOs and Room models never cross into `:domain` or Presentation | Compiler for `:domain`; CI for Presentation |
| Data sources and PlatformKit return their own records and throw their own exceptions; repositories map | Compiler for PlatformKit; review for data sources |
| Each data source and each PlatformKit folder ships an interface next to its real class; repositories hold the interface | Review |
| Data sources and PlatformKit take configuration (base URL, bucket) through the constructor; never read `BuildConfig` | Review |
| Data sources never use repositories or each other; a store may implement a PlatformKit interface (`UploadStoreImpl` → `uploadkit.UploadStore`) | Review |
| PlatformKit is a library ([A16](#a16--platformkit-is-a-library-not-a-ring)): no Domain or Data types, no business rules | CI + review |
| `uploadkit` and `realtime` use nothing from the rest of the app, and not each other | Review |
| Types other modules use are `public` (Kotlin default); everything else `internal` | Compiler + detekt |

</dd></dl>

### 4.3 MVVM, use cases, repositories

<dl><dd>

| Type | Rules |
|---|---|
| Screen | `@Composable fun <Name>Screen(viewModel: <Name>ViewModel = hiltViewModel(), …)`: collects state, calls view model functions; a stateless `<Name>Content(state, onAction)` holds the layout and the previews |
| ViewModel | `@HiltViewModel class … @Inject constructor(use cases)`; exposes `StateFlow<<Name>State>`; work in `viewModelScope` |
| State | `sealed interface <Name>State` (`Loading`, `Loaded(rows)`, `Failed(messageRes)`) or one `data class` with an enum status |
| Row | `data class LocationRow(...)`, `IssueSnapshot`; never an entity or a Room model |
| Route | `@Serializable sealed interface <Feature>Route : NavKey`, cases carry ids only; `<feature>Entries()` turns a route into its screen ([A19](#a19--feature-routes-and-a-coordinator-per-back-stack)) |
| Coordinator | `NavigationCoordinator`, one per back stack (`CoordinatedStack`); screens call `coordinator.push(route)`; dialogs and bottom sheets stay local state |
| Use case | One per user action, a class with one `suspend operator fun invoke`, one file each in `:domain` `usecases/<feature>/`; depends on repository interfaces only |
| Repository | One per aggregate: `ProjectRepository`, `CaptureRepository`, `IssueRepository`, … |

</dd></dl>

### 4.4 Composition root (Hilt)

<dl><dd>

- All bindings live in `:app` `di/`, one `@Module @InstallIn(SingletonComponent::class)` file per layer ([A20](#a20--di-modules-split-by-layer)):

| File | Provides |
|---|---|
| `CoreModule.kt` | `AppClock`, `AppLogger`, dispatchers |
| `PlatformModule.kt` | PlatformKit services and engines |
| `DataSourcesModule.kt` | Data sources, `SiteLogDatabase`, DAOs, DataStore |
| `RepositoriesModule.kt` | Repositories (`<Noun>RepositoryImpl` as `<Noun>Repository`) |
| `UseCasesModule.kt` | Use cases, one `// region <Feature>` per feature |
| `StartupModule.kt` | `StartupSequence` and its steps |
| `SyncTablesModule.kt` | The map of syncable tables for the sync engine (F04) |
| `debug/…/DevDataModule.kt` | `usecases/devdata/` use cases — debug source set only |

- Bindings use `@Provides` functions in `:app`, so `:domain`, `:data` and `:platformkit` never import Hilt or `javax.inject`.
- A view model is the only Presentation class Hilt builds; it imports `HiltViewModel` and `Inject`, nothing else from DI.

```kotlin
@HiltViewModel
class SplashViewModel @Inject constructor(
    private val startupSequence: StartupSequence,
) : ViewModel() {

    private val _state = MutableStateFlow<SplashState>(SplashState.Running)
    val state: StateFlow<SplashState> = _state.asStateFlow()

    fun onAppear() {
        viewModelScope.launch {
            _state.value = SplashState.Running
            val ready = startupSequence.run()
            if (ready) {
                _state.value = SplashState.Ready
            } else {
                _state.value = SplashState.Failed
            }
        }
    }
}
```

```kotlin
@Composable
fun SplashScreen(viewModel: SplashViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.onAppear()
    }

    SplashContent(
        state = state,
        onTryAgain = viewModel::onAppear,
    )
}
```

| Rule | Detail |
|---|---|
| Scope | PlatformKit services, databases, stateful services `@Singleton`; use cases unscoped |
| Resolution | View models take use cases through the constructor ([A14](#a14--view-models-take-use-cases-through-the-constructor)); route ids arrive through `@AssistedInject` |
| Previews | `<Name>Content` previews take a hand-built state; no Hilt in previews |
| Tests | View models: built directly on test doubles, no Hilt; bindings: `@HiltAndroidTest` + `@TestInstallIn` only where a wiring test needs it |
| Injected seams | Repositories, `AppClock`, `UploadStore`, `UploadTransport`, `CaptureSessionControlling`, `kotlin.random.Random`, `CoroutineDispatcher` |

</dd></dl>

### 4.5 Startup sequence

<dl><dd>

- Each step guarded and logged separately — never one `try` around init.
- `SiteLogApplication.onCreate` runs steps 1–2; `SplashViewModel` runs the rest through `StartupSequence`.

| # | Step | On failure |
|---|---|---|
| 1 | Crashlytics (`Application.onCreate`) | Continue — never blocks launch |
| 2 | Logger | Continue with a no-op logger |
| 3 | Keystore: audit HMAC key, encrypted secrets file | Block, show recovery screen |
| 4 | Room database open + migration | Block, offer diagnostic export |
| 5 | Reconcile WorkManager upload work ↔ upload store | Continue, mark jobs `PENDING` |
| 6 | Audit chain verify (async) | Flag in Diagnostics |
| 7 | Sweep orphan files, stale derived images, old tiles | Log only |
| 8 | App lock gate in `RootScreen` (when Lock app is on) | Biometrics → app passcode → forgot-passcode flow |

</dd></dl>

### 4.6 Error handling

<dl><dd>

| Layer | Returns / throws | Catches to |
|---|---|---|
| Data source / PlatformKit | Its own exceptions (`SecureStoreException`, `NetworkException`, `CameraException`) | Map platform exceptions (`IOException`, `SQLiteException`, `KeyPermanentlyInvalidatedException`) to its own; retry transient ones |
| Data (repository impl) | Never throws: `Outcome<Entity, <Feature>Error>` | Every data source / PlatformKit call: map to `Outcome.Failure`; a case that is not a failure (nothing stored on first launch) returns `Outcome.Success` |
| Domain (use case) | Returns the `Outcome` as is, or chains it (`map`, `flatMap`) | Nothing to catch; a business rule returns `Outcome.Failure` |
| ViewModel | — | No catch: `when (outcome)`, `Failure` → state `Failed(messageRes)` |
| Screen | — | Never; shows the state |

```kotlin
sealed interface Outcome<out T, out E> {
    data class Success<out T>(val value: T) : Outcome<T, Nothing>
    data class Failure<out E>(val error: E) : Outcome<Nothing, E>
}
```

- Own type, not `kotlin.Result` (its failure is always a `Throwable`) and not Arrow ([A21](#a21--own-outcome-type)).
  - Interface: `suspend fun loadAccessSetting(): Outcome<AccessSetting, AccessError>`.
  - Domain errors: one `sealed interface <Feature>Error` per feature in `entities/<feature>/`.
  - Exceptions stop at the repository impl; Domain and Presentation never `try`.
- A result that is a business answer stays a type, not an error (`PasscodeCheck`).
- **Never swallow `CancellationException`:** catch `Exception`, then rethrow if it is a `CancellationException`, or the coroutine never cancels.
- Every catch logs the exception (`logger.error(message, throwable)`); data source / PlatformKit log the technical detail (status, path), the repository the meaning (“Load access settings failed → StorageUnavailable”).
- Errors never skip a layer.

</dd></dl>

</dd></dl>

## 5. Data flows

<dl><dd>

### 5.1 Capture → stored evidence

<dl><dd>

| # | Step | Dispatcher | Failure |
|---|---|---|---|
| 1 | Write file to `filesDir/media/…` | CameraX executor / `Dispatchers.IO` | Abort, surface error |
| 2 | Stream SHA-256 in 1 MiB chunks | `Dispatchers.IO` | Abort, delete file |
| 3 | Build `TrustedTimestamp` + GPS | `Dispatchers.Default` | Degrade confidence, never abort |
| 4 | Insert `Capture` through the Room DAO (`withTransaction`) | Room's IO executor | Retry once, then orphan sweep |
| 5 | Enqueue upload with `priority` | `UploadCoordinator` | Stays `PENDING` |
| 6 | Thumbnail to UI | `Dispatchers.Main` | — |

- **Invariant:** no DB row without a hash; no file without a row for longer than one launch.

</dd></dl>

### 5.2 Concurrency map

<dl><dd>

| Context | Type | Holds |
|---|---|---|
| UI | `Dispatchers.Main`, `viewModelScope` | View model state |
| DB | Room suspend DAOs (own executor) | Transactions |
| Upload | `UploadCoordinator` (one `Mutex`, `CoroutineWorker`) | Job table, effects |
| Capture | CameraX executors, `Dispatchers.IO` for hashing | Camera use cases, recorder |
| BLE | `DeviceLinkManager` (single-thread dispatcher) | `BluetoothGatt` |
| WebSocket | `RealtimeChannel` (`Mutex` + `CoroutineScope`) | Socket, ping job |

- Pass ids across layers, never Room objects.
- Never block `Dispatchers.Main`; inject dispatchers so tests use `StandardTestDispatcher`.
- `GlobalScope` is banned (detekt rule); long-lived work uses an injected `@ApplicationScope CoroutineScope`.

</dd></dl>

</dd></dl>

