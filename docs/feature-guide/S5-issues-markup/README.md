# S5 — Issues and markup

<dl><dd>

- Sprint 5 · 2 weeks · 45h · [all sprints](../README.md)

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

- **Goal:** issues are logged from the camera, carried into the next session and closed as before/after pairs; photos get a markup layer and a stamped copy with trusted time.
- **Demo — offline:** log an issue by voice (API 31+), re-check it next session, close it with a verification photo; mark it up; share a stamped copy.
- **Demo — signed in:** the issue, its pair and the markup appear on the second device; markup drawn on iOS renders the same on Android.

| F | Feature | Guide | Hours |
|---|---|---|---|
| F08 | Issues and before/after | [F08-issues.md](F08-issues.md) | 24 |
| F09 | Markup and stamps | [F09-markup.md](F09-markup.md) | 21 |

```mermaid
flowchart LR
    Cam["Camera"] --> Sheet["Issue bottom sheet"] --> IssueRepo["IssueRepositoryImpl"]
    Sheet --> Dict["SpeechRecognizer<br/>on device"]
    Recheck["next session"] --> Pair["before/after pair"] --> IssueRepo
    Photo["capture"] --> Markup["Markup editor (Canvas)"] --> AnnRepo["AnnotationRepositoryImpl"]
    AnnRepo --> Stamp["stamped copy<br/>trusted time"]
    IssueRepo --> Sync["outbox → F04"]
    AnnRepo --> Sync
```

</dd></dl>

## Tasks

<dl><dd>

| Task | Step | Layer | Hours | Status |
|---|---|---|---|---|
| F08.1 | Issue, issue capture, phrase DTOs | Contract | 1 | not start |
| F08.2 | Issue entities, pairing, close, carry-over use cases | Domain | 3 | not start |
| F08.3 | Repositories + data sources in memory | Fake data | 3 | not start |
| F08.4 | Issues screens (split: sheet · list · detail · verification) | View models | 7 | not start |
| F08.5 | Room, on-device dictation (split: storage · dictation) | Local | 8 | not start |
| F08.6 | Shared backend: issues tables | Backend | 0 | not start |
| F08.7 | Issues, issue captures, phrases join sync | Sync | 2 | not start |
| F09.1 | Annotation DTO | Contract | 1 | not start |
| F09.2 | Annotation types | Domain | 1 | not start |
| F09.3 | Repository + data sources in memory | Fake data | 1 | not start |
| F09.4 | Markup and stamp screens | View models | 4 | not start |
| F09.5 | Vector rendering (split: layer · stamp) | Local | 7 | not start |
| F09.6 | Shared backend: annotations table | Backend | 0 | not start |
| F09.7 | Annotations sync; trusted time from the server clock | Sync | 7 | not start |

</dd></dl>

## Done

<dl><dd>

- [ ] Offline: issue → re-check → before/after pair; markup saved as a layer; original hash unchanged
- [ ] Signed in: issues and markup converge with iOS; stamp time confidence shown
- [ ] F08 and F09 tests written and green

</dd></dl>

## Self-check

<dl><dd>

- Closing an issue never edits the old one. What gets created instead?
- Why must the `shapes` JSON match the iOS app byte for byte in structure?
- What does dictation do on an Android 10 phone?

</dd></dl>
