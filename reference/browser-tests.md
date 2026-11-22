# Browser tests — every supported shape, and the traps

Grounded in TASK-1218's live browser matrix (`e2e/browser-matrix/`, 7 forms × chrome/edge/firefox
× happy/fail = 42 cells). Every claim here was confirmed on a real run against a real browser,
not inferred from source. Where something is NOT supported, that is stated plainly.

## The forms, and when to use each

| Form | What the user uploads | Pick it when |
| --- | --- | --- |
| **Playwright spec** | `.ts`/`.js` importing `@playwright/test`, with `test(...)` blocks | New browser test, no existing asset. **Default.** |
| **Playwright library script** | `.ts`/`.js` importing `playwright`, driving `chromium.launch()` itself | The user already has a plain Playwright automation script |
| **Playwright IIFE script** | Library script whose work is inside `(async () => { … })()` | Same as above — extremely common real-world shape |
| **Selenium test script** | `.py` with a `def test_*()` function | The user has a Selenium script already written as a test |
| **Selenium bare script** | `.py` with top-level code or `if __name__ == '__main__':` | The user has an ordinary Selenium automation script |
| **Interactive (scriptless)** | No file at all — `authoringMode: 'interactive'` + `browserSteps` | The user has no code and wants to describe a journey |
| **Taurus YAML** | `entrypoint.yaml` referencing a script | Multiple scenarios/executions in one test |

**All of these are supported. Upload the user's script as-is** — never ask them to rewrite it into
a different shape. The platform detects the shape from content and wraps it when needed:

- A Playwright **library** script is wrapped in a generated one-test spec so `playwright test` can
  run it (a library script declares no tests; unwrapped it fails with `No tests found`).
- A Selenium **bare** script is wrapped in a generated `test_maxoperf_entry.py` using
  `runpy.run_path(..., run_name='__main__')` — the `run_name` is what makes an
  `if __name__ == '__main__':` guard actually fire (unwrapped: `Nothing to test`).

## Playwright: which file runs, and the user's own config

- **The uploaded script runs, whatever it is named.** `checkout.js`, `flow.ts` and `a.spec.ts` all
  work — the platform generates the `playwright.config.ts` that points at exactly that file, for a
  bare upload AND for a Taurus YAML `executor: playwright` scenario `script:`. Never rename a file
  to `*.spec.ts` to "make it run".
- **Keep the user's `playwright.config.*` — upload it as an asset, never as the entrypoint**
  (a config as entrypoint fails validation with `playwright_config_as_entrypoint`). It is wrapped,
  not replaced: their `use` (baseURL, viewport, timeouts…), `timeout`, `expect`, `retries` and
  reporters (except html/json) apply; the platform forces `testDir`/`testMatch`/`testIgnore` (only the
  entrypoint runs) and the browser `channel`/launch args. A CommonJS `.js` config next to an ESM
  script is handled for you.
- **Config `projects` → the test's browsers**: chromium / Desktop Chrome → `chrome`, msedge /
  Desktop Edge → `edge`, firefox → `firefox`. **webkit, mobile devices and other channels are not
  supported** (warning
  `playwright_project_unsupported`; it will not run). The synced set shows on `get_test` as
  `browsers` with `browsersSource: "playwright_config"`; a user edit (`browsersSource: "user"`)
  wins over a later re-upload of the same config set.
- **Validation is static and catches "no tests" before a run**: `playwright_no_runnable_tests`
  (no `test(...)`/`describe(...)` and no library launch), `playwright_script_not_found` (the YAML
  scenario's `script` was not uploaded), `playwright_script_not_supported` (not `.js/.ts/.mjs/
  .cjs/.mts/.cts`). Warnings never block a run but explain one — read `validation.warnings` on
  `get_test` and relay them.
- **`waitUntil: 'networkidle'` / `waitForLoadState('networkidle')`** (warning `playwright_networkidle`, with file + line) may never
  settle on pages with analytics, chat widgets or bot protection → navigation timeouts. Suggest
  `waitUntil: 'load'` plus a locator wait (`await expect(page.getByRole(...)).toBeVisible()`).
- **Runner size follows browsers per runner** on managed clouds: ≤3 → `s`, ≤6 → `m`, more → `l`
  (the larger of that and the configured size). Each browser is a real process; overloaded runners
  now fail as `runner_overloaded` — suggest fewer browsers (VUs) per runner or a larger size.

## Browser selection

`browsers` on the test is an ordered list of `chrome` | `edge` | `firefox` (`browser` is always
`browsers[0]`, kept for old clients). All three are live-validated across every scripted form.

- **`chrome` means real vendor Google Chrome**, not Chromium. A script calling `chromium.launch()`
  gets vendor Chrome automatically (the runner injects `channel: 'chrome'`); `edge` maps to
  `channel: 'msedge'`. Firefox is a separate engine, not a channel.
- For a **Selenium** script the browser the script itself constructs wins (`webdriver.Chrome()` vs
  `webdriver.Firefox()`); the platform pins the matching driver binary but cannot launch a
  different browser than the code asks for. Say so if the user's `browser` selection and their
  script disagree.
- **Multi-browser in ONE run**: set `browsers: ['chrome','firefox']` (`create_test`, or
  `PATCH /v1/tests/:id` `{"browsers": [...]}` via `call_platform_api` — send a JSON array, never a
  string). Virtual users are split evenly (any remainder goes to the first browsers, in order), each browser gets its
  own runner group, and VU-hours stay the same. Errors: `BROWSERS_MISMATCH` (`browser` ≠
  `browsers[0]`), `BROWSERS_EXCEED_VUS` (each browser needs ≥1 VU), `BROWSER_NOT_STEERABLE` on save
  (`PATCH` once the script is uploaded) and at run start (a Selenium test that picks its own
  browser — a bare script, or a Taurus YAML scenario with a `script:` or its own
  `browser`/`capabilities` — also a `browser_not_steerable` validation warning; select one browser,
  or rebuild the flow as a Taurus `requests:` scenario or in the scriptless browser builder).
  **Check `browsersSteerable` on `get_test` before sending more than one browser** — `false`
  means `browsers` takes a single entry.
- Chromium-family launches without a `channel` (`chromium.launch()`, `launchPersistentContext`,
  `launchServer`) run on vendor Chrome (Edge on an `edge` run); Playwright's bundled Chromium is
  not installed on runners.

## Why a browser run failed — read it, don't guess

`get_run_status` on a `failed`/`cancelled` run returns `failure { cause, title, summary, detail }`
and `runnerFailures[]` (`runnerId`, `browser`, `cause`, `summary`, `detail`); `get_run_errors`
attaches the same when the run has no error rows. `detail` is the engine's own error text
(redacted, ≤1024 chars) — quote it. Named causes: `playwright_no_tests_found`, `script_load_error`,
`script_dependency_missing`, `browser_launch_failed`, `browser_navigation_timeout`,
`target_unreachable`, `script_error_all_iterations`, `engine_no_results`, `taurus_config_error`,
`runner_overloaded` (fewer browsers per runner / larger size). They cover Selenium too. A browser run where no iteration passed is `failed` with one of these causes even if the engine exited 0 (a Playwright run cut by its duration whose finished tests all failed also ends `failed`; one cut while a test is still running writes no result and still ends `passed`). Correlate
further only when the cause is generic.

Playwright captures every failing response (status ≥400, or a `requestfailed` network error) and
Selenium only while recording (Chrome: full body; Edge/Firefox: metadata only, samples written at
session end) — read them with `get_run_error_bodies { signature }`.

## What a healthy browser run produces

Assert these when validating; a run missing any of them is not healthy even if it says `passed`:

| Signal | Notes |
| --- | --- |
| **samples** | KPI rows. Zero samples means nothing was measured, regardless of verdict. |
| **video** | Live HLS. Requires a *headful* browser — see traps. |
| **browser registered** | A `browserId` must resolve, and its name must match the selected browser. Video with no browser behind it means no per-browser views. |
| **steps** | REAL steps. `Before Hooks`/`After Hooks` alone is an empty Steps view. |
| **HAR** | In-browser requests, with usable labels. |
| **console** | Only present if the page actually logs — absence is not automatically a bug. |
| **transactions** | Transaction labels back the Transactions view and every label filter. |

## Traps — each of these shipped as a real bug

**`headless: true` is overridden when the run records.** Video is an ffmpeg grab of the Xvfb
framebuffer, so a headless browser paints nothing → no video → no browser registers → the
per-browser Steps/HAR/console/transaction views are all empty. Do not "fix" a user's script by
removing `headless: true`; the runner handles it.

**An un-awaited top-level IIFE.** `(async () => { … })();` is not awaited by the module, so a naive
wrapper's `await import(...)` resolves the instant the IIFE *starts* and the browser is torn down
mid-journey. The platform waits for the script's browsers to close. Do not tell the user to
restructure their script.

**Top-level `await` needs ESM.** A library script using `const b = await chromium.launch()` at
module scope only parses as a module; the generated bundle ships `package.json {"type":"module"}`.
A CommonJS `.js` using `require()` should be named `.cjs`.

**A bare `webdriver.Chrome()` needs container flags.** The runner injects `--no-sandbox` and
`--disable-dev-shm-usage` itself. User scripts do not need them; harmless if present.

**Never spell out the word `apiritif` inside a Selenium `.py`** — not even in a comment. Engine
classification content-sniffs the whole file for it and will silently reclassify the test.

**A criterion's `label` is a scope filter, not a display name.** Use `''` to match all labels; a
human-readable label matches zero KPI rows and the criterion can never breach.

**`apiritif` is not browser-capable.** Its executor never translates the `actions:` DSL into
browser-driving code, and the API rejects `browser` for that engine. Use `selenium` for a
steps-DSL browser test.

**`.side` (Selenium IDE) and TypeScript *Selenium* scripts do not run.** bzt rejects `.side`
outright and routes `.ts` to Playwright's runner. Tell the user to export a Selenium **Python**
script, or use the interactive builder. Do not promise these.

## Failure criteria for browser tests

- Plan tiers are **per engine class**: browser engines get a single-digit VUs-per-runner ceiling
  (typically 3–10), protocol engines hundreds to thousands. Enforcement follows the **detected**
  engine, so a browser test always gets the browser tier.
- A criterion evaluated server-side (`evaluator: global-backend`) is ignored under
  `verdictMode: 'per_runner'`. If the test carries only backend-evaluated criteria, its settings
  must use `either`/`combined`, or the gates are inert — the run reports `passed` at 100% error
  rate. Tests created through the API get this right by default.
- Evaluate cumulatively rather than on a narrow rolling window, or a sparse browser journey never
  trips the criterion.
- A failure buried at the end of one long transaction cannot be observed until that transaction
  closes. Prefer several short transactions over one long one.

## Interactive (scriptless) browser tests

`authoringMode: 'interactive'` + `interactiveKind: 'browser'` + `browserSteps`. No file upload.
Use it when the user has no code. It produces the same live signals as a scripted run.

**Page-contains assertion** (no locator; works in every browser-test authoring path):

```jsonc
{ "action": "assertPageContains", "text": "Order Confirmed", "match": "contains", "ignoreCase": false, "negate": false }
{ "action": "assertPageContains", "text": "Order #\\d+", "match": "regex", "ignoreCase": true }
{ "action": "assertPageContains", "text": "Fatal error", "match": "contains", "negate": true }   // fails when the text IS present
```

It tests the visible text (`innerText`) of the CURRENT frame at run time (after `switchFrame`; the TOP
frame at compile time; child-frame text is never merged). `contains` collapses whitespace; `regex` sees
the raw text (use `[\s\S]` to span lines). Max 500 characters; invalid patterns are rejected at save, and
a pattern that hangs the page can still stall a run, so keep patterns simple. A miss FAILS the step after
waiting up to ~5 s for the text to appear; use an explicit `waitFor`/`pauseFor` for longer waits.

## Dependency references (interactive `urlBase` / `go` / `openWindow`)

A `go` step's `url` and an `openWindow` step's `target` (also the interactive HTTP builder's
`urlBase` — see `reference/platform-features.md`'s "Test data in scripts" for the analogous CSV
path) can carry a **live dependency reference** instead of a literal string, so the target tracks a
tunnel or virtual service instead of a URL frozen at build time:

```jsonc
{ "kind": "dependency", "dependencyKind": "virtual_service" /* or "tunnel" */, "entityId": "vs-0123456789", "envVarName": "PETSTORE" }
```

It renders to `${MAXOPERF_RUNTIME__MAXOPERF_VS_PETSTORE_URL}` (tunnel: `MAXOPERF_TUNNEL_<name>_URL`)
at run time — the same runtime-secrets placeholder form secret references already use, so no runner
change is needed. To reach a path under it, add `"path": "/checkout"` to the browser step value
(appended after the URL); in the HTTP builder the request's own `url` is appended to `urlBase`.
Valid ONLY on `go.url` / `openWindow.target` (and the HTTP builder's `urlBase`) —
any other step/field rejects it. Use a **tunnel** when the target is running locally or not yet
deployed, a **virtual service** when it's a mock standing in for a flaky/costly real dependency (see
`reference/platform-features.md`'s decision table) — build the reference with `bind_test_dependency`
first (Dependencies, `reference/mcp-tools.md`) so the binding exists, then reference its
`entity_id`/`env_var_name` here; saving the test auto-creates/reconciles an `origin: 'inline'`
binding for you either way, but the console's own picker always binds first.

## Validating a browser test you just built

Run it, then confirm samples + video + a correctly-named browser + real steps + HAR + transactions.
If steps show only hooks, or video exists with no browser behind it, the run is not healthy — say
so rather than reporting success.
