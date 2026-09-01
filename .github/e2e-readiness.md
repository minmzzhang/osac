# E2E readiness gate (OSAC-3370)

Expensive e2e (`e2e-vmaas-full-install`, `e2e-bmaas-full-install`, `e2e-caas-full-install`) **waits** (required `e2e-*-gate` stays pending) until an unlock signal is present. Goal: save runner bandwidth until CodeRabbit is happy, without marking the PR e2e red.

Action, `/e2e-ready`, label cleanup, the **e2e-on-label starter**, and **fork CR replay** live in [osac-test-infra](https://github.com/osac-project/osac-test-infra). This repo keeps thin listeners (`e2e-on-label.yml`, `e2e-on-approval.yml`, `e2e-on-approval-fork.yml`) that `uses` those reusables and pass this repo's caller workflow filenames. `fork-handoff` stays a top-level job here (reusable job names are prefixed; the replay gate matches `fork-handoff` exactly).

`GITHUB_TOKEN` cannot start other workflows from a `labeled` event. `/e2e-ready` therefore applies the label **and** starts e2e from the test-infra handler (`workflow_dispatch` of this repo's `e2e-on-label.yml`, which calls the reusable). A human UI `e2e-ready` label still does not start e2e.

## Signals (any one is enough)

| Signal | Notes |
|--------|--------|
| `coderabbitai[bot]` `APPROVED` on exact HEAD | Primary. **Starts** e2e. Same-repo: `e2e-on-approval` (`pull_request_review`). Fork: that event has a read-only token / no secrets, so `e2e-on-approval` only hands off; `e2e-on-approval-fork.yml` (`workflow_run`, YAML from default branch) verifies APPROVED on exact HEAD and reruns. Blocked while any human still has outstanding `CHANGES_REQUESTED`. Abbreviated SHA does not count. The replay workflow must already be on `main` (this PR cannot replay itself before merge). |
| `lgtm` label | Alternate. **Starts** e2e (`e2e-on-label`). Prow removes the label on new pushes. |
| `e2e-ready` via `/e2e-ready` | Quiet override. Slash command applies the label as `github-actions[bot]` **and** starts e2e (`workflow_dispatch` of `e2e-on-label`; a GITHUB_TOKEN `labeled` event would not). Manual UI labels are rejected. Cleanup removes the label on push. |

**Human `APPROVED` reviews do not unlock expensive e2e.**

`/test` and `/retest` only **rerun** workflows; they do not apply unlock labels.
If readiness is still waiting, a rerun stays pending until a signal is present.

Unlock replay (`e2e-on-label` / `e2e-on-approval`) looks up the original
`pull_request` run by `head_sha`. Do not scan only the newest 100 runs — this
repo exceeds that in a day, so delayed `/lgtm` would miss the head (see
[osac#550](https://github.com/osac-project/osac/pull/550)).
If that run is still queued (no runner yet), skip replay — the same run will
evaluate current labels when it starts ([osac#538](https://github.com/osac-project/osac/pull/538)
timed out after 3 minutes while `changes` waited 48 minutes).

Non-`pull_request` events (schedule, `workflow_dispatch`, `merge_group`) skip the gate.

Fork PRs: readiness is a **cost** gate only. Secrets / cluster e2e still need
`ok-to-test` (or org membership) via `authorize-fork-pr` — same as before.
CodeRabbit APPROVED **does** auto-start fork e2e after this workflow is on
`main` (`workflow_run` replay). `/ok-to-test` is still required for secrets,
not for the cost unlock.

## Author flow

1. Open PR (code change) → cheap `e2e-readiness` succeeds with `ready=false`; required `e2e-*-gate` stays **pending**; no heavy runners.
2. Docs-only PR → `e2e-readiness` skipped; `e2e-*-gate` reports success.
3. When ready: CodeRabbit `APPROVED` on current head, **or** `/lgtm`, **or** `/e2e-ready`.
4. Push more commits → `lgtm` (Prow) and `e2e-ready` (cleanup) drop → gate pending again until CodeRabbit re-approves or a label is re-applied.

## Smoke checklist

- [ ] Open a PR with a code change (not docs-only).
- [ ] Confirm `e2e-readiness` finished (`ready=false`; the job is green because it is only a cheap API check, not the merge gate) and required `e2e-*-gate` is **yellow/pending** (no CR / `lgtm` / `/e2e-ready`).
- [ ] Confirm no EC2 / full-install reusable jobs start while `e2e-*-gate` is pending.
- [ ] CodeRabbit `APPROVED` on exact head → e2e starts (same-repo: `e2e-on-approval`; fork: `e2e-on-approval-fork` after the handoff run completes).
- [ ] Apply `lgtm` → `e2e-on-label` starts e2e.
- [ ] `/e2e-ready` (bot-applied) unlocks **and** starts e2e (`e2e-on-label` run appears). A manual `e2e-ready` label does not.
- [ ] Push a new commit → unlock labels removed → gate pending again.
- [ ] Optional: schedule / `workflow_dispatch` still runs without the label.
- [ ] Docs-only PR: `e2e-*-gate` succeeds (merge not blocked on e2e).
