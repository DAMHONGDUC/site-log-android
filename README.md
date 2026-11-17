# SiteLog Android

<dl><dd>

- Android (Jetpack Compose) edition of the SiteLog docs: same product, same backend, same sprints and feature numbers as the iOS app.
- Repository: [site-log-android](https://github.com/DAMHONGDUC/site-log-android); specs, guides and roadmap live in `docs/`.
- Package and application id `com.dd.sitelog` (from the Android Studio template). iOS uses `app.dd.site.log`: Firebase has one app per platform, and the backend does not care which app calls it.
- Source docs it mirrors: [iOS docs](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs) · backend: [site-log-be](https://github.com/DAMHONGDUC/site-log-be).

</dd></dl>

## Contents

<dl><dd>

1. [Docs map](#1-docs-map)
2. [Getting started](#2-getting-started)
3. [Repo layout](#3-repo-layout)
4. [iOS → Android map](#4-ios--android-map)
5. [What is shared, what is not](#5-what-is-shared-what-is-not)
6. [Design](#6-design)

</dd></dl>

## 1. Docs map

<dl><dd>

| Doc | Contents |
|---|---|
| [ROADMAP.md](docs/ROADMAP.md) | Sprints S0–S9, features F00–F18, tasks with hours and ticks |
| [feature-guide/](docs/feature-guide/README.md) | One folder per sprint, one guide per feature: contract → domain → fake data → view models → local → backend → sync → tests |
| [00-overview.md](docs/feature-docs/00-overview.md) | Product, stack, data model, architecture, conventions, decisions A01–A22 |
| [01-field-work.md](docs/feature-docs/01-field-work.md) | 01 Projects · 02 Capture · 03 Issues · 06 DeviceLink · 11 Plans · 12 Annotation · 13 Checklists |
| [02-upload-sync.md](docs/feature-docs/02-upload-sync.md) | 04 Upload engine · 08 Auth and sync · 10 Realtime |
| [03-backend.md](docs/feature-docs/03-backend.md) | 14 Backend API: what the Android app calls and how |
| [04-reporting-security.md](docs/feature-docs/04-reporting-security.md) | 05 PDF and CSV export · 07 Security and audit log |
| [05-platform.md](docs/feature-docs/05-platform.md) | Dev setup · 09 Diagnostics · Testing · 17 CI/CD · 18 Push and deep links · 19 Profiling |
| [compose-rules.md](docs/compose-rules.md) | Compose rules: small composables, state, design tokens, strings |
| Design | Shared with iOS: HTML pages and Figma links in [§6 Design](#6-design) |

- Docs rule: content sits indented under its title. After editing any `.md`, run `python3 tools/docs/indent-md.py`.

</dd></dl>

## 2. Getting started

<dl><dd>

| Step | Action |
|---|---|
| 1 | Install the latest stable Android Studio and Android SDK 36 + 37; Gradle downloads its own JDK 25 on the first build |
| 2 | Clone `site-log-android`; open the root folder in Android Studio; let Gradle sync |
| 3 | Create an emulator: **Device Manager → Pixel 9, API 36** (keep a real phone for camera and Bluetooth) |
| 4 | Run the `app` configuration (⌃R / Shift+F10); tests: `./gradlew testDebugUnitTest` |
| 5 | `./gradlew spotlessApply detekt lint` before every commit; CI runs the same |
| 6 | Full tooling (format on save, Live Edit, previews): [05-platform § Dev setup](docs/feature-docs/05-platform.md#dev-setup) |

</dd></dl>

## 3. Repo layout

<dl><dd>

- Clean Architecture + MVVM; layers are Gradle modules; the `app` module holds the composition root and Presentation.
- Target layout: today only the Android Studio template `app/` exists; modules arrive with S0 ([F00](docs/feature-guide/S0-foundation-access/F00-foundation.md)).

```text
site-log-android/
├── app/                                  # composition root + Presentation (Compose)
│   └── src/
│       ├── main/java/com/dd/sitelog/
│       │   ├── SiteLogApplication.kt     # @HiltAndroidApp; Crashlytics first
│       │   ├── MainActivity.kt           # single activity, setContent { RootScreen() }
│       │   ├── di/                       # Hilt modules, one file per layer
│       │   ├── startup/                  # StartupSequence + steps/
│       │   ├── fakes/                    # skeleton sample data; shrinks as each feature lands
│       │   └── presentation/             # Screen + ViewModel; 2+ screens → one folder per screen
│       │       ├── navigation/  root/  access/  maintab/  <feature>/
│       ├── main/res/values/strings.xml   # every UI string key
│       └── debug/                        # dev menu, DevDataModule, network security config
├── domain/                               # pure Kotlin (JVM) module: entities/ repositories/ usecases/
├── data/                                 # repositories/ datasources/{local,remote}/ models/{local,remote}/ mappers/
├── platformkit/                          # system services + engines: camera/ bluetooth/ biometrics/ … uploadkit/ realtime/
├── core/                                 # logger/ clock/ crypto/
├── designsystem/                         # tokens/ components/ (Compose, Material 3)
├── build-logic/                          # Gradle convention plugins
├── gradle/libs.versions.toml             # every dependency and version
├── tools/ci/  tools/docs/                # guard scripts · indent-md.py
├── fastlane/  .github/workflows/
├── docs/                                 # specs, guides, ROADMAP.md, compose-rules.md
└── README.md
```

</dd></dl>

## 4. iOS → Android map

<dl><dd>

| Area | iOS | Android |
|---|---|---|
| UI | SwiftUI | Jetpack Compose, Material 3 |
| Language | Swift 6, strict concurrency | Kotlin 2.x, coroutines + Flow |
| Layers | Local Swift packages | Gradle modules (`:domain` is pure JVM) |
| Project file | XcodeGen `project.yml` | Gradle Kotlin DSL + `libs.versions.toml` + `build-logic/` |
| DI | Factory (`Container.shared`) | Hilt (`@Module`, `hiltViewModel()`) |
| View model | `@Observable` class | `androidx.lifecycle.ViewModel` + `StateFlow` |
| Navigation | `NavigationStack` + coordinator | Navigation 3 back stack + coordinator |
| Result | `Result<T, E>` | `Outcome<T, E>` (own sealed type in `:domain`) |
| Local store | SwiftData | Room |
| Small settings | `UserDefaults` | DataStore (Preferences) |
| Secrets | Keychain | Android Keystore + encrypted DataStore file |
| Biometrics | Face ID (`LAContext`) | `BiometricPrompt` |
| Camera | AVFoundation | CameraX (+ Media3 Transformer to join clips) |
| Background upload | Background `URLSession` | WorkManager + OkHttp |
| HTTP / WebSocket | `URLSession` | OkHttp |
| Remote images | Kingfisher | Coil 3 |
| PDF render / plans | PDFKit, `CATiledLayer` | `PdfDocument` / `PdfRenderer`, Compose tiles |
| BLE | CoreBluetooth | `android.bluetooth.le` |
| Speech | `SFSpeechRecognizer` | `SpeechRecognizer` (on-device) |
| Push | Firebase Cloud Messaging → APNs | Firebase Cloud Messaging |
| Logging | `os.Logger` | `AppLogger` over `android.util.Log` |
| Lint / format | SwiftLint, swift-format | detekt, ktlint (Spotless), Android Lint |
| Tests | Swift Testing | JUnit 4, kotlinx-coroutines-test, Turbine, Truth, Robolectric |
| UI tests | XCUITest | Compose UI test |
| Profiling | Instruments | Android Studio Profiler, Perfetto, Macrobenchmark |
| Beta / store | TestFlight / App Store | Play internal testing / Google Play |
| Hot reload | Inject + InjectionNext | Compose Live Edit + `@Preview` |

</dd></dl>

## 5. What is shared, what is not

<dl><dd>

| Shared with iOS | Android only |
|---|---|
| Product, data model, feature numbers, sprint order | Gradle modules, Hilt, Room, WorkManager |
| Backend API, sync protocol, upload protocol, WebSocket messages, FCM push (`POST /devices`, [03-backend §10](docs/feature-docs/03-backend.md#10-push-registration)) | `platform: "android"` gets a data-only FCM message |
| Firebase project (`sitelog-dev`, `sitelog-prod`) | An Android app in each Firebase project, SHA-1/SHA-256 fingerprints |
| Design plan, mockups, Figma | Material 3 components, 48 dp touch targets |
| Backend guides in site-log-be | Emulator reaches the Mac at `10.0.2.2`, not `localhost` |

</dd></dl>

## 6. Design

<dl><dd>

- Shared with iOS: the HTML pages and the design plan live in the [iOS repo](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/design); Figma: [Site Log iOS Design](https://www.figma.com/design/uJZcCP6lzw8QTSkcu8uqvV/Site-Log-iOS-Design).
- Open the HTML pages through a local server from a clone of the iOS repo (`python3 -m http.server 8765 --directory docs/design`); GitHub shows only their source.

| Page | HTML (iOS repo) | Figma |
|---|---|---|
| 01 Foundations: app icon (adaptive layers), text styles (Inter / JetBrains Mono), colours, spacing | [1-foundations.html](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/design/html/1-foundations.html) | [01 Foundations](https://www.figma.com/design/uJZcCP6lzw8QTSkcu8uqvV/Site-Log-iOS-Design?node-id=0-1) |
| 03 Android: 69 screens in Material 3 + 3 dark | [3-android.html](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/design/html/3-android.html) | [03 Android](https://www.figma.com/design/uJZcCP6lzw8QTSkcu8uqvV/Site-Log-iOS-Design?node-id=54-3) |
| 05 System design: 21 components, Android beside iOS | [5-system-design.html](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/design/html/5-system-design.html) | Not imported yet |

- Plan and screen inventory: [00-design-plan.md](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/design/00-design-plan.md); Figma sync status: [figma-sync](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/design/figma-sync/README.md); Material 3 mapping: [compose-rules §5](docs/compose-rules.md#5-design-system).

</dd></dl>
