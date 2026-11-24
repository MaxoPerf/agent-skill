# Platform features beyond the test-run loop

The core workflow in `SKILL.md` is: build a test → upload it → run it → read results. Four more
platform capabilities exist alongside that loop. This page is the conceptual map — when each one
is the right reach, not an API dump (see `reference/mcp-tools.md` for exact args/returns).

## The four things, one line each

- **Reverse tunnel** — makes a target *reachable* (local/undeployed → public URL).
- **Browser fleet** — gives the user raw parallel real-browser capacity to drive *themselves*.
- **Virtual service** — makes a *dependency* of the target predictable (mock/simulate a backend).
- **PDC (execution location)** — decides *where* any of the above physically runs.

## Decision table

| Situation | Reach for | Tool names |
| --- | --- | --- |
| The target is running locally or not deployed yet | A reverse tunnel — exposes it at `tunnel-<name>.maxoperf.com` so a MaxoPerf test can hit it | `create_tunnel`, `list_tunnels`, `get_tunnel`, `stop_tunnel`, `get_tunnel_stats` |
| The user wants parallel real-browser capacity to drive with their **own** script/CI/exploratory automation — not a MaxoPerf-orchestrated test run | A browser fleet — managed Selenium-Grid/Playwright-`connect()`/CDP endpoint | `create_browser_fleet`, `get_browser_fleet`, `list_browser_fleets`, `list_browser_fleet_sessions`, `list_browser_fleet_runs`, `scale_browser_fleet`, `release_browser_fleet_browsers`, `start_browser_fleet`, `stop_browser_fleet`, `update_browser_fleet`, `delete_browser_fleet` |
| The test's target depends on something you don't want to hit for real (a third-party API, a flaky/rate-limited/costly downstream) | A virtual service — deploy a mock/simulated backend, point the test at its `endpointUrl` | `list_virtual_services`, `get_virtual_service`, `create_virtual_service`, `deploy_virtual_service`, `stop_virtual_service`, `delete_virtual_service`, `list_virtual_service_catalog`, `list_virtual_service_bindings` |
| The mock needs to remember things across requests — "give each virtual user its own cart", a login-then-act flow, a full fake CRUD API, a counted flow, backing up/restoring a demo | Stateful mocks on that virtual service — variables, captures, transforms, cases, counters, the one-click CRUD generator, backups, and Try it | `create_vs_transaction`/`update_vs_transaction` with `state`/`cases`/`then_ops`, `create_vs_crud_resource`, `get_virtual_service_state`, `list_vs_state_callers`, `get_vs_state_value`/`set_vs_state_value`/`delete_vs_state_value`, `forget_vs_state_caller`, `reset_vs_state_flows`, `clear_vs_state`, `list_vs_state_backups`/`create_vs_state_backup`/`restore_vs_state_backup`/`delete_vs_state_backup`, `export_vs_state`, `try_virtual_service`, `list_transform_snippets` |
| A fleet or virtual service needs a placement — which region/PDC it lands on | List execution locations first, then pass the chosen id | `list_execution_locations` |
| A page is a SPA / client-rendered, and `read_page_dom`/`fetch_url` come back empty or unhelpful (static HTML only, no JS) | On-demand browser inspection — real JS-rendered DOM | `inspect_url` |
| A run is running and is too hot or too cold right now ("double the load", "cap it at 200 RPS", "pause it"), or the user asks what was changed mid-run | Live controls — steer the running run without restarting it | `get_live_controls` first, then `set_live_virtual_users`, `set_live_throughput`, `set_live_properties`, `set_live_config_files`, `pause_run`, `resume_run`; `get_live_control_history` to confirm |
| The task needs to actually DO something in a browser — log in, click through a flow, fill a form, then read the result or grab a screenshot — not just fetch one page | Driving a fleet browser through an ordered step list | `browser_run` |
| The user wants to see/manage everything a test depends on in one place, pick a tunnel/VS/fleet/secret inline instead of a literal URL, add a browser UX check to a load test, or get notified on ecosystem events (dependency failure, tunnel disconnect, VS/fleet failure) | Dependencies (+ EUX probe, workspace notification rules) | `list_test_dependencies`, `bind_test_dependency`, `unbind_test_dependency`, `get_run_dependencies`, `get_dependency_usage`, `set_eux_probe`, `list_workspace_notification_rules`, `upsert_workspace_notification_rule`, `delete_workspace_notification_rule` |
| The user has a non-secret value many tests share (base URL, tenant id, public cert, feature toggle) | Config values | `list_config_values`, `create_config_value`, `update_config_value`, `delete_config_value`, `get_config_value_usage`, `bind_test_dependency { kind: 'config' }` |

Note what's absent from this table: none of these four replace the build → upload → run loop in
`SKILL.md`. A tunnel/fleet/virtual-service/PDC choice is something you resolve *during* intake and
setup (Step 0-3), before `create_test`/`start_run`.

## Workspaces & projects (owner, counts, delete)

`list_workspaces` rows carry `createdAt`, `owner` (`{ userId, name }`, or `null` when the creator was
never recorded) and `counts` (`projects`, `tests`, `activeRuns`); `list_projects` rows add
`description`, `createdAt`, `owner` and `counts` (`tests`, `activeRuns`). Use `update_project
{ project_id, name?, description? }` to rename or describe a project (`description: null` clears it).
`delete_workspace { workspace_id, confirm_name }` is **destructive**: it soft-deletes the workspace and
cascades to its tests, projects, datasets, dashboards and data entities. It needs workspace admin and is
refused (409) for the account's last workspace or while it still has active runs, running browser
fleets, live virtual services or private datacenters — stop those first. Always show the user what is
inside (counts from `list_workspaces` / `GET /v1/workspaces/:id`) and get an explicit yes; `confirm_name`
must equal the workspace name exactly. It is not available to Loadrigo — send the user to the console.

## Reverse tunnels

**What it is.** A public TLS URL (`https://tunnel-<name>.maxoperf.com`) that forwards to a port on
the user's machine once their client connects (`npx @maxoperf/tunnel http <port> --token
<clientToken>`). The tunnel and URL persist; `stop_tunnel` disconnects the client without deleting
either.

**When to reach for it.** As soon as the target turns out to be "running locally" or "not deployed
yet" — offer a tunnel immediately rather than asking the user to deploy something just to test it,
and don't try to test `localhost` directly (the runner is remote and can't reach it).

**The one-time token.** `create_tunnel` shows the client token exactly once. Tell the user to
store it; it is not retrievable again (only rotatable, via `call_platform_api` — not a curated
tool).

## Proxy recorder & traffic recordings

**What it is.** A proxy recorder is a MaxoPerf-hosted HTTP(S) proxy that records the traffic of a
browser or phone pointed at it. Exposure is either **gateway** (the shared MaxoPerf proxy gateway) or
**direct** (a proxy on the user's own private datacenter); `get_proxy_recorder_capabilities` says which
PDCs support which. The capture (a **traffic recording**) is sealed when the recorder stops and can be
converted into an HTTP test (with correlation suggestions) or a Virtual Service library entry.

**Recipe.** `get_proxy_recorder_capabilities` → `create_proxy_recorder` → `start_proxy_recorder` →
`get_proxy_recorder` until `ready` → tell the user to open the console recorder page to copy the proxy
password (or pair a device) and install the CA on the device → `query_traffic_flows` with `tail` to
confirm capture → `stop_proxy_recorder` → `convert_traffic_recording` preview → review `suggestions` and
`warnings` with the user → persist (`preview:false`, once) → `start_run`. A persisted Virtual Service is a
library entry: deploy it with the virtual-service tools.

**Tools.** Thin wrappers over `/v1/proxy-recorders*` and `/v1/traffic-recordings*`; everything returned is redacted.

| Tool | Args | Returns |
| --- | --- | --- |
| `get_proxy_recorder_capabilities` | `{}` | `{ gateway:{available,host,port}, defaults, limits, pdcs:[{id,name,visibility,gateway,direct}] }` — the only way to learn which PDCs can host a recorder and with which exposure. Call it first. |
| `list_proxy_recorders` | `{ status?, q?, page?, page_size? }` | `{ items, total, page, pageSize }` (name, status, exposure, PDC, `currentRecordingId`, `canStart`, `canStop`). |
| `get_proxy_recorder` | `{ recorder_id }` | The recorder incl. `connect{scheme,host,port,username}` while `ready`. **The proxy password is never returned.** |
| `create_proxy_recorder` (write) | `{ name, private_datacenter_id, exposure?, capture_mode?, host_include?, host_exclude?, redact_rules?, max_body_bytes?, idle_timeout_minutes?, max_duration_minutes?, allowed_source_cidrs?, workspace_id? }` | A stopped recorder + `next: start_proxy_recorder`. Pick the PDC and exposure from `get_proxy_recorder_capabilities` (`gateway:false` and `direct:false` gives `422 EXPOSURE_UNSUPPORTED`). Not idempotent. |
| `start_proxy_recorder` (write) | `{ recorder_id }` | `{ recorder, recording:{id,status} }`. Poll `get_proxy_recorder` until `ready` (typically under 60 s). `403 PROXY_RECORDER_PLAN_LIMIT`, `409 PROXY_RECORDER_NOT_STARTABLE`. |
| `stop_proxy_recorder` (write) | `{ recorder_id }` | `{ recorder }` (status `stopping`); the recording is sealed. |
| `list_traffic_recordings` | `{ recorder_id?, status?, page?, page_size? }` | `{ items, total, page, pageSize }` (status, `flowCount`, `bytesIn`/`bytesOut`, `startedAt`, `sealedAt`, `stopReason`). |
| `get_traffic_recording` | `{ recording_id, include_analytics? }` | The recording (+ `analytics`: totals incl. `hosts`, status mix, latency percentiles, per-host, attention hints). |
| `query_traffic_flows` | `{ recording_id, tail?, after_seq?, before_seq?, limit?, host?, method?, status?, q?, flow_id?, max_body_chars? }` | List: `{ items, nextSeq, recordingStatus }` (default `limit` 20, max 50; `tail=N` = the last N flows; `after_seq=nextSeq` polls live). With `flow_id`: one redacted flow, bodies cut to `max_body_chars` (default 2000, max 8000) + `truncated`. |
| `convert_traffic_recording` | `{ recording_id, target:'http_test'\|'virtual_service', preview?=true, filter?, headers?, correlations_accept?, test?, service?, flow_ids?, response_format? }` | **Preview (default, side-effect free):** `http_test` → `{ requestCount, skipped, suggestions, warnings, truncatedAt50 }` (full `model` with `response_format:'detailed'`); `virtual_service` → `{ service, transactionCount, transactions, skipped, warnings }`. **Persist (`preview:false`, write):** `{ testId, requestCount, skipped, warnings }` or `{ createdServiceId, transactionCount, skipped, warnings }`. `test:{name,projectId,kind:'performance'\|'functional-api',loadProfile?,locationPlan?}` — persisting a `performance` test **requires** `test.loadProfile:{virtualUsers, loadStopMode, durationSeconds\|iterations, rampUpSeconds?}` (ask the user; nothing is invented; there is no `agentCount` — runner count is `locationPlan.rows[].runnerCount`); `functional-api` sends none. `service:{name}` for virtual_service. `warnings` are structured `[{code, flowId?, header?, count?}]`. Correlation suggestions are `body_jsonpath` only (a value in a JSON response body reused later); apply by id with `correlations_accept`. |

Reachable through `call_platform_api` (not curated): HAR download `GET /v1/traffic-recordings/{id}/har`
(always redacted), `POST /v1/proxy-recorders/{id}/pause` and `/resume`, `PATCH`/`DELETE` of recordings
and recorders. Deploy a persisted Virtual Service library entry with the virtual-service tools.

**Safety.**
- Never ask for or print the proxy password. The password, `rotate-secret`, pairing-code creation and
  `reveal=true` are console-only (workspace admin) and blocked for agents; send the user to the console
  recorder page.
- Captured data is redacted (auth headers, cookies, tokens); a converted test never contains the original
  secret values — bind real ones from workspace secrets.
- Only record traffic the user is authorised to capture. Apps with certificate pinning cannot be
  intercepted.

**When to use it vs. the rest.** A tunnel exposes a local service to the internet; a proxy recorder
captures what a client sends to someone else's service. Use a recorder to build a realistic HTTP test from
real usage or to mock a dependency (Virtual Service) from real responses; use the browser tools when the
user wants a scripted browser test instead.

## Browser fleets

**What it is.** N `chrome`/`firefox`/`edge` browser sessions, spread across one or more locations,
behind a single endpoint the user's own client drives directly — via Playwright `connect()`,
WebDriver/Selenium Grid, or raw CDP. Billed as browser VUs (1 browser = 1 VU per location).

**When to reach for it vs. a scripted browser test.** A browser fleet is **not** how you run a
MaxoPerf test — it's raw capacity for when the user already has their own test runner/CI/tooling
and just wants managed real-browser infrastructure to point it at. If the user wants MaxoPerf to
run and grade a browser journey (pass/fail, KPIs, video/HAR/steps), that's a Playwright/Selenium
test via the normal build → upload → run flow (`reference/browser-tests.md`), not a fleet.

**The location grammar.** Every browser/location slice uses `<provider>:<region>` (`aws:us-east-1`,
`gcp:europe-west1`, `azure:eastus`) for managed cloud, or `byoc:<pdcId>` to land on a specific PDC
— see `list_execution_locations` below for valid ids.

**The token.** Shown inline only in `create_browser_fleet`'s response; re-fetch it later with
`get_browser_fleet` (omitted once the fleet is terminal).

**Lifecycle.** `create_browser_fleet` can create the fleet idle (`start: false` — no runners, no
quota charged) and provision it later with `start_browser_fleet`. `stop_browser_fleet` releases a
running fleet's runners but keeps the fleet/endpoints/token so it can be `start_browser_fleet`'d
again — unlike the permanent `delete_browser_fleet`. `update_browser_fleet` edits
name/browsers/packing/TTL/idle timeout/`step_capture`; a `browsers`/`browsers_per_runner` change on
a running fleet re-provisions a fresh run, but `step_capture` alone never does (it takes effect on
the fleet's next Start). `scale_browser_fleet` adds browser/location slices to a live fleet;
`release_browser_fleet_browsers` is its inverse — scale a live fleet DOWN by releasing whole runner
VMs LIFO for named slices.

**Finding fleets.** `list_browser_fleets` takes optional `q` (name search, max 100 chars) and
`lifecycle` (`created`/`running`/`stopped`/`failed`) filters, pages up to `page_size: 200`, and
always returns a `summary` (`{ total, running, stopped, activeSessions }`) unaffected by those
filters — account-wide, or scoped to one workspace when `workspace_id` is sent.
`list_browser_fleet_sessions` (paged, default 25; optional ISO `started_from`/`started_to` keep sessions that
started in a window, and then `total` is omitted) lists every session across ALL of the fleet's
runs with its capture summary; `list_browser_fleet_runs` lists every run the fleet has ever had
(including 0-session runs) for jumping into a specific run's KPIs/video.

**Sizing a fleet before you create it.** `get_browser_fleet_quota` (read-only) returns the plan's
`limits` (concurrent browsers/runners, max lifetime, browsers-per-runner range), what the account is
`usage`-ing now, the `remaining` headroom and the `cost` model: 1 browser = 1 VU, accruing
`vuHoursPerBrowserHour` VU-hours per hour of runtime, drawn from `remainingVuHours`
(`overageCentsPerVuHour` applies past the allowance; `hardCap` plans refuse instead). `null` means
unlimited or unavailable, never 0. Estimate N browsers for H hours as N × H VU-hours and check it
against `remaining.browsers` and `cost.remainingVuHours` before `create_browser_fleet` /
`scale_browser_fleet`.

## Virtual services

**What it is.** A mock/simulated backend, deployed as its own pod on a chosen outpost/PDC, with a
stable `endpointUrl`. It answers according to configured transactions/responses (`noMatch` policy
governs anything unmatched: `passthrough`/`respond`/`reject`/`notFound`). It has an explicit
deploy → running → stop lifecycle — it isn't always-on.

**Virtual-service catalog import** (`POST /v1/virtual-services/services/{serviceId}/import`) is
`multipart/form-data` (`file` + `format`), so it is **not reachable through `call_platform_api`
either**. Either build the mock from a recorded run — `POST /v1/virtual-services/import/from-run/{runId}`,
a normal JSON body, so the escape hatch CAN do it — or hand the import to the user in the console.

**When to reach for it.** The test's real target depends on something you don't want to exercise
for real during a load/browser test — a third-party payment API, a rate-limited partner service,
a downstream that's flaky or expensive to hit at volume. Deploy a virtual service standing in for
that dependency, then point the test's config/secrets at its `endpointUrl` instead of the real one.

**Placement is required at create.** Every virtual service runs as its own pod on a selected PDC,
so `create_virtual_service` **must** carry `placement.private_datacenter_id` — pick one from
`list_execution_locations`. Omitting it, or naming a PDC the account cannot use (the two answer
identically, on purpose, so the error never reveals whether someone else's datacenter exists), is
rejected with `400 VS_CONFIG_INCOMPLETE` and creates nothing. Virtual services created before this
rule may still have no PDC; they load fine and fail at `deploy_virtual_service` instead. Placement
can only be changed while stopped.

## API Library (vendor-API replicas)

**What it is.** Ready-made, editable replicas of well-known vendor APIs (payments, messaging,
identity, …). `install_vs_library_pack` **copies** a pack into a NEW virtual service — groups,
transactions, state config, hostname — so the copy never changes when the pack is updated (install
the newer version side by side; `get_virtual_service` shows `library.updateAvailable`).

**When to reach for it.** The test's target talks to a vendor API (Stripe, Twilio, GitHub, …) you
should not hit for real: `list_vs_library_packs` → `get_vs_library_pack` (operations, stateful
resources, webhook events, auth example, `unverifiableOperations`, disclaimer) →
`install_vs_library_pack` (needs `workspace_id`, `pack_id`, `name`, `private_datacenter_id` from
`list_execution_locations`; a missing/unusable PDC is `400 VS_CONFIG_INCOMPLETE`) → `deploy_virtual_service`
→ point the app at the `endpointUrl` with the pack's example credential.

**Webhooks.** Replicas can send signed outbound webhooks. Set `webhook_url` (+ `signing_secret_id`, a
workspace secret id) at install, or `webhook_target` via `update_virtual_service` (null clears).
`list_vs_webhook_deliveries` shows outcomes (`delivered`/`failed`/`blocked` = SSRF guard/`dropped` = queue
full), failures first. Needs a runtime on config >= 3: `get_virtual_service.runtimeSupportsLibrary` false
means redeploy. Custom transactions take `response.webhooks` (<= 5, snake_case in MCP) and template
captures take `as: "json"`; invalid specs are `400 VS_WEBHOOK_INVALID` naming the path.
The `hmac` signer also has a template mode for vendors that sign `timestamp + body` into their own headers: `message` (must contain `{body}`; also `{timestamp}`, `{id}`, `{url}`), `headers` (1-4, values use `{signature}`/`{timestamp}`/`{id}`), `timestamp` (`s`|`ms`|`iso`), `secret_encoding` (`utf8`|`hex`); needs runtime config >= 4, else `409 VS_RUNTIME_UPGRADE_REQUIRED` (redeploy). The library tools omit each pack's `logoDataUri` (console-only).

**Limits.** State is capped per collection per caller (the pack's `limits`); installed groups are hidden
from the Groups list by default and removed with the VS.

## State, Capture, Scenarios & Sequences (stateful mocks, Round 2)

**What it is.** Every virtual service can remember data across requests, durably, in Postgres,
shared by its 2 replicas (any replica serves any caller) — not in-memory, so it costs +30–120 ms
per stateful request and answers `503` + header `x-maxoperf-vs-state: unavailable` if the store is
briefly unreachable (never a stale or guessed answer). Buckets: one **global** ("everyone") bucket,
plus one **session** bucket per distinct session-key value. `update_virtual_service`'s
`state_config.session_key` picks what identifies a session — a header (e.g. `Authorization`), a
cookie, or a query param. No session key configured ⇒ `session` aliases `global` (everyone shares
one bucket). This is how "give each virtual user its own cart" works: point the session key at
whatever the test sends per-VU, and each value gets its own isolated bucket.

**Data egress (disclosed).** A stateful request's headers, cookies, query and body reach the
MaxoPerf API, and captured values are stored centrally (in Postgres and in backups). Raw session
keys are never stored, logged or returned — only an opaque HMAC caller id + a `keyMask`
(`…<last 4 chars>`). **No raw stored value ever reaches an LLM**: every read tool (inspector,
`get_vs_state_value`, a caller's `data`/`label`, `try_virtual_service`'s `diff`, `export_vs_state`)
returns secret-named keys as `[REDACTED]`. The only unredacted copies are the editor-only console
backup download and a raw console export — neither is reachable through MCP (the generic
`call_platform_api` escape hatch is denied on both paths, always). **One named exception:** the
`response` a LIVE `try_virtual_service` renders is the mock's real public output — exactly what any
client calling the mock would receive — so it is returned as-is (not redacted); only its `diff`
entries are redacted.

**When to reach for it.** As soon as the mock needs to remember something instead of answering the
same canned response every time — a login that gates later requests, a running counter, a
cart/order that accumulates across calls. Reach for `create_vs_crud_resource` first for "I need a
fake `/orders` API" — one call instead of authoring seven transactions by hand.

**Variables vs captures.** A **variable** (`state_config.variables`) is a DECLARED value with a
`type` (`text` | `number` | `flag` | `list` | `object` | `counter`), a `scope` (`caller` |
`everyone`) and an optional `initial` — declaring one gives every fresh bucket a starting value and
makes it show up in `get_virtual_service_state` even before anything writes it. A **capture**
(`state.captures[]` on a transaction) is how a value gets WRITTEN: it extracts something from a
matched request (`from.source`: `body` | `header` | `query` | `path` | `cookie` | `template` |
`value`), optionally runs it through a **transform chain**, and writes it with an `action` (`set` |
`append` | `increment` | `delete` | `merge`) to `to: {scope: "global"|"session"|"local", key}`.

**Transform chain (`captures[].transforms[]`, ≤ 16 steps, applied in order).** Kinds: `trim`,
`lowercase`, `uppercase`, `capitalize`, `prefix`, `suffix`, `template`, `regexMatch` (yields
boolean), `regexExtract`, `regexReplace`, `dateFormat`, `numberFormat`, `path`, `pick`, `omit`,
`join`, `default`, `required`, `javascript`. A `javascript` step runs server-side in a QuickJS
sandbox (≤ 50 ms, ≤ 16 MiB) with `(value, request, state)` in scope — pin a saved revision with
`list_transform_snippets` + `{snippet_id, snippet_revision_id}`, or pass unsaved `inline` (≤ 8 KiB)
while iterating. A step that reads `state` fails `javascript_state_too_large` once the buckets pass
256 KiB; `request.body` is visible to JS/regex steps only up to 256 KiB (over that, a step reading
`body` fails `javascript_request_too_large` and gets `request` with no `body`); regex steps fail
`regex_input_too_large` on inputs over 64 KiB. `captures[].default` is written when the source is
missing, or a transform returns `null`/`""` (a transform ERROR never uses the default).

**Cases, Otherwise, and counters (`then_ops`).** A transaction's top-level `cases[]` (≤ 10, each
with 1–10 `when` conditions) let ONE transaction answer differently depending on state or the
request — the first case whose conditions all hold wins; none matching falls through to the
transaction's own response (the "Otherwise"). `then_ops[]` (≤ 10) are counters
(`{counter, scope: "caller"|"everyone", op: "increase"|"decrease"|"reset", by?, initial?}`) that
need NO variable declaration — but **they run on every winning response of the transaction, every
case AND the Otherwise**, so a response that must NOT count (e.g. "cart is full → 409") must be its
OWN transaction, matched earlier (lower `priority`), with no `then_ops` at all.

**Worked example — "count cart adds per caller, remember the username lowercase, 409 when full":**

```jsonc
{ "name": "update_virtual_service", "arguments": { "virtual_service_id": "vs_...",
  "state_config": { "session_key": { "source": "header", "name": "Authorization" },
    "variables": [ { "name": "cartCount", "type": "counter", "scope": "caller", "initial": 0 } ] } } }

// Matched FIRST (lower priority) — a rejected add must not count, so it has no then_ops:
{ "name": "create_vs_transaction", "arguments": { "virtual_service_id": "vs_...", "name": "Cart full",
  "priority": 5, "request": { "method": "POST", "url": { "kind": "equals", "value": "/cart/items" } },
  "state": { "conditions": [ { "scope": "session", "key": "cartCount", "match": "gt", "value": 9 } ] },
  "response": { "mode": "mock", "status": 409, "body_template": "{\"error\":\"cart full\"}" } } }

{ "name": "create_vs_transaction", "arguments": { "virtual_service_id": "vs_...", "name": "Add to cart",
  "priority": 10, "request": { "method": "POST", "url": { "kind": "equals", "value": "/cart/items" } },
  "response": { "mode": "mock", "status": 201, "body_template": "{\"count\": {{session.cartCount}} }" },
  "then_ops": [ { "counter": "cartCount", "scope": "caller", "op": "increase" } ],
  "state": { "captures": [ { "from": { "source": "body", "path": "$.user" },
    "to": { "scope": "session", "key": "userName" }, "action": "set",
    "transforms": [ { "kind": "trim" }, { "kind": "lowercase" } ], "default": "guest" } ] } } }
```

**Editing rules.** `update_vs_transaction` replaces `state`/`cases`/`then_ops` **as a unit** (like
`request`/`response`) — get the transaction first and send the complete object back. Edits reach
live traffic within ~2 s (no redeploy needed) — preview an unsaved case/counter/transform first
with `try_virtual_service` + `draft_transactions` before saving. `update_virtual_service` merges
`state_config` at the top level (keys you send replace — `variables` replaces the WHOLE list, `null`
resets one to its default, keys you omit are kept) — never a client-side read-modify-write. Args
are snake_case while responses are camelCase — when sending back a fetched transaction/stateConfig
rename `requiredState`→`required_state`, `newState`→`new_state`, `afterLast`→`after_last`,
`bodyTemplate`→`body_template`, `sessionKey`→`session_key` (a camelCase key is rejected as
unrecognized, never silently dropped).

**Trying it first.** `sandbox_vs_transaction` runs ONE transaction against a sample request and a
sample starting state, without deploying anything or touching a real virtual service's data — its
result is unchanged from Round 1 (`scenarioTransition`, no `diff`/`scenario`). `try_virtual_service`
is the bigger tool: it runs the FULL priority walk across every transaction, in two modes.
**Scratch** (default) mutates nothing — start from declared-variable initials or pass a previous
`stateAfter` back in to chain steps, and optionally preview unsaved `draft_transactions` (pass
`get_vs_transaction` output straight through — server fields are stripped for you). **Live**
(`live: true`) requires `caller` (the raw session-key value) and runs against REAL live state — its
`response` is the mock's real public output (not redacted); `diff` is. A passthrough winner answers
`204` with header `x-maxoperf-vs-try: passthrough-not-forwarded` in both modes.

**Live state, callers, and clearing.** `get_virtual_service_state` is a straight (bounded) read —
optionally pass `caller` (a caller id from `list_vs_state_callers`, `"~everyone"`, or
`"~anonymous"` for callers that carried no session key) to see one caller's data too;
`runtimeSupportsState: false` means the VS runs an old runtime — redeploy it; `null` means it has
not polled its config yet. A value over 8 KiB shows as `{truncated: true, bytes}` there —
`get_vs_state_value` reads the full thing (redacted). To clear data: `delete_vs_state_value` (one
value), `forget_vs_state_caller` (one caller's whole bucket), `reset_vs_state_flows` (scenario
positions only, optionally VS-wide), or `clear_vs_state` (everything, VS-wide — 503
`VS_STATE_BUSY` means retry in a few seconds). A VS-wide reset/restore takes a short exclusive lock
— live stateful requests may 503 for its duration.

**Counters, flow positions, safe clear.** `set_vs_state_value { name, caller?, increment_by }`
adds atomically to a counter — declared, or one a transaction's behavior counts (instead of `value`; exactly one of the two) so it never
overwrites increments made by live traffic in between — `400 VS_STATE_NOT_COUNTER` for any other
type. `get_vs_flow_positions { flow }` counts callers per step of one flow (it includes the shared
`~everyone` bucket, so with no session key the count is 0 or 1);
`reset_vs_state_flows { flow, step }` then moves only the callers at that step back to the flow
start (flow+step cannot be combined with `caller_id`). `clear_vs_state` first takes a "Before clear"
backup (kind `pre_restore`, skipped when state is empty; one of the 3 newest, purged after 7 days)
that `restore_vs_state_backup` brings back. `try_virtual_service` always evaluates the saved
config (plus `draft_transactions`) and never calls the mock's public URL; live mode writes real
stored values.

`delete_virtual_service` is permanent: it stops a running service on its outpost first, then deletes its state + backups and frees the hostname (`stop_virtual_service` keeps it). Custom domains are not supported on MaxoPerf shared datacenters (`409 custom_domain_unsupported_on_shared_datacenter`).

**Backups & export.** `create_vs_state_backup` takes a manual snapshot (one of 20 manual slots,
freed by `delete_vs_state_backup` — an automatic or `pre_restore` backup can't be deleted this way,
they age out after 7 days). `restore_vs_state_backup` replaces all live state (taking a
`pre_restore` safety backup first). `export_vs_state` reads CURRENT live state in the backup
format without spending a slot — always redacted; the raw form is console-only. Uploading a backup
file is **console-only** (State → Backups & restore → Upload) — there is no curated tool and no
`call_platform_api` example for it, because that endpoint needs a real multipart file.

**One-click CRUD.** `create_vs_crud_resource { virtual_service_id, path: "/orders", scope }`
generates a full in-memory REST resource (list/get-one/create/update/patch/delete + 404-when-absent)
in one call. `scope: "session"` gives every session its own collection (one stored value, ≤ 1 MiB,
rewritten whole on every write); `scope: "global"` shares one collection and is the only scope a
`seed` array (items need a string `id`) may be used with — the seed is merged into a never-written,
everyone-scoped `object` variable, so items are served IMMEDIATELY and again after `clear_vs_state`.

## Dependencies (secrets, VS, tunnels, fleets — one picture)

**What it is.** Every secret, virtual service, tunnel and browser-fleet binding a test has, in one
list (`list_test_dependencies`), plus which tests/live runs currently use a given one
(`get_dependency_usage` — check this before deleting/stopping a dependency). `bind_test_dependency`
/ `unbind_test_dependency` are the curated way to attach one — prefer them over the raw
`PUT /virtual-services/bindings` / `PUT /tests/{testId}/secret-bindings` escape-hatch calls; they
route to the right per-kind endpoint for you. Workspace **config values** (non-secret, plaintext) bind the same
way with `kind: 'config'` and are listed in `configItems` - see "Config values" below.

**Manual vs. inline.** A binding's `origin` is `manual` (created explicitly, e.g. via
`bind_test_dependency`) or `inline` (created because the test's interactive model references it —
a `DependencyRef` chip in an HTTP `urlBase` or a browser `go`/`openWindow` step, see "Dependency
references" in `reference/browser-tests.md`). `unbind_test_dependency` only ever removes a manual
binding — to remove an inline one, remove the chip from the model instead (deleting the binding
directly is rejected with `409 binding_in_use_by_model`). A manual binding the model *also* still
references is kept as `inline` rather than deleted. On a VarioTest member, bindings inherited from
the master (`source: 'vario_master'`) are never removed — unbind them on the master.

**Which kind, when.** Use a **tunnel** when the target is running locally or not yet deployed (a
literal `localhost`/private URL never resolves for a remote runner). Use a **virtual service** when
the target depends on something you don't want to hit for real — a mock standing in for a flaky,
rate-limited or costly downstream. Use a **browser fleet** binding when the test's own SCRIPT drives
custom WebDriver/Playwright automation and needs a live endpoint to connect to mid-run (injected as
`MAXOPERF_FLEET_<name>_WEBDRIVER_URL` / `_PLAYWRIGHT_URL` / `_CDP_URL` / `_TOKEN`; `_CDP_URL` has no token, so
send `Authorization: Bearer <_TOKEN>`, Chromium only, and set a generous connect timeout such as 60 s because
the first connect starts Chrome: 7.1 s cold vs 3.5 s warm in testing; the fleet's Playwright minors are a
rolling set, now 1.59/1.58/1.57/1.56/1.63, always including the runner's, and an explicit
`X-Playwright-Version`/`?pwVersion=` outside it gets HTTP 428; a script's video/HAR/console live on the fleet's
Sessions tab, not the run's tabs) — different from the standalone
browser-fleet feature above, which the user's OWN client drives directly, outside any MaxoPerf run.
Use a **secret** for credentials, always (never a literal in the script). Add an **EUX probe**
(below) when a performance test also needs a real-browser UX read while the load runs.

**Run dependency leases vs. bindings.** A binding is test-level config; `get_run_dependencies`
shows what a specific RUN actually prepared/held (virtual services/tunnels/fleets it started, with
lease state and timestamps) — secrets are not leased and never appear there; the config values the run
injected are in its `configValues` array. A run that can't
prepare a dependency fails loudly with `failure_cause: "dependency_unavailable"`, naming which one
and why.

**Tool arguments.** `create_tunnel`/`create_browser_fleet` take an optional `workspace_id`
(default: the account's oldest active workspace); `list_tunnels`/`list_browser_fleets` take an
optional `workspace_id` filter. `get_tunnel_stats` takes `window_minutes`/`step_seconds`, or
`from`/`to` ISO-8601 (takes precedence — e.g. a run's time range; span ≤ 7 days).

| Tool | Args | Returns |
| --- | --- | --- |
| `list_test_dependencies` | `{ test_id }` | `{ testId, items: [{ kind, entityId, name, envNames, state, url, autoStart, origin: 'manual'\|'inline', source: 'test'\|'vario_master', sourceTestId?, sourceTestName?, bindingId? }], warnings }`. Supersedes `list_virtual_service_bindings` for the full picture. |
| `bind_test_dependency` (write) | `{ test_id, kind: 'virtual_service'\|'tunnel'\|'browser_fleet'\|'secret'\|'config', entity_id, env_var_name?, auto_start? }` — `env_var_name` is the REQUIRED stem `[A-Z][A-Z0-9_]*` (≤40 chars) for virtual_service/tunnel/browser_fleet; OPTIONAL and must match `SECRET_[A-Za-z0-9_]{1,120}` for secret (omit to inherit the secret's own name) and `CONFIG_[A-Za-z0-9_]{1,120}` for config (omit to inject under the key name). `auto_start` only accepted for kind=virtual_service; fleet bindings have no auto-start flag (the run always starts a bound fleet). | The created/updated binding (upsert — re-binding updates the env var name and, for virtual_service, `auto_start`, which resets to `true` when omitted). |
| `unbind_test_dependency` (write) | `{ test_id, kind, entity_id }` | Removes the test's own manual binding: `404` when there is none (`binding_not_found`; `tunnel_not_found` / `browser_fleet_not_found` for those kinds) (bindings inherited from a VarioTest master are never removed), `409 binding_in_use_by_model` when it only exists because the model references it inline — remove the chip from the model instead. A manual binding the model still references is kept as inline. Secret and config unbind preserve the other manual rows of that kind. |
| `get_run_dependencies` | `{ run_id }` | A run's dependency leases (VS/tunnel/fleet it prepared/held), state + `startedByRun` + timestamps; aggregated from every child on a composite parent. Secrets are never leased/listed here. Also `configValues: [{ configValueId, name, envName, value, source, childRunId, memberLabel }]` - the frozen config values the run injected. |
| `get_dependency_usage` | `{ kind: 'secret'\|'virtual_service'\|'tunnel'\|'browser_fleet', entity_id }` | Which tests/live runs use it — check before deleting/stopping a dependency. For a config value use `get_config_value_usage`. |
| `set_eux_probe` (write) | `{ test_id, probe_test_id }` (`null` clears it) | Points a single performance test at a browser test that runs alongside its load as an **EUX probe** (real-browser UX under load). Only valid single performance test → browser test, same workspace; else `400 eux_probe_invalid_target` (`details.reason`, `not_found` for a missing probe test); setting a probe needs the `browser_testing` plan entitlement (`403 eux_probe_requires_browser_testing`). |
| `list_workspace_notification_rules` | `{ workspace_id }` | `{ rules: [{ id, workspaceId, integrationId, trigger, tagFilter, enabled, createdBy, createdAt, updatedAt }] }` |
| `upsert_workspace_notification_rule` (write) | `{ workspace_id, rule_id?, integration_id, trigger, tag_filter?, enabled? }` — omit `rule_id` to create, send one to replace its FULL config (never a partial patch — an omitted `tag_filter`/`enabled` resets to none/`true`); `trigger` ∈ `run_finished`/`run_passed`/`run_failed`/`run_cancelled`/`run_status_changed`/`dependency_preflight_failed`/`tunnel_disconnected_during_run`/`criteria_breached`/`virtual_service_failed`/`browser_fleet_failed` | The created/updated rule. Requires workspace admin. |
| `delete_workspace_notification_rule` (write) | `{ workspace_id, rule_id }` | Deletes the rule. Requires workspace admin. |

## Config values (non-secret workspace settings)

**What it is.** A workspace-level list of named plaintext values (`BASE_URL` -> `https://staging.example.com`)
that tests bind exactly like secrets and receive as env vars. Any workspace **viewer** can read them
(`list_config_values`, values included); only an **editor** can create, edit or delete. Because the value
is visible to every viewer, **never put credentials in a config value - use a workspace secret.** The key
is immutable (to rename, create a new value); a workspace holds up to 200 values of up to 64 KiB each, and
a run's injected config totals at most 512 KiB (`400 config_values_too_large` at run start otherwise).

| Tool | Args | Returns |
| --- | --- | --- |
| `list_config_values` | `{ workspace_id }` | `{ configValues: [{ id, name, value, version, createdAt, updatedAt, createdBy, updatedBy }], canEdit }` |
| `create_config_value` (write) | `{ workspace_id, name, value }` - `name` matches `[A-Za-z_][A-Za-z0-9_]{0,127}` | `{ configValue }`; `409 config_value_name_taken`, `400 config_value_invalid`, `config_value_limit` |
| `update_config_value` (write) | `{ workspace_id, config_value_id, value }` | `{ configValue }` with `version` bumped (the key never changes: `config_value_key_immutable`) |
| `delete_config_value` (write) | `{ workspace_id, config_value_id }` | `{ workspaceId, configValueId, deleted: true }`; `409 config_value_in_use` carries the usage in `details` - unbind first |
| `get_config_value_usage` | `{ workspace_id, config_value_id }` | `{ configValueId, tests: [{ testId, name, projectId, envName, origin }], hiddenCount, notificationIntegrations: [{ id, name }] }` |

**Binding to a test.** `bind_test_dependency { test_id, kind: 'config', entity_id: <config value id>, env_var_name? }`
and `unbind_test_dependency { test_id, kind: 'config', entity_id }` - same gesture as every other kind.
`env_var_name` is OPTIONAL and must match `CONFIG_[A-Za-z0-9_]{1,120}`; omit it and the value is injected
under its key name verbatim. `list_test_dependencies` returns config bindings in a separate `configItems`
array (value included, never in `items`). Inline: a `{ kind: 'config', configValueId, envName }` chip is
accepted everywhere a `{ kind: 'secret' }` chip is (HTTP header/query values, browser step values and
scenario variables) and renders `${MAXOPERF_RUNTIME__<envName>}`. Precedence when the same env name
collides: secrets > config > VS/tunnel/fleet env; a VarioTest member's config wins over its master's.

**Runs.** Every run freezes the config values it injected; `get_run_dependencies` returns them in
`configValues: [{ configValueId, name, envName, value, source, childRunId, memberLabel }]` (not leases). A
`run_snapshot` rerun reuses the frozen values even if the value was since edited or deleted; a
`current_test` rerun reads the live bindings.

**Other consumers (via `call_platform_api`, no curated tool).** A notification-integration webhook's
`config.headers[]` / `config.queryParams[]` items accept `{ name, configValueId }` beside literal values (a
header can also use `secretId`). A virtual service's custom TLS certificate takes
`PUT /virtual-services/{id}/custom-cert { certConfigValueId, keySecretId }` - the certificate may be a
config value, the private key must always be a secret. `POST /tests/sample-request` headers accept
`{ name, configValueId }`.

## EUX probe (browser UX under load)

**What it is.** A single performance test can point `euxProbeTestId` (`set_eux_probe`) at a browser
test in the same workspace; the platform runs that browser test alongside the load as a probe, so
you get real-browser UX numbers (not just server-side latency) while the target is under load.

**When to reach for it.** The user wants to know "does the page still feel fast while it's under
load", not just whether the backend held up. Only valid single performance test → browser test,
same workspace, not itself, not a VarioTest master — every other combination is rejected with
`400 eux_probe_invalid_target` naming the reason (`details.reason`). Setting a probe also needs the
`browser_testing` plan entitlement (`403 eux_probe_requires_browser_testing`). Pass
`probe_test_id: null` to clear it.

## Workspace notification rules

**What it is.** One rule set per workspace (`list_workspace_notification_rules`,
`upsert_workspace_notification_rule`, `delete_workspace_notification_rule`) instead of per test —
covers the run-lifecycle triggers plus five ecosystem ones: `dependency_preflight_failed`,
`tunnel_disconnected_during_run`, `criteria_breached`, `virtual_service_failed`,
`browser_fleet_failed`. An optional `tag_filter` scopes a rule to tests/runs carrying at least one
of those tags. `upsert_workspace_notification_rule` always takes the FULL desired config (send
`rule_id` to replace an existing rule's config, omit it to create) — never a partial patch; on a
replace, an omitted `tag_filter`/`enabled` resets to none/`true`.

## Workspace alerts, APM integrations and Git connections

**What it is.** Three workspace settings pages saved as configuration only: alert rules (metric
threshold + duration + scope + severity, routed to existing notification integrations) with
maintenance windows, APM provider settings (Datadog, Grafana Cloud, New Relic, Honeycomb; the API key
is a workspace secret id, never a value) and Git repository connections (provider, repo, branch, test
path, run trigger). Tools: `list/upsert/delete_alert_rule`, `list/upsert/delete_alert_maintenance_window`,
`list/upsert/delete_apm_integration` + `test_apm_integration`, `list/upsert/delete_git_connection`.

**Be honest about what runs.** No alert is evaluated, no run marker or metric is sent to an APM, and no
Git sync or webhook runs yet (`evaluationEnabled`/`deliveryEnabled`/`syncEnabled` are `false`). Never
tell the user a rule "will fire", an APM is "connected" or a repo is "synced". `test_apm_integration`
really checks the credential only for Datadog and Honeycomb; other providers answer `supported: false`.
For "tell me when a run fails or a criterion breaches" use workspace notification rules, which do run.

Tool arguments (all take `workspace_id`; `upsert_*` creates when the id is omitted, patches when sent; writes need workspace admin):

| Tool | Args | Returns |
|------|------|---------|
| `list_alert_rules` | | `{rules, evaluationEnabled:false}` |
| `upsert_alert_rule` (write) | `rule_id?`, `name`, `metric` (`p95_latency_ms`/`error_rate_pct`/`rps`), `operator` (`above`/`below`), `threshold`, `duration_minutes`, `scope_kind` (`all`/`tags`), `tag_filter?`, `severity`, `destination_ids` (notification integration ids), `respect_maintenance?`, `enabled?` | the rule |
| `delete_alert_rule` (write) | `rule_id` | deleted |
| `list_alert_maintenance_windows` | | `{windows}` with derived `state` |
| `upsert_alert_maintenance_window` (write) | `window_id?`, `name`, `scope_kind`, `tag_filter?`, `starts_at`, `ends_at` | the window |
| `delete_alert_maintenance_window` (write) | `window_id` | deleted |
| `list_apm_integrations` | | `{integrations, deliveryEnabled:false}`; `status` is `configured`/`verified`/`needs_attention` |
| `upsert_apm_integration` (write) | `integration_id?`, `provider` (`datadog`/`grafana_cloud`/`new_relic`/`honeycomb`, create only), `site?`, `api_key_secret_id` (a workspace secret id), `send_markers?`, `tags?` | the integration |
| `delete_apm_integration` (write) | `integration_id` | deleted |
| `test_apm_integration` (write) | `integration_id` | `{supported, ok, message, integration}`; real check only for `datadog`/`honeycomb` |
| `list_git_connections` | | `{connections, syncEnabled:false}` |
| `upsert_git_connection` (write) | `connection_id?`, `provider` (`github`/`gitlab`/`bitbucket`), `repository` (`owner/name`), `branch`, `test_path`, `run_on` (`push`/`pull_request`/`manual`) | the connection |
| `delete_git_connection` (write) | `connection_id` | deleted |

## PDCs (execution locations)

**What a PDC is.** Private Data Center = an **Outpost site**: a place running the MaxoPerf outpost
agent that can host per-VS pods and browser-fleet runners. Two kinds:

- **`visibility: "system"`** — a shared, MaxoPerf-operated PDC. The **Shared Sandbox** is the
  always-present one: every account can place a fleet or virtual service there with zero
  infrastructure of their own.
- **`visibility: "account"`** — a self-hosted PDC the caller's own workspace registered (BYOC —
  bring your own compute).

`list_execution_locations` returns both kinds merged in `byoc.privateDatacenters`, plus curated
managed cloud regions in `managed.providers` (for protocol-test/run location plans — unrelated to
PDC placement).

**Placement preference order** (cheapest/most-available first): a **shared system-catalog PDC**
(Shared Sandbox — no setup needed) → an **alive, healthy user-owned BYOC PDC** (if the account has
one) → a **managed cloud region** (for run/test location plans that don't use PDCs at all, e.g.
protocol tests). Default to the shared PDC unless the user has their own and prefers it (data
residency, network proximity to their real backend, etc.) — ask rather than assume if it's
unclear which they want.

## `locationPlan` — where a test's runners go

`create_test` (the MCP tool) takes the simple flat form: `managed_regions`, `cloud_provider`,
`private_datacenter_id`. Anything richer — several locations, per-location runner counts, a
managed/BYOC mix — is a `locationPlan` object sent through `call_platform_api` to
`POST /v1/tests` or `PATCH /v1/tests/{testId}`.

Managed AWS regions are only `us-east-1`, `eu-west-1` and `ap-southeast-1`; any other AWS region id is rejected (400). Discover valid ids with `list_execution_locations`.

**Every key lives at exactly one level.** The common failures are putting a row key on the plan
(`property cloudProvider should not exist`) or a plan key on a row (`property managedRegions should
not exist`) — unknown properties are rejected outright, they are not ignored.

```jsonc
// ✅ Modern form — `rows`, one entry per location, each with its own runner count
{ "name": "call_platform_api", "arguments": {
  "method": "PATCH", "path": "/v1/tests/tst-0123456789",
  "body": { "locationPlan": { "rows": [
    { "kind": "managed", "cloudProvider": "aws", "region": "us-east-1", "runnerCount": 4, "weight": 2 },
    { "kind": "managed", "region": "europe-west1", "runnerCount": 2 },
    { "kind": "byoc", "privateDatacenterId": "pdc-0123456789", "runnerCount": 1 }
  ] } } } }

// ✅ Legacy flat form — one implied runner per entry. `rows` and these two are ALTERNATIVES:
//    send `rows`, or send these; never mix them in one plan.
{ "locationPlan": { "managedRegions": ["us-east-1", "europe-west1"],
                    "privateDatacenterIds": ["pdc-0123456789"] } }

// ❌ `cloudProvider` is a ROW key, never a plan key
{ "locationPlan": { "cloudProvider": "aws", "rows": [ … ] } }
// ❌ `managedRegions` is a PLAN key, never a row key
{ "locationPlan": { "rows": [{ "kind": "managed", "managedRegions": ["us-east-1"], "runnerCount": 1 }] } }
```

**Row keys** — `kind` (`'managed' | 'byoc'`, required); `region` (required when `kind: 'managed'`);
`privateDatacenterId` (required when `kind: 'byoc'`); `cloudProvider` (`aws`/`gcp`/`azure`, optional —
only needed when a region id exists under more than one provider); `runnerCount` (required integer,
**1–500**); `weight` (optional, 0.01–1 000 000, splits total virtual users across locations, default 1);
`id` (optional client key).

**Bounds** — `rows` ≤ 32 entries · `managedRegions` ≤ 10 · `privateDatacenterIds` ≤ 20 ·
`runnerCount` ≤ 500 per row. Region ids must be real `list_execution_locations` ids, not
guesses at a cloud's own naming.

## Plan limits — read them before composing the request

Runner counts, virtual users and test duration are all capped by the account's billing plan, and the
cap is enforced at `create_test`/`start_run` time with messages like `Plan limits 10000 concurrent
VUs; requested 50000` or `Plan limits tests to 20 minutes`. Those are correct answers, not bugs — but
the user gets a better one if you read the ceiling first and say so up front, instead of proposing a
plan that is going to bounce.

```jsonc
// 1. Which plan is this account on?
{ "name": "call_platform_api", "arguments": { "method": "GET", "path": "/v1/billing/summary" } }
//    → { "planSku": "…", "subscriptionStatus": "…", … }

// 2. What does that plan allow?
{ "name": "call_platform_api", "arguments": { "method": "GET", "path": "/v1/billing/plans" } }
//    → { "plans": [ { "sku": "…", "limits": { … }, "consumables": [ … ] } ] }
//    Match `sku` to step 1's `planSku` and read its `limits`.
```

The `limits` block carries `maxConcurrentVus`, `maxConcurrentRunners`, `maxConcurrentRuns`,
`maxTestDurationMinutes`, `maxWorkspaces`, `maxProjectsPerWorkspace`, `retentionDays`,
`allowedEngines`, and the BYOC/SSO/APM entitlement flags. **`-1` means unlimited.** Live
consumption against the metered dimensions is `GET /v1/billing/quotas`.

When the user asks for more than the plan allows, say what the ceiling is and offer the largest
run that fits — do not silently shrink the request, and do not send it knowing it will be rejected.

**Prepaid credit (one-time payments).** A plan can be paid either by a PayPal subscription or from a
prepaid credit balance funded by one-time PayPal payments. `GET /v1/billing/credit` returns the
balance, the ledger and — for a credit-funded plan — `fundingSubscription`. `subscriptionStatus:
"past_due_credit"` means the balance did not cover the renewal: the plan and its limits stay in force
until `fundingSubscription.graceEndsAt`, then the account moves to `free` (the balance is kept). A
top-up needs the customer's approval in PayPal, so never try to pay from an agent: point the user to
**Billing → Prepaid credit → Top up** in the console. Prepaid credit does not expire and is not
refundable.

## On-demand browser inspection (`inspect_url`)

**What it is.** Spins up (or reuses) a real browser — via the same placement preference order
above — navigates to a public URL, and returns the JS-rendered page: DOM/text/outline, same shape
family as `read_page_dom`.

**When to reach for it vs. `fetch_url`/`read_page_dom`.** Those two are static-HTML-only — no
JavaScript execution. They're the first thing to try (cheaper, faster). Reach for `inspect_url`
when the target is a single-page app or otherwise renders its real content client-side, and
`read_page_dom` comes back empty, or you need to see the page as a real browser would while
planning a browser test's selectors.

## Driving a browser (`browser_run`)

**What it is.** `inspect_url`'s multi-step sibling: instead of loading one URL and returning its
rendered DOM, it DRIVES the same fleet browser through an ordered list of steps
(`goto`/`click`/`fill`/`waitFor`/`evaluate`/`extract`/`screenshot`) against ONE page, in a single
call, and reuses the same `loadrigo-inspect` fleet + placement order.

**When to reach for it vs. `inspect_url`.** `inspect_url` is a snapshot — one page load, one
rendered-DOM read. Reach for `browser_run` when the task requires DOING something first: signing
in, clicking through a flow, filling and submitting a form, then reading what the browser shows
afterward or capturing a screenshot of the result.

**Step semantics.** Steps run in order; the tool stops after the first failed step (later steps
come back `skipped` rather than being attempted, so the result count always matches the step
count). Every `goto` step is SSRF-guarded exactly like the initial url. `screenshot` steps return
a base64 PNG appended to the top-level `screenshots` array.

## Datasets, shared CSV and CSV repair

A **dataset** is a row-level table a test binds to (`get_test_data_bindings`); a **shared CSV file** is the uploaded source a dataset column reads. Use `list_datasets` / `list_csv_files` to find them; both report who uses what (`linkedTestCount`, `usedByDatasets`).

- **Parse options** (delimiter `, ; tab |`, encoding `utf-8 / utf-16 / iso-8859-1 / windows-1252`, quote `" '`, first-row-is-header) are chosen at import (`POST /v1/datasets/sniff`, `POST /v1/workspaces/:id/csv-files` via `call_platform_api`), applied when the file is parsed and **persisted on the file**; every later preview, export and run reads it the same way.
- **Repair:** a dataset flagged `isCorrupted` has malformed rows. `scan_dataset_repair` lists each issue with the fixes the server can apply (`eligibleActions`); then `apply_dataset_repair` with one decision per line (`escape` doubles stray quotes, `exclude` drops the row, `keep` leaves it). The original file is never modified: a repaired copy becomes a new dataset revision and the dataset is re-scanned. Show the user the `escapePreview` and ask before excluding rows.

Tool returns (exact shapes):

| `list_datasets` | `{ workspace_id }` | `{ datasets: [{ id, name, sourceType, rowCount, columnCount, isCorrupted, activeRevisionNumber, linkedTestCount, csvFileId, updatedAt }] }`. Read-only. `isCorrupted` = malformed CSV rows (`scan_dataset_repair`); `linkedTestCount` = distinct tests that bind it. See `reference/platform-features.md` "Datasets, shared CSV and CSV repair". |
| `list_csv_files` | `{ workspace_id }` | `{ csvFiles: [{ id, filename, fileSizeBytes, rowCount, columnCount, delimiter, encoding, quote, hasHeader, usedByDatasets, linkedTestCount, uploadedBy }] }`. Read-only; the parse options are the ones persisted at import. |
| `scan_dataset_repair` | `{ workspace_id, dataset_id }` | `{ source: 'file'\|'recorded', scannedRows, scannedAt, canApply, issues: [{ line, errorType, message, rawLine, eligibleActions, escapePreview, explanation }] }`. Read-only server scan of the stored CSV. |
| `apply_dataset_repair` | `{ workspace_id, dataset_id, decisions: [{ line, action: 'escape'\|'exclude'\|'keep' }] }` | `{ revisionNumber, escaped, excluded, kept, skipped, csvFileId, scan }`. Writes a NEW shared CSV and a new dataset revision, then re-scans. Needs editor; lines shift after an exclude, so scan again before a second apply. |

## Test data in scripts

**What it is.** A test's data bindings (CSV files from Datasets, bound in the console's `Data` tab)
land at a stable path inside every runner: `MAXOPERF_DATA_DIR` (`/work/run/data`) plus one
`MAXOPERF_DATASET_<SLUG>_CSV` env var per binding, pointing at `<dataDir>/<slug>.csv`. The `slug` is
frozen on the binding at bind time — renaming the dataset, or swapping it for a different dataset on
the same binding, never moves the file or changes the env var name. `get_test_data_bindings
{ test_id }` (`reference/mcp-tools.md`) returns every binding's `slug`, `relativePath`,
`containerPath`, `envVar`, `columnNames[]` and `scriptVariable`, plus `executionContext.dataDir` /
`dataDirEnvVar` — read it before telling a user where their CSV is, rather than guessing the path.

**When to reach for it.** The user asks "where does my CSV go", "what env var has my test data", or
wants a script to read a bound dataset. Also reach for it before writing a script that consumes test
data, so the generated code uses the real slug/env var instead of a placeholder.

**Two routes — use one, not both.** For k6, Locust and Gatling, MaxoPerf can auto-inject the CSV
read via the `@maxoperf-data-bindings` marker, or the script can read the file itself from the env
var (self-read requires every binding's EOF mode to stay `recycle`):

- **Auto-inject (route a).** Place the marker as a comment alone on its own line where the injected
  code belongs — `// @maxoperf-data-bindings` (k6, Gatling) or `# @maxoperf-data-bindings`
  (Locust), exactly once across the test's scripts — then reference `scriptVariable`:
  - k6: marker inside the default function (the injected code may `return` on exhaustion under a
    `stop` EOF mode), then `<scriptVariable>.<column>`.
  - Locust: marker inside the task, then `<scriptVariable>["<column>"]` — `scriptVariable` already
    carries the `self.` prefix (e.g. `self.maxoperf_row_checkout_skus`), never add another.
  - Gatling: marker where the `.feed` belongs in the scenario chain — MaxoPerf inserts the `.feed`
    itself, never add your own — then reference columns as `#{column}`. `scriptVariable` is `null`
    for Gatling (the feed session key isn't a script-visible variable).
- **Self-read (route b).** Open `containerPath` (or the file at `envVar`) directly and parse the
  CSV yourself. Works on any engine because of the unpatched-self-read run path, but the binding's
  EOF mode must stay `recycle` — a `stop` EOF mode on a self-read script, or an ambiguous marker
  (twice, or two marked scripts), is refused at run start with `422 dataset_eof_unsupported`.

**JMeter** only has one route: exactly one `CSVDataSet` per binding whose `filename` is `<slug>.csv`
or `data/<slug>.csv` (a binding with no matching `CSVDataSet`, or two, keeps the run from starting)
— MaxoPerf points it at the delivered file; `scriptVariable` is `null` (the `CSVDataSet`'s own
`variableNames` are the column names). A `.jmx` with any `CSVDataSet` is always patched, so an
env-var filename is not supported there.

**Taurus YAML + apiritif** get `data-sources` added automatically; other Taurus-wrapped engines
follow their own engine's route above (`eofCapability.resolvedExecutor` in the response reflects the
underlying engine, not `taurus`, so a Taurus-wrapped k6/Locust script still counts for
`scriptVariable`). Engines not listed here (Playwright, Selenium, …) always self-read via the env
var.

**Renaming.** Changing a binding's `slug` (via `PUT /tests/{testId}/data-bindings`,
see "Writing bindings" below) renames the file, the env var, and — for k6/Locust — the injected
variable. Warn the user: scripts that reference the old names must be updated to the new ones.

**Writing bindings.** Writes go through the escape hatch — no curated `set_test_data_bindings` tool (a full-replace write
tool risks silently dropping other bindings). Rename the file backing an existing binding:

```jsonc
// PUT /tests/{testId}/data-bindings is a FULL REPLACE — GET first, then send EVERY binding back,
// each with its slug (unchanged ones keep their current slug; the one you're renaming gets the new
// slug). Copy these write fields from the GET response unchanged: datasetId, datasetRevisionId,
// entityId, variantId, distribution, position, localOverrides, scope — an omitted one is NOT kept:
// distribution resets to "split", a pinned datasetRevisionId is unpinned, localOverrides/scope are
// cleared. Omit eofMode to keep the current EOF mode (sending eofMode on an artifact whose
// eofCapability.supported is false is refused with 422 dataset_eof_unsupported). Send nothing
// else — echoing any response-only field (id, testId, effectiveEofMode, relativePath,
// containerPath, envVar, columnNames, scriptVariable, …) is rejected with 400.
{ "name": "call_platform_api", "arguments": {
  "method": "PUT", "path": "/tests/{testId}/data-bindings",
  "body": { "bindings": [
    { "datasetId": "ds-0123456789", "datasetRevisionId": "dsr-0123456789", "distribution": "replicate",
      "position": 0, "localOverrides": null, "scope": null, "slug": "checkout-skus-v2" },
    { "datasetId": "ds-0987654321", "distribution": "split", "position": 1, "slug": "users" }
  ] } } }
```

Renaming the slug renames the file, the env var, and (for k6/Locust) the injected row variable —
warn the user that scripts using the old names must be updated.

## Dataset Library (ready-made test data)

Free catalog of ~97 datasets across 18 industries + 8 categories (credentials, search terms, addresses per country, payments sandbox cards, …). Flow: `list_dataset_library` (full concise catalog — filter it yourself by group/title) → `get_dataset_library_item` to check columns/preview → `use_dataset_library_item` `{ workspace_id, slug, name?, row_count? }` to add an editable copy to a workspace (returns `{ datasetId, name, rowCount, csvFileIds }`; `rowCount` may be clamped) → bind it to a test in the console (Test → Test data → "Add from Dataset Library" does add + bind in one step), or via `call_platform_api` `PUT /tests/{testId}/data-bindings` after reading the current bindings with `get_test_data_bindings` (verify the request body arrives as a JSON object, not a double-encoded string — known `call_platform_api` body issue). Console: /datasets/library. Guests can browse (`list_`/`get_`); adding needs a signed-in workspace editor.

## Sync start (runners start together)

A multi-runner run waits at a barrier: every runner finishes booting and loading the test, then all start together. This is the default; pass `sync_start: false` to `start_run`/`rerun_run` (or untick it in the console) to let each runner start as soon as it is ready.

- `get_run_status` shows `syncStart`, `readyRunners` / `plannedRunners`, `startDeadlineAt`, and `startReleasedAt` + `startReleasedReason` (`all_ready` | `force`) once released.
- `force_start_run { run_id }` releases a waiting run now with the runners that are ready (write tool; refused in read-only sessions). 409 `not_waiting` if the run is not waiting, 409 `not_applicable_for_fleet_run` for a browser-fleet run.
- The wait is capped (default 15 minutes, incl. dependency prepare for composite members); if not every runner is ready by then the run is cancelled with cause `sync_start_timeout`. Tell the user to force start or opt out if boot is slow.
- Runners on old or pinned runner images, and runners without a control channel, start immediately and are not waited for. Single-runner runs and browser-fleet runs are unaffected.

## Live controls (steering a running run)

**What it is.** Changes applied to a run WHILE it runs, without restarting it: the virtual-user
total, a requests/second cap, engine properties, data files a script reads, and pause/resume.
Every change is recorded as a numbered revision with a per-runner acknowledgement.

**When to reach for it.** The user is watching a live run and wants it different *now*: "double
the load", "cap it at 200 RPS", "pause it while I check the database", "flip the feature flag".
Also when they ask "what did I change?" — that is `get_live_control_history`. To change what the
NEXT run does, edit the test instead; to end a run, use `cancel_run`.

**Always read first.** Call `get_live_controls { run_id }` before any write. For each `kind` it
reports `support` (`null` = this executor cannot do it), `requires` + `requirementMet` (`false` =
the script lacks a precondition; tell the user exactly what `requires` names instead of calling
the write, which would return `422 CONTROL_PRECONDITION_UNMET`), and `writable` (false once the
run is not running).

**Per-executor support.**

| Control | Where it works |
| --- | --- |
| Virtual users | k6 (the `externally-controlled` executor), JMeter (Concurrency Thread Group), Locust, Gatling (with the MaxoPerf helper) |
| Throughput | JMeter (Constant Throughput Timer), Gatling (with the helper), k6 (`externally-controlled` executor) — **approximate** (MaxoPerf moves VUs to approach the rate) |
| Pause / resume | k6 (the `externally-controlled` executor), Locust, JMeter (Concurrency Thread Group), Gatling (with the helper) |
| Properties, config files | Every executor (delivered as files a cooperating script reads; JMeter and Locust apply properties live) |

**The split.** `total` and `requests_per_second` are fleet-wide totals: MaxoPerf divides them
across the runners in `scope` (default: every runner) and returns `perRunner`. A share is rounded
up and is never below 1, so if the divide is inexact the response has `clamped: true` and
`effectiveTotal` — report that number, not the one you asked for. Properties and config files are
not divided; every runner in scope gets the same values.

**Throughput `0` means unlimited**, never "stop". To stop load without ending the run, use
`pause_run`; `resume_run` returns to the exact pre-pause state rather than re-ramping. Always say
"≈" when reporting a k6 throughput value.

**Tools and args.**

| Tool | Args | Returns |
| --- | --- | --- |
| `get_live_controls` | `{ run_id }` | What a RUNNING run lets you change right now: `{ writable, liveRunnerCount, controls: [{ kind, support, requires, requirementMet, payload }] }`. `support: null` = not available for this executor; `requirementMet: false` = the script lacks what `requires` names; `support.approximate: true` = MaxoPerf projects the number onto another lever (k6 throughput). **Call it before any live change.** |
| `get_live_control_history` | `{ run_id, revision?, limit?, cursor? }` | The live-change audit trail, newest first (`revisions[]`, `nextCursor`). With `revision`: that change's per-runner acks (`acks[]`, `tally{ applied, failed, target }`) — how you confirm a change reached every runner. |
| `set_live_virtual_users` (write) | `{ run_id, total, scope? }` | Changes the fleet-wide VU **total** mid-run; MaxoPerf divides it across the runners in `scope`. `202 { revision, runnerCount, matchedRunnerCount, perRunner, clamped?, effectiveTotal?, dispatchState, delivered }` — when `clamped` is true the share was rounded up (never below 1 per runner), so report `effectiveTotal`. Needs the right plan shape: JMeter a Concurrency Thread Group (`requires: jmeter.concurrency-thread-group`), Gatling the MaxoPerf helper (`requires: gatling.maxoperf-helper`), k6 the `externally-controlled` executor (`requires: k6.externally-controlled`); Locust needs nothing. Anything else is `422 CONTROL_PRECONDITION_UNMET` naming that id. |
| `set_live_throughput` (write) | `{ run_id, requests_per_second, scope? }` | Fleet-wide RPS cap, divided across runners. **`0` = unlimited** (removes the cap; it never stops load — use `pause_run`). On k6 it is **approximate** (MaxoPerf moves VUs); say "≈" when you report it. Needs a rate lever in the plan: JMeter a Constant Throughput Timer (`requires: jmeter.throughput-timer`), Gatling the helper (`requires: gatling.maxoperf-helper`), k6 the `externally-controlled` executor (`requires: k6.externally-controlled`). **Locust has no throughput control at all** — the tool answers `422 CONTROL_UNSUPPORTED_FOR_EXECUTOR`. |
| `set_live_properties` (write) | `{ run_id, properties: { key: "value" }, scope? }` | Sets engine properties every runner in `scope` receives (not divided). `maxoperf_*` keys are reserved. |
| `set_live_config_files` (write) | `{ run_id, files: [{ name, content }], scope? }` | Drops data files (`.json`/`.yaml`/`.csv`/`.properties`, …) a cooperating script reads. Send plain-text `content`; the tool base64-encodes it. ≤256 KiB per file. |
| `pause_run` (write) / `resume_run` (write) | `{ run_id, scope? }` | Stops generating load without ending the run (runners stay allocated), then resumes to the exact pre-pause state — not a re-ramp. On k6 only with the `externally-controlled` executor (otherwise `422 CONTROL_PRECONDITION_UNMET`, `requires: k6.externally-controlled`); JMeter needs a Concurrency Thread Group (`requires: jmeter.concurrency-thread-group`) and Gatling the helper (`requires: gatling.maxoperf-helper`). To END a run use `cancel_run`. |

Live-control writes (`set_live_*`, `pause_run`, `resume_run`) take an optional `scope`:
omitted = every runner; `{ type: 'locations', locations: [{ kind: 'managed', provider, region } | { kind: 'byoc', privateDatacenterId }] }`
(real ids from `list_execution_locations`); or `{ type: 'runners', runnerIds: [...] }`.

**Errors to explain, not retry blindly.** `422 CONTROL_UNSUPPORTED_FOR_EXECUTOR` /
`CONTROL_PRECONDITION_UNMET` (read `details.requires`), `422 SELECTOR_EMPTY` /
`SELECTOR_NO_MATCHING_RUNNERS` (the scope matched no live runner; `details.unmatchedRunnerIds`
names runners that are not in this run, `details.unmatchedLocations` locations with none), `400 LIVE_CONTROL_SCOPE_LOCATION_UNKNOWN` (use a real id
from `list_execution_locations`), `403 PLAN_LIMIT_EXCEEDED` (increases only; a decrease always
passes), `409 RUN_NOT_RUNNING` and `409 LIVE_CONTROL_REVISION_CONTENDED` (concurrent writes;
retry once). **Every live-control tool, read or write, can also answer `404` (no such run on this
account — it is not yours, so never retry it) and `403` (no permission to view or control this
run).** With `revision`, `get_live_control_history` answers `404 REVISION_NOT_FOUND` for a revision
this run never had.

**Confirm it landed.** A `202` means the change was recorded, not that it applied: `delivered:
false` means it has not reached the runners yet, and `dispatchState: coalesced` means a newer
change to the same control replaced it before dispatch.
`get_live_control_history { run_id, revision }` shows each runner's ack; report any runner that
failed it, with its error.

## Failure criteria labels (`failure_criteria[].label`)

A criterion's `label` is an exact, case-sensitive **sample-label filter**, not a display name.

- `"ANY"` = every transaction label (the `COMBINED` rollup excluded). It is the default: an empty,
  missing or whitespace label is stored as `"ANY"`, and so is any label on a subject that cannot be
  label-scoped (e.g. `concurrency`). `"ANY"` is reserved — a sample literally named `ANY` cannot be
  targeted; `any` (lower case) is an ordinary label.
- Otherwise use an exact transaction label. `list_test_labels { test_id }` returns the labels of
  the latest finished run (`source: "last-run"`) and of the interactive definition
  (`source: "definition"`).
- An unknown label is **saved, never rejected** (a new test has no labels yet), but the
  `create_test` result carries `warnings: [{ code: "CRITERION_LABEL_UNKNOWN", criterionId, label }]`.
- A criterion that matched zero samples in a run is reported with status `no-data` — it did not
  evaluate, so fix the label (or use `"ANY"`) rather than reading it as a pass.
- Human-readable text belongs in `message`, which becomes the criterion's summary line.

## Root cause analysis

The run report's **Root cause** tab, over MCP as `get_run_root_cause { run_id, response_format? }`
(read-only; same data as `GET /v1/runs/{runId}/root-cause`). Errors are grouped into buckets
(`auth`, `rate_limit`, `network`, `server`, `database`, `bad_request`, `test_script`, plus
`failure_criteria`, `load_generator`, `other`); each entry has a count, first/last seen, and an AI
`why` + `fix` steps, and the run gets a one-line verdict with top actions.

- Analysis runs in the background while the run is live and once more when it ends — an entry can be
  `pending`/`failed`/`skipped`; re-read later rather than guessing. `ai.state` is
  `ready | analyzing | unavailable | disabled | not_available`.
- `ai.state: "disabled"` + `reason: "account_opt_out"`: the **account owner** turned AI analysis off
  (console → Account settings → AI & data). It is not settable over MCP/API keys; say so, then fall
  back to `get_run_errors` / `get_run_error_bodies`. Browser-fleet runs return a 422 (unsupported).
- Privacy: error signatures, redacted samples and log lines are sent to the AI subprocessor
  (DeepSeek) for the analysis; the stored output is the account's own run data.
- `concise` (default) = verdict + top buckets x top 3 entries; `detailed` = the full body. Raw
  evidence: `get_run_error_bodies { signature }` or `get_run_logs`.

## Run errors & captured bodies

`get_run_errors { run_id }` groups error rows by signature (normalized message, ids/values masked)
and category — `http_5xx` | `http_4xx` | `network` | `assertion` | `other`. A row recorded before
the error-grouping update has `signature: null` and `category: null`; fall back to `response_code`
to identify it instead.

Drill into a group with `get_run_error_bodies { run_id, signature }` — a sample's `errorKey` IS its
signature (samples carry no separate `category`; category comes from the `get_run_errors` group).
If a signature returns no rows (a browser/Locust error message differs from the sample's status
line), fall back: retry with `label` + `response_code`, then `label` + `category` (the group's) —
never `label` alone (other errors' bodies). Legacy row (`signature: null`): pass `response_code`.

**Coverage** (whether a body is actually captured, not just counted):

| Engine | Coverage |
| --- | --- |
| jmeter, k6, gatling, locust, apiritif, playwright | `full` |
| selenium | `recording` — Chrome captures the full body; Edge/Firefox capture metadata only, both written at session end |
| others | messages only, no body |

**Known limits** (state these rather than presenting a gap as a bug): k6 samples only from the
first execution and VUs 1–2; Selenium needs the first execution to be Selenium, then covers every
Selenium execution; Locust FastHttpUser not captured; k6 http.batch/asyncRequest not wrapped.

Locust custom failures (`response.failure(...)`, `CatchResponseError`) are recorded by Taurus with
code 500 and appear under Server errors (5xx).

Playwright test failures (`Test failed: …`, code 500 or the last navigation status) are categorized
`assertion`, or `network` for timeouts and connection errors, never `http_5xx`.

**Redaction** applies to every captured body: secret headers, body/form/query keys, XML/SOAP
elements, multipart parts, URL credentials (`user:pass@`), Bearer/Basic/JWT tokens, URLs, and
response messages are redacted; sample labels are kept raw.

**Turning it off.** Capture defaults to on and is **single tests only** — a VarioTest master
rejects the setting and its members always inherit their source test's own setting. Set it at
create time with `create_test { capture_error_bodies: false }`, or change it later with
`call_platform_api PATCH /v1/tests/{testId} {"captureErrorBodies": false}` (no dedicated MCP
update-test tool).

**Blocked traffic.** `get_run_summary` also returns `blockedTraffic` (`detected`, `share`, `codes`, `wafSignature`, `mitigations[]`): detected when 100+ requests ran and 30%+ answered 403/429 or a WAF/CDN vendor was seen on captured 403/429 headers alongside at least one blocked response. Counts only samples the engine recorded a response code for (under-reports otherwise). If `detected`, tell the user the target blocked the load and walk the `mitigations` (allowlist runner IPs, WAF bypass header, lower the rate) before tuning the test. Absent = not computed, never "not blocked".

## AI intent — turning a description into an object

When the user describes something in words that maps to a builder object — requests for an HTTP
test, a load profile ("ramp to 200 users over 5 min"), failure criteria ("fail if p95 > 800ms"), or
a cron schedule ("every weekday at 9am") — call `ai_convert_intent` with the matching `surface` and
the user's `test_id` (existing test) or `project_id` (new test), show the returned `object`, and
apply it with the normal create/update tools after the user agrees. It does not save anything. Keep
the `aiSessionId` and pass it back (with the same `text` plus a `guidance` list) to refine the
result cheaply. Not available to guest trials. Arguments and error codes: "AI intent tool arguments" just below. The same tool also covers virtual services (see "Virtual-service surfaces" there).

### AI intent tool arguments

Three tools turn a plain-language description into a validated MaxoPerf object, as the API key
owner. Nothing is saved — apply the result with the normal create/update tools.

| Tool | Args | Returns |
|------|------|---------|
| `ai_convert_intent` | `surface` (`http-builder.requests` \| `http-builder.request` \| `load-profile.section` \| `failure-criteria.section` \| `schedule.cron` \| `vs-new.generate` \| `vs-transactions.add` \| `vs-transaction.edit` \| `vs-behavior.transaction` \| `vs-behavior.fallback` \| `vs-state.config` \| `vs-flow.journey` \| `vs-flow.try-script` \| `vs-deployment.settings` \| `vs-settings.section` \| `vs-analytics.mock-unmatched`), `text` (1..2000), `ai_session_id?` (uuid), `guidance?` (≤10 × ≤500), `test_id?` \| `project_id?` (test surfaces: exactly one; `schedule.cron` needs `test_id`), `virtual_service_id?` \| `workspace_id?` (VS surfaces: exactly one; `vs-new.generate` needs `workspace_id`), `attachment?` (≤ 80000 chars of pasted OpenAPI/curl; `vs-new.generate` and `vs-transactions.add` only), `current?` | `{surface, aiSessionId, object, warnings, usage}` |
| `ai_intent_suggestions` | `surface` (a convert id, `browser-steps.line`, or `<id>.guidance`), `q?`, `limit?` (1..20) | `{items:[{id,text,source}]}` |
| `ai_intent_history_delete` | `id` (a history suggestion id) | `{deleted:true}` (write; blocked read-only) |

Billed on `loadrigo_tokens`. Reuse `aiSessionId` on the same surface for cheaper cached calls. To
correct a result, call again with the same `ai_session_id` and `text` plus the full `guidance` list.
Error codes: `AI_INTENT_CONTEXT_REQUIRED` (no or two anchors), `AI_INTENT_UNRESOLVABLE`,
`AI_QUOTA_EXCEEDED`, `AI_UNAVAILABLE`. There is no history-record tool: saving autocomplete history
is a console action, so agent text never pollutes the owner's personal suggestions.

### Virtual-service surfaces

For a mock API described in words (or a pasted OpenAPI / curl `attachment`), call `ai_convert_intent`
with `vs-new.generate` + `workspace_id` to draft a whole virtual service (transactions, CRUD resources,
state variables), then save it with `create_virtual_service` / `create_vs_transaction` — nothing is
saved by convert. On an existing virtual service pass `virtual_service_id`: `vs-transactions.add`
(add transactions or CRUD resources — batch big APIs, at most 15 per call), `vs-transaction.edit`,
`vs-behavior.transaction` (state, cases, then-ops), `vs-behavior.fallback` (no-match policy),
`vs-state.config` (variables), `vs-flow.journey`, `vs-flow.try-script`, `vs-deployment.settings`,
`vs-settings.section`, `vs-analytics.mock-unmatched`. Pass the returned text back as
`source_text: {text, edited: false}` on `create_vs_transaction` / `update_vs_transaction` so the console
shows the AI origin; a later manual edit flips `edited` server-side. These calls can take up to ~80 s.

## AI steps — numbered lines into a compiled browser test

When the user describes a browser flow as numbered lines ("1 go to … 2 add an item to the basket …
5 ensure page says …"), call `start_browser_steps_compile` with the existing interactive browser test's
`test_id` and the lines, then poll `get_browser_steps_compile` (`session_id` + `compile_id` from the start
reply, every ~5-10 s) until `state` is `done`. It drives a real cloud browser (billed on `loadrigo_tokens`),
creates a NEW version on every compile and, when every line compiled, makes it the test's active steps.
Not available to guest trials. `start_` returns in under ~45 s: `{state:"running", sessionId, compileId}`, or
`{state:"warming", sessionId}` while the cloud browser boots (nothing started yet: call `start_` again with
the same arguments). `compile_browser_steps` is the same compile as one call that blocks up to 5 min; prefer
start/poll (agent clients and Loadrigo give up after 60-90 s).

Lines such as "ensure page says X", "ensure the page does not contain X" or "ensure the page matches
/regex/" compile to an `assertPageContains` step (see `reference/browser-tests.md`) and now FAIL the
line when it does not hold, after waiting ~5 s, instead of silently waiting. Phrase negation ("does not
contain"), regex ("matches /…/") and "ignore case" explicitly.

Lines can say more than clicks. Scrolling ("scroll to the bottom of the screen"), hover, right-click, tab/frame
switching and "remember X as NAME" (later lines reference `${NAME}`) compile too; where no ready-made step
exists the AI writes JavaScript that runs in the test's own browser and shows as an AI-written script the user
can read and edit. Never put secrets in lines. Indent steps with two spaces under a header line ending in a
colon (`If …:`, `Otherwise:`, `Repeat N times:`) to make a conditional or loop block; `Repeat 3 times: scroll
down` is the one-line form. Lines in a branch that was not taken while compiling come back `resolved_live` with
`not verified: …` or `skipped` with `not verified: … missing from the test until compiled`: show those to the
user, never report them as passing. There is no repeat-until, for-each, drag, upload or native dialog. In the
user's own Chrome (the extension) scroll/script steps, remembered values, `If` blocks and window/frame
switching fail by name with `not supported in Your Chrome: <action>`; hover, right-click and `Repeat` work.

| Tool | Args | Returns |
|------|------|---------|
| `start_browser_steps_compile` | same args as `compile_browser_steps` | `{state:"running" or "warming", sessionId, compileId?}` |
| `get_browser_steps_compile` | `session_id`, `compile_id` | `{state:"running"}` or `{state:"done", versionId, version, active, lineResults}`; failures are errors |
| `compile_browser_steps` (blocking) | `test_id`, `lines` (`N: text`, `# Heading` = transaction, ≤100), `recompile_line_id?` (e.g. `L2`), `guidance?` (≤10 × ≤500, only with `recompile_line_id`), `activate?` | `{versionId, version, active, lineResults:[{lineId, text, status, stepIds, error?, candidates?}]}` |
| `list_browser_step_versions` | `test_id` | `{items:[{id, version, active, counts, …}]}` newest first |
| `get_browser_step_version` | `test_id`, `version_id` | version with source lines, line results, `compiledModel` |
| `activate_browser_step_version` | `test_id`, `version_id` | the activated version (also rollback); write |

Reference a secret by its workspace secret NAME in the line text ("login using secret1 and
secret2"); values are never returned and no tool reveals them. A version with a failed line is saved
inactive and the test keeps its previous steps; fix it by compiling again (`start_browser_steps_compile`) with
`recompile_line_id` plus `guidance` (e.g. `["use the second product"]`) and the same `lines`, then
activate if it did not auto-activate. The console's per-line / "compile up to here" / auto compiles save a
`draft: true` version (inactive; the same user's next one overwrites it; `counts.notRun` > 0 or `failed` > 0
means `activate_browser_step_version` answers `AI_STEP_VERSION_HAS_FAILED_LINES`): do not treat a draft as
the test's steps, and compile for a full version. Errors: `AI_QUOTA_EXCEEDED`, `AI_STEPS_TEST_NOT_BROWSER`,
`AI_STEPS_COMPILE_TIMEOUT`, `AI_UNAVAILABLE`, `AI_STEPS_SESSION_LOST`, `AI_STEPS_COMPILE_FAILED`, `AI_STEP_VERSION_HAS_FAILED_LINES`.
Starting again while the SAME test's compile runs returns the same `running` + `compileId` (keep polling it); it attaches regardless of the new request body (new lines/guidance are ignored).
A compile running for a DIFFERENT test answers `AI_STEPS_COMPILE_IN_PROGRESS` (it is never replaced; the message
carries its `sessionId`/`compileId`): poll that one, or wait a minute and call `list_browser_step_versions` to find the new version.

## Deleting things (soft delete)

Tests, runs, PDCs and teams are **soft-deleted**: hidden from lists and not reachable by id, never physically removed. `delete_test {test_id}` and `delete_run {run_id}` (also on Loadrigo) are blocked while a run is in progress / the run is not finished (409 `TEST_HAS_ACTIVE_RUN`, `RUN_NOT_TERMINAL`); a test used by a VarioTest is blocked (`TEST_IN_VARIOTEST`). A deleted test's past runs stay visible, labelled as a deleted test. `delete_private_datacenter` (409 `PDC_IN_USE` with a blocker list) and `delete_team` (409 `SYSTEM_TEAM`) need `confirm_name` equal to the entity name and are **not** on Loadrigo. CLI: `maxoperf delete test|run|pdc|team <id>`.

`maxoperf push` matches tests **by name**, so it re-creates a test that was deleted in the console.

## Account administration

- **Tags** are free-form labels on tests and runs. `list_account_tags` is the catalogue; renaming or deleting
  one rewrites every item in every workspace, so it is account-owner only and you should quote the counts to
  the user first. Deleting a tag never deletes the tests/runs; a notification rule whose only tag was deleted
  is disabled rather than widened.
- **Members** — roles are `owner`, `billing_manager` and `member`. `lastActiveAt` is the last authenticated
  request, not a login time. Removing a person revokes the API keys they created in this account.
- **Billing contact** — if one is set, receipts and payment-failed notices go to it instead of the default
  recipient. Ask the user before changing it.

Tool arguments:

| Tool | Args | Returns |
| --- | --- | --- |
| `list_account_tags` | *(none)* | Distinct tags with `testCount`/`runCount` and `canManage` (account owners only). |
| `rename_account_tag` (write) | `{ tag, new_name }` | Renames the tag on every test, run and notification-rule filter; merges if `new_name` exists. Returns the blast radius. |
| `delete_account_tag` (write) | `{ tag }` | Removes the tag everywhere (tests/runs are kept). Rules left with no tag are disabled, never widened. |
| `list_account_members` | `{ q?, team_id?, page?, page_size? }` | Members with `accountRoles`, teams and `lastActiveAt` (null = never seen). |
| `set_account_member_role` (write) | `{ member_id, account_role: 'owner'\|'billing_manager'\|'member' }` | Account owners only; the last owner cannot be demoted (409). |
| `remove_account_member` (write) | `{ member_id }` | Removes the person from every team, revokes their access and the API keys they made here. Owners only; never the last owner. |
| `resend_account_invite` (write) | `{ invite_id }` | Re-sends a pending invite and extends expiry 7 days; one resend per minute. |
| `get_billing_contact` | *(none)* | `{ configured, name, email, canManage }` — where receipts and payment notices go. |
| `set_billing_contact` (write) | `{ email, name? }` | Owner or billing manager. Billing mail then goes to `email`. |

## Dashboard layout and build workflow

#### `layout` shape

`layout` is passed straight through to the API's dashboard-layout validation — build it, don't
guess at it. Top level: `{ version: 1, charts: [...] }`; each chart is
`{ id, title, chartType, span, order, queries }` (full example below).

- `chartType`: one of `line | area | bar | pie | donut | radar | treemap | table`.
- `span`: one of `4 | 6 | 12` (a backward-compatible width hint; new layouts may omit it).
- `queries[]`: same query-row shape `query_run_metrics` uses, camelCase (`queryId`, `metricId`,
  `aggregation`, `alignmentPeriod`, `groupBy`, `filters`) — reuse the exact rows you already
  validated via `query_run_metrics` before saving them into a chart.
- `chartOptions.table`: for `chartType: "table"`, optional `columns`/`defaultSort`/`pageSize`.

Two-chart example (line + treemap), from a latency + error dashboard:

```jsonc
{
  "version": 1,
  "charts": [
    { "id": "c1", "title": "p95 by location", "chartType": "line", "span": 6, "order": 0,
      "queries": [{ "queryId": "q1", "metricId": "latency.p95", "aggregation": "p95",
        "alignmentPeriod": "auto", "groupBy": ["location"] }] },
    { "id": "c2", "title": "Errors", "chartType": "treemap", "span": 6, "order": 1,
      "queries": [{ "queryId": "q1", "metricId": "errors.count", "aggregation": "sum",
        "alignmentPeriod": "auto", "groupBy": ["errorCode"] }] }
  ]
}
```

#### Build workflow

1. `get_metrics_catalog` — discover real metric ids, categories, units, dimensions, aggregations.
2. `query_run_metrics` — validate the queries you intend to chart actually return data.
3. `create_dashboard` — same query rows, now embedded per-chart in `layout.charts[].queries`;
   pass `run_id` to get a `consolePath` back.
4. Navigate the user to `consolePath` (or tell them the dashboard is saved and where to find it).
5. To change an existing dashboard, `get_dashboard` → edit the returned `layout` → `update_dashboard`.

An invalid `layout` comes back as a normal tool error (`isError: true`) with the server's
validation message — read it and fix the layout rather than guessing again from scratch.

## Resources as code (export, schemas, applying)

Every resource kind (tests, configs, virtual services, fleets, alert rules, datasets and more) has a file format. `list_resource_kinds` returns the kinds (kind, plural = the `.maxoperf/<plural>/` directory, scope, apply order, schema URL); `get_resource_schema {kind, version?}` returns the published JSON Schema to author or validate a file against.

- `export_resource {kind, id}` returns `{ kind, id, key, placement, filename, yaml, files }` for one resource by its entity id (never a `maxoperf://` URI). Without `id` it exports many (`workspace_id`, `project_id`, `kind` narrow it) as `{ documents, files, skipped, truncated, nextCursor }`. If `truncated` is true the export is incomplete: call again with `cursor = nextCursor` until `truncated` is false, and say so rather than presenting a partial result as complete. `skipped` lists what could not be exported (`{kind, count, reasons:["forbidden"]}` = resources the caller cannot view). A resource you cannot view is a 404; one the format cannot express exactly is `422 EXPORT_UNSUPPORTED_RESOURCE` (relay `details.reasons`). Secret values are never exported, only `valueFrom` bindings. `export_test_as_code` keeps its older behaviour; for the tests it refuses (vario, builder, functional, browser) use `export_resource` with `kind: test`.
- References between resources are `maxoperf://<kind>/<key>` URIs (`maxoperf://<kind>/name:<name>` for a target that is not exported). Apply order follows the kind order from `list_resource_kinds` (lower first), so bind a virtual service, secret or config to a test by reference in the test file.
- Applying: plan and apply run in the `maxoperf` CLI or the GitHub Action (`maxoperf plan`, `maxoperf apply`, `--prune`), never through MCP, because they need the repo files and a client-side engine. Through MCP, create or update with the per-resource tools (`create_test`, `create_config_value`, `upsert_alert_rule`, ...). `list_tests` takes `tag` (repeatable, matches ANY).
- The ownership ledger (`/v1/resources/ledger*`: which repo file owns which resource, plus the sync lock) is engine-internal. `call_platform_api` may read it with GET, but every write (PUT/POST/PATCH/DELETE) is rejected: the engine writes it with its own CLI key, and an agent that could write it could forge or steal prune ownership (the ability to make `--prune` delete a resource).

## CI/CD and GitHub Actions

Tests as code, the GitHub Action, deploy hooks (console-created) and per-run `config_overrides`: `reference/ci-cd.md`.

## Finding docs and API endpoints

Three read-only tools answer "how do I" and "which endpoint" questions; prefer them to guessing.

- `search_docs {query}` — ranked excerpts from the public docs, each with a citable `https://maxoperf.com/docs/...` URL. Excerpts are short.
- `read_doc {url}` — the whole docs page, in order, for a URL from `search_docs` (anchor ignored) or a slug like `tunnels/create-a-tunnel`. Call it when an excerpt does not settle the answer, then cite the page URL.
- `search_endpoints {query?, tag?, limit?}` — ranked, word-based search over the public API. Use plain words ("create a load test", "cancel run"); results are best-first. Then call the endpoint with `call_platform_api`.
