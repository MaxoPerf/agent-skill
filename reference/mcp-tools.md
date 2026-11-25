# MCP tool reference

Exact tool/prompt names and arg shapes of `apps/mcp-server`; `SKILL.md` never guesses them. If this
disagrees with the live `tools/list`, trust the live server.

All tools are called over `POST /mcp` (Streamable HTTP), authenticated with
`Authorization: Bearer <mpak_...>`. Reads default to `response_format: "concise"` (pass
`"detailed"` for the full payload). Write tools require a non-read-only session.

## Context

| Tool | Args | Returns |
| --- | --- | --- |
| `whoami` | *(none)* | Account identity + default workspace. |
| `list_workspaces` | *(none)* | Visible workspaces (+ createdAt, owner, counts). |
| `delete_workspace` | `{ workspace_id, confirm_name }` | DESTRUCTIVE cascade (admin); confirm with user first. |
| `set_active_workspace` | `{ workspace_id }` | Sets the active workspace for the rest of the session; validates visibility first. |

## Projects

| Tool | Args | Returns |
| --- | --- | --- |
| `list_projects` | `{ workspace_id? }` | Projects visible in the account, with `canEdit`/`canDelete` flags. |
| `create_project` | `{ workspace_id, name, description? }` | New project (workspace editor). |
| `update_project` | `{ project_id, name?, description? }` | Rename / edit description (null clears). |

## Tests

| Tool | Args | Returns |
| --- | --- | --- |
| `list_tests` | `{ project_id?, workspace_id?, type?: 'single'\|'vario'\|'all', response_format? }` | Test list. |
| `get_test` | `{ test_id, response_format? }` | One test, incl. validation (status/reason/checkedAt/engineKind/`warnings[]`/`detectedBrowsers`), `browsers`, `browsersSource`. |
| `create_test` | `{ project_id, name, type?: 'single'\|'vario', engine_kind, managed_regions?, cloud_provider?: 'aws'\|'gcp'\|'azure', private_datacenter_id?, virtual_users, duration_seconds \| iterations, ramp_up_seconds?, agent_count?, browsers?, failure_criteria?, script_thresholds?, capture_error_bodies? }` | New test shell. **A `single` test needs `virtual_users` + `duration_seconds` (or `iterations`) + `engine_kind` in BOTH location shapes.** Incomplete → `400 TEST_CONFIG_INCOMPLETE` (`details.missing`) / `400 ENGINE_KIND_REQUIRED`; nothing is defaulted: **ask the user, never guess**. `ramp_up_seconds` defaults `0`; a `private_datacenter_id`-only test runs on exactly 1 runner. Managed regions must be real `/v1/execution-locations` ids. Then `upload_test_file` the entrypoint. `browsers?: ('chrome'\|'edge'\|'firefox')[]` splits VUs evenly. `failure_criteria[].label`: `"ANY"` (default) or a transaction label (`list_test_labels`); unknown → `CRITERION_LABEL_UNKNOWN` warning + `no-data`. `capture_error_bodies` (single tests only; VarioTest masters reject it) default true, false disables Errors-tab capture. |
| `set_test_script_thresholds` | `{ test_id, mode }` | See platform-features → Script thresholds. |
| `get_test_overview` | `{ test_id }` | Run-history overview (console Overview tab). |
| `list_test_labels` | `{ test_id }` | Labels a criterion can target (latest run + definition). |

## Test files (upload / list / download)

| Tool | Args | Returns |
| --- | --- | --- |
| `upload_test_file` (write) | `{ test_id, filename, content? \| local_path?, role: 'entrypoint'\|'test_asset', content_type? }` — exactly one of `content` (inline text) or `local_path` (repo-relative, must resolve inside cwd) | `{ fileId, uploadState, sizeBytes }`. Wraps the real 3-step presigned flow (initiate → PUT bytes → complete) in one call — `call_platform_api` cannot do this (the middle hop is an off-base presigned URL). One `entrypoint` per test; supporting files use `test_asset`. Never logs the presigned URL or file bytes. |
| `list_test_files` | `{ test_id }` | `{ files: [{ fileId, filename, fileRole, uploadState, sizeBytes }] }` |
| `download_test_file` | `{ test_id, file_id }` | `{ downloadUrl, expiresInSeconds, filename }` — a short-lived presigned GET URL, not raw bytes. |

## Test data bindings

| Tool | Args | Returns |
| --- | --- | --- |
| `get_test_data_bindings` | `{ test_id }` | `{ bindings: [{ datasetId, distribution, position, slug, relativePath, containerPath, envVar, columnNames, eofMode, scriptVariable, … }], executionContext: { dataDir, dataDirEnvVar, effectiveExecutor, eofCapability, … } }`. Read-only; renaming a binding's file (full-replace `PUT` via `call_platform_api`) and per-engine usage: `reference/platform-features.md` "Test data in scripts". |
| `list_datasets` | `{ workspace_id }` | Datasets (`rowCount`, `isCorrupted`, `linkedTestCount`); read-only. |
| `list_csv_files` | `{ workspace_id }` | Shared CSV files + parse options; read-only. |
| `scan_dataset_repair` | `{ workspace_id, dataset_id }` | Read-only scan: `issues[]` + `canApply`. |
| `apply_dataset_repair` | `{ workspace_id, dataset_id, decisions: [{ line, action: 'escape'\|'exclude'\|'keep' }] }` | Writes a NEW CSV + revision, re-scans; editor only; scan again before a second apply. Shapes: `reference/platform-features.md` "Datasets, shared CSV and CSV repair". |

## Runs

| Tool | Args | Returns |
| --- | --- | --- |
| `start_run` (write) | `{ test_id, engine_kind?, recording_layout?: 'tiled'\|'solo', config_overrides?: {KEY: value} }` | `{ runId, status }`. Idempotent — identical retry reuses the run instead of double-launching. |
| `get_run_status` | `{ run_id, response_format? }` | Lifecycle status: `queued \| allocating \| starting \| running \| stopping \| passed \| failed \| cancelled`. Poll this after `start_run`/`rerun_run`. Failed runs add `failure` + `runnerFailures[]`. Fleet run adds `fleetId`/`fleetName`. CI runs add `ciContext`. |
| `list_runs` | `{ workspace_id?, test_id?, status?, engine?, q?, from?, to?, page?, response_format? }` | Paginated run list; concise by default. |
| `cancel_run` (write, **destructive**) | `{ run_id, mode?: 'graceful'\|'kill', reason? }` | Cancels a run — cannot be undone. **Always confirm with the user before calling.** Not registered at all in read-only sessions. |
| `rerun_run` (write) | `{ run_id, source: 'run_snapshot'\|'current_test' }` | New run derived from an existing one. `run_snapshot` (recommended) reproduces exactly what ran; `current_test` rebuilds from the test as it exists today. Idempotent like `start_run`. Fleet run → 409 `not_applicable_for_fleet_run`. |
| `force_start_run` | `{run_id}` | platform-features.md (Sync) |
| `add_runners` (write) | `{ run_id, additions: [{ location_plan_row_id, count }] }` | Scales up a live run at existing location-plan rows; added runners inherit the run's resolved size. Fleet run → 409 `not_applicable_for_fleet_run`. |

Live controls steer a RUNNING run (`get_live_controls` first; `set_live_*`, `pause_run`/`resume_run`, `get_live_control_history`): args, `scope` and errors are in `reference/platform-features.md`.

## Results

| Tool | Args | Returns |
| --- | --- | --- |
| `get_run_results` | `{ run_id, ...optional time-range/location/label/runner/scenario filters }` | KPI summary cards (throughput, latency, error rate, duration) — the run dashboard Overview tab. **Always pair this with the console deep link.** |
| `query_run_metrics` | `{ run_id, from?, to?, locations?, transactions?: 'all'\|'passed'\|'failed_error_code'\|'failed_assertion', queries: [{ query_id, metric_id, display_name?, aggregation, alignment, filters?, group_by?, rollup_scope? }], response_format? }` (≤12 queries, ≤5 filters/query, ≤3 group-bys/query; server caps each series at 300 groups) | Time series for the requested metric descriptors. Use `get_metrics_catalog` (or `get_run_results`) first to discover metric ids. `transactions` scopes every query in the call to that transaction outcome. |
| `get_run_errors` | `{ run_id, response_format? }` | Error rows grouped by signature + category (`http_5xx\|http_4xx\|network\|assertion\|other`); concise top 25 groups. No rows → adds `failure`. Then `get_run_error_bodies { signature }`. |

Labels: none/COMBINED = all; a list = only those (never COMBINED). VUs: run-wide or per label.

## Platform features (fleets · virtual services · execution locations · on-demand browser)

Beyond the test-run loop, four capabilities have their own tools. Their **when-to-use** guide and
placement order (shared system-catalog PDC → alive user PDC → managed cloud) live in
`reference/platform-features.md` (load on demand); exact args/returns come from the live tool list:

- **Browser fleets** — managed Selenium-Grid / Playwright-`connect()` / CDP capacity the user drives themselves:
  `create_browser_fleet` (`start:false`=idle; `step_capture`/`workspace_id` optional),
  `get_browser_fleet` (+ `stepCapture`, `sessionCount`),
  `get_browser_fleet_quota` (plan limits + usage + remaining + VU-hour cost, call before create/scale),
  `list_browser_fleets` (optional `q`/`lifecycle`/`workspace_id`/`page_size` filters and a
  `summary` — see `reference/platform-features.md`), `list_browser_fleet_sessions`
  (paged, + capture summary), `list_browser_fleet_runs` (incl. 0-session runs),
  `scale_browser_fleet` (403 `limit_live_concurrent_vus`),
  `release_browser_fleet_browsers` (scale down, LIFO),
  `start_browser_fleet`, `stop_browser_fleet` (reversible — keeps
  fleet/endpoints/token; `mode: 'kill'` skips the drain),
  `update_browser_fleet` (edit name/browsers/packing/TTL/idle/`step_capture`; re-provisions only
  on a browsers/packing change),
  `delete_browser_fleet` (**permanent**: runners, endpoints and token die; history/billing kept; prefer `stop_browser_fleet`).
  Every fleet payload carries server-derived `canStart`/`canStop`/`terminal` (never re-derive from `status`; a **failed** fleet is terminal *and* restartable, so keep Start visible).
  `location` is `<aws|gcp|azure>:<region>` or `byoc:<pdcId>`, a **real** id from
  `list_execution_locations` (never invent a region). An unknown or invisible location is rejected
  identically with `400 FLEET_LOCATION_UNKNOWN` (`details.discover` → locations endpoint).
- **Virtual services** — a mock/simulated backend deployed to a PDC with a stable `endpointUrl` (deploy → running → stop):
  `list_virtual_services`, `get_virtual_service`, `create_virtual_service`, `deploy_virtual_service`, `stop_virtual_service`,
  `delete_virtual_service`, `list_virtual_service_catalog`, `list_virtual_service_bindings`.
  `create_virtual_service` **requires** `placement.private_datacenter_id` (else `400 VS_CONFIG_INCOMPLETE`).
- **Stateful mocks** (args + when-to-use: `reference/platform-features.md`): `list_vs_transactions` `get_vs_transaction` `create_vs_transaction` `update_vs_transaction` `delete_vs_transaction` `sandbox_vs_transaction` `update_virtual_service` `get_virtual_service_state` `get_vs_flow_positions` `create_vs_crud_resource` `list_vs_state_callers` `get_vs_state_value` `set_vs_state_value` `delete_vs_state_value` `forget_vs_state_caller` `reset_vs_state_flows` `clear_vs_state` `list_vs_state_backups` `create_vs_state_backup` `restore_vs_state_backup` `delete_vs_state_backup` `export_vs_state` `try_virtual_service` `list_transform_snippets`
- **Execution locations** — `list_execution_locations` lists managed regions + PDCs (`byoc.privateDatacenters`, incl. the shared Sandbox) for fleet/VS placement.
- **On-demand browser** — `inspect_url` spins up/reuses a real browser (same placement order) and returns the JS-rendered DOM/text of a URL — use it over static `fetch_url`/`read_page_dom` for SPAs.
  `browser_run` (write, ≤40 steps) is its multi-step sibling: DRIVES that same fleet browser through
  an ordered `goto`/`click`/`fill`/`waitFor`/`evaluate`/`extract`/`screenshot` step list against one
  page (log in, fill a form, read the result) instead of loading one URL.
  SSRF-guarded on the initial url and every `goto` step; stops after the first failed step (later
  steps come back `skipped`); returns `{ fleetId, location, finalUrl, steps, screenshots, consoleErrors }`.

## Dashboards (metrics-dashboard builder parity)

Parity with the console's run-detail metrics-tab dashboard builder: discover the metric catalog,
query with `query_run_metrics`, then persist a saved dashboard (8 chart types). Rendered on `?tab=metrics`.

| Tool | Args | Returns |
| --- | --- | --- |
| `get_metrics_catalog` | `{ run_id, from?, to?, locations?, labels?, runners?, scenario_id? }` | Condensed metric catalog for the run: `{ metrics: [{ id, label, category, unit, dimensions: [id], aggregations }], limits }`. Call this before `query_run_metrics` or `create_dashboard` to get real metric ids — omits preset recipes and per-dimension value lists to stay small. |
| `list_dashboards` | `{ workspace_id, scope?: 'user'\|'workspace' }` | Saved dashboards visible in the workspace (user-private and/or workspace-scoped), each with its full layout, usage stats, and canEdit/canDelete/canSetDefault permissions. |
| `get_dashboard` | `{ dashboard_id }` | One saved dashboard by id, including its full layout. |
| `create_dashboard` (write) | `{ workspace_id, scope: 'user'\|'workspace', title (≤120 chars), description? (≤500 chars), layout, run_id? }` | `{ dashboardId, revision, consolePath? }` — `consolePath` (`/runs/{run_id}?tab=metrics&dashboardId={id}`) is only present when `run_id` was passed. Workspace scope requires workspace editor access. **Not idempotent** — if a call fails ambiguously (e.g. timeout), `list_dashboards` before retrying rather than re-creating blindly. |
| `update_dashboard` (write) | `{ dashboard_id, title?, description?, layout? }` | `{ dashboardId, revision }` — a partial patch: only the fields you pass change. User dashboards require ownership; workspace dashboards require workspace editor access. |

There is no `delete_dashboard` tool (destructive, excluded the same way `add_runners` is).

Dashboard `layout` shape (`{ version: 1, charts: [{ id, title, chartType, span, order, queries }] }`) and a worked example: `reference/platform-features.md` "Dashboard layout and build workflow". Workflow: `get_metrics_catalog` (real metric ids) → `query_run_metrics` (prove the queries return data) → `create_dashboard` with those same query rows in `layout.charts[].queries` (pass `run_id` to get a `consolePath`, then send the user there) → to change one: `get_dashboard`, edit `layout`, `update_dashboard`. `chartType` is one of `line|area|bar|pie|donut|radar|treemap|table`; `span` is `4|6|12` (optional); each query row is the exact camelCase row `query_run_metrics` took. An invalid `layout` returns a tool error; fix it, don't guess.

## Escape hatch (complete surface)

| Tool | Args | Returns |
| --- | --- | --- |
| `call_platform_api` | `{ method: 'GET'\|'POST'\|'PATCH'\|'PUT'\|'DELETE'\|'OPTIONS'\|'HEAD', path, query?, body? }` | Reaches any long-tail public endpoint not covered above (billing, **secrets** — see below, BYOC, notifications, schedules, data-entities). Admin/internal/key-management paths are always rejected locally before any upstream call. Read-only sessions may use only the safe methods (`GET`/`HEAD`/`OPTIONS`). `TRACE` is not offered. |
| `get_openapi` | `{ section?: 'full'\|'paths_only' }` | The public OpenAPI document (cached ~5 min). Large — pass `paths_only`, or prefer `search_endpoints`. |
| `search_endpoints` | `{ query?, tag?, limit? }` | Keyword/tag search over the OpenAPI document — use before `call_platform_api` to find the right path/method. |

### Discovering the API surface — never fetch the spec through `call_platform_api`

There is **no** `/v1/openapi.json`, `/v1/openapi` or `/v1/api-json` path. Asking for one returns
`Cannot GET /v1/openapi.json` and burns a tool call. The spec has its own tools:

```jsonc
// ✅ Find the right path/method for a task
{ "name": "search_endpoints", "arguments": { "query": "schedule", "limit": 10 } }

// ✅ List every public path, cheaply
{ "name": "get_openapi", "arguments": { "section": "paths_only" } }

// ✅ Read an EXTERNAL spec (the user's own API), not MaxoPerf's
{ "name": "read_openapi", "arguments": { "url": "https://api.example.com/openapi.json" } }

// ❌ Never — no such endpoint exists
{ "name": "call_platform_api", "arguments": { "method": "GET", "path": "/v1/openapi.json" } }
```

`get_openapi`/`search_endpoints` serve MaxoPerf's own spec; `read_openapi` is for a spec the user
points you at.

### Uploading bytes — `call_platform_api` cannot send a file

`call_platform_api` sends **JSON only**. Every "attach a file" endpoint is either multipart or a
presigned 3-step flow, so guessing a JSON body key for one always fails — `property
body/data/file/spec/source/url should not exist`, or `import_file_missing` when the guess is
accepted as an empty body.

```jsonc
// ✅ Test scripts and assets: one call, the tool performs initiate → PUT bytes → complete
{ "name": "upload_test_file", "arguments": {
  "test_id": "tst-0123456789",
  "filename": "load.js",
  "role": "entrypoint",              // exactly one entrypoint per test; supporting files use test_asset
  "content": "import http from 'k6/http';\nexport default () => { http.get(__ENV.TARGET); };"
} }

// ❌ Never — there is no JSON file-upload endpoint to guess at
{ "name": "call_platform_api", "arguments": { "method": "POST", "path": "/v1/tests/tst-0123456789/files",
  "body": { "filename": "load.js", "content": "…" } } }
```

Pass `local_path` instead of `content` only when the client can read a filesystem. Confirm with
`list_test_files` — `uploadState` must read `uploaded`.

### Secrets & bindings via the escape hatch

There is no curated secrets tool — use `call_platform_api` for these, and never inline a secret
value into an uploaded script:

```jsonc
// Create/update a workspace secret (value is never echoed back)
{ "name": "call_platform_api", "arguments": {
  "method": "POST", "path": "/workspaces/{workspaceId}/secrets",
  "body": { "name": "STRIPE_SECRET", "value": "<user-provided, never printed back>" } } }

// Bind (prefer bind_test_dependency): replaces only origin:"manual" rows — GET the list first
{ "name": "call_platform_api", "arguments": {
  "method": "PUT", "path": "/tests/{testId}/secret-bindings",
  "body": { "bindings": [ { "secretId": "sec-0123456789", "envName": "SECRET_STRIPE_KEY" } ] } } }
```

`envName` (optional) must match `^SECRET_[A-Za-z0-9_]{1,120}$`; omit it to inherit the secret's
own name. Never print a secret's value back to the user — only confirm a binding exists.

## Diagnostics & RCA

Post-run root-cause and anomaly tools (TASK-1172) — see `reference/diagnostics.md` for the full
signal catalog, the VU-shortfall check, and the anomaly method's false-positive controls.
`SKILL.md` Step 8.5 drives these after every terminal run.

| Tool | Args | Returns |
| --- | --- | --- |
| `get_run_summary` | `{ run_id, transactions?, response_format? }` | Executive summary: outcome, tripped criteria, `blockedTraffic`. |
| `get_run_error_bodies` | `{ run_id, row_id?, from?, to?, runner?, label?, scenario?, error_key?, signature?, response_code?, category?, limit?, response_format? }` | Sampled error bodies + status codes; `signature`/`response_code` scope to one error group. No rows → `label`+`response_code`, then `label`+`category` (never `label` alone). Full body text only via `row_id`. |
| `get_run_logs` | `{ run_id, severity?: 'error'\|'warn', stream?: 'all'\|'system'\|'engine', from?, to?, limit?, cursor?, response_format? }` | Error-level engine/system log lines for the run. |
| `get_runner_health` | `{ run_id, from?, to?, runners?, response_format? }` | Runner heartbeat/CPU/mem trend + `targetVus`/`peakAchievedVus`/`vuShortfallPct`. |
| `detect_run_anomalies` | `{ run_id, metric_ids?, sensitivity?: 'low'\|'normal'\|'high', allow_partial? }` | Robust MAD outliers on the run's charts; terminal-gated (`RUN_NOT_TERMINAL` otherwise). |

## Reverse tunnels

Expose a local/undeployed target on a public `https://tunnel-<name>.maxoperf.com` URL: `create_tunnel`
(returns the URL + a one-time `clientToken` + a ready `runCommand`), `list_tunnels`, `get_tunnel`,
`stop_tunnel`, `get_tunnel_stats`. The user brings it online with the CLI
(`npx @maxoperf/tunnel http <port> --token <clientToken>`; omit `--token` for a zero-signup guest tunnel). The
token is shown once — never reprint it. Delete/rotate/patch: `call_platform_api` (`/v1/tunnels/*`). Tunnel vs.
virtual service: `reference/platform-features.md`.

## Account & workspace admin (tags · members · billing contact · alerts · APM · Git)

Args: `reference/platform-features.md`. Account: `list_account_tags`, `rename_account_tag`, `delete_account_tag`, `list_account_members`, `set_account_member_role`, `remove_account_member`, `resend_account_invite`, `get_billing_contact`, `set_billing_contact` (all but the `list_*`/`get_*` are writes; confirm first). Workspace (config only, nothing evaluated/synced yet), each `list_*`/`upsert_*`/`delete_*`: `*_alert_rule`, `*_alert_maintenance_window`, `*_apm_integration` (+ `test_apm_integration`), `*_git_connection`.

## Guided prompts (chat-only clients: Claude Desktop, ChatGPT)

Text-only recipes (no network calls) for clients without filesystem access:

| Prompt | Args | Purpose |
| --- | --- | --- |
| `plan-and-build-test` | `{ goal, target? }` | Turns a plain-language goal into the project → test → upload_test_file → run sequence. |
| `choose-executor` | `{ test_kind }` | Recommends an engine/executor with rationale (mirrors `reference/engines.md`'s decision table). |
| `scan-endpoints-for-hotspots` | `{ openapi_or_paste }` | Ranks likely hotspots from an OpenAPI spec or pasted endpoint list. |
| `setup-secrets-and-envs` | `{ test_id }` | Guides secrets/multi-env setup before the first run. |
| `diagnose-run-failure` | `{ run_id }` | Ranked root cause: app error vs failed assertion/SLO vs runner under-provisioning (VU shortfall) vs external dependency. |
| `explain-run-anomalies` | `{ run_id }` | Explains each outlier with a likely cause, correlated to errors/runner health. |

## Response shaping

Read tools default to `response_format: "concise"` (small field sets) and truncate
oversized series; pass `response_format: "detailed"` when the full
payload is genuinely needed (e.g. debugging a specific field).

## AI browser steps

`start_browser_steps_compile` + `get_browser_steps_compile` (poll), `compile_browser_steps`, `*_browser_step_version(s)`: `platform-features.md`.

## Error envelope

Every tool error is `{ isError: true, content: [{ type: "text", text: "<status> <code>: <detail>" }], structuredContent: { status, code, detail } }`.
Common codes: `READ_ONLY_MODE` (write attempted in a read-only session), `FORBIDDEN_PATH`
(`call_platform_api` deny-listed path), `TOO_MANY_QUERIES`/`TOO_MANY_FILTERS`/`TOO_MANY_SERIES`
(`query_run_metrics` limits), `UPLOAD_FAILED`, `PATH_NOT_ALLOWED` (`upload_test_file` local_path
traversal), `INVALID_ARGUMENTS`.

**CI/CD** (`reference/ci-cd.md`): `export_test_as_code {test_id}`, `list_deploy_hooks {workspace_id}`, `delete_deploy_hook {workspace_id, hook_id}`; `list_runs` takes `branch`. Hooks are created/rotated in the console only.
