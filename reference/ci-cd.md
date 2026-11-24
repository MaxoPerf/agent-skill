# CI/CD and GitHub Actions

Load this on demand when the user wants MaxoPerf tests to run from CI (GitHub Actions, GitLab, Jenkins,
any pipeline) or after a deploy (Vercel, Netlify, a generic webhook). Nothing here is needed for a normal
build, upload and run loop.

## Pick the path

| Goal | Use |
|------|-----|
| Tests live in the repo, run on push or pull request | GitHub Action `MaxoPerf/github-action@v1` |
| Same, other CI | `npx @maxoperf/cli push` or `npx @maxoperf/cli run` |
| Run after a deploy lands (preview/staging URL) | a **deploy hook** (a URL that CI or Vercel/Netlify calls) |

## Test file v1 (`.maxoperf/tests/*.yml`)

One YAML file per test, committed with its script and data files. `push` syncs every file to MaxoPerf
(creating or updating the test); `run` pushes, then starts the runs, waits and gates on the failure
criteria. Start from an existing test: `export_test_as_code { test_id }` returns `{ filename, yaml, files }`
where each file has the repository path to commit it at. Secret values are never exported, only binding
names. VarioTests, builder tests, browser tests and tests with aliased or inline bindings are refused with
`422 EXPORT_UNSUPPORTED_TEST` and `details.reasons`: relay the reasons, do not hand-write around them. `export_resource { kind: "test", id }` exports the kinds it refuses (see "Resources as code" in `reference/platform-features.md`).

## GitHub Action

```yaml
- uses: MaxoPerf/github-action@v1
  with:
    api-key: ${{ secrets.MAXOPERF_API_KEY }}   # an mpak_ key
    account-id: ${{ vars.MAXOPERF_ACCOUNT_ID }}
    config-overrides: |
      TARGET_URL=https://shop-pr-${{ github.event.number }}.preview.example.com
```

Inputs: `command` (`run`|`push`), `tests` (glob), `project-id`, `config-overrides` (`KEY=VALUE` lines), `wait`,
`fail-on` (`criteria`|`never`), `timeout-minutes`, `pr-comment`. Outputs: `results`, `run-id`, `status`,
`report-url`, `passed`.

## Per-run config overrides

`start_run { test_id, config_overrides: { KEY: "value" } }` (and `POST /v1/runs` `configOverrides`) sets
config values for that one run only, overriding the workspace Config values of the same name. Rejected:
a key that collides with a secret (`CONFIG_OVERRIDE_SECRET_COLLISION`), a reserved key, and engines that do
not support overrides (`CONFIG_OVERRIDES_UNSUPPORTED`).

## Deploy hooks

A deploy hook is a secret URL `https://app.maxoperf.com/v1/hooks/deploy/mpdh_...` that starts one run per
test carrying the hook's tag. The deploy URL is passed as a config override (the hook's `urlConfigKey`).
Optional signature check (a workspace secret) for GitHub, Vercel, Netlify or the generic
`X-Maxoperf-Signature: sha256=<hex>`.

**Console only.** Hooks are created, rotated and edited in the console (Workspace settings, Deploy hooks)
because the URL is a capability and is shown once. You can only `list_deploy_hooks { workspace_id }`
(no URL returned) and `delete_deploy_hook { workspace_id, hook_id }` (destructive: confirm first;
every CI job calling it then fails). `call_platform_api` refuses hook create, rotate and update. Tell the
user to create the hook in the console and paste the URL into their CI secret store; never ask them to
paste the URL into the chat.

## Run provenance and filtering

Runs started by CI carry `ciContext` (provider, repository, branch, sha, PR number, run URL).
`get_run_status` returns it; `list_runs { branch }` filters to one git branch.

## Idempotency

`POST /v1/runs` enforces `Idempotency-Key` for 24 hours: the same key replays the first response
(`200`, `Idempotent-Replayed: true`) and starts nothing new. CI clients send a per-attempt key. The MCP
`start_run` dedupes a double-fire within the same minute only; a later start is a new run.
