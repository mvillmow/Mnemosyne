---
name: automation-review-authorization-ci-boundary
description: "Keep automation-loop source-review authorization independent of CI/CD and preserve adopted PR branches through detached review. Use when: (1) review approval must bind to an exact head, (2) a clean detached worktree may contain an agent-precommitted fix, (3) an adopted PR head is confused with a disposable branch reservation, (4) repeated NOGO rounds need fail-closed evidence handling before merge, or (5) merge-wait must consume only current-head approval."
category: architecture
date: 2026-07-28
version: "1.6.0"
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
  - detached-review
  - publication-lease
  - adopted-pr-branch
---

# Automation Review Authorization: CI Boundary

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-07-28 |
| **Objective** | Keep a code-automation loop's strict source-review decision inside that loop rather than delegating its authorization to CI/CD, and enforce it at the auto-merge boundary. |
| **Outcome** | ProjectHephaestus keeps source-review authorization in the loop and binds publication and merge progression to the reviewed head. PR #2508 added the missing detached-review distinction: the reviewed remote head is a publication lease, not a disposable direct-scope branch reservation. A clean worktree with commits ahead publishes the exact detached `HEAD`; a clean worktree at the reviewed head performs no publication and never deletes the adopted PR branch. |
| **Verification** | verified-ci — PR #2508 passed required checks on final head `55300bb4`, received `state:implementation-go` only after the current-head review, and merged as `6a3cb437`; the review decision remained source-review-only. |

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
- A PR has repeated NOGO reviews involving unproven test claims, stale heads, or environment-sensitive integration validation and must not advance to merge on prose alone.
- A detached adopted-PR review returns a clean worktree and the coordinator must distinguish “no change” from “the address agent already committed the change.”
- A commit/push helper receives both `publish_detached_head=True` and `expected_remote_sha`; the latter is a publication lease for the reviewed PR head, not ownership of a disposable branch reservation.

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

detached adopted-PR publication
  1. commit any dirty changes, then compare expected_remote_sha..HEAD
  2. ahead > 0: publish exact HEAD with the expected remote SHA as the lease
  3. ahead == 0: return no-publication; never release/delete the adopted PR branch
  4. keep reservation cleanup only for coordinator-created direct-scope branches

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

13. Treat each NOGO as a fail-closed review result, not as a merge-retry hint. Remove unsupported full-suite or tool-success claims, fix concrete findings (including ambient environment/configuration that can change integration behavior), and re-review the new head. The loop's source review remains CI-free, but merge readiness still requires independently observed required checks on that exact final head; after those checks pass, retain the ordinary `state:implementation-go` → `merge_wait` → merged-state confirmation path.

14. Classify detached review publication by ancestry, not worktree dirtiness. `commit_if_changes() == False` proves only that there are no uncommitted edits; the address agent may already have committed a fix while leaving the detached worktree clean. Count `expected_remote_sha..HEAD`: when nonzero, publish the exact detached `HEAD` through the SHA-leased push path; when zero and `publish_detached_head=True`, return “nothing to publish” without calling branch-release cleanup. Only coordinator-created direct-scope reservations may use `delete_reserved_branch_if_unchanged`.

15. Revoke review proof when a correction changes the head. Re-review the new exact head, confirm the complete thread state, and only then apply `state:implementation-go`. PR #2508 moved from an earlier reviewed head to `55300bb4` after the clean-worktree ambiguity was found; the final audit, label, and conditional merge all targeted that corrected head. Passing CI on the final head was merge-readiness evidence, not the source-review authorization.

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
| Treat a repeated NOGO as an actionable merge retry | PR #2344 / issue #2232 was reviewed repeatedly while the changed integration test had unproven full-suite claims, an ambient Git configuration hazard, and a formatting defect. | The implementation was substantively correct, but the evidence gate and concrete findings still prevented safe authorization; merging on the claim or on a prior head would have bypassed the review contract. | Keep the PR in NOGO/no-go until each finding is fixed, independently review the new head, and require current-head checks before normal merge-wait progression. |
| Assume read-only review execution can validate real temporary-repository integration | The reviewer attempted to run the full/integration evidence in a read-only environment. | The test needed a writable temporary directory, so the claimed test count could not be independently confirmed there. | Treat sandbox-blocked execution as unavailable evidence; do not repeat the claim in PR prose, and use writable local or CI execution for merge evidence. |
| Treat a clean detached worktree as “no fix” | Publication logic relied only on whether `commit_if_changes()` created a commit. | An address agent can commit its own fix, leaving the worktree clean while `HEAD` is ahead of the reviewed remote SHA. | After handling dirty changes, compare `expected_remote_sha..HEAD`; publish the exact detached head whenever it is ahead. |
| Reuse direct-scope reservation cleanup for an adopted PR | A zero-ahead clean review routed `expected_remote_sha` through `delete_reserved_branch_if_unchanged`. | In detached review, that SHA identifies the adopted open PR head; successful cleanup would delete the contributor's remote branch. | Treat the reviewed SHA as a publication lease in detached mode. Zero-ahead means no publication and no deletion. |
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
| NOGO-to-merge evidence | PR #2344 / issue #2232 received repeated NOGO results for unproven full-suite claims, an ambient Git config injection risk, and formatting; after fixes, independently observed required checks passed on the final head and the PR merged at `2026-07-27T09:42:33Z` as `03e31fcd4e0af48ab5ccdbabda2b40c9b460fa3c`. |
| Detached publication decision | With `publish_detached_head=True`, use `git rev-list --count expected_remote_sha..HEAD`: nonzero publishes exact `HEAD` with the expected SHA lease; zero returns false without branch-release cleanup. |
| PR #2508 sequence | Corrected head `55300bb4` passed required checks; final current-head review posted; `state:implementation-go` applied at `2026-07-28T19:18:15Z`; conditional merge completed 12 seconds later as `6a3cb437`. |
| Local validation example | `uv run pytest` over pipeline stage/coordinator and active-documentation/ADR tests: 85 passed; `git diff --check` passed. |
| Historical-policy migration | Preserve accepted ADRs; record the new label-only rule in a superseding ADR and its index entry. |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | PR #2280 / issues #2053 and #2276 | CI-free source review and loop-owned `state:implementation-go` authorization. The direct repository-wide PR route now supplies PR context to strict review. Local swarm review then found that dynamic review-payload preservation could retain an aliased proof or survive a NOGO retry; a fixed allowlist removes those fields only after the label's current-head readback. Local verification only; no CI/CD state was queried. |
| ProjectHephaestus | PR #2306 / issue #2177 | Docs PR that reached merged state through the normal review-to-merge path: review GO, loop-owned `state:implementation-go`, and merge_wait. Post-merge `gh pr view` showed `state=MERGED` with `autoMergeRequest=null`, confirming reruns must short-circuit on terminal PR state. |
| ProjectHephaestus | PR #2422 / issue #2417 | After #2418 landed, the implementation preserved ownership-first polling: owned lost approval disarmed and failed back to `PR_REVIEW`; external arms stayed blocked; containment failures terminalized. The PR body contained `Closes #2417`, its full suite reported 6676 passed and 6 skipped, required checks passed, `state:implementation-go` was applied, and merge_wait completed the normal merge at `2026-07-24T16:01:09Z` as `d544776c`. Verified in CI. |
| ProjectHephaestus | PR #2344 / issue #2232 | Repeated NOGO reviews stayed fail-closed: unproven “6474 passed” prose, read-only inability to run the writable-temp integration test, an ambient Git config injection hazard, and Ruff formatting were treated as blockers. After fixes and current-head required checks passed, the PR completed the normal review/label/merge path at `2026-07-27T09:42:33Z` with merge commit `03e31fcd4e0af48ab5ccdbabda2b40c9b460fa3c`. Verified in CI. |
| ProjectHephaestus | PR #2508 / issue #2507 | A review correction distinguished two clean detached states: unchanged reviewed head (no publish, no branch release) and agent-precommitted head (publish exact `HEAD` through the reviewed-SHA lease). The corrected head `55300bb4` passed required checks, received a fresh current-head review and `state:implementation-go`, then merged as `6a3cb437`. Verified in CI. |
