# Diagnostics: root-cause analysis & anomaly detection

Post-run diagnostics for the MaxoPerf agent skill. `SKILL.md` Step 8.5 drives these after **every
terminal run** (`passed` / `failed` / `cancelled`). Tool names + args are in
[`mcp-tools.md`](./mcp-tools.md) → *Diagnostics & RCA*.

## When to run

- **Anomaly scan — every terminal run**, regardless of verdict. A clean run still gets a one-line
  "no outliers found." Never present anomalies from a **non-terminal** run as a final finding —
  `detect_run_anomalies` returns `RUN_NOT_TERMINAL` for a running run unless you deliberately pass
  `allow_partial: true` for a live peek.
- **Root-cause analysis — auto-offer on `failed`** (or a `passed`/`cancelled` run the user flags as
  suspicious), and on demand any time the user asks "why did it fail?"
- Always include the console deep link (`<console origin>/runs/<runId>`) with any finding — the user
  can break out to the full charts / error viewer.

## Read the run's own failure first

`get_run_status` returns `failure { cause, title, summary, detail }` + `runnerFailures[]`; quote
`detail` (the engine's own words). Cause ids: `reference/browser-tests.md`.

## The signal catalog (what RCA correlates)

| Signal | Tool | What it tells you |
| --- | --- | --- |
| **Which failure criteria tripped** | `get_run_summary` | The first-order "why failed" — the SLO/assertion/threshold that flipped the run red. Start here. |
| **Actual errors — codes + bodies** | `get_run_errors` → `get_run_error_bodies { signature }` | Status/response codes always; the real request/response **body text** only via the `row_id` detail route (concise = a ≤200-char snippet; `response_format:"detailed"` for the full redacted body). Quote the real error, don't just count it. |
| **Error logs** | `get_run_logs` | Error/warn engine + system log lines (`severity`, `stream` filters) — engine-side failures, connection resets, script exceptions. |
| **Runner health & VU shortfall** | `get_runner_health` | Heartbeat + CPU/mem/process trend, and the headline diagnostic **`targetVus` vs `peakAchievedVus` → `vuShortfallPct`**. |
| **Chart outliers** | `detect_run_anomalies` | Sustained statistical outliers on latency/throughput/error-rate/concurrency series. |

## The VU-shortfall check (do this before blaming the target)

A large `vuShortfallPct` (requested `targetVus` far above `peakAchievedVus`) means **the runner
fleet never generated the requested load** — the run under-delivered, so high latency or a "slow"
verdict may be an artifact of insufficient load generation, not the target application. Always
state this explicitly before concluding "the app is slow": e.g. *"Note: the fleet only reached
620 of 1000 target VUs (38% shortfall), so these latencies are under partial load."*

## Ranking a root cause

From the signals, `diagnose-run-failure` produces a **ranked** cause, distinguishing:

1. **Application errors** — real 5xx/4xx from the target (error bodies + codes + logs agree).
2. **Failed assertion / SLO** — the app responded fine but a threshold (p95, error-rate) tripped
   the failure criteria (`get_run_summary`).
3. **Runner under-provisioning** — high `vuShortfallPct`, runner CPU/mem saturation; the load side,
   not the target.
4. **External dependency** — errors concentrated on calls to a third-party host / specific label.

Correlate in time: an anomaly window that lines up with an error spike and healthy runners points
at the target; one that lines up with runner saturation points at the load side.

## Anomaly method & false-positive controls

`detect_run_anomalies` uses a deterministic robust-statistics pass (`apps/mcp-server/src/anomaly.ts`),
chosen over mean/stdev because a handful of real spikes would inflate a stdev-based bound and hide
them. Per metric series (post-ramp-up points):

```
robustZ = |x − median| / (1.4826 · MAD)          # MAD = median absolute deviation
flag x when robustZ > threshold(sensitivity)      # low 4.5 · normal 3.5 · high 2.5
```

False-positive controls — each guards a real noise source:

- **Ramp-up exclusion** — points inside the run's `rampUpSeconds` warm-up are dropped *before*
  analysis; latency rising while VUs ramp is expected, not an anomaly.
- **Minimum sample size** — fewer than **20** post-ramp points → `insufficient_data` (never a guess).
- **Flat-series guard** — `MAD == 0` → `flat_series`, no anomaly (and no divide-by-zero).
- **Sustained-window gate** — a single outlier point ("blip") is ignored; an anomaly is emitted only
  when **≥3 points are flagged within a 5-point window**, and contiguous qualifying windows merge
  into one event. This is the main defense against crying wolf on transient noise.
- **Terminal-gate** — analysis runs only once the run ended (`RUN_NOT_TERMINAL` otherwise), because
  a partial series has no stable baseline.

Each emitted anomaly carries `{ metric_id, window {from,to}, severity, baseline (median),
observed (peak/trough), robustZ }`. The tool never editorializes — the skill explains the outlier
and correlates it with the other signals.
