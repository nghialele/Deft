# Deft Fork — Hermes Integration Roadmap & Session Handoff

> **Status: ACTIVE.** This file is the durable handoff between agent sessions.
> Any agent picking up work on this fork must read this file FIRST, update it
> after every work session, and commit changes to `fork/*` branches.
> Keep this file truthful: record what was *actually* run, not what was planned.

## Fork identity

- **Upstream:** `git@github.com:Maneek21/Deft.git` (remote `upstream`, branch `master`, AGPL-3.0-only)
- **Fork:** `git@github.com:nghialele/Deft.git` (remote `origin`)
- **Working branch:** `fork/main` (forked from upstream `master` at `b567af6`)
- **Fork rule:** never force-push or rewrite `origin/master`; `origin/master` stays
  a clean fast-forward mirror of `upstream/master`. All fork work lives on
  `fork/*` branches and merges into `fork/main` only via fast-forward or clean
  merge commits with reviewed diffs.

## Deployment target

- Deft **v0.3.0-preview.15** self-hosted (Docker Compose), on a host separate
  from the Hermes runtime. Endpoint/employee details are deployment-specific
  and intentionally not recorded here — see the operator's private notes.
- Hermes **0.21.5** (NousResearch/hermes-agent). Keep test profiles isolated
  from any personal/default profile.
- Compat target pair for all integration work: Deft preview.15 + Hermes 0.21.5.

## Compat goal (fork policy)

| Layer | Version policy | Status |
|---|---|---|
| Hermes runtime | Support `>=0.20.5 <0.22.0` (upstream minimum pin through latest stable 0.21.x) | **Aspirational — only 0.21.5 is tested.** 0.20.5 path untested since the API drift. Do not claim support for a version that has not been re-run. |
| Deft | Fork of upstream master (post preview.15) | Tracking upstream. |

The fork's integration manifest change (`>=0.21.0 <0.22.0`) is **experimental,
uncertified** until the full 11-stage certification + restart/recovery tests
pass on the actual deployment. `official_certification: false` until the Deft
maintainer supplies an official gate for this pair.

## Workstream status

### 1. Certification prompt fix (record_decision batching) — IN PROGRESS

**Status note (session 3):** the test-drift work below supersedes the premise
of the historical "batched local tool calls rejected" framing. Hermes 0.21.x
supports parallel MCP tool calls; the one-tool-per-turn prompt instruction is
a conservative mitigation, not a proven root-cause fix. Keep it, but treat a
Deft-side investigation (does the server tolerate the certification tool-call
sequence 0.21.5 actually produces?) as still open.

**Problem:** Hermes 0.21.5 rejects batched/parallel local tool calls in a single
operation. The certification prompt told the model to "Call these tools now:
…" with 8 tools listed, inviting a batched response. The `record_decision`
nonce never gets recorded, so the `cooperative_nonce_seen` stage stays pending
and certification can never pass. This is a tool-call orchestration issue, NOT
an auth/credential problem.

**Fix (this branch):** `buildCertificationPrompt` and
`certificationInstructions` in `apps/api/src/routes/agent-employees.ts` now
instruct one tool call per assistant turn, with Hermes-specific wording about
batched local tool calls. `runtime_setup.troubleshooting` gained a matching
line. Regression assertions added in
`apps/api/test/agent-certification-stability.test.ts`.

**Validation (local):** Typecheck clean (`tsc --noEmit`), and
`agent-certification-stability.test.ts` passed **7/7** against a disposable
pgvector test container (thrown away afterward; no real database touched).

**Validation (deployment):** NOT YET RUN. Deploy to the preview.15 instance
and re-run certification end-to-end: start certification, paste prompt into
`hermes chat --cli --max-turns 20`, restart the Hermes gateway once for the
restart-proof stages.

**Open sub-items:**
- [ ] Run certification end-to-end on the deployment with the updated prompt.
- [ ] Watch for: nonce recorded via `record_decision`, restart stage green,
      exactly one delivery + one reply (no duplicate channel events).

### 2. Experimental manifest update — RESOLVED (pending deploy validation)

Widened to `>=0.20.5 <0.22.0` on 2026-10-04, marked experimental:

- [x] `deft_release` / `deft_release_compatibility` → `0.3.0-preview.15`
      (validator requires exact match with `package.json`).
- [x] `hermes_compatibility` → `>=0.20.5 <0.22.0` with
      `"x-fork-experimental": true` + note field (0.20.x is carried as the
      upstream-declared floor; only 0.21.5 is suite-verified).
- [x] `hermes_tested` → 0.21.5, `refs/tags/v2026.9.24`, commit
      `f97608f178d1ffeca59860195ab7da295f7c8e5f`.
- [x] `testedMinorRange` strictness in `hermes-integration-bundle.mjs` now
      allows a wider range **only** for `x-fork-experimental` manifests, and
      still requires the range to cover `hermes_tested.version`
      (`rangeCovers` helper). Non-experimental manifests keep the strict
      single-minor derivation unchanged.
- [x] Audit doc gained a dated 2026-10-04 fork revalidation section whose
      provenance lines match the new manifest pin (validator does substring
      matching against it).
- [x] `scripts/hermes-integration-bundle.test.mjs` assertions updated to
      the new pins.
- [x] Setup-step troubleshooting now documents `DEFT_HERMES_BUNDLE_URL`
      for fork operators (default URL stays upstream).
- [x] `HERMES_INTEGRATION_VERSION` left at `0.5.1` (no semantic contract
      change; consumers key on manifest fields, not this constant).

**Validation (local, all green):** bundle build+verify (deterministic,
content sha256 `5aa85f40…`), bundle tests 8/8, release-workflow tests 16/16,
gate-contract tests 5/5, onboarding tests 2/2, certification-stability 7/7
(disposable pgvector container), python adapter suite 27/27 against the
0.21.5 tag worktree (see audit doc), `tsc --noEmit` clean.

**Not done (deliberately):** regenerating/publishing a fork bundle tarball
(upstream assets untouched; fork publish step is a user decision) and
end-to-end certification on the deployment.

### 3. Hermes 0.21.5 source access — RESOLVED

A local read-only checkout of NousResearch/hermes-agent is available at a
sibling workspace root (see session log for the exact facts about checkout
version vs the 0.21.5 release tag). Version-sensitive claims must be checked
against the release tag, not the checkout HEAD.

### 4. UI/UX + core feature changes — QUEUED

After the integration is stable, not before. Keep feature work off the same
commits as integration fixes so reverts stay surgical.

## Pitfalls (do not re-learn these)

- **A passing handshake proves almost nothing.** Agent Channel compatibility is
  protocol-string only (`deft.agent_channel.v2` + capabilities) — no Hermes
  version check. "Chat works" ≠ compatible. The certification stages are the
  real probe.
- **`record_decision` failure is orchestration, not auth.** Don't chase tokens.
- **Stale adapter journal.** If the adapter errors "state belongs to a different
  Deft endpoint, employee, or owner profile": move the old journal aside as a
  backup and restart the gateway. Never silently reuse state across
  endpoint/employee/profile.
- **Scratch on the Hermes host gets pruned.** Commit durable work into this
  repo (this file + branch) immediately. The previous experiment was lost this way.
- **Don't replay old channel events or run parallel adapters** to clear pending
  certification state — inspect Deft challenge state + adapter journal first.
- **Don't touch the personal Hermes `default` profile or production Deft
  workspace data** when testing; no volume deletes, no key rotation without
  explicit approval.
- **Repo-root `AGENTS.md` is stale** (describes a Next 14 / better-auth /
  OpenClaw-era architecture that no longer matches source). Trust `CLAUDE.md`
  and the source for architecture facts.

## Git workflow (agent-managed)

```bash
git checkout fork/main          # integration branch for fork work
# ...work, commit...
git push origin fork/main       # publish fork work
# upstream sync (user-triggered, agent prepares but does not push master):
git fetch upstream
git merge upstream/master       # on fork/main, resolve, push origin fork/main
```

- Commits: imperative subject ≤50 chars, capitalized, no trailing punctuation;
  body wrapped at 72 chars, only when useful.
- Never commit secrets/tokens. `docs/fork/vendor/` must be pre-scanned.
- Untracked files that are user-local notes (like the report) belong under
  `docs/fork/` and get committed; scratch output does not.

## Session log

- **2026-10-04 (session 1):** Read repo, report, certification code. Created
  `fork/main` from upstream `master` @ `b567af6`. Wrote certification prompt fix
  (one tool call per turn, Hermes batching warning) in
  `agent-employees.ts` + regression assertions in
  `agent-certification-stability.test.ts`. Committed `b68151d` (user pushes
  manually — SSH key needs a passphrase). Typecheck + stability suite run
  locally: 7/7 pass on a disposable test DB.
- **2026-10-04 (session 1, amend):** Removed the historical experiment report
  from the repo entirely (personal stack details; not needed). This roadmap
  is now the single durable handoff. Also fixed a typo in fork rules
  ("eps merges" → "merge commits").
- **2026-10-04 (session 2):** Reproduced the upstream suite against a local
  hermes-agent checkout. Created the local test runner
  `.fork-bin/run-deft-platform-tests.sh` (untracked by design). Established
  the checkout facts: tag `v2026.9.24` is the Hermes **0.21.5** release;
  the checkout is `main` newer than that tag. Suite run reproduced 27 tests
  with 2 errors — both test-side staleness, not adapter bugs (see session 3).
- **2026-10-04 (session 3):** Diagnosed and fixed both suite failures;
  committed `44fdd82` "Fix Hermes 0.21 test drift in deft-platform suite":
  1. `_run_on_mcp_loop` moved to `tools/mcp_tool_loop.py` and
     `_make_tool_handler` to `tools/mcp_tool_handlers.py` (Hermes 0.21 module
     decomposition); test now patches the modules where production reads
     them, matching Hermes's own test seams.
  2. New Hermes 0.21 gateway runtime-status writer
     (`gateway_state.json`, process-global daemon thread) raced
     `TemporaryDirectory` cleanup under patched `HERMES_HOME`, flaking the
     named-profile journal test intermittently (reproduced ~50% of runs).
     Neutralized by patching `gateway.status.publish_runtime_status` in
     `setUp`. Root cause verified by stack tracing the writer.
  Suite: 27/27 green, stable across 11 consecutive full runs. Adapter code
  unchanged — plugin surface is stable in 0.21.x. Both fixes are clean
  upstream-PR candidates (upstream will hit the same drift re-certifying
  against 0.21). Roadmap item 3 resolved; item 2 (manifest + validators)
  is now unblocked and its full file list is recorded above.
  ```
  .fork-bin/run-deft-platform-tests.sh
  ```
  is the canonical way to re-verify compatibility against any hermes-agent
  checkout (`HERMES=/path` overrides; defaults to the sibling workspace).
- **2026-10-04 (session 5):** Implemented ROADMAP item 2 (experimental
  manifest update). Manifest: `deft_release` → 0.3.0-preview.15,
  `hermes_compatibility` → `>=0.20.5 <0.22.0`, `hermes_tested` → 0.21.5 @
  `refs/tags/v2026.9.24` (`f97608f…`), new `x-fork-experimental: true` + note.
  Validator: `x-fork-experimental` manifests may widen the range past
  `testedMinorRange` but must still cover the tested version (`rangeCovers`).
  Audit doc: 2026-10-04 fork revalidation section (0.21.5 worktree, 27/27).
  Tests: bundle 8/8, release-workflow 16/16, gate-contract 5/5, onboarding
  2/2, certification-stability 7/7, python suite 27/27, tsc clean. Also:
  troubleshooting line documents `DEFT_HERMES_BUNDLE_URL` for fork operators
  (default bundle URL intentionally stays upstream). Range choice:
  `>=0.20.5 <0.22.0` with an honest "only 0.21.5 tested" note — the honest
  floor is 0.21.0 but upstream's own pin declares 0.20.5, so the fork keeps
  the upstream floor while flagging 0.20.x as untested via the note field.
  **Test container note:** `deft-fork-test-postgres` (port 5434) uses
  `POSTGRES_PASSWORD=forktest`, db `deft_test` — not the `postgres:postgres`
  creds used in earlier sessions (container was recreated).
