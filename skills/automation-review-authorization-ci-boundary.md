---
name: automation-review-authorization-ci-boundary
description: "Keep automation-loop source-review authorization independent of CI/CD. Use when: (1) a strict PR review is mistakenly implemented as a required CI check, (2) an agent loop cannot observe or control the external workflow that is supposed to authorize it, (3) loop approval must survive restart using only loop-owned PR state, (4) merge-wait loses implementation approval after arming, or (5) a downstream rerun must short-circuit on a merged PR because GitHub clears autoMergeRequest after merge."
category: architecture
date: 2026-07-24
version: "1.4.0"
user-invocable: false
verification: verified-ci
history: automation-review-authorization-ci-boundary.history
tags:
  - automation-loop
  - pr-review
  - ci-independent
  - authorization-boundary
  - implementation-go
  - restart-safety
  - state-label
  - source-review
  - merge-wait
  - fail-back
---

# Automation Review Authorization: CI Boundary

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-07-24 |
| **Objective** | Keep a code-automation loop's strict source-review decision inside that loop rather than delegating its authorization to CI/CD, and enforce it at the auto-merge boundary. |
| **Outcome** | ProjectHephaestus moved strict-review proof, workflow triggers, artifacts, leases, and CI-status contracts out of the authorization path. The loop's own CI-free PR review applies `state:implementation-go` after a GO verdict; its review-local payload is then reduced to a fixed non-authorizing context before `merge_wait` can consume the label. PR #2422 confirmed that a lost approval must be routed by arm ownership: a current-run arm is safely disarmed and failed back to fresh PR review, while an external arm is blocked without mutation. |
| **Verification** | verified-ci — PR #2422's full test suite and required checks passed before the normal label-driven merge; the review decision itself remained source-review-only. |

## When to Use

- A strict source review has been added as a required CI check even though the automation loop, not CI, is expected to decide whether implementation may advance.
- The loop is blocked by a workflow artifact, status, lease, or trigger that it cannot start, observe reliably, or repair.
- A restart path needs to determine whether a PR may proceed without reconstructing a CI-run proof artifact.
- You need to retire an external review-proof system while preserving historical ADRs and making the active contract unambiguous.
- A direct `--prs` discovery route can create an issue-less work item that reaches merge-wait with a stale or externally applied GO label.
- A process-local strict-review mutex is released at a stage handoff even though the successor has not yet confirmed the live head and armed auto-merge.
- Review-local head, verdict, or evidence data remains in a work item after the label has been applied, where a later stage could accidentally turn it into a second authorization requirement.
- A downstream rerun evaluates a PR after merge and must short-circuit on PR state instead of expecting `autoMergeRequest` to still be present; GitHub clears `autoMergeRequest` on merged PRs.
- A `merge_wait` poll sees implementation approval missing and must distinguish a current-run arm from an externally armed PR before choosing fail-back, blocking, or terminal containment.

## Verified Workflow

### Quick Reference

```text
automation loop owns source-review authorization
  1. read the live PR source, diff, and loop-owned state
  2. run the strict PR review in the loop (CI-free)
  3. if and only if the review returns GO, apply state:implementation-go
  4. merge_wait consumes that loop-owned label and the live PR head

merge-wait is also the authorization boundary of last resort
  1. reject an item without required issue/requirements context before any arm
  2. durably defer any existing auto-merge request and re-read its state
  3. only then terminally fail the orphaned item
  4. retain the strict-review guard until the first successful arm confirmation

lost approval during merge-wait polling
  1. classify arm ownership before evaluating missing implementation approval
  2. current-run arm: disable it, re-read to confirm it is gone, then FAIL_BACK not_implementation_go
  3. external arm: return BLOCKED without mutation or PR-review routing
  4. if disarm cannot be confirmed, terminalize containment failure; never review or merge with a live arm

CI/CD is outside this decision:
  - do not query checks, workflow runs, artifacts, or deployments
  - do not create review-proof workflows or triggers on review/implementation-go
  - do not make an external CI result a prerequisite for loop progress
```

### Detailed Steps

1. Establish a single decision owner. Source-review authorization belongs to the automation loop when that loop is responsible for planning, implementation, review, and advancement. CI/CD may validate a repository independently, but it is not evidence the loop can depend on for this decision.

2. Run the strict PR review as an in-loop, CI-free operation against the live PR diff. Require an explicit GO result before transition; a missing, ambiguous, or NO-GO result must not apply the approval label.

3. Record the completed loop decision with one loop-owned state marker such as `state:implementation-go`. `merge_wait` should consume that marker and the live PR head when it restarts; it must not require a workflow artifact, lease, status context, or an external proof document.

4. Discard review-local state after the label's post-write current-head confirmation and before transition to `merge_wait`. Use a fixed allowlist of ordinary issue/implementation context, the cleanup worktree path, and the process-local handoff mutex. Do not retain review heads, verdicts, attempts, artifacts, leases, evidence, or a dynamically captured ingress-key list. A denylist cannot anticipate aliases; a dynamic ingress list can preserve a forged or stale proof after a retry. The current-head check must precede sanitization so a concurrent head change still follows the normal containment path with the original review state available.

5. Enforce the requirements-context invariant at `merge_wait.on_enter`, not only at strict review. An unlinked direct PR may have an externally retained GO label, and a stage-routing regression can otherwise bypass the strict-stage orphan check. Before recovery, label consumption, or arming, invoke the same fail-closed helper used for unsafe arm state: request auto-merge deferral, re-read live PR state to confirm it is disarmed, then return terminal failure. This makes stale labels non-authoritative when the work item lacks its required context.

6. Treat the strict-review guard as a handoff mutex, not merely a strict-stage mutex. Keep it held after strict review advances to merge-wait. Release it only when merge-wait's first arm operation has returned its successful continuation after live-label/head verification and arm confirmation. Preserve ownership through fail-back/retry to strict review; release idempotently on terminal finish, shutdown parking, or exception handling.

7. Keep the boundary mechanically enforceable. Delete CI workflows and automatic tasks that trigger from review or implementation-go solely to produce authorization proof. Remove their references from active documentation, agent directions, prompt contracts, and tests.

8. Cover direct PR discovery as well as issue-driven discovery. If the strict stage needs issue/comment context, pass the PR number as its work-item context for a PR discovered without a closing issue. Passing `None` converts a valid PR into a terminal strict-review failure. If direct PRs deliberately remain issue-less, they must stop at merge-wait under step 5; never treat a label alone as enough to compensate for missing requirements context.

9. Preserve ADR history. Do not rewrite accepted historical decisions just to erase obsolete policy. Add a new superseding ADR and update the ADR index so active readers find the current contract while audits retain the original record.

10. Validate the source-only behavior locally: resolve the PR identity and exact head, inspect the source diff and active contracts, run targeted stage/documentation tests, and run `git diff --check`. Tests must cover (a) an orphaned merge-wait item with GO label: defer is attempted, no arm occurs, terminal failure; (b) a competing strict reviewer remains blocked during merge-wait arm and proceeds only after the successful arm continuation; and (c) known, unknown, and forged review-proof aliases do not cross the successful GO handoff. State clearly that this is not CI evidence.

11. Short-circuit downstream reruns on terminal PR state. If a later workflow or review pass reruns after the PR has merged, fetch PR `state` first and exit 0 when it is not `OPEN`. GitHub clears `autoMergeRequest` on merged PRs, so a null arm is expected and not a blocker.

12. In `merge_wait.POLL`, determine whether the live auto-merge arm was created by the current run before handling a missing implementation-approval label. Only the current run may safely return `FAIL_BACK, not_implementation_go`: first request disarm and verify the live arm is gone, then route to a fresh PR review. An externally armed PR remains `BLOCKED` with no mutation. A failed or unverifiable disarm is terminal containment failure, never permission to continue.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Put strict-review proof in CI/CD | A required workflow generated proof artifacts and gated progress. | The automation loop did not control the CI loop, so it could neither make progress nor make a trustworthy decision from that external state. | The component that owns the review decision must execute and persist it; make strict source review an in-loop CI-free operation. |
| Restart from external proof data | `merge_wait` depended on workflow artifacts, leases, and status records. | Those records were external coupling, could be absent or stale after restart, and were unnecessary once loop-owned approval existed. | Restart from the loop-owned approval label and live PR state only. |
| Preserve strict-review ingress fields dynamically | A review pass snapshotted its ingress payload keys and kept those keys after GO. | An unknown alias or forged snapshot entry could preserve review evidence; after a NOGO retry, the stale snapshot could drop fresh implementation context. | At the GO boundary, retain a small fixed set of non-authorizing context keys instead of trying to classify or remember review fields. |
| Treat a repository-wide PR as an issue-less strict review | Direct PR discovery constructed a work item with `issue=None`. | The strict stage requires issue/comment context and rejected the work item as terminal. | For direct PR review, use the PR number as the work-item context unless the design supplies an equivalent explicit context. |
| Trust strict review as the only orphan check | An unlinked direct PR with a stale `state:implementation-go` label was routed around strict review and into merge-wait. | The strict-stage check never ran on that path, so merge-wait could otherwise arm auto-merge without requirements context. | Repeat the invariant at the irreversible side-effect boundary: defer, confirm deferral, and fail before merge-wait reads labels or arms. |
| Release strict guard at the stage transition | The guard was released as soon as strict review routed to merge-wait. | A competing strict reviewer could enter while the first item was between review approval and confirmed arming. | Retain ownership through merge-wait's first successful arm and release on its `POLL` continuation; all finish/park/exception paths remain idempotent releases. |
| Treat `autoMergeRequest` as a post-merge signal | A rerun checked `autoMergeRequest` after the PR had already merged. | GitHub clears `autoMergeRequest` on merged PRs, so the rerun misread a terminal PR as still pending. | Post-merge consumers must check `state` and short-circuit on non-`OPEN` instead of treating `autoMergeRequest` as durable. |
| Treat every missing approval in `merge_wait.POLL` as terminal | The stage returned a terminal failure when the implementation label disappeared from a current-run arm. | The open PR was stranded even though the routing table already mapped `not_implementation_go` to `PR_REVIEW`. | For a current-run arm, disarm and verify containment, then fail back for a fresh review. |
| Check approval before arm ownership | A poll evaluated the missing label before classifying an already-armed PR as external. | An external arm could be mutated or incorrectly routed into review, crossing the ownership boundary. | Classify ownership first; external arms are `BLOCKED` and untouched. |
| Continue after a failed disarm or stale live arm | The stage attempted to recover review without proving auto-merge was disabled. | Review could proceed while an irreversible arm remained live. | Failed containment is terminal; require a fresh live-state read confirming disarm before fail-back. |
| Rewrite accepted ADRs to remove obsolete instructions | Historical ADR text was modified in place. | It obscured the decision record and broke the repository's ADR immutability convention. | Preserve accepted ADRs verbatim; add a superseding ADR and make the index point to the active policy. |

## Results & Parameters

| Item | Result |
|------|--------|
| Decision marker | `state:implementation-go` is the sole loop-owned authorization after a current review GO. |
| Prohibited dependencies | CI checks, workflow runs, artifacts, deployments, external status contexts, and review-proof leases. |
| Review-state handoff | After the label's current-head readback, retain only fixed non-authorizing context, cleanup, and the ephemeral handoff mutex; discard all review result and proof aliases. |
| Direct-PR correction | Use the PR number as strict-review work-item context rather than `None`. |
| Defense in depth | `merge_wait.on_enter` rejects `issue=None` before consuming labels or arming; deferral is confirmed by a fresh PR-state read. |
| Guard lifetime | Strict-review ownership covers the strict-to-merge-wait handoff and first successful arm; it is released only after that arm confirms continuation to polling. |
| Lost approval routing | Current-run arm: `disable → verify disarmed → FAIL_BACK, not_implementation_go → PR_REVIEW`; external arm: `BLOCKED`, no mutation; failed containment: terminal failure. |
| Post-merge terminality | PR #2306 / issue #2177 merged at `2026-07-21T01:53:35Z` with `state=MERGED`; `autoMergeRequest` is `null` and `mergeStateStatus` was `UNKNOWN` after merge. Downstream reruns must key off PR state and treat terminal PRs as complete. |
| Review/merge evidence | Issue #2417 required rebase after #2418, exact `Closes #2417`, signed+DCO commits, stage/coordinator regressions, full validation, and the normal `state:implementation-go` label path. |
| Local validation example | `uv run pytest` over pipeline stage/coordinator and active-documentation/ADR tests: 85 passed; `git diff --check` passed. |
| Historical-policy migration | Preserve accepted ADRs; record the new label-only rule in a superseding ADR and its index entry. |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | PR #2280 / issues #2053 and #2276 | CI-free source review and loop-owned `state:implementation-go` authorization. The direct repository-wide PR route now supplies PR context to strict review. Local swarm review then found that dynamic review-payload preservation could retain an aliased proof or survive a NOGO retry; a fixed allowlist removes those fields only after the label's current-head readback. Local verification only; no CI/CD state was queried. |
| ProjectHephaestus | PR #2306 / issue #2177 | Docs PR that reached merged state through the normal review-to-merge path: review GO, loop-owned `state:implementation-go`, and merge_wait. Post-merge `gh pr view` showed `state=MERGED` with `autoMergeRequest=null`, confirming reruns must short-circuit on terminal PR state. |
| ProjectHephaestus | PR #2422 / issue #2417 | After #2418 landed, the implementation preserved ownership-first polling: owned lost approval disarmed and failed back to `PR_REVIEW`; external arms stayed blocked; containment failures terminalized. The PR body contained `Closes #2417`, its full suite reported 6676 passed and 6 skipped, required checks passed, `state:implementation-go` was applied, and merge_wait completed the normal merge at `2026-07-24T16:01:09Z` as `d544776c`. Verified in CI. |
