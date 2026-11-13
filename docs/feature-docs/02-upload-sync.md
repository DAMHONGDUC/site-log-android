# Upload and sync (Android)

<dl><dd>

- How bytes and metadata leave the phone and reach other devices (iOS or Android). What the app calls: [03-backend.md](03-backend.md).
- Same protocol as iOS ([iOS 02-upload-sync](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/feature-docs/02-upload-sync.md)); this file covers the Android engine.
- Code blocks are contract sketches, not complete files.

| Path | Carries | Spec |
|---|---|---|
| HTTPS presigned PUT | Bytes | [04](#04-upload-engine) |
| Delta sync | Metadata | [08](#08-auth-and-sync) |
| WebSocket | Server-side job state | [10](#10-realtime-progress-channel) |
| BLE | Measuring tools only | [06](01-field-work.md#06-devicelink-ble-measuring-tools) |

</dd></dl>

## Contents

<dl><dd>

- [04 Upload engine](#04-upload-engine) — boundary, protocol, Android background constraints, state machine, policy
- [08 Auth and sync](#08-auth-and-sync) — offline-first and auth, protocol, client engine
- [10 Realtime progress channel](#10-realtime-progress-channel)
- [Beginner terms](#beginner-terms)

---

</dd></dl>

## Beginner terms

<dl><dd>

- **Presigned URL:** a temporary address that allows one upload to S3.
- **ETag:** S3's receipt for an uploaded part, sent back on `complete`.
- **Idempotent:** repeating a request has the same result as sending it once.
- **Cursor / tombstone:** where the next sync page starts / a small deletion record so other devices delete too.
- **Doze / App Standby:** Android modes that delay network and jobs when the phone is idle; WorkManager waits them out.
- **Foreground service:** work the user can see in a notification; the OS keeps the process alive for it.

</dd></dl>

## 04. Upload engine

<dl><dd>

- Module: `:platformkit` `uploadkit/` — standalone, uses nothing else from SiteLog ([A02](00-overview.md#a02--hand-written-upload-engine))

**Goal:** move 150–250 files and several GB to S3, surviving network loss, Doze, process death, reboot and presigned-URL expiry.

### 04.1 Boundary

<dl><dd>

- In: a file path, metadata, an endpoint, an injected store. Does not import `:domain`.

```kotlin
interface UploadStore {
    suspend fun upsert(job: UploadJobRecord)
    suspend fun job(id: UploadJobId): UploadJobRecord?
    suspend fun pendingJobs(limit: Int): List<UploadJobRecord>
    suspend fun updatePart(part: UploadPartRecord)
    suspend fun delete(id: UploadJobId)
}
```

- `:data` implements it over Room (`UploadStoreImpl` in `datasources/local/persistence`).
- `uploadkit` tests use `InMemoryUploadStore` in `src/test/`: the state machine runs with no app, DB or network.
- Files: `UploadCoordinator` (entry point, one `Mutex`), `UploadStateMachine` (pure), `ChunkPlanner`, `UploadTransport` (OkHttp), `UploadWorker` (`CoroutineWorker`), `UploadScheduler` (WorkManager wrapper).

</dd></dl>

### 04.2 Protocol

<dl><dd>

- Same calls as iOS; the backend never touches bytes ([03-backend §7](03-backend.md#7-uploads)).

| Step | Call | Returns |
|---|---|---|
| 1 | `POST /uploads {key, byteSize, contentType, sha256}` | `single {uploadId, url}` or `multipart {uploadId, partSizeBytes, parts[]}` |
| 2 | `PUT <presigned url>` (the file, or a byte range of it) | 200 + ETag |
| 3 | `POST /uploads/:id/complete {parts[{number, etag}]}` | `{remoteKey, verificationCode, verified}` — idempotent |
| 4 | `POST /uploads/:id/parts/refresh {numbers}` | Fresh URLs — **mandatory** after a long stop |
| 5 | `DELETE /uploads/:id` | Abort |

- The backend picks the mode: under 5 MiB single PUT, else uniform 8 MiB parts.
- Every call carries a Firebase ID token; the key prefix must match its uid.

</dd></dl>

### 04.3 Android background constraints

<dl><dd>

| Constraint | Consequence |
|---|---|
| A transfer dies with the process; nothing like a background `URLSession` exists | `UploadWorker` (WorkManager) owns the run; after death WorkManager restarts it and the coordinator resumes **at the next unfinished part** |
| WorkManager work has a ~10-minute execution limit unless it runs in the foreground | Long runs call `setForeground(ForegroundInfo(id, notification, FOREGROUND_SERVICE_TYPE_DATA_SYNC))` with a progress notification |
| Android 15+: `dataSync` foreground services get 6 h per 24 h | `onTimeout` / `isStopped` → finish the current part, persist, return `Result.retry()` |
| Android 13+: posting the progress notification needs `POST_NOTIFICATIONS` | Denied → upload still runs; only the notification is hidden |
| Doze and App Standby buckets delay work | Constraint `CONNECTED`/`UNMETERED`; never fight Doze; Diagnostics shows "will upload when the phone wakes" |
| OEM battery savers (Xiaomi, Oppo, Samsung) may kill work | Diagnostics links to the maker's settings page; nothing is lost, only delayed |
| No temp part files needed | OkHttp `RequestBody` streams a byte range of the original file with an exact `contentLength()` (S3 rejects chunked PUTs) |

- WorkManager: one unique work `upload-queue` (`ExistingWorkPolicy.KEEP`), `setExpedited(RUN_AS_NON_EXPEDITED_WORK_REQUEST)` for the first batch from the foreground, `BackoffPolicy.EXPONENTIAL` only as a safety net (the state machine owns real backoff).
- The manifest declares `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_DATA_SYNC` and the `SystemForegroundService` `foregroundServiceType="dataSync"`.

**Launch reconciliation** (the usual cause of "stuck at 60%"):

| WorkManager | Store record | Action |
|---|---|---|
| Work enqueued or running | Job `UPLOADING`/`PENDING` | Keep |
| Work for a job id the store does not know | Missing | Cancel by tag, log |
| No work | Job `UPLOADING` | Reset to `PENDING`, enqueue the queue work |

- A [realtime](#10-realtime-progress-channel) `snapshot` adds server state; jobs only move forward.

</dd></dl>

### 04.4 State machine

<dl><dd>

- Same machine as iOS, in Kotlin; `reduce` is pure and deterministic; `UploadCoordinator` is the only executor of effects.

```
pending → planning → uploading ⇄ waitingForURL / waitingForNetwork → completing → synced
                         └────────────► failed (retry budget exhausted only)
```

```kotlin
sealed interface UploadEvent {
    data class Planned(val parts: List<UploadPartRecord>) : UploadEvent
    data class PartSucceeded(val number: Int, val etag: String) : UploadEvent
    data class PartFailed(val number: Int, val reason: FailureReason) : UploadEvent
    data class UrlsRefreshed(val parts: List<UploadPartRecord>) : UploadEvent
    data object NetworkBecameAvailable : UploadEvent
    data class Completed(val remoteKey: String) : UploadEvent
    data object CancelledByUser : UploadEvent
}

sealed interface UploadEffect {
    data class EnqueuePart(val number: Int) : UploadEffect
    data class RefreshUrls(val numbers: List<Int>) : UploadEffect
    data class CallComplete(val etags: Map<Int, String>) : UploadEffect
    data class ScheduleRetry(val number: Int, val afterMs: Long) : UploadEffect
    data class Abort(val reason: FailureReason) : UploadEffect
}

class UploadStateMachine(private val random: Random) {
    fun reduce(state: UploadJobState, event: UploadEvent): Pair<UploadJobState, List<UploadEffect>> = TODO()
}
```

| Cause | Reason | Handling |
|---|---|---|
| 401/403 on a presigned URL | `URL_EXPIRED` | `WAITING_FOR_URL`, **no budget consumed** |
| `UnknownHostException`, `ConnectException`, no network | `OFFLINE` | `WAITING_FOR_NETWORK`, no budget consumed |
| 5xx, `SocketTimeoutException`, 429 (`Retry-After`) | `SERVER_TRANSIENT` / `TIMEOUT` | Retry, budget consumed |
| 400, 404, 411, 413 | `PERMANENT` | Abort |
| ETag or checksum mismatch | `INTEGRITY` | Abort, flag for the user |

- Retry: `delay = min(2 s × 2^attempt, 300 s) × jitter(0.8…1.2)`, max 5 attempts; jitter from the injected `Random`.
- Jitter is mandatory: 200 files failing at one signal loss must not retry together.

</dd></dl>

### 04.5 Policy

<dl><dd>

| Policy | Detail |
|---|---|
| Priority queue | Host-supplied priority ([03](01-field-work.md#03-issues-and-before-after-pairing)); plans low; stamped derivatives never upload |
| Wi-Fi only | Constraint `NetworkType.UNMETERED`, on by default; the transport also checks `NetworkCapabilities.NET_CAPABILITY_NOT_METERED` before each part |
| Dedupe | Same `sha256` in a project → skip, reuse the existing `remoteKey` |
| Concurrency | 3 jobs (`Semaphore(3)`), OkHttp `Dispatcher.maxRequestsPerHost = 4` |
| Local files | Never auto-deleted; cleanup is user-initiated and `SYNCED`-only ([09](05-platform.md#09-diagnostics-and-observability)) |
| Logging | `{jobId, part n/N, bytes, elapsedMs, attempt, status, reason}`; never presigned URLs |

```kotlin
object UploadConstants {
    const val PART_SIZE_BYTES: Long = 8L * 1024 * 1024
    const val SINGLE_PUT_THRESHOLD_BYTES: Long = 5L * 1024 * 1024
    const val MAX_CONCURRENT_JOBS: Int = 3
    const val MAX_CONNECTIONS_PER_HOST: Int = 4
    const val MIN_FREE_DISK_BYTES: Long = 500L * 1024 * 1024
    const val RETRY_BASE_DELAY_MS: Long = 2_000
    const val RETRY_CAP_DELAY_MS: Long = 300_000
    const val MAX_RETRY_ATTEMPTS: Int = 5
    const val UNIQUE_WORK_NAME: String = "upload-queue"
    const val NOTIFICATION_CHANNEL_ID: String = "uploads"
}
```

| Risk | Handling |
|---|---|
| Orphaned parts after abort | `DELETE /uploads/:id` + 7-day bucket lifecycle rule |
| Disk fills | Stop planning new jobs, keep running ones, show numbers |
| Account switch with pending jobs | Cancel the unique work, abort old uid's jobs, never write into the new prefix |
| User taps **Force stop** | All work is cancelled by the OS until next launch; reconciliation restarts it |
| Clock skew vs `expiresAt` | 403 is the truth; `expiresAt` only drives proactive refresh |

**Done**

- 200 files / 2 GB complete with matching server hashes
- `adb shell am kill com.dd.sitelog` mid-transfer resumes without re-uploading finished parts
- Airplane mode → `WAITING_FOR_NETWORK` with budget intact
- 60 s URL TTL refreshes automatically
- Wi-Fi only on mobile data sends zero bytes
- Reboot mid-queue: the queue resumes after unlock

**Tests**

- Full (state × event) matrix; final part failure never calls `complete`
- Budget rules; exhausted budget keeps the original file
- Backoff with a seeded `Random`
- `ChunkPlanner` sizes and byte ranges; single-PUT threshold; dedupe
- All three reconciliation cases (WorkManager test helpers: `WorkManagerTestInitHelper`)
- Priority order
- `MockWebServer` flow with 503 at part 3
- Real kill on a device

---

</dd></dl>

</dd></dl>

## 08. Auth and sync

<dl><dd>

- Modules: `:app` `presentation/sync`, `:data` (sync engine), `:platformkit` `networking/`

**Goal:** sign in once, work fully offline, sync all metadata across devices over the hand-written delta protocol.

- Firebase is identity, crash reporting and push only; all data is in Postgres.

| In | Out |
|---|---|
| Firebase Auth: Google (Credential Manager), Apple (`OAuthProvider("apple.com")`), email | Firestore, in any role |
| Cursor-based delta pull, batched push, tombstones, LWW | Multi-user sharing, co-editing, SSO |
| Offline mutation queue (outbox) in Room | Media in the sync protocol |
| Account deletion | |

- **Cross-platform accounts:** the same Firebase project as iOS, so an Apple-ID user signs in on Android with Apple and sees the same data.
- **Remote media:** a pulled `Capture` carries its object key → `POST /media/urls` → shown with Coil (size-bound, disk-cached, cache key = object key). Local originals always read from disk ([A05](00-overview.md#a05--coil-for-remote-images)).

### 08.1 Offline-first and auth

<dl><dd>

- Room is the source of truth while recording; writes return immediately; no screen waits on the network.
- Launching offline after a prior sign-in goes straight in (`FirebaseAuth.currentUser` is cached on device).
- `currentUser.getIdToken(false).await()` on every request — never hand-cache it. One OkHttp `Authenticator` refreshes on 401 (`getIdToken(true)`), shared by concurrent calls; no retry budget consumed.

</dd></dl>

