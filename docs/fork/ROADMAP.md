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
- [ ] Hermes-side: fix the 2 stale test assumptions in
      `test_deft_platform.py` against 0.21.5 internals
      (`_run_on_mcp_loop` gone, journal-path behavior changed). Requires
      Hermes 0.21.5 source — see "Hermes source access" below.

### 3. Hermes 0.21.5 source access — BLOCKING item 2

The scratch workspace on the Hermes host was pruned. To trace the tool-loop
batching rejection and the `_run_on_mcp_loop` replacement, the coding agent
needs Hermes 0.21.5 source. Options (in preference order):

- [ ] Copy the runtime's `gateway/` package (or the whole repo at v0.21.5)
      from the Hermes host into `docs/fork/vendor/hermes/` **without** secrets,
      so the agent can read it across sessions. Do not commit secrets.
- [ ] Or grant fetch access to `github.com/NousResearch/hermes-agent` and
      let the agent read tag v0.21.5 source from upstream.
- [ ] Without either, item 2 is limited to manifest text + adapter test
      stubs; the real 0.21.5 tool-loop behavior stays unverified.

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
  `fork/main` from upstream `master` @ `b567af6`. Moved
  `deft_hermes_report.md` → `docs/fork/deft_hermes_report.md` (untracked →
  tracked, preserving experiment history). Wrote certification prompt fix
  (one tool call per turn, Hermes batching warning) in
  `agent-employees.ts` + regression assertions in
  `agent-certification-stability.test.ts`. Committed `b68151d` (user pushes
  manually — SSH key needs a passphrase). Typecheck + stability suite run
  locally: 7/7 pass on a disposable test DB. Follow-up commit: sanitized the
  deployment-target section (personal stack details removed) and updated this
  session log; the previously recorded personal endpoints/profiles/provider
  were removed from this file.
- **2026-10-04 (session 1, amend):** Roadmap edits re-sanitized after user
  request; also fixed a typo in fork rules ("eps merges" → "merge commits").
