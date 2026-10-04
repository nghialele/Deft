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

### 2. Experimental manifest update — PENDING

`integrations/hermes/integration-manifest.json` still pins
`deft_release_compatibility: "=0.3.0-preview.14"` and
`hermes_compatibility: ">=0.20.5 <0.21.0"` — the compatibility gap that forced
the blind-bind. On a `fork/*` branch (never in place on upstream):

- [ ] Bump `deft_release_compatibility` to `>=0.3.0-preview.14 <0.3.1`
      (fork tracks post-preview.15 master).
- [ ] Bump `hermes_compatibility` to `>=0.20.5 <0.22.0`, clearly labeled
      experimental: add a `"x-fork-experimental": true` marker and a note field.
- [ ] Regenerate bundle + checksums; keep official preview.14/15 assets
      untouched (bundle URL is versioned by tag, so upstream assets are safe
      by construction — verify the fork's bundle URL differs).

**Validator/test surface to update together (they encode the same pins):**

- `scripts/lib/hermes-integration-bundle.mjs` — `validateManifest` L255-259
      (exact release pins), L296 (`testedMinorRange` makes `>=0.20.5 <0.22.0`
      structurally impossible), L299-311 (calendar-tag ref + provenance).
- `apps/api/src/scripts/hermes-employee-release-gate.ts` L383-387 (same
      invariants; `probeHermesRuntime` needs a clean checkout matching pins).
- `scripts/generate-release-manifest.mjs` L92.
- Tests: `scripts/hermes-integration-bundle.test.mjs` (L141-152),
      `scripts/release-workflow.test.mjs` (L234, L278-279),
      `apps/api/test/hermes-employee-release-gate-contract.test.ts`,
      `apps/api/test/hermes-native-onboarding.test.ts` (L17 asserts
      integration_version 0.5.1).
- `hermesIntegrationBundleUrl()` in `apps/api/src/routes/agent-employees.ts`
      L600-603 defaults to the upstream GitHub releases URL; fork must
      override via `DEFT_HERMES_BUNDLE_URL` or change the default — the
      runtime_setup step text tells the operator that URL, so they change
      together.
- Release pipeline stays gated by `release/release-scope.json`
      (`scope: "core"`); keep fork releases core-scope.

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
