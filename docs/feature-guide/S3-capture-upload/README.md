# S3 — Capture and upload

<dl><dd>

- Sprint 3 · 2 weeks · 50h · [all sprints](../README.md)

</dd></dl>

## Contents

<dl><dd>

1. [Overview](#overview)
2. [Tasks](#tasks)
3. [Done](#done)
4. [Self-check](#self-check)

</dd></dl>

## Overview

<dl><dd>

- **Goal:** photos are captured with CameraX and hashed offline; small files reach S3 through a WorkManager upload that survives process death.
- **Demo — offline:** airplane mode → capture → photo linked to a location, kept after relaunch; the Sync tab shows it waiting.
- **Demo — signed in:** network back → the photo uploads (notification shows progress); the server confirms its hash; the capture row reaches the second device.

| F | Feature | Guide | Hours |
|---|---|---|---|
| F05 | Capture | [F05-capture.md](F05-capture.md) | 22 |
| F06 | Upload (single PUT) | [F06-upload.md](F06-upload.md) | 28 |

```mermaid
flowchart LR
    Camera["CameraX"] --> Capture["CaptureRepositoryImpl"]
    Capture --> Disk[("filesDir + SHA-256")]
    Capture --> Queue["UploadStoreImpl (Room)"]
    Queue --> Worker["UploadWorker<br/>WorkManager + OkHttp"]
    Worker -->|"POST /uploads · complete"| BE["Backend"]
    Worker -->|"PUT bytes"| S3[("S3")]
```

</dd></dl>

## Tasks

<dl><dd>

| Task | Step | Layer | Hours | Status |
|---|---|---|---|---|
| F05.1 | Capture DTO | Contract | 1 | not start |
| F05.2 | Record a capture | Domain | 1 | not start |
| F05.3 | Repository + capture data source on a bundled photo | Fake data | 2 | not start |
| F05.4 | Camera view models | View models | 3 | not start |
| F05.5 | CameraX, disk, SHA-256, location (split: camera · files · hash) | Local | 14 | not start |
| F05.6 | Shared backend: captures table | Backend | 0 | not start |
| F05.7 | Captures join sync | Sync | 1 | not start |
| F06.1 | Begin / complete DTOs | Contract | 1 | not start |
| F06.2 | `uploadkit` API and job types | Domain | 1 | not start |
| F06.3 | `UploadStoreImpl` in memory | Fake data | 2 | not start |
| F06.4 | Sync tab view models | View models | 6 | not start |
| F06.5 | Upload store on Room | Local | 4 | not start |
| F06.6 | Shared backend: uploads endpoints | Backend | 0 | not start |
| F06.7 | Worker, foreground info, single PUT, state machine, backoff (split: 3 tasks) | Sync | 14 | not start |

</dd></dl>

## Done

<dl><dd>

- [ ] Offline: capture in airplane mode, kept after relaunch; "waiting" never red
- [ ] Signed in: a photo uploads after `adb shell am kill`; server hash matches
- [ ] F05 and F06 tests written and green

</dd></dl>

## Self-check

<dl><dd>

- Why is the SHA-256 computed at capture time, not at upload time?
- iOS keeps uploading after the app is killed; Android does not. What does WorkManager do instead?
- What should happen when `complete` is called twice?

</dd></dl>
