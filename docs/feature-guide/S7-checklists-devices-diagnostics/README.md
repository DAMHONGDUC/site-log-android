# S7 — Checklists, DeviceLink, diagnostics · internal testing #2

<dl><dd>

- Sprint 7 · 2 weeks · 47h · [all sprints](../README.md)

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

- **Goal:** checklist runs that turn a Fail into an issue; laser readings over Bluetooth, tested with a second Android phone as the meter; integrity checks, cleanup and crash reports; internal testing #2.
- **Demo — offline:** “Finishing handover” in a room, a Fail opens one prefilled issue; phone B (mock meter) sends 3.42 m → it shows on phone A's camera; diagnostics say “All 1,284 photos check out”.
- **Demo — signed in:** checklist runs reach the second device; a test non-fatal shows in Crashlytics.

| F | Feature | Guide | Hours |
|---|---|---|---|
| F12 | Checklists | [F12-checklists.md](F12-checklists.md) | 12 |
| F13 | DeviceLink (BLE meters) | [F13-devicelink.md](F13-devicelink.md) | 18 |
| F14 | Diagnostics · internal testing #2 | [F14-diagnostics.md](F14-diagnostics.md) | 17 |

```mermaid
flowchart LR
    Run["checklist run"] -->|"Fail"| Issue["prefilled issue (F08)"]
    PhoneB["phone B<br/>mock meter, debug"] -->|"BLE notify"| PhoneA["phone A<br/>DeviceLinkManager"]
    PhoneA --> Overlay["reading on the camera"]
    Diag["diagnostics"] --> Check["integrity · storage · bundle · battery"]
    Tag["v* tag"] --> Play["internal testing #2"]
```

</dd></dl>

## Tasks

<dl><dd>

| Task | Step | Layer | Hours | Status |
|---|---|---|---|---|
| F12.1 | Template, item, run, result DTOs | Contract | 1 | not start |
| F12.2 | Checklist types, Fail → issue use case | Domain | 1 | not start |
| F12.3 | Repository + data sources in memory | Fake data | 1 | not start |
| F12.4 | Checklists screens | View models | 4 | not start |
| F12.5 | Built-in templates and results on Room | Local | 4 | not start |
| F12.6 | Shared backend: checklist tables | Backend | 0 | not start |
| F12.7 | Checklist tables join sync | Sync | 1 | not start |
| F13.1 | Mock meter profile (same as iOS) | Contract | 1 | not start |
| F13.2 | DeviceLink types | Domain | 1 | not start |
| F13.3 | Repository + data source on scripted readings | Fake data | 1 | not start |
| F13.4 | Pairing and camera overlay | View models | 3 | not start |
| F13.5 | BLE central (split: manager · profiles · data source) | Local | 8 | not start |
| F13.6 | Mock meter on the second phone | Mock | 3 | not start |
| F13.7 | Feature flag, off for users | Release | 1 | not start |
| F14.1 | Diagnostics snapshot | Domain | 1 | not start |
| F14.2 | Repository + data source with fixed values | Fake data | 1 | not start |
| F14.3 | Diagnostics and cleanup screens | View models | 2 | not start |
| F14.4 | Storage checks, cleanup, bundle, Crashlytics (split: storage · bundle · Crashlytics) | Local | 8 | not start |
| F14.5 | Profiling pass and internal testing #2 | Release | 5 | not start |

</dd></dl>

## Done

<dl><dd>

- [ ] Offline: a Fail creates exactly one issue; phone B's reading shows on phone A; capture never waits on Bluetooth
- [ ] Signed in: checklist runs on both devices
- [ ] Release build: DeviceLink hidden by the flag; profiling budgets met; internal testing #2 installed
- [ ] F12, F13 and F14 tests written and green

</dd></dl>

## Self-check

<dl><dd>

- A template is edited after three runs used it. What must the old runs still show?
- Why must every GATT call wait for the previous callback?
- Cleanup may delete which files, and never which?

</dd></dl>
