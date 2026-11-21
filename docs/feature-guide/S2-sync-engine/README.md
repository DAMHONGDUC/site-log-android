# S2 — Sync engine

<dl><dd>

- Sprint 2 · 2 weeks · 37h · [all sprints](../README.md)

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

- **Goal:** one sync engine that every later feature plugs into; projects, sessions and locations are the first synced tables.
- **Demo — offline:** edits made offline queue up in the outbox; the Sync tab says how many are waiting; process death keeps them.
- **Demo — signed in:** an Android phone and an iPhone on one account converge; a delete on one stays deleted on the other.
- Lighter sprint on purpose: sync is the riskiest piece.

| F | Feature | Guide | Hours |
|---|---|---|---|
| F04 | Sync engine | [F04-sync-engine.md](F04-sync-engine.md) | 37 |

```mermaid
sequenceDiagram
    participant A as Android phone
    participant BE as Backend
    participant B as iPhone
    A->>A: edit offline → outbox row (same Room transaction)
    A->>BE: POST /sync/push (batch)
    BE-->>A: applied / superseded
    B->>BE: POST /sync/pull since rev
    BE-->>B: changed rows + tombstones
```

</dd></dl>

## Tasks

<dl><dd>

| Task | Step | Layer | Hours | Status |
|---|---|---|---|---|
| F04.1 | Pull and push DTOs | Contract | 2 | not start |
| F04.2 | Sync status types and use cases | Domain | 1 | not start |
| F04.3 | Engine over an in-memory outbox and server (split: queue · push · pull · conflicts · deletes) | Fake data | 18 | not start |
| F04.4 | Sync tab status, account view models | View models | 3 | not start |
| F04.5 | Outbox and cursor in Room; F03 data sources write outbox rows | Local | 6 | not start |
| F04.6 | Shared backend: `/sync/*` on staging | Backend | 0 | not start |
| F04.7 | Remote data source on `HttpClient`, triggers, periodic work, convergence | Sync | 7 | not start |

</dd></dl>

## Done

<dl><dd>

- [ ] Offline: outbox survives process death and reboot
- [ ] Signed in: Android ↔ iOS converge; deletes propagate; a session never moves back a state
- [ ] F04 tests written and green

</dd></dl>

## Self-check

<dl><dd>

- Why is the outbox row written in the same Room transaction as the change?
- WorkManager's shortest periodic interval is 15 minutes. How does the app still sync every 5 minutes in the foreground?
- After S2, what is the smallest change a new feature needs to become synced?

</dd></dl>
