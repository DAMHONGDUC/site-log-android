# Reporting and security (Android)

<dl><dd>

- The paid deliverable and the protection of the evidence behind it.
- Same product rules as iOS ([iOS 04-reporting-security](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/feature-docs/04-reporting-security.md)); this file covers Android rendering, storage and app lock.
- Code blocks are sketches.

</dd></dl>

## Contents

<dl><dd>

- [05 PDF and spreadsheet export](#05-pdf-and-spreadsheet-export) — sections, rendering, spreadsheet
- [07 Security and audit log](#07-security-and-audit-log) — at rest, app lock, audit log, leaks
- [Beginner terms](#beginner-terms)

---

</dd></dl>

## Beginner terms

<dl><dd>

- **File-based encryption (FBE):** Android encrypts each user's internal storage; files are unreadable until the first unlock after boot.
- **Android Keystore:** hardware-backed store for keys; the app can use a key but never read its bytes.
- **HMAC:** a keyed fingerprint that proves data was not changed.
- **AES-GCM:** encryption that also detects changes. **PBKDF2:** a deliberately slow way to derive a key from a passcode.
- **`FLAG_SECURE`:** a window flag that blanks the app in Recents and blocks screenshots.

</dd></dl>

## 05. PDF and spreadsheet export

<dl><dd>

- Modules: `:domain` `usecases/reports` (pagination), `:platformkit` `rendering/pdf` (drawing)

**Goal:** turn a `Session` into a branded, signed PDF plus CSV/XLSX, shareable on the spot with no network — never truncated, never missing images, never crashing on page 38 of 40.

| In | Out |
|---|---|
| Branded cover, summary by severity / floor / assignee | Fully custom layouts (v2) |
| Plan pin maps, checklist tables, issue blocks with annotated photos | PKI digital signatures — handwritten signatures with internal acceptance value, and the app says so |
| Before/after pairs, stale-issue section, two-party signature page | |
| Grouping by location or by assignee (a parameter, not a second code path) | |
| PDF + CSV/XLSX through the share sheet (`Intent.ACTION_SEND` + `FileProvider`) or Save to… (`CreateDocument`) | |

### 05.1 Sections

<dl><dd>

| Section | Source |
|---|---|
| Cover — project, client, dates, surveyor, logo | Project + `ReportBranding` |
| Summary — counts by severity, floor, assignee | Issues |
| Plan pin maps, one page per sheet | [11](01-field-work.md#11-floor-plans-and-issue-pins) |
| Checklist results per location | [13](01-field-work.md#13-checklist-templates) |
| Issue detail, before/after pairs, stale issues | [03](01-field-work.md#03-issues-and-before-after-pairing), [12](01-field-work.md#12-annotation-and-verifiable-stamps) |
| Signatures | Compose `Canvas` pen capture → PNG + `signedAt`, `signerName`, `signerRole` per party |

```kotlin
data class ReportBranding(
    val companyName: String,
    val logoFilePath: String?,
    val accentColorHex: String,
    val clientBlock: String,
    val footerNote: String,
)
```

- Branding: one `report_branding` row per user; the logo uploads like any file.
- Signing sets `Session.state = EXPORTED`; re-signing creates a `-R2` revision, keeps the original.

</dd></dl>

### 05.2 Rendering

<dl><dd>

- `android.graphics.pdf.PdfDocument` + `Canvas`: no third-party library, no WebView printing.

```
Session → SessionSnapshot (one query) → LayOutReportUseCase (:domain, pure) → List<ReportPage>
        → ReportRepositoryImpl maps to List<PdfPageLayout>
        → PdfPageRenderer (:platformkit, one page at a time, bitmaps recycled after each page)
        → file in cacheDir → atomic rename into filesDir/exports/ → share
```

| Rule | Reason |
|---|---|
| Layout counts images, renderer draws them | Guarantees no dropped evidence |
| Decode with `ImageDecoder` target size / `inSampleSize` to `MAX_IMAGE_PIXEL_SIZE` | 160 full-size bitmaps = `OutOfMemoryError` |
| `Dispatchers.Default` / `IO`; UI gets progress only | |
| Write to a temp file, then `renameTo` | A kill mid-render never leaves a partial file |
| `PdfDocument` writes pages as they finish (`finishPage`), one open page at a time | Memory stays flat over 40 pages |
| Renders the stamped, annotated derivative | The archive keeps the clean original |
| Text with `StaticLayout` + bundled Noto Sans (Vietnamese diacritics) | System fonts differ per OEM; the PDF must look the same everywhere |

**Integrity on every page**

- Under each image: first 8 hash characters, the code and its state (`registered`/`verified`), time confidence when below `SERVER_VERIFIED`.
- Unsynced captures print "not yet verified".
- Final page: SHA-256 over the `captureId`-sorted hash list.
- Spacing, color and type come from `:designsystem` tokens (plain values usable outside Compose), inside the PDF too.

**Spreadsheet**

- CSV always (UTF-8 with BOM so Excel shows Vietnamese); XLSX above `CSV_ONLY_ROW_THRESHOLD`, written as a minimal OOXML zip with `ZipOutputStream`; streamed with a `BufferedWriter`.
- One row per issue: `report_id, project, session_date, location_code, plan_sheet, pin_x, pin_y, issue_id, title, detail, severity, status, assignee, trade, due_date, capture_count, first_capture_sha256, verification_code, created_at`.

```kotlin
object ReportConstants {
    const val PAGE_WIDTH_PT: Int = 595  // A4 @72dpi
    const val PAGE_HEIGHT_PT: Int = 842
    const val MARGIN_PT: Float = 40f
    const val IMAGES_PER_ROW: Int = 2
    const val MAX_IMAGE_PIXEL_SIZE: Int = 1400
    const val JPEG_QUALITY: Int = 80
    const val HASH_PREFIX_LENGTH: Int = 8
    const val STALE_ISSUE_SESSION_THRESHOLD: Int = 3
    const val CSV_ONLY_ROW_THRESHOLD: Int = 5000
    const val MAX_LOGO_PIXEL_SIZE: Int = 600
}
```

| Risk | Handling |
|---|---|
| Image missing from disk | Placeholder "Image no longer on device — {hash}"; never crash or skip |
| Non-ASCII file names | `FileProvider` URI + `DISPLAY_NAME`; test with "Ánh Dương – Block B" |
| Too large for chat apps (Zalo, Messenger) | Show estimated size first, offer a compression level |

**Done**

- 200 images / 40 issues on a 4 GB RAM phone under 200 MB peak
- Image count equals non-excluded capture count (asserted)
- Pairs matched and labeled with `Location.code`
- Works in airplane mode; opens in Google Drive PDF viewer and Acrobat

**Tests**

- Image count; empty / 1 / 500 issues; 20-image issue paginates
- Missing file placeholder; hash-of-hashes order-stable
- Both groupings give the same total; CSV escaping; "not yet verified"
- Pin numbering matches detail; 200-image memory (instrumented)

---

</dd></dl>

</dd></dl>

## 07. Security and audit log

<dl><dd>

- Modules: `:app` `presentation/access/` (`biometriclock`, `passcodeunlock`, `privacy`), `presentation/settings/applock`, `:domain`

**Goal:** media and reports are unreadable if the phone is lost, unmodifiable without a trace, and never leaked into logs or analytics.

### 07.1 At rest

<dl><dd>

| Data | Protection |
|---|---|
| Media, plans, exports | Internal storage (`filesDir`), FBE credential-encrypted |
| Room database | Internal storage, FBE |
| Export bundles leaving the app | AES-GCM, per-bundle key |
| Audit chain | HMAC-SHA256 with a Keystore key (`KeyProperties.PURPOSE_SIGN`, never exported) |
| Passcode hash, lockout counter, Firebase-related secrets | `datastore/secrets.preferences_pb` DataStore; values sealed with a Keystore AES-GCM key |

**Why not app-level AES for media**

- Uploads stream from the original file; encrypted media would need a plaintext copy for the whole upload window: worse than nothing, and double the disk.
- FBE already keeps files unreadable before the first unlock, and readable afterwards so uploads continue while the screen is locked.
- Reports that leave the app are the copies that need extra protection: the export bundle is AES-GCM.

</dd></dl>

### 07.2 App lock

<dl><dd>

| Screen | Purpose |
|---|---|
| S4 **Settings → Privacy → App lock** | Two groups: *Lock app*, *Lock after* |
| S5 | Set passcode |
| A1 | Biometric unlock (fingerprint or face, whichever the phone has) |
| A3 | App passcode unlock |

| Setting | Values | Default |
|---|---|---|
| Lock app | On / Off | Off |
| Passcode | 6 digits, app-only; required before Lock app turns on | — |
| Unlock with biometrics | On / Off; hidden when `BiometricManager.canAuthenticate(BIOMETRIC_STRONG)` is not `SUCCESS` | On once a passcode is set |
| Lock after | Immediately · 1 minute · 5 minutes · 15 minutes · 1 hour | 5 minutes |

| Rule | Detail |
|---|---|
| Turning on | Set passcode (enter + confirm) first, then offer biometrics |
| Turning off, changing passcode | Requires the current passcode or biometrics |
| Biometrics | `BiometricPrompt` with `BIOMETRIC_STRONG` and a `CryptoObject` over a Keystore key; negative button = **Use app passcode** |
| Biometry changed | The key is created with `setUserAuthenticationRequired(true)` + `setInvalidatedByBiometricEnrollment(true)`; a new fingerprint makes `Cipher.init` throw `KeyPermanentlyInvalidatedException` → biometrics off, passcode required |
| Passcode storage | Never stored: PBKDF2-HMAC-SHA256 + 16-byte random salt in the secure store |
| Wrong attempts | 5 free, then waits of 1 · 5 · 15 · 60 min; counter and wait-until in the secure store, so killing the app does not reset them; wait measured against `elapsedRealtime` + boot count |
| Forgot passcode | Device credential check (`BiometricPrompt` with `DEVICE_CREDENTIAL` on API 30+, `KeyguardManager.createConfirmDeviceCredentialIntent` on 29) → set a new app passcode; no data erased |
| When it locks | Cold start, and return to foreground (`ProcessLifecycleOwner` `ON_START`) after *Lock after*, measured with `SystemClock.elapsedRealtime()` (not wall time) |
| Recents | While Lock app is on: `window.addFlags(FLAG_SECURE)` (blank thumbnail, no screenshots) and `setRecentsScreenshotEnabled(false)` on API 33+ |

- A UI gate in `RootScreen`, not encryption.

</dd></dl>

### 07.3 Audit log

<dl><dd>

- Records every view, delete, export, report exclusion, metadata change, pin move, annotation edit.

```kotlin
data class AuditEntry(
    val id: UUID,
    val action: AuditAction,
    val subjectId: String,
    val actorId: String,             // Firebase uid, or "offline"
    val occurredAt: Instant,
    val previousEntryHmac: String,
    val hmac: String,                // HMAC-SHA256(key, canonical(fields) + previousEntryHmac)
)
```

- The chain makes edits and deletions detectable; canonical form matches iOS so the same verifier logic applies.
- Key generated once in the Keystore, never leaves the device; verified at launch; a break raises a Diagnostics flag and a Crashlytics non-fatal.
- Entries survive project deletion. Losing the key on a new phone is accepted: the log is per device.

</dd></dl>

### 07.4 Leaks and secrets

<dl><dd>

| Rule | Detail |
|---|---|
| Never log presigned URLs, tokens, secure-store values | They carry credentials; OkHttp logging interceptor only in debug, at `BASIC`, with `redactHeader("Authorization")` |
| Never log user text | Log `issueId`, not `issue.detail` |
| Crashlytics | IDs and enums only; no free text, no GPS |
| Logger | `AppLogger` only; no `println`, no `Log.*` elsewhere (CI guard); release logs stop at `INFO` |
| Secrets | `google-services.json`, signing keys written by CI; `.example` files committed |
| Privacy | GPS disclosed in onboarding and can be turned off; Play **Data safety** form declares location, device IDs, crash data; account deletion is immediate locally, within 30 days on the server, and reachable from a web link (Play requirement) |
| Backups | `data_extraction_rules.xml` excludes media, DB and the secure store from cloud backup and device transfer |
| Components | Every `Activity`, `Service`, `Receiver` `exported="false"` unless it must receive intents (launcher, deep link, FCM) |

```kotlin
object SecurityConstants {
    val LOCK_AFTER_OPTIONS_MS: List<Long> = listOf(0, 60_000, 300_000, 900_000, 3_600_000)
    const val DEFAULT_LOCK_AFTER_MS: Long = 300_000
    const val PASSCODE_LENGTH: Int = 6
    const val PASSCODE_SALT_BYTES: Int = 16
    const val PASSCODE_PBKDF2_ROUNDS: Int = 210_000
    const val FREE_PASSCODE_ATTEMPTS: Int = 5
    val PASSCODE_LOCKOUT_STEPS_MS: List<Long> = listOf(60_000, 300_000, 900_000, 3_600_000)
    const val SECRETS_KEY_ALIAS: String = "com.dd.sitelog.secrets"
    const val AUDIT_HMAC_KEY_ALIAS: String = "com.dd.sitelog.audit.hmac"
    const val BIOMETRIC_KEY_ALIAS: String = "com.dd.sitelog.biometric"
    const val EXPORT_KEY_LENGTH_BYTES: Int = 32
}
```

**Done**

- Media is not reachable from `adb backup`, cloud backup or a USB file browser
- Background upload continues while the screen is locked
- A hand-edited entry fails at the right index
- No `println(`, no stray `Log.`
- A 20-photo session's logs hold no URL, token or free text

**Tests**

- 100-entry chain verifies; edit at 50 and mid-chain deletion detected at the right index
- AES-GCM bundle round-trip; wrong key throws
- Passcode hash round-trip; wrong passcode rejected
- Lockout steps survive process death (Robolectric secure store)
- Lock after boundary (just under / at the value)
- `KeyPermanentlyInvalidatedException` turns biometrics off
- Redacted bundle equals the expected text

</dd></dl>

</dd></dl>
