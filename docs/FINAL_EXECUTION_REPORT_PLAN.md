# Final Execution Report Implementation Plan

This plan captures the minimal, production-ready setup to render the ARCHON “Final Execution Report” page with a live SSE feed for operation state, timeline, and logs.

## Final State (within 3 seconds)
- **ARCHON STATUS:** `LIVE` + `LOCKED` badge with lock icon.
- **Operation Card:** `OP-CIENFUEGOS / Cargo 25,000 MT ±10% / Incoterm / Laycan`.
- **Execution Scope:** `Colombia (Primary)` + `Brazil (Backup)`.
- **Locked Constraints (non-negotiable):** `Refinery COA only / No trader blending / Execution-Only / No upfront fees / Zero tolerance`.
- **Timeline:** `NOW → T+7 Shortlist → T+10 Compliance Lock → T+30 Contract & Fixing`.
- **Live Feed:** last 50 log lines with filters (`EWS/Bank/RFQ/Heartbeat`).

## Unified Data Model
- **Operation State:** `{ status: LIVE|PAUSED|STOPPED, lock: LOCKED|UNLOCKED, health: NOMINAL|DEGRADED|CRITICAL }`.
- **Timeline Step:** `{ id, title, eta, state: DONE|RUNNING|PENDING|FAILED }`.
- **Log Entry:** `{ ts, level: INFO|WARN|ERROR, tag: EWS|BANK|RFQ|TIMELINE|HEARTBEAT, message }`.

## Backend (SSE)
- Endpoint: `GET /api/stream` emits JSON every second `{ opState, timeline, lastLogs }`.
- Use Node/Express or Next.js API Routes; SSE is sufficient for server → client pushes.
- Logs source: in-memory ring buffer (e.g., 500 lines) or `logs/op-cienfuegos.log` read and tailed.

```js
import express from "express";
import fs from "fs";

const app = express();
const logBuffer = [];

function pushLog(line) {
  logBuffer.push(line);
  if (logBuffer.length > 500) logBuffer.shift();
}

function getLastLogs(n) {
  return logBuffer.slice(-n);
}

// Seed example logs
pushLog("[22:16:39]HEARTBEAT: SYSTEM NOMINAL");
pushLog("[22:17:02]SENDING RFQ: OP-CIENFUEGOS/LAYCAN");
pushLog("[22:17:20]BANK CLEARANCE: PRE-APPROVED");

app.get("/api/stream", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const interval = setInterval(() => {
    const payload = {
      opState: { status: "LIVE", lock: "LOCKED", health: "NOMINAL" },
      timeline: [
        { id: "now", title: "Auto-Execution Start", eta: "NOW", state: "RUNNING" },
        { id: "t7", title: "Shortlist Confirmed", eta: "T+7", state: "PENDING" },
        { id: "t10", title: "Compliance Lock", eta: "T+10", state: "PENDING" },
        { id: "t30", title: "Contract & Fixing", eta: "T+30", state: "PENDING" },
      ],
      lastLogs: getLastLogs(50),
    };

    res.write(`data: ${JSON.stringify(payload)}\n\n`);
  }, 1000);

  req.on("close", () => clearInterval(interval));
});

app.listen(3001, () => console.log("SSE stream on :3001"));
```

## Frontend (Next.js + Tailwind)
- Use `EventSource` to subscribe once and push updates into state.
- Parse incoming log lines on the client to derive `tag`:
  - Contains `EWS` → `EWS`
  - Contains `BANK CLEARANCE` → `BANK`
  - Contains `SENDING RFQ` → `RFQ`
  - Contains `SYNCING TIMELINE` → `TIMELINE`
  - Contains `HEARTBEAT` → `HEARTBEAT`

```tsx
import { useEffect, useState } from "react";

type OpState = { status: string; lock: string; health: string };
type TimelineStep = { id: string; title: string; eta?: string; state: string };
type LogEntry = { raw: string; tag: string };

type StreamPayload = { opState: OpState; timeline: TimelineStep[]; lastLogs: string[] };

function parseTag(line: string): string {
  if (line.includes("EWS")) return "EWS";
  if (line.includes("BANK CLEARANCE")) return "BANK";
  if (line.includes("SENDING RFQ")) return "RFQ";
  if (line.includes("SYNCING TIMELINE")) return "TIMELINE";
  if (line.includes("HEARTBEAT")) return "HEARTBEAT";
  return "GENERAL";
}

export default function ExecutionReport() {
  const [data, setData] = useState<StreamPayload | null>(null);

  useEffect(() => {
    const es = new EventSource("/api/stream");
    es.onmessage = (e) => setData(JSON.parse(e.data));
    es.onerror = () => es.close();
    return () => es.close();
  }, []);

  if (!data) return <div className="text-slate-300">Connecting…</div>;

  const logs: LogEntry[] = data.lastLogs.map((raw) => ({ raw, tag: parseTag(raw) }));

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 p-6 space-y-4">
      <header className="flex items-start justify-between">
        <div>
          <p className="text-sm text-amber-200">OP-CIENFUEGOS</p>
          <h1 className="text-2xl font-semibold">Final Execution Report</h1>
          <p className="text-xs text-slate-400">Cargo 25,000 MT ±10% • Incoterm • Laycan</p>
        </div>
        <div className="text-right space-y-1">
          <div className="inline-flex items-center gap-2 rounded-full bg-emerald-500/10 px-3 py-1 text-sm text-emerald-200">
            <span className="h-2 w-2 rounded-full bg-emerald-400 animate-pulse" />
            ARCHON STATUS: LIVE
            <span className="ml-2 inline-flex items-center gap-1 rounded-full bg-emerald-600/20 px-2 py-0.5 text-xs">LOCKED 🔒</span>
          </div>
          <p className="text-xs text-slate-400">Health: {data.opState.health}</p>
        </div>
      </header>

      <section className="grid gap-4 lg:grid-cols-3">
        <div className="rounded-xl border border-slate-800 bg-slate-900/60 p-4">
          <h2 className="text-sm font-semibold text-slate-200">SYSTEM.LOG</h2>
          <div className="mt-3 max-h-[520px] space-y-2 overflow-y-auto text-xs">
            {logs.slice(-50).map((log, idx) => (
              <div key={idx} className="flex items-start gap-2">
                <span className="rounded-full bg-slate-800 px-2 py-0.5 text-[10px] text-slate-300">{log.tag}</span>
                <span className="text-slate-100">{log.raw}</span>
              </div>
            ))}
          </div>
        </div>

        <div className="rounded-xl border border-slate-800 bg-slate-900/60 p-4">
          <h2 className="text-sm font-semibold text-slate-200">EXECUTION TIMELINE</h2>
          <ol className="mt-4 space-y-3 text-sm">
            {data.timeline.map((step) => (
              <li key={step.id} className="flex items-center gap-3">
                <span className="h-3 w-3 rounded-full bg-emerald-400" />
                <div>
                  <p className="font-medium text-slate-100">{step.title}</p>
                  <p className="text-xs text-slate-400">{step.eta ?? ""} • {step.state}</p>
                </div>
              </li>
            ))}
          </ol>
        </div>

        <div className="space-y-4">
          <div className="rounded-xl border border-slate-800 bg-slate-900/60 p-4">
            <h2 className="text-sm font-semibold text-slate-200">EXECUTION SCOPE</h2>
            <p className="mt-3 text-sm text-slate-100">Colombia (Primary)</p>
            <p className="text-sm text-slate-300">Brazil (Backup)</p>
          </div>
          <div className="rounded-xl border border-slate-800 bg-slate-900/60 p-4">
            <h2 className="text-sm font-semibold text-slate-200">LOCKED CONSTRAINTS</h2>
            <ul className="mt-3 space-y-2 text-sm text-slate-100">
              <li>Refinery COA only</li>
              <li>No trader blending</li>
              <li>Execution-Only</li>
              <li>No upfront fees</li>
              <li>Zero tolerance</li>
            </ul>
          </div>
        </div>
      </section>
    </div>
  );
}
```

## Snapshot Export
- Add a button that downloads `JSON.stringify({ opState, timeline, lastLogs })` as `execution-snapshot.json` for audit.

## Optional Indicators
- **Kill-Switch:** badge showing `ARMED` + conditions (placeholder logic acceptable).
- **EWS Panel:** latest scan status `CLEAN/WARNING/TRIGGERED` with timestamp.
- **Bank Pre-Clearance:** latest status + timestamp.
- **Audit Footer:** include `VΩ.Σ.ARCHON` + hash/session id for traceability.

## Folder/Route Suggestions
- `frontend/` for Next.js app (or use repo root if starting fresh).
- `frontend/pages/api/stream.ts` for SSE route when using Next.js.
- `frontend/app/execution-report/page.tsx` for the UI.
- `logs/op-cienfuegos.log` optional if using file tailing.
- `scripts/seed-logs.js` (optional) to push synthetic lines into the ring buffer.
