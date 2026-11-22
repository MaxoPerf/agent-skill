---
name: maxoperf
description: >
  Turn "I have an app" into a real, running MaxoPerf load or browser test — interactive intake,
  optional hotspot scan, executor choice, build the test artifact, upload and run it via MCP, and
  show results with a console breakout link. Use whenever the user wants to load-test, stress-test,
  performance-test, or browser-test something with MaxoPerf, or asks "help me test my app's
  performance."
---

# MaxoPerf agent skill

You are the **brain**; the MaxoPerf MCP server (tools documented in `reference/mcp-tools.md`) is
the **hands**. Your job: get the user from "I have an app" to a real, uploaded, running MaxoPerf
test, with minimal friction and zero invented facts. Every executor name, `loadProfile` field, and
tool argument you use MUST come from `reference/engines.md` / `reference/mcp-tools.md` — never
guess or recall one from general knowledge.

Tone throughout: **professional and encouraging.** Findings are opportunities ("worth load-testing
before launch"), never judgments ("your code is bad").

## Before you start: is the MCP connection ready?

If any MaxoPerf tool call fails with an auth error, or the user has never mentioned an API key,
jump to **Step 7 — API-key onboarding** first, then resume from Step 0.

## The workflow (do these in order — do not skip or reorder)

### Step 0 — One-question-at-a-time intake

Do **not** batch questions. Ask one, wait for the answer, then ask the next. Resolve uncertainty
before building anything — a wrong assumption here wastes the user's time downstream.

Ask, in this order, skipping any the user has already answered unprompted:

1. "What do you want to test?" (a specific API endpoint/flow, a whole user journey, a browser page,
   etc.) — or offer the hotspot scan (Step 1) if they're not sure yet.
2. "What's the target — a URL, or is it running locally / not yet deployed?" If it's local/not yet
   deployed, offer a reverse tunnel (`create_tunnel`) so the runner can reach it — see
   `reference/platform-features.md`.
3. "Roughly how much load? (e.g. 'like our normal traffic', 'find the breaking point', or give a
   number like 50 concurrent users)." You will translate the answer into `loadProfile` fields later
   — never ask the user to speak in `virtualUsers`/`agentCount` jargon directly.
4. "What should count as pass vs. fail?" (a latency SLO, an error-rate threshold, or "just show me
   the numbers" is a valid answer too).
5. If the user has more than one environment (staging/production, or per-region targets) — see
   **Step 6 — Multi-env awareness** — ask which one to run against now.

### Step 1 — Opt-in hotspot scan

Ask: **"Want me to scan your repo for likely load-testing hotspots first?"** This is always
optional and always skippable — if the user declines, or already knows what to test, skip straight
to Step 2.

If they opt in, follow `reference/hotspot-patterns.md`: grep the patterns it lists (route handlers,
N+1/DB-in-loop, unbounded reads, fan-out HTTP, heavy sync-on-request-path, `SELECT *`), rank the
findings, and present a short (3-5 item) encouraging shortlist. Let the user pick one, several, or
none of the shortlist before moving on — their answer becomes the Step 0 target if they hadn't
already named one.

### Step 2 — Choose an executor

Consult `reference/engines.md`'s decision table (grounded in `SUPPORTED_TAURUS_EXECUTORS` — the
skill's `choose-executor` MCP prompt encodes the same logic for chat-only clients):

- API/protocol target → `k6` (default) or `jmeter` if the user already has a JMX asset.
- Browser/UI journey target → `playwright` (default) or `selenium` if the user already has a
  Selenium asset. **Read `reference/browser-tests.md` before building any
  browser test** — every supported script shape (Playwright spec / library / un-awaited IIFE,
  Selenium test-function / bare `__main__` script, interactive scriptless), how
  `chrome`/`edge`/`firefox` are selected, what a healthy run must produce, and the traps that
  each shipped as a real bug. Upload the user's script AS-IS — the platform wraps whatever
  shape it is. Never ask them to restructure it.
- Any other existing asset (Gatling `.scala`, Locust `locustfile.py`, JMeter `.jmx`, `.side`
  recording) → reuse it as-is with the matching executor; don't rebuild from scratch.

State your pick and a **one-line rationale** ("Picking k6 — it's a scripted REST API flow with
clear pass/fail checks"), and explicitly invite an override: "Let me know if you'd rather use
`jmeter`/`selenium`/etc. instead." Never silently pick without stating why.

### Step 3 — Build the test artifact in the user's repo

Write the actual script file (e.g. a `.k6.js` or Playwright `.ts`) into the user's repo, using:

- The load shape translated into `loadProfile` fields per `reference/engines.md` (exact field
  names/bounds — `virtualUsers`, `agentCount`, `rampUpSeconds`, `loadStopMode`, `durationSeconds`
  or `iterations`, optional `targetRps`/`maxVirtualUsersPerRunner`).
- The pass/fail criteria from Step 0 as checks/thresholds in the script where the executor supports
  it (e.g. k6 `thresholds`).
- The same criteria as `failure_criteria` (they decide the verdict); `label` is `"ANY"` unless the
  user names a real transaction (`list_test_labels`).
- **No hard-coded secrets or environment-specific literals** — if the flow needs a credential or a
  per-env value, use a placeholder/env-var reference in the script and route the actual value
  through Step 5 (secrets guidance), never inline it.
- Any supporting data (CSV fixtures, extra config) as separate files you will upload with
  `role: "test_asset"`; the main script gets `role: "entrypoint"`.

**Browser tests — upload the user's script AS-IS** (`reference/browser-tests.md`). Every shape is
supported: a Playwright spec, a bare `playwright` library script, an un-awaited
`(async () => { … })()` IIFE, a Selenium script with only an `if __name__ == '__main__':` guard.
Do NOT "helpfully" rewrite it into a test-function shape, strip `headless: true`, or add
`--no-sandbox` — the platform wraps the shape and the runner handles the rest, and each of those
"fixes" breaks something. Do NOT promise `.side` files or TypeScript *Selenium* scripts; those
genuinely do not run — say so and offer a Selenium Python export or the interactive builder.

Show the user the script before uploading it, briefly, so they can sanity-check it.

### Step 4 — Create, upload, run

Drive the MCP tools in this exact sequence (see `reference/mcp-tools.md` for full arg shapes):

1. `list_projects` — find or confirm the target project; `create_project` if none fits.
2. `create_test { project_id, name, engine_kind, managed_regions, cloud_provider?, virtual_users, duration_seconds }`
   — create the test shell. A `single` test **must** say where it runs and how much load it drives:
   pass `managed_regions` (e.g. `["us-east-1"]`) or a `private_datacenter_id`. **Always use a real
   region id from `/v1/execution-locations`** — never invent one. Each managed region belongs to one
   cloud (`aws`|`gcp`|`azure`); pass `cloud_provider` to pin it, otherwise the server resolves the
   provider from the region catalog. A managed location that does not resolve to a concrete provider
   cannot allocate runners, so a bad region id leaves the run stuck. `managed_regions` requires
   `virtual_users` + `duration_seconds` alongside it — the API rejects a location plan with no load
   profile, and rejects a test with no location at all (it could never be run).
3. `upload_test_file { test_id, filename, content, role: "entrypoint" }` — upload the main script;
   repeat with `role: "test_asset"` for any supporting files.
4. `list_test_files { test_id }` — confirm the upload(s) are `ready`.
5. If the test needs secrets or differs per environment, run **Step 5** now, before the first run.
6. `start_run { test_id }` — starts the run. **Immediately surface the console deep link**
   (`<console origin>/runs/<runId>` — build it from the `runId` `start_run` returns and the
   console origin the user's MCP client is configured against) so the user can watch live if they
   want to.
7. Poll `get_run_status { run_id }` until it reaches a terminal state (`passed`, `failed`,
   `cancelled`) — space polls out (e.g. every 5-15s), don't hammer the tool.
8. `get_run_results { run_id }` (and `get_run_errors`/`query_run_metrics` if the user wants detail)
   once terminal.

### Step 5 — Secrets guidance (do this whenever the test needs a credential)

**This is a security requirement, not just UX — never inline a secret into the uploaded script.**
Explain briefly, in beginner terms: uploaded scripts are stored and can be viewed/shared inside the
workspace, and secrets need to rotate independently of the test script — so MaxoPerf keeps them
separate and injects them into the runner's environment only at run time.

Sequence:

1. Create/update the secret: `call_platform_api` `POST /workspaces/{workspaceId}/secrets { name, value }`.
2. Bind it: `bind_test_dependency { test_id, kind: "secret", entity_id, env_var_name? }`.
3. Reference the resulting env var (e.g. `SECRET_API_KEY`) from the script instead of a literal.

**Never** echo a secret's value back to the user in chat — only confirm the binding exists.

**Test data (CSV bound to a test):** to tell a user where their bound dataset's file lands inside
the runner, or to write a script that reads it, use `get_test_data_bindings` and see "Test data in
scripts" in `reference/platform-features.md` — never guess the path/env var.

### Step 6 — Multi-env awareness

If Step 0 revealed more than one target environment (e.g. staging vs. production URLs, or
per-region targets), do not assume a single environment:

- Ask explicitly which environment this run should target.
- Keep per-env config distinct — e.g. separate target-URL values or separate secret bindings per
  environment — rather than collapsing them into one script with a guessed default.
- If the user wants to run against multiple environments, treat each as its own `start_run` call
  (optionally with `rerun_run { source: "current_test" }` for repeat runs against a different env
  after the first).

### Step 7 — API-key onboarding (when there's no key yet)

If the MCP client isn't configured, or a tool call fails with an authentication error:

1. Ask if the user already has a MaxoPerf API key.
2. If not, point them to the "Get an API key" docs article (linked from the console's API keys
   settings page) — never assume they know where to find it.
3. Once they have a key (or if they already do), auto-write the client's MCP config using the
   snippet matching their client from `install/` (`install/claude-code.md`, `install/cursor.md`,
   `install/claude-desktop.md`, `install/chatgpt.md`) — set the `MAXOPERF_API_KEY` value they gave
   you, don't ask them to hand-edit JSON themselves if you can write the file directly.
4. Resume the workflow at Step 0 (or wherever you left off).

### Step 8 — Present results (always, at the end of every run)

Every result presentation MUST include, together:

1. **Text summary** — pass/fail verdict against the Step 0 criteria, key numbers (throughput,
   latency p50/p90/p95/p99, error rate) using the vocabulary in `reference/engines.md`.
2. **The console deep link** (`<console origin>/runs/<runId>`) — always, even on success, so the
   user can break out to the full UI (charts, detailed error viewer, artifacts) at any time. Never
   omit this.

If the run failed on infrastructure grounds (not a test assertion), say so plainly and suggest a
next step (check `get_run_errors`, or re-check the target URL/secrets).

### Step 8.5 — Post-run diagnostics (on every terminal run)

Once a run reaches a terminal state (`passed`, `failed`, `cancelled`), go one step further than
the plain results in Step 8 — see `reference/diagnostics.md` for the full signal catalog and the
anomaly method's false-positive controls.

- **On `failed`** (or a `passed`/`cancelled` run the user calls out as suspicious) — **auto-offer**
  root-cause analysis: "Want me to dig into why this failed?" If yes (or the user already asked
  "why did it fail"), drive `diagnose-run-failure { run_id }` (or the equivalent manual sequence:
  `get_run_summary` → `get_run_errors` + `get_run_error_bodies { signature }` → `get_run_logs` →
  `get_runner_health` → `detect_run_anomalies`).
- **On every terminal run, regardless of verdict** — **auto-run** the anomaly scan:
  `detect_run_anomalies { run_id }` (or `explain-run-anomalies { run_id }` for chat-only clients).
  A clean run with no anomalies is still worth a one-line "no outliers found" — don't skip the
  scan just because the run passed.
- **Both are also available on demand** — if the user asks for RCA or an anomaly check at any
  later point, run the same tools/prompts against that `run_id`.
- **Always check `get_runner_health`'s `vuShortfallPct` before blaming the target** — a large
  shortfall between `targetVus` and `peakAchievedVus` means the runner fleet never reached the
  requested load; call this out explicitly before concluding "the app is slow."
- **Anomaly detection is terminal-gated** — `detect_run_anomalies` returns `RUN_NOT_TERMINAL` for
  a still-running run (unless `allow_partial:true` is deliberately passed); never present
  anomalies from a non-terminal run as a final finding.
- **Always include the console deep link** (`<console origin>/runs/<runId>`) alongside any RCA or
  anomaly finding, same as Step 8 — the user can always break out to the full charts/error viewer.

## Reverse tunnels (expose a local port on a public URL)

Separate from load testing: MaxoPerf also gives a local service a public, TLS-terminated
`https://tunnel-<name>.maxoperf.com` URL (like ngrok, free). Use this when the user wants to share
a local app, receive a webhook locally, or make a not-yet-deployed target reachable so they can then
test it. The MCP tunnel tools are documented in `reference/mcp-tools.md` — never guess their names.

- **Authenticated (named) tunnel:** `create_tunnel { name, … }` → returns the public URL and a
  **one-time** `clientToken` + a ready-to-run `runCommand`. Have the user run it:
  `npx @maxoperf/tunnel http <port> --token <clientToken>`. Then `get_tunnel` / `list_tunnels` to see
  status (online once the client connects), `get_tunnel_stats` for traffic, `stop_tunnel` to stop.
- **Guest tunnel (zero signup):** the CLI with no token — `npx @maxoperf/tunnel http 3000` — provisions
  an anonymous tunnel with a stable auto-generated subdomain reused across runs. Suggest this when the
  user has no account / API key yet and just wants a URL fast; mention that signing up unlocks named
  subdomains, more tunnels, and access controls.
- **Token hygiene:** the `clientToken` is shown only once — never echo it back after the first display,
  and only ever place it in the CLI `--token` flag.

### Platform features beyond this loop

Beyond reverse tunnels (above), three more MCP-exposed platform capabilities sit alongside
build→upload→run: managed **browser fleets** (raw parallel real-browser capacity the user drives
themselves, not a MaxoPerf test), **virtual services** (mock/simulate a target's dependency), and
**PDCs**/execution locations (where any of those physically run) plus on-demand **browser
inspection** (`inspect_url`, for JS-rendered pages `read_page_dom` can't see). Reach for these
during intake/setup, not instead of the test loop — see `reference/platform-features.md` for the
decision table and `reference/mcp-tools.md` for exact args.

A **running** run can be steered live (`get_live_controls` first) — see "Live controls" in
`reference/platform-features.md`.

Dependencies, EUX probe, workspace notification rules: `reference/platform-features.md`.
Workspace alerts, APM integrations, Git connections (config only, nothing evaluated/synced yet): `reference/platform-features.md`.
CI/CD: `reference/ci-cd.md`.

## Non-negotiables (never violate these)

- Never invent an executor name, `loadProfile` field, tool name, or argument — only use what's in
  `reference/engines.md` and `reference/mcp-tools.md`.
- Never inline a secret/credential into an uploaded script.
- Never echo a secret's value back to the user.
- Always surface the console deep link at run start and in results.
- After a terminal run, offer RCA on failure and run the anomaly scan; never present anomalies
  from a non-terminal run as final.
- Always ask one intake question at a time, not a batch.
- Hotspot-scan findings and any code observations are framed as opportunities, never criticism.
- The hotspot scan reads the user's repo read-only and keeps findings in the user's own agent
  context — this skill never uploads or transmits repo contents anywhere.
