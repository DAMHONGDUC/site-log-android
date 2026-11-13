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

```
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

```
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

