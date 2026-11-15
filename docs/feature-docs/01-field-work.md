# Field work (Android)

<dl><dd>

- What the supervisor does on site: organize the walk, capture, log and verify issues, pin them on plans, mark up photos, run checklists, read measuring tools.
- Architecture and data model: [00-overview.md](00-overview.md). Product rules are the same as iOS ([iOS 01-field-work](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/feature-docs/01-field-work.md)); this file changes the platform parts.
- Code blocks are sketches of APIs and data shapes, not complete files.

</dd></dl>

## Contents

<dl><dd>

| Spec | Topic |
|---|---|
| [01](#01-projects-sessions-locations) | Projects, sessions, locations |
| [02](#02-capture-pipeline) | Capture pipeline — CameraX, metadata and hashing, pre-record buffer |
| [03](#03-issues-and-before-after-pairing) | Issues and before-after pairing |
| [06](#06-devicelink-ble-measuring-tools) | DeviceLink BLE measuring tools |
| [11](#11-floor-plans-and-issue-pins) | Floor plans and issue pins |
| [12](#12-annotation-and-verifiable-stamps) | Annotation, stamps, dictation, trusted time, verification code |
| [13](#13-checklist-templates) | Checklist templates |

- [Beginner terms](#beginner-terms)

---

</dd></dl>

## Beginner terms

<dl><dd>

- **Offline-first:** save work on the phone before relying on a network.
- **Hash:** a fixed-size fingerprint of file bytes; SHA-256 shows whether a saved video changed.
- **BLE:** Bluetooth Low Energy, used by laser and moisture meters.
- **CameraX use case:** one job bound to the camera: `Preview`, `ImageCapture`, `VideoCapture`, `ImageAnalysis`.
- **OOM:** `OutOfMemoryError`, Android's version of iOS jetsam: decoding full-size bitmaps causes it.

</dd></dl>

## 01. Projects, sessions, locations

<dl><dd>

- Modules: `:app` `presentation/projects`, `:domain`, `:data` `datasources/local/persistence`

**Goal:** build the `Project → Session → Location` tree faster than paper, fully offline, one-handed.

| In | Out |
|---|---|
| CRUD for all three; 3-level tree (`floor → unit → room`) | Multi-user sharing (v2) |
| Bulk generation from a template; cloning a tree from another project | Excel/CAD import |
| Closing and reopening sessions | |

```text
Project list → [+] New project → Project detail: [Sessions] [Locations] [Plans]
  → [+] Start session → Survey screen: locations + capture/issue badges
      → Select location → Capture (02) / Checklist (13) / Plan (11)
  → [End session] → state = CLOSED
```

- Camera entry point in the lower half (FAB or bottom bar); badges read from Room and never wait on upload.
- Bulk generation (`Floors 1..20 × 4 units, "{floor}-{unit:02}"` → 80 locations) runs in one `withTransaction`, previews first 5 / last 3 / total, rejects duplicates, caps at `maxGeneratedPerBatch`.
- Badge counts come from one grouped DAO query per screen (`GROUP BY locationId`), never per row.
- The survey list is a `LazyColumn` with `key = { it.id }` and stable row classes.

```kotlin
data class LocationRow(
    val id: UUID,
    val code: String,
    val captureCount: Int,
    val openIssueCount: Int,
    val pendingUploadCount: Int,
)

object LocationTemplateLimits {
    const val MAX_GENERATED_PER_BATCH: Int = 2000
    const val MAX_DEPTH: Int = 3
    const val PREVIEW_HEAD_COUNT: Int = 5
    const val PREVIEW_TAIL_COUNT: Int = 3
}
```

| Risk | Handling |
|---|---|
| Closing a session by accident | Reopen button; state shown in the top bar |
| Project delete removes GB of media | Confirmation dialog states numbers: "3 sessions, 412 photos, 2.1 GB, 87 not uploaded" |
| Delete while files are `PENDING` | Blocked outright |

**Done**

- Project → session → 80 locations → capture in airplane mode
- 500-row list without dropped frames (JankStats / Perfetto)
- Kill mid-generation leaves no partial records

**Tests**

- Template edge cases (1 floor, 1 unit, collisions); batch cap; depth-4 rejected
- `CLOSED` rejects `addCapture`
- Cascade delete leaves no orphans (Room `ForeignKey.CASCADE`)
- Migration from a 100-location version-1 fixture database

---

</dd></dl>

## 02. Capture pipeline

<dl><dd>

- Modules: `:platformkit` `camera/`, `:data` `datasources/local/filestorage`

**Goal:** one CameraX binding for photo, video and audio without rebinding between shots; every file hashed before it reaches the DB.

| In | Out |
|---|---|
| Photo, video + audio, standalone audio; torch, zoom, flip, timer | Photo editing |
| Continuous capture with a thumbnail strip | Gallery import |
| GPS, `deviceModel`, `capturedAt`, `trustedTime`; SHA-256 on close | |
| 30-second pre-record buffer | |

### 02.1 Session and state

<dl><dd>

```text
ProcessCameraProvider.bindToLifecycle(owner, selector,
    Preview          → PreviewView (AndroidView) or CameraXViewfinder (camera-compose)
    ImageCapture     → photos (CAPTURE_MODE_MINIMIZE_LATENCY)
    VideoCapture<Recorder> → video + audio (Quality.FHD, fallback HD)
)
idle → binding → ready → capturingPhoto → ready
                   ↓                          ↑
               recording → finalizing (Finalize event + hash)
```

- Bind `Preview + ImageCapture + VideoCapture` once; check `CameraInfo` for the supported combination, fall back to binding `VideoCapture` only while recording on low-end devices.
- Camera work on CameraX executors; hashing on `Dispatchers.IO`; UI on `Main`.
- Interruption (call, app backgrounded, camera taken by another app) while recording → `Recording.stop()` → **finalize, never discard**; `VideoRecordEvent.Finalize` with an error still keeps a playable file when `ERROR_SOURCE_INACTIVE`.
- Standalone audio: `MediaRecorder` (AAC in `.m4a`).

</dd></dl>

### 02.2 Metadata and hashing

<dl><dd>

| Field | Source | When unavailable |
|---|---|---|
| `capturedAt` | `Instant.now()` at `onImageSaved` | Always present |
| `trustedTime` | Wall clock + `SystemClock.elapsedRealtimeNanos()` + cached server offset ([12.4](#124-trusted-time)) | Degrades to `DEVICE_ONLY` |
| GPS + accuracy | `FusedLocationProviderClient.getCurrentLocation(PRIORITY_BALANCED_POWER_ACCURACY)` | `null`; a fix older than `LOCATION_STALE_AFTER` is `null`, never wrong |
| `deviceModel` | `Build.MANUFACTURER + " " + Build.MODEL` | Always present |
| `sha256` | Streamed from the closed file | Always present |

```kotlin
fun sha256(file: File): String {
    val digest: MessageDigest = MessageDigest.getInstance("SHA-256")
    val buffer = ByteArray(HashConstants.STREAMING_CHUNK_BYTES)
    file.inputStream().use { input ->
        while (true) {
            val read: Int = input.read(buffer)
            if (read == -1) {
                break
            }
            digest.update(buffer, 0, read)
        }
    }
    return digest.digest().joinToString(separator = "") { byte -> "%02x".format(byte) }
}
```

- Lives in `:core` `crypto/FileHasher.kt`; it throws `IOException` up to the data source.
- Stream in chunks: `file.readBytes()` on a 1.5 GB video is an `OutOfMemoryError`.
- Files: `filesDir/media/<project>/<session>/<capture>.<ext>`, excluded from backup.
- Photos are JPEG (`ImageCapture` output); HEIF only when the device supports it (`ImageCapture.OUTPUT_FORMAT_*` check).

</dd></dl>

### 02.3 Pre-record buffer

<dl><dd>

- Keeps the last 30 s when record is pressed.
- Never raw frames in RAM: an `ImageAnalysis`-free path — a second `Recorder` writes 3-second segments to `cacheDir/prebuffer/` in a ring of `SEGMENT_COUNT` (stop + start a new `Recording` every segment); on record, the segments plus the new clip are joined with **Media3 Transformer** (`Composition` of one `EditedMediaItemSequence`), no re-encode when formats match.
- Off by default; disabled below `MIN_BATTERY_PERCENT` or on devices without concurrent recording support; a known gap of a few frames between segments is accepted and documented in the report.

```kotlin
object CaptureConstants {
    const val VIDEO_BITRATE: Int = 8_000_000
    const val MAX_VIDEO_DURATION_MS: Long = 600_000
    const val LOCATION_STALE_AFTER_MS: Long = 120_000
    const val MIN_FREE_DISK_BYTES_FOR_VIDEO: Long = 500L * 1024 * 1024
}

object PreRecordConstants {
    const val SEGMENT_DURATION_MS: Long = 3_000
    const val SEGMENT_COUNT: Int = 10
    const val MIN_BATTERY_PERCENT: Int = 20
}

object HashConstants {
    const val STREAMING_CHUNK_BYTES: Int = 1_048_576
}
```

| Risk | Handling |
|---|---|
| Disk fills mid-recording | `StatFs` check first; below threshold, block video and state the remainder |
| Thermal throttling | `PowerManager.currentThermalStatus` `SEVERE` → lower bitrate; `CRITICAL` → stop deliberately |
| Permissions | `CAMERA`, `RECORD_AUDIO`, `ACCESS_FINE_LOCATION` requested separately, each when first needed |
| Rotation | `ImageCapture.targetRotation` / `VideoCapture.targetRotation` follow `OrientationEventListener`, or video is sideways in the PDF |
| OEM camera quirks | Test on Samsung, Xiaomi, Pixel; CameraX `CameraInfo` checks, no hard-coded resolutions |

**Done**

- 50 photos with flat memory
- 3-minute 1080p plays back and the hash reproduces
- A call mid-recording finalizes to `PENDING`
- Airplane mode + denied location still captures

**Tests**

- Hash against known vectors and `sha256sum` on 200 MB
- State machine rejects recording during photo
- Interruption finalizes once
- Stale GPS → `null`
- Ring buffer eviction. All run without a camera via `CaptureSessionControlling`

---

</dd></dl>

</dd></dl>

## 03. Issues and before-after pairing

<dl><dd>

- Modules: `:app` `presentation/issues`, `:domain`, `:data`
- Core business feature: prove "broken last week, fixed now" with two photos of the same spot.

**Goal:** log an issue in under 20 s, one-handed; in a later session reopen it at the same location, shoot verification, render the pair in the PDF.

| In | Out |
|---|---|
| Issue CRUD, multiple captures, severity, status, due date | Account-linked assignment |
| Assignee and trade (free text, autocompleted) | Multi-user comment threads (v2) |
| Carry-over of open issues; pairing and closure by verification photo | |
| Phrase library, on-device dictation, voice notes | |

### 03.1 Flows

<dl><dd>

```text
Log:     Capture → shoot → [Attach issue] → ModalBottomSheet (description, severity=MAJOR,
         assignee, due, 🎤 dictate · 🔊 voice note · 📍 pin) → Save → back to camera
Verify:  New session → location → banner "3 open issues from 12 Aug" → issue
         → [Shoot verification] with prior photo at 30% overlay → [Resolved] [Still open] [Worse]
Close:   new Issue (status VERIFIED, verification photo); old.resolvedByIssueId = new.id,
         old.status = RESOLVED; original captures never touched
```

| Decision | Reason |
|---|---|
| Bottom sheet over the camera, camera stays bound | Rebinding CameraX costs ~0.5–1 s |
| Default severity `MAJOR` | Defaulting to minor makes everything minor |
| `assigneeName` is free text | A name that groups the PDF is what gets used on site |
| 30% overlay (`Image` with `alpha = 0.3f` over the preview) | Without it the angles differ and the pair is worthless |

- Phrase library: ~30 seeded phrases; new typed phrases added; sorted by usage; synced via the backend's `phrases` table.

</dd></dl>

### 03.2 Design

<dl><dd>

```kotlin
data class IssuePairing(
    val previous: IssueSnapshot,
    val current: IssueSnapshot?,
    val outcome: PairingOutcome, // RESOLVED / STILL_OPEN / WORSENED
)

object IssueDefaults {
    val SEVERITY: IssueSeverity = IssueSeverity.MAJOR
    const val DUE_DATE_OFFSET_DAYS: Long = 7
    const val MAX_CAPTURES_PER_ISSUE: Int = 20
    const val OVERLAY_OPACITY: Float = 0.30f
    const val STALE_ISSUE_SESSION_THRESHOLD: Int = 3
}
```

- `PairIssuesUseCase` lives in `:domain`, pure Kotlin — the most error-prone logic in the app.
- Upload priority: critical 100 · major 50 · minor 10 · no issue 5 · video −20.

| Risk | Handling |
|---|---|
| Location deleted or renamed between sessions | Delete blocked while open issues exist; rename keeps `id` + audit entry |
| Verification at the wrong spot | Overlay; the PDF prints `Location.code` under every image |
| Issues piling up | Past `STALE_ISSUE_SESSION_THRESHOLD` the banner changes color and the PDF gets a section |

**Done**

- One issue with photo in under 20 s, measured
- Session 2 surfaces exactly the 3 open issues offline
- Closing leaves the original byte-identical

**Tests**

- Pairing with 0/1/n open issues; skips `VERIFIED`
- Closing never mutates prior captures
- Priority across severity × kind
- Delete-with-open-issues rejected; phrase ordering tie-break

---

</dd></dl>

</dd></dl>

## 06. DeviceLink BLE measuring tools

<dl><dd>

- Module: `:platformkit` `bluetooth/` · mock: a debug-only peripheral screen on a second Android phone ([F13](../feature-guide/S7-checklists-devices-diagnostics/F13-devicelink.md))

**Goal:** scan → connect → read a laser distance or moisture meter (Bosch GLM, Leica DISTO) → attach the reading to a `Capture`.

- BLE for control and small values; files never go over BLE.

```kotlin
interface DeviceProfile {
    val serviceUuid: UUID
    val notifyCharacteristicUuid: UUID
    fun parse(bytes: ByteArray): DeviceReadingRecord // pure, tested on captured bytes
}

data class DeviceReadingRecord(
    val value: Double,
    val unit: MeasurementUnit, // METER / MILLIMETER / PERCENT_MOISTURE
    val kind: MeasurementKind, // DISTANCE / AREA / MOISTURE
    val measuredAt: Instant,
    val deviceName: String,
)
```

- One file per vendor profile; `:data` maps the record to the Domain `DeviceReading`.
- `adapterOff → adapterOn → scanning → connecting → discoveringServices → ready`; `disconnected` is normal (meters sleep). `DeviceLinkManager` runs every GATT call on one single-thread dispatcher: Android GATT allows one operation at a time.
- Android BLE rules:
  - Permissions: `BLUETOOTH_SCAN` (with `neverForLocation`) + `BLUETOOTH_CONNECT` on API 31+; `ACCESS_FINE_LOCATION` on 29–30.
  - Scan with a `ScanFilter` on the service UUID and `SCAN_MODE_LOW_LATENCY`, stop after `SCAN_TIMEOUT`; never more than 5 scan starts per 30 s (the OS silently blocks).
  - `connectGatt(context, autoConnect = false, callback, TRANSPORT_LE)`; enable notifications by writing the CCCD descriptor; `close()` the old `BluetoothGatt` before reconnecting (status 133 otherwise).
  - Reconnect to a known address with `autoConnect = true` after the first success.
- UX: reading overlays the preview; status is a colored dot, no dialogs; **never blocks capture**. Readings older than `MEASUREMENT_STALE_AFTER` are not attached.
- The mock (second phone, `BluetoothLeAdvertiser` + `BluetoothGattServer`) scripts invalid values, disconnects and garbage bytes; CI covers `parse` on captured bytes. DeviceLink ships to users only after one real meter passes (feature flag).

```kotlin
object DeviceLinkConstants {
    const val SCAN_TIMEOUT_MS: Long = 15_000
    const val CONNECT_TIMEOUT_MS: Long = 10_000
    const val RECONNECT_BASE_DELAY_MS: Long = 2_000
    const val RECONNECT_MAX_DELAY_MS: Long = 60_000
    const val MINIMUM_RSSI: Int = -85
    const val MEASUREMENT_STALE_AFTER_MS: Long = 30_000
    const val BACKGROUND_DISCONNECT_DELAY_MS: Long = 300_000
}
```

**Risks**

- No hardware for byte formats → start on the mock and ship a raw-bytes "generic" profile
- Permission denied → feature hides itself
- Battery → no background scanning

**Done**

- Reads from the mock into a capture
- Killing the mock greys the dot, capture still works
- Relaunch reconnects within 10 s

**Tests**

- Vendor `parse` on real bytes; truncated / bad-checksum payloads throw
- Unit conversion; stale reading not attached
- Reconnect backoff capped
- Full flow against the mock

---

</dd></dl>

## 11. Floor plans and issue pins

<dl><dd>

- Modules: `:platformkit` `rendering/` (tiles), `:data` `datasources/local/filestorage` (plan files), `:app` `presentation/plans`

**Goal:** import a floor plan, pin issues at exact coordinates, print a pin map per sheet.

- Out: DWG/CAD, measuring, plan editing.

```kotlin
@Entity(
    tableName = "plan_sheets",
    foreignKeys = [ForeignKey(entity = ProjectModel::class, parentColumns = ["id"], childColumns = ["projectId"], onDelete = ForeignKey.CASCADE)],
    indices = [Index("projectId")],
)
data class PlanSheetModel(
    @PrimaryKey val id: String,
    val projectId: String,
    val name: String,
    val sourceFilePath: String,
    val pageIndex: Int,
    val pixelWidth: Int,
    val pixelHeight: Int,
    val sha256: String,
    val locationId: String?,
)
```

- The pin lives on the issue row (`planSheetId`, `pinX`, `pinY`), matching the backend.
- **Coordinates are 0…1, never pixels**, so pins survive re-rendering at any scale. `Offset` only inside `presentation/plans`.
- One pin per issue; every move writes an audit entry.
- Rendering: tiles on demand (A0 at 300 dpi never becomes one bitmap): `PdfRenderer.Page.render(bitmap, destClip, transform, RENDER_MODE_FOR_DISPLAY)` per 512 px tile, `BitmapRegionDecoder` for JPEG/PNG plans; tile cache in `cacheDir/plans/`; Compose `Canvas` draws visible tiles; pins in an overlay `Box`.
- Interaction: `detectTransformGestures` for pan/zoom; **long-press drops a pin** (tap is for panning), tap opens the issue, drag confirms on release. Pins colored by severity, hollow when resolved, clustered below `CLUSTER_ZOOM_THRESHOLD`.
- Import: Storage Access Framework (`ActivityResultContracts.OpenDocument`, `application/pdf`, `image/*`) or a share intent; the file is copied into `filesDir/plans/`, hashed, uploaded at low priority, capped at `MAX_PLAN_FILE_BYTES`.

```kotlin
object PlanConstants {
    const val MAX_PLAN_FILE_BYTES: Long = 100L * 1024 * 1024
    const val MAX_ZOOM_SCALE: Float = 8f
    const val CLUSTER_ZOOM_THRESHOLD: Float = 2f
    const val CLUSTER_RADIUS_DP: Float = 24f
    const val PIN_DROP_LONG_PRESS_MS: Long = 400
    const val TILE_SIZE_PX: Int = 512
    const val UPLOAD_PRIORITY: Int = 1
}
```

| Risk | Handling |
|---|---|
| Revised drawing | New `PlanSheet`, offer to migrate pins by normalized coordinate, keep the old sheet |
| Encrypted or corrupt PDF | `PdfRenderer` throws `SecurityException` / `IOException` → rejected at import with the reason |
| Pin on the wrong sheet | Sheet name in the top bar and under every pin map |

**Done**

- 20-page A1 PDF, 50 pins, smooth pan/zoom
- A0 peak memory under 150 MB (Memory Profiler)
- Pins land on the same spot after relaunch at a different zoom
- Fully offline

**Tests**

- Normalized point round-trip through zoom/pan; same point at two scales
- Clustering; oversized and encrypted imports rejected
- Pin migration; sheet delete blocked while pins exist

---

</dd></dl>

## 12. Annotation and verifiable stamps

<dl><dd>

- Modules: `:platformkit` `rendering/` (stamped copies), `:domain` (models)

**Rule:** the original file is never touched; markup is a separate layer; stamps are a derived copy.

### 12.1 Annotation layer

<dl><dd>

```kotlin
@Serializable data class NormalizedPoint(val x: Double, val y: Double)
@Serializable data class NormalizedRect(val origin: NormalizedPoint, val width: Double, val height: Double)

@Serializable
sealed interface AnnotationShape {
    @Serializable @SerialName("arrow") data class Arrow(val from: NormalizedPoint, val to: NormalizedPoint, val style: AnnotationStyle) : AnnotationShape
    @Serializable @SerialName("rect") data class Rect(val rect: NormalizedRect, val style: AnnotationStyle) : AnnotationShape
    @Serializable @SerialName("ellipse") data class Ellipse(val rect: NormalizedRect, val style: AnnotationStyle) : AnnotationShape
    @Serializable @SerialName("freehand") data class Freehand(val points: List<NormalizedPoint>, val style: AnnotationStyle) : AnnotationShape
    @Serializable @SerialName("text") data class Text(val text: String, val at: NormalizedPoint, val style: AnnotationTextStyle) : AnnotationShape
}
```

- `:domain` is pure JVM, so these use kotlinx.serialization annotations only (the `serialization-core` artifact is allowed in `:domain`); the JSON must match the iOS `shapes` JSON exactly — one sample file is shared in both test suites.
- `Annotation { id (= capture id), shapes, createdAt, updatedAt }`, 1:1 with a capture.
- Editor: Arrow (default), Box, Circle, Pen, Text, drawn with Compose `Canvas` + `pointerInput`; stroke scales with image size; undo capped at `MAX_UNDO_STEPS`; commits on exit. Edits write an audit entry.

</dd></dl>

### 12.2 Stamped derivatives

<dl><dd>

- File: `filesDir/derived/<…>/<captureId>-stamped.jpg`, drawn with `android.graphics.Canvas` on a downsampled bitmap (`ImageDecoder` target size `DERIVED_IMAGE_MAX_PIXEL_SIZE`).
- Carries date/time, coordinates and accuracy, project, location code, logo.
- Generated on demand, cached; never hashed, never evidence.

</dd></dl>

### 12.3 Dictation and voice notes

<dl><dd>

- API 31+: `SpeechRecognizer.createOnDeviceSpeechRecognizer(context)` — site audio never leaves the device; check `SpeechRecognizer.isOnDeviceRecognitionAvailable`.
- API 29–30 or no on-device model: the mic button opens the phrase library instead; never fall back to cloud recognition.
- Vietnamese (`vi-VN`) and English; language packs may need a download, offered once.
- A voice note is an ordinary audio `Capture`; dictated text lands in `Issue.detail`.

</dd></dl>

### 12.4 Trusted time

<dl><dd>

```kotlin
data class TrustedTimestamp(
    val deviceTime: Instant,
    val elapsedRealtimeNanos: Long, // detects clock jumps within a boot
    val bootCount: Int,             // Settings.Global.BOOT_COUNT: a reboot resets elapsed time
    val serverOffsetMs: Long?,      // serverTime − deviceTime
    val offsetMeasuredAt: Instant?,
    val confidence: TimeConfidence,
)
```

| Confidence | Condition |
|---|---|
| `SERVER_VERIFIED` | Offset measured within `SERVER_OFFSET_VALIDITY`, under `MAX_ACCEPTABLE_OFFSET` |
| `SESSION_CONSISTENT` | No server contact, elapsed realtime matches wall-clock progression in the same boot |
| `DEVICE_ONLY` | Clock jumped, rebooted, or no server contact |

- Offset comes from the `Date` header of any backend response (`HttpClient` reports it).
- Confidence is **printed in the report**; a capture is never rejected for low confidence.

</dd></dl>

### 12.5 Verification code

<dl><dd>

- On upload completion the backend records the hash and returns a short code; printed under each image; resolvable at `GET /verify/:code` ([03-backend §8](03-backend.md#8-verify)).

| State | Meaning | Report wording |
|---|---|---|
| `registered` | Server recorded a client-supplied hash | "registered {code}" |
| `verified` | Server re-read the object and recomputed the hash | "verified {code}" |
| unsynced | Nothing left the device | "not yet verified" |

```kotlin
object AnnotationConstants {
    const val MAX_UNDO_STEPS: Int = 20
    const val STROKE_WIDTH_RATIO: Float = 0.004f // × image width
    const val DERIVED_IMAGE_MAX_PIXEL_SIZE: Int = 2000
}

object TrustedTimeConstants {
    const val SERVER_OFFSET_VALIDITY_MS: Long = 3_600_000
    const val MAX_ACCEPTABLE_OFFSET_MS: Long = 120_000
}
```

**Done**

- 5 shapes survive relaunch over an original whose hash still verifies
- Phone markup is proportional on A4; iOS-made markup renders the same on Android
- Deleting the derived cache loses nothing
- Moving the clock a day ahead drops confidence to `DEVICE_ONLY` and the report says so
- Dictation works in airplane mode (API 31+)

**Tests**

- `AnnotationShape` JSON round-trip against the shared iOS sample
- Same render at two sizes; original never mutated; undo bounds
- Confidence for fresh/stale offset, clock jump, reboot
- "not yet verified" wording; stamp position in three orientations

---

</dd></dl>

</dd></dl>

## 13. Checklist templates

<dl><dd>

- Modules: `:app` `presentation/checklists`, `:domain`, `:data`

**Goal:** walk a location against a template, tick items, turn a failed item into an issue without retyping.

- Out: weighted scoring, branching, regulatory libraries.

| Table (Room) | Key columns |
|---|---|
| `checklist_templates` | `id`, `name`, `appliesTo?`, `isBuiltIn`, `usageCount` |
| `checklist_template_items` | `id`, `templateId`, `order`, `text`, `defaultSeverity`, `requiresPhotoOnFail` |
| `checklist_runs` | `id`, `templateId`, `sessionId`, `locationId`, `startedAt`, `completedAt?` |
| `checklist_results` | `id`, `runId`, `templateItemId`, `outcome`, `note`, `issueId?`, `answeredAt?`; unique `(runId, templateItemId)` |

- A run stores its own results; editing a template never rewrites completed runs.
- Flow: pick a template (recent first) → one row per item `[✓ Pass] [✕ Fail] [– N/A]` (segmented buttons) → fail opens the camera if required and prefills the issue sheet ([03](#03-issues-and-before-after-pairing)) → top bar progress "14/22"; finish at 100% answered. Runs are resumable.
- Built-ins (handover: living room, bedroom, bathroom, kitchen, balcony; electrical; plumbing; structural) are editable and duplicable, not deletable; edits fork a copy. Templates export/import as JSON (same format as iOS).
- The PDF gets a per-location table that shows what **passed** too.

```kotlin
object ChecklistConstants {
    const val MAX_ITEMS_PER_TEMPLATE: Int = 200
    const val RECENT_TEMPLATE_COUNT: Int = 5
    const val TEMPLATE_EXPORT_VERSION: Int = 1
}
```

**Done**

- 22-item run offline, two fails → two prefilled issues
- Kill mid-run resumes
- Template edit leaves the completed run and its PDF section unchanged

**Tests**

- Fail creates one issue with item text and severity; fail → pass resolves it
- Template edit does not alter runs; completion percentage
- JSON import validation; built-ins fork on edit

</dd></dl>
