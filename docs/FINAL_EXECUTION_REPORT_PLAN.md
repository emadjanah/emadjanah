# Final Execution Report — Implementation Plan

This document defines a minimal, production-ready implementation for the ARCHON
**Final Execution Report**: a dark, audit-grade confirmation page that exposes a
LOCKED + LIVE execution state via Server-Sent Events (SSE).

---

## Objective (3-Second Rule)

Any viewer must confirm the execution state within **3 seconds**:

- **ARCHON STATUS:** `LIVE` + `LOCKED` (non-negotiable)
- **Operation Card:** `OP-CIENFUEGOS · Cargo 25,000 MT ±10% · Incoterm · Laycan`
- **Execution Scope:** `Colombia (Primary)` + `Brazil (Backup)`
- **Locked Constraints:**  
  `Refinery COA only · No trader blending · Execution-Only · No upfront fees · Zero tolerance`
- **Timeline:** `T+0 → T+7 → T+10 → T+30`
- **Live Feed:** last 50 log lines with tags (EWS / BANK / RFQ / TIMELINE / HEARTBEAT)
- **Snapshot Export:** one-click JSON for audit

This page is **confirmation**, not monitoring.

---

## Architecture Decision

### Primary (This Repository)
- **Backend:** Express
- **Frontend:** Static HTML (`public/index.html`)
- **Transport:** SSE (`GET /api/stream`)
- **State Model:** In-memory + optional seeded log file
- **Build Step:** None

### Alternative (Optional)
- **Backend:** Next.js App Router SSE (`app/api/stream/route.ts`)
- **Frontend:** React (Next)
- Use only if SSR/React composition is required.

> Do **not** mix both in the same deployment.

---

## Data Contract (SSE Payload)

The `/api/stream` endpoint emits **once per second**:

```json
{
  "opState": {
    "status": "LIVE",
    "lock": "LOCKED",
    "health": "NOMINAL",
    "killSwitch": "ARMED"
  },
  "timeline": [
    { "id": "now", "title": "NOW · Auto-Execution Start", "eta": "T+0", "state": "RUNNING" },
    { "id": "t7",  "title": "T+7 · Shortlist Confirmed", "eta": "T+7", "state": "PENDING" },
    { "id": "t10", "title": "T+10 · Compliance Lock",    "eta": "T+10", "state": "PENDING" },
    { "id": "t30", "title": "T+30 · Contract & Fixing",  "eta": "T+30", "state": "PENDING" }
  ],
  "lastLogs": [
    "[22:16:39] HEARTBEAT: SYSTEM NOMINAL",
    "[22:16:42] BANK CLEARANCE: PRIMED FOR DISBURSEMENT"
  ],
  "ews": { "status": "CLEAN", "timestamp": "2025-12-17T22:16:42Z" },
  "bank": { "status": "PRIMED", "timestamp": "2025-12-17T22:16:42Z" }
}
```

---

## Log Line Standard

All log lines **must** follow this exact format:

```
[HH:MM:SS] <MESSAGE>
```

Examples:

```
[22:16:39] HEARTBEAT: SYSTEM NOMINAL
[22:17:05] SENDING RFQ: REFINERY COA ONLY
[22:17:10] EWS: SIGNAL CLEAN
```

This guarantees deterministic parsing and filtering.

---

## Timeline State Rules

For each step with `target` (seconds since operation start):

* `elapsed < target` → `PENDING`
* `target ≤ elapsed < nextTarget` → `RUNNING`
* `elapsed ≥ nextTarget` → `DONE`

For the final step (`T+30`), use a small window (e.g., +8s) before marking `DONE`.

Timeline **must** be computed from a **global operation start timestamp**, not per-client connection.

---

## Backend Requirements (Express)

* SSE headers:

  * `Content-Type: text/event-stream`
  * `Cache-Control: no-cache`
  * `Connection: keep-alive`
  * `X-Accel-Buffering: no`
* Send an initial comment to flush buffers:

  ```
  : connected
  ```
* Global `OP_STARTED_AT` timestamp
* In-memory ring buffer (max 500 lines)
* Optional seed file: `logs/op-cienfuegos.log`
* Synthetic log generator allowed (demo mode)

---

## Frontend Requirements (Static HTML)

* Dark, high-contrast, audit-grade UI
* No framework dependency
* SSE via `EventSource('/api/stream')`
* Client-side features:

  * Tag-based log filtering
  * Timeline rendering
  * Health / EWS / Bank indicators
  * Snapshot export (JSON)
* **No mutations** to server state

---

## Snapshot Export

The UI must allow exporting the **current payload**:

```json
{
  "capturedAt": "2025-12-17T22:18:10Z",
  "...": "full SSE payload"
}
```

Filename example:

```
archon-final-state-20251217-221810.json
```

Purpose: audit, traceability, evidence.

---

## File Structure (Primary)

```
.
├─ server.js                  # Express + SSE
├─ public/
│  └─ index.html              # Final Execution Report UI
├─ logs/
│  └─ op-cienfuegos.log       # Optional seed logs
├─ package.json
└─ docs/
   └─ FINAL_EXECUTION_REPORT_PLAN.md
```

---

## Non-Goals

* No authentication
* No bidirectional control
* No WebSockets
* No real-time trading logic
* No mutable operations

This page **confirms** execution; it does not manage it.

---

## Audit Footer (Mandatory)

Every render must display:

```
VΩ.Σ.ARCHON
Execution-only · Refinery COA only · Zero tolerance
```

---

## Status

**APPROVED FOR USE**
Final confirmation artifact for OP-CIENFUEGOS execution.

VΩ.Σ.ARCHON

