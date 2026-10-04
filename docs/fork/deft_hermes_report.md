---
title: "Deft ↔ Hermes Integration — Experimental Coding-Agent Handoff"
tags:
  - "deft"
  - "hermes"
  - "integration"
  - "experimental"
  - "coding-agent"
  - "handoff"
folder: "reports"
favorite: false
created: "2026-10-04 11:51:22"
updated: "2026-10-04 11:51:22"
---

# Deft ↔ Hermes Integration — Coding-Agent Handoff

**Purpose:** Give the coding agent working locally on the Deft codebase a factual history of the Hermes integration attempts, the experimental/custom builds, what worked, what failed, and a safe next step. This is an engineering handoff, not a claim of official compatibility or certification.

## Executive summary

We investigated connecting Hermes Agent to Deft as a governed Agent Employee. The deployed target was Deft **v0.3.0-preview.15** with Hermes **v0.21.5**. Deft preview.15 was a core release; its GitHub release did not provide the expected Hermes integration archive. The available preview.14 bundle declares compatibility with Deft **=0.3.0-preview.14** and Hermes **>=0.20.5 <0.21.0**, so it does not officially support the target pair.

We nevertheless ran an isolated, explicitly experimental compatibility effort. The adapter loaded in Hermes 0.21.5; employee-scoped MCP discovery and basic live Agent Channel behavior worked. A readiness probe passed. Full certification did **not** pass cleanly: a `record_decision` step failed when Hermes attempted to batch local tool calls, and recovery testing remains incomplete. Do not label the target pair compatible/certified based on these partial successes.

## Target system and integration design

- Deft: **v0.3.0-preview.15**.
- Hermes: **v0.21.5**.
- Integration model: Deft's native `deft-platform` Agent Channel adapter plus employee-scoped HTTP MCP server named `deft`; common plugins included `deft-employee` and `deft-memory` where used.
- Test employee used in experiments: `hermes-test`; `lele` was also discussed/used in setup planning. Confirm which employee/profile still exist before resuming; do not reuse credentials or assume old runtime state is current.
- Experimental Hermes profile(s) discussed/created: `deft-exp` and `deft-lele`. Keep Deft testing isolated from the personal `default` profile. Do not run the legacy Agent Channel bridge alongside the native adapter for the same employee.
- Provider/model in the working experimental setup: BazaarLink with **gpt-oss-20b**. Other model choices were discussed in plans; verify actual configuration rather than assuming.
- The preview.15 MCP endpoint used in the plan was `https://hq.nghia.im/api/mcp/v1`. Treat this as historical context and check the current employee setup page for the deployed endpoint before testing.

## What was tried

### 1. Official artifact and compatibility checks

- Checked the Deft preview.15 GitHub release for the Hermes integration archive. The expected `deft-hermes-integration-0.3.0-preview.15.tar.gz` URL returned **HTTP 404**; release notes described a core release without a newly certified Hermes bundle.
- Downloaded/inspected the preview.14 integration archive in scratch, without installing it into the production/default profile. It contained `deft-platform`, `deft-employee`, `deft-memory`, an example config, and a manifest.
- The preview.14 manifest pinned Deft `=0.3.0-preview.14` and Hermes `>=0.20.5 <0.21.0`; its published gate was for Deft preview.14 + Hermes 0.20.5. That gate cannot be generalized to preview.15 + Hermes 0.21.5.
- Preview.14 and preview.15 release assets/manifests were therefore treated as an explicit compatibility gap, not papered over by changing version metadata.

### 2. Deft deployment and local build work

- A Deft Compose workflow was used to build the application and one-shot utility images, then bring up the stack:
  ```bash
  docker compose build deft init doctor smoke
  docker compose up -d
  ```
- Deployment work included PostgreSQL with pgvector and Deft initialization/health checks. Troubleshooting also covered duplicate Compose port mappings and moving port settings into `.env`; the Deft host/deployment is distinct from the Hermes host. Before changing the live deployment, inspect the current Compose files, effective merged config, containers, volumes, and port ownership. Do not destroy volumes as cleanup without explicit approval and a verified backup.
- This Compose build/deployment activity is distinct from building the experimental Hermes integration bundle. Do not assume the current local Deft checkout or image corresponds to the old test state—verify source revision, image tag/digest, and working tree.

### 3. Experimental integration source/custom build

- Compatibility work was kept on experimental source branches/tags rather than rewriting upstream release metadata. Recorded names include `exp/v0.3.0-preview.15-exp` / `v0.3.0-preview.15-exp`; a local experiment also used branch `exp/preview.15-hermes` and tag `v0.3.0-preview.15` in a scratch Deft source workspace.
- A documentation change was recorded at commit **`267bace`** on the experimental branch, with notes at `deft/docs/hermes-preview15-exp.md` in the scratch experiment workspace. The working tree/path may not exist on the coding agent's machine; locate the current checkout and verify branch/commit/tag before relying on this history.
- Experimental intent: preserve preview.14 inputs unchanged, make only real fixes needed for the preview.15/Hermes 0.21.5 pair, and label any locally built package/image **experimental and uncertified**. Do not claim an upstream release, signed bundle, or official certification unless verified in the repository/release metadata.
- A preview.15 compatibility test run against Hermes 0.21.5 was recorded as **48 passed, 1 failed, 1 errored**. The reported failures involved stale test assumptions (`_run_on_mcp_loop` missing and old journal-path behavior). Treat those as concrete leads to reproduce in the current checkout; they do not establish that all code paths pass.

### 4. Live Agent Employee checks

- The readiness probe for `hermes-test` returned `ready=true`, channel identity confirmed, 48 MCP tools discovered, and no missing required tools.
- Basic end-to-end behavior was observed: adapter loaded, MCP authenticated/discovered tools, Agent Channel delivered messages, employee-bound replies worked, and Deft space mentions/DM behavior were exercised in the experiment.
- A certification run completed several checks successfully: workspace context, task query, module list, `ping_alive`, memory recall/write, and conversation-turn recording.
- The `record_decision` certification step failed because Hermes attempted to batch multiple local tool calls in one operation, which that runtime rejected. This is evidence of a tool-call orchestration issue to reproduce—not proof that Deft auth/credentials are invalid, and not a passed certification.
- A stale adapter transport journal once caused `Deft platform state is invalid: state belongs to a different Deft endpoint, employee, or owner profile`. The old journal was moved aside as a backup and the shared gateway restarted; negotiation then succeeded. This demonstrates why endpoint/employee/profile-scoped state must not be silently reused.
- Earlier runs also encountered duplicate accepted events and a pending certification state. Do not replay old events or start parallel adapters to try to clear them; inspect the current Deft challenge state and adapter journal first.

## Findings for the coding agent

1. **Release packaging/metadata is a real issue.** Preview.15 lacked a published Hermes integration archive, and preview.14's compatibility manifest excludes the deployed Hermes version.
2. **Do not fix this by editing old compatibility claims.** Any code or package changes belong on a clearly experimental branch/tag and must retain provenance, exact target versions, and hashes.
3. **Hermes API drift is plausible.** The adapter uses Hermes internals, and Hermes 0.21.5 may differ from the bundle's tested 0.20.5 runtime. Reproduce failures against the actual 0.21.5 APIs before patching.
4. **Separate layers while debugging:** Deft server/API behavior, integration adapter compatibility, Hermes tool-loop orchestration, packaging/manifest, and gateway environment/credentials. A successful MCP catalog or readiness probe is not end-to-end certification.
5. **`deft-memory` had an environment-boundary issue in an earlier setup:** the systemd gateway did not automatically inherit the profile `.env`, so the plugin reported unavailable while channel/MCP worked. Verify the actual service environment without printing secrets before attributing memory failures to Deft.
6. **Keep the result bounded.** Until end-to-end certification and recovery testing pass, use only isolated employees/test assignments; keep destructive and external writes approval-gated.

## Suggested local investigation sequence

1. Inspect the current Deft repo path, `git status`, remotes, branch, tags, and recent commits. Locate the experimental branch/tag and `docs/hermes-preview15-exp.md` if present. Do not reset, clean, or switch away from uncommitted work.
2. Record exact deployed Deft source commit/image digest and Hermes version/commit. Find the preview.14 archive, manifest, and checksums; preserve the original bytes.
3. Reproduce the recorded test suite on the current tree. Report the exact 48/1/1 outcome only if reproduced; otherwise clearly separate historical notes from current results.
4. Trace and fix the `record_decision` tool batching failure at the Hermes tool-loop boundary. Add a regression test that uses the actual supported tool-call granularity. Do not disguise a runtime failure as a Deft adapter failure.
5. Recheck adapter imports/API assumptions and journal scoping against Hermes 0.21.5. Add tests for changed internals and for endpoint/employee/profile mismatch handling.
6. Validate release packaging: archive contents, plugin manifests, exact Deft/Hermes compatibility range, checksums, source revision, and clean reproducible build. Keep the experimental artifact clearly named `preview.15-exp`; never overwrite official preview.14/preview.15 assets or manifests.
7. Only with authorization and an isolated test employee/profile, run readiness, a fresh nonce challenge, one delivery/reply, employee-scoped MCP calls, and then restart/duplicate/lease/failure tests. Verify each result in Deft before concluding.
8. Produce a final experimental gate report with exact tested versions, commits, artifact digest, commands/tests and results, unresolved defects, and `official_certification: false` unless the Deft maintainer supplies an official gate for this exact target pair.

## Known artifacts/identifiers from the earlier experiment

- Scratch source workspace recorded as `/root/.hermes/cache/scratch/deft-exp/deft` on the Hermes host. This may have been pruned or may not be accessible to the local coding agent.
- Documentation path: `docs/hermes-preview15-exp.md`.
- Documentation commit: `267bace`.
- Experimental branch/tag names: `exp/v0.3.0-preview.15-exp`, `v0.3.0-preview.15-exp`; another local scratch branch/tag was recorded as `exp/preview.15-hermes`, `v0.3.0-preview.15`.
- Compatibility test result recorded: 48 passed, 1 failed, 1 errored (old Hermes API/journal-path assumptions cited; reproduce before treating as current).
- Preview.15 release bundle URL returned 404; preview.14 bundle was not a release-matched artifact for the target pair.

## Safety and reporting constraints

- This note contains no employee tokens or secrets. Never paste secrets into chat, issue reports, commits, or test logs.
- Do not touch the personal Hermes `default` profile or production Deft workspace as an experiment.
- Do not delete Deft data, reset a database, remove volumes, or rotate Deft signing/encryption keys without an explicit, informed approval and the Deft-specific recovery procedure.
- Preserve upstream tags/manifests and the original preview.14 archive. Keep experimental changes auditable and reversible.
- Report what was actually run in the current checkout separately from this historical handoff. Do not claim that a build, certification, recovery test, or deployment succeeded unless its output has been verified.
