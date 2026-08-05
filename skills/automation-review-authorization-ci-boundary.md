---
name: automation-review-authorization-ci-boundary
description: "Separate automation-loop source-review authorization from repository policy, required-check readiness, and the final exact-head merge. Use when: (1) review approval must bind to an exact head, (2) CI duplicates a ruleset-owned signature policy, (3) an advisory workflow is mistaken for merge authority, (4) merge-wait must preserve reviewed-head proof while checks finish, (5) native auto-merge is already armed, (6) repeated NOGO or skipped rounds need fail-closed handling, (7) detached-review publication and handoff evidence must remain head-bound, or (8) batched thread replies need replay-safe head and snapshot identity."
category: architecture
date: 2026-08-04
version: "2.1.0"
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
  - iterative-review
  - review-batch
  - pr-handoff
  - test-receipt
  - validation-classification
  - ruff-format
  - skip-revival
  - slow-required-checks
  - conditional-merge
  - github-ruleset
  - required-signatures
  - advisory-check
  - reply-journal
  - thread-snapshot
---

# Automation Review Authorization: CI Boundary

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-08-04 |
| **Objective** | Keep a code-automation loop's strict source-review decision inside that loop rather than delegating its authorization to CI/CD, and enforce exact-head review/merge progression across repeated remediation rounds. |
| **Outcome** | ProjectHephaestus PR #2604 / issue #2330 removed duplicate signature verification from `pr-policy` and deleted the non-authorizing `auto-merge-policy` job. The live loop kept source-review authority in the exact-head GO label, left signature and required-check enforcement with GitHub, and conditionally merged the reviewed head without native auto-merge. |
| **Verification** | verified-ci — final review head `76ab9692`; three review threads resolved; required checks green; exclusive GO/NOGO transition completed; merge commit `b9a27e62` landed at `2026-08-04T22:15:59Z`. |
| **Issue #2361 / PR #2612** | verified-ci — five remediation batches journaled 31 replies against exact heads and thread-snapshot digests. A later same-head review revoked an interim GO. The final review matched `db9bd1ef`; all 37 threads were resolved, GO replaced NOGO, required checks completed, and conditional merge `d9d53fa0` followed with native auto-merge null. |

## When to Use

- A strict source review has been added as a required CI check even though the automation loop, not CI, is expected to decide whether implementation may advance.
- The loop is blocked by a workflow artifact, status, lease, or trigger that it cannot start, observe reliably, or repair.
- A restart path needs to determine whether a PR may proceed without reconstructing a CI-run proof artifact.
- You need to retire an external review-proof system while preserving historical ADRs and making the active contract unambiguous.
- A direct `--prs` discovery route can create an issue-less work item that reaches merge-wait with a stale or externally applied GO label.
- A strict-review handoff reaches `merge_wait` without preserving the exact reviewed-head receipt needed for a conditional merge.
- Review-local head, verdict, or evidence data remains in a work item after the label has been applied, where a later stage could accidentally turn it into a second authorization requirement.
- A downstream rerun evaluates a PR after merge and must short-circuit on PR state instead of expecting `autoMergeRequest` to still be present; GitHub clears `autoMergeRequest` on merged PRs.
- A live PR has native auto-merge armed and the loop is tempted to disable, adopt, or replace an external actor's request.
- A PR has repeated NOGO reviews involving unproven test claims, stale heads, or environment-sensitive integration validation and must not advance to merge on prose alone.
- A generated PR title/body could contain agent-supplied summaries, absolute worktree paths, stale working-tree claims, or unsupported test totals.
- A Python PR review needs a bounded pytest receipt for changed unit tests, tied to the new side of the diff and excluding deleted or non-hermetic verifier tests.
- Ruff format reports `unformatted:` or `N files would be reformatted`, and the result must route to implementation remediation rather than be classified as a host/runner fault.
- A detached adopted-PR review returns a clean worktree and the coordinator must distinguish “no change” from “the address agent already committed the change.”
- A commit/push helper receives both `publish_detached_head=True` and `expected_remote_sha`; the latter is a publication lease for the reviewed PR head, not ownership of a disposable branch reservation.
- A PR needs several review/address rounds; a later finding or head change must invalidate earlier authorization.
- A prior review round was exhausted or marked `state:skip`; re-entry must require a new pushed head and a fresh current-head review.
- `state:implementation-go` appears before slow required checks finish, so merge-wait must retain the exact-head proof and wait without treating CI as review authorization.
- A completed run needs an event audit that separates review evidence, GO/NOGO label transitions, required checks, and the final merge.
- `pr-policy` duplicates a commit-signature check already enforced by an active GitHub ruleset.
- A required or advisory CI job reports auto-merge state but does not own the loop's GO label or final merge mutation.
- A remediation round replies to several review threads and its durable handoff must not replay against a changed head or changed thread set.

## Verified Workflow

### Quick Reference

```text
automation loop owns source-review authorization
  1. capture the live PR head and review a clean checkout of that exact head
  2. require complete review-thread state and an explicit in-loop GO
  3. re-read OPEN, autoMergeRequest:null, and the unchanged reviewed head
  4. replace NOGO with state:implementation-go and verify the exclusive label state
  5. pass the exact reviewed-head receipt to merge_wait for this process only

merge-wait is also the authorization boundary of last resort
  1. reject missing requirements context, missing/drifted head proof, or nonexclusive GO
  2. block a non-null native auto-merge request without mutating it
  3. wait for repository merge readiness and server-enforced conversation resolution
  4. re-read admission facts, then issue one SHA-conditional normal squash merge
  5. reconcile the terminal merged state; never enable, disable, adopt, or poll native auto-merge

detached adopted-PR publication
  1. commit any dirty changes, then compare expected_remote_sha..HEAD
  2. ahead > 0: publish exact HEAD with the expected remote SHA as the lease
  3. ahead == 0: return no-publication; never release/delete the adopted PR branch
  4. keep reservation cleanup only for coordinator-created direct-scope branches

authority separation
  - strict review + state:implementation-go: loop-owned source authorization
  - required_signatures: GitHub ruleset-owned commit policy
  - pr-policy and required checks: repository merge readiness
  - conditional exact-head merge: merge_wait's only merge mutation
  - advisory workflow output: evidence only; never authorization

review handoff evidence
  1. derive PR text from host-owned facts; keep summaries deterministic, repo-relative, and command-level
  2. bind bounded pytest receipts to changed new-side unit-test paths on the reviewed head
  3. exclude deleted paths and non-hermetic host-verifier tests from that receipt plan
  4. classify Ruff format output as validation remediation; keep bootstrap/sandbox failures fail-closed
  5. re-review and merge only the resulting exact head
  6. journal batched thread replies with PR number, head SHA, unique batch nonce, reply map, and thread-snapshot digest

review revival and slow-check merge
  1. treat `state:skip` or exhausted unresolved rounds as containment, never approval
  2. require a new pushed head and a fresh review whose `commit_id` matches that head
  3. treat a no-commit reply as review evidence only; it does not authorize the PR
  4. after GO, wait for the exact-head required checks and merge-gate result before merging

CI/CD is outside the source-review decision:
  - pr_review does not treat checks, runs, artifacts, or deployments as GO authority
  - merge_wait may observe repository readiness after GO, but readiness cannot create GO
  - do not create review-proof or advisory workflows and mistake their output for authorization
```

### Detailed Steps

1. Establish a single decision owner. Source-review authorization belongs to the automation loop when that loop is responsible for planning, implementation, review, and advancement. CI/CD may validate a repository independently, but it is not evidence the loop can depend on for this decision.

2. Run the strict PR review as an in-loop, CI-free operation against the live PR diff. Require an explicit GO result before transition; a missing, ambiguous, or NO-GO result must not apply the approval label.

3. Record the completed loop decision with one loop-owned state marker such as `state:implementation-go`. `merge_wait` should consume that marker and the live PR head when it restarts; it must not require a workflow artifact, lease, status context, or an external proof document.

   Treat each addressed review batch as a fresh authorization attempt. If a later review finds
   another major issue, retract `state:implementation-go`, apply NOGO, and repeat the exact-head
   review; do not carry the earlier GO into merge-wait. PR #2596 followed NOGO → GO → NOGO → GO,
   with final head `81331951` as the only revision authorized for merge. PR #2612 additionally
   showed that a later review on the same head can invalidate a transient GO; label exclusivity
   must return to NOGO until another exact-head review authorizes progression.

4. After the label's post-write current-head confirmation, retain one process-local reviewed-head receipt for `merge_wait`; discard verdict prose, grades, artifacts, leases, and aliases. The receipt is deliberately non-durable: restart, refresh, checkout mismatch, or head drift routes back to fresh review even when the durable GO label remains.

5. Enforce the requirements-context invariant at `merge_wait.on_enter`, not only at strict review. An unlinked direct PR may have an externally retained GO label, and a stage-routing regression can otherwise bypass the strict-stage orphan check. Reject it before consuming labels or issuing a merge request; do not mutate GitHub state while rejecting it.

6. Treat native auto-merge as externally owned. Every advancing label write and final merge admission must explicitly observe `autoMergeRequest: null`; an omitted field is partial data, not proof. If a request is present, block without enabling, disabling, adopting, or polling it because the loop cannot prove ownership of that external state.

7. Keep the boundary mechanically enforceable. Delete CI workflows and automatic tasks that merely report review or auto-merge state without authorizing anything. Keep policy in its real owner: GitHub rulesets enforce commit signatures; `pr-policy` enforces structural PR/commit rules such as the closing line, Conventional Commits, and DCO; the loop owns review GO; branch protection and required checks gate the merge.

8. Cover direct PR discovery as well as issue-driven discovery. If the strict stage needs issue/comment context, pass the PR number as its work-item context for a PR discovered without a closing issue. Passing `None` converts a valid PR into a terminal strict-review failure. If direct PRs deliberately remain issue-less, they must stop at merge-wait under step 5; never treat a label alone as enough to compensate for missing requirements context.

9. Preserve ADR history. Do not rewrite accepted historical decisions just to erase obsolete policy. Add a new superseding ADR and update the ADR index so active readers find the current contract while audits retain the original record.

10. Validate the source-only behavior locally: resolve the PR identity and exact head, inspect the source diff and active contracts, run targeted stage/documentation tests, and run `git diff --check`. Tests must cover (a) an orphaned merge-wait item with GO fails before mutation, (b) missing or drifted process-local head proof fails back to review, (c) a non-null or unreadable auto-merge request blocks advancement without mutation, and (d) the only successful merge request includes the reviewed head SHA. State clearly that this is not CI evidence.

11. Short-circuit downstream reruns on terminal PR state. If a later workflow or review pass reruns after the PR has merged, fetch PR `state` first and exit 0 when it is not `OPEN`. GitHub clears `autoMergeRequest` on merged PRs, so a null arm is expected and not a blocker.

12. In `merge_wait`, admit the PR only when it is open against `main`, explicitly unarmed, exclusively GO-labeled, thread-complete, protected by server-side conversation resolution, and still at the process-local reviewed head. Wait for ordinary merge readiness, repeat all admission reads, then perform one SHA-conditional normal squash merge. A 409/head drift routes to fresh review; a present native auto-merge request remains blocked and untouched.

13. Treat each NOGO as a fail-closed review result, not as a merge-retry hint. Remove unsupported full-suite or tool-success claims, fix concrete findings (including ambient environment/configuration that can change integration behavior), and re-review the new head. The loop's source review remains CI-free, but merge readiness still requires independently observed required checks on that exact final head; after those checks pass, retain the ordinary `state:implementation-go` → `merge_wait` → merged-state confirmation path.

14. Classify detached review publication by ancestry, not worktree dirtiness. `commit_if_changes() == False` proves only that there are no uncommitted edits; the address agent may already have committed a fix while leaving the detached worktree clean. Count `expected_remote_sha..HEAD`: when nonzero, publish the exact detached `HEAD` through the SHA-leased push path; when zero and `publish_detached_head=True`, return “nothing to publish” without calling branch-release cleanup. Only coordinator-created direct-scope reservations may use `delete_reserved_branch_if_unchanged`.

15. Revoke review proof when a correction changes the head. Re-review the new exact head, confirm the complete thread state, and only then apply `state:implementation-go`. PR #2508 moved from an earlier reviewed head to `55300bb4` after the clean-worktree ambiguity was found; the final audit, label, and conditional merge all targeted that corrected head. Passing CI on the final head was merge-readiness evidence, not the source-review authorization.

16. Treat the generated PR handoff as a host-owned journal, not an agent transcript. Use a deterministic conventional title, a stable summary, repo-relative change references, command-level testing facts, and the exact `Closes #N` line. Do not persist agent summaries containing local paths, stale “uncommitted” claims, or unsupported suite totals.

17. Make review test evidence both bounded and head-bound. Derive changed unit-test receipts from real diff headers and the new-side `+++` path; skip `/dev/null` deletions and known non-hermetic verifier tests. Run each selected test file with a bounded command such as `uv run pytest -o addopts= <path> -q --tb=short`, then attach the receipt to the checked-out review head before the primary reviewer runs.

18. Classify tool output by meaning, not only exit code. Ruff format exit 1 with `unformatted:` or `<N> files would be reformatted` is an implementation validation failure and should route back to remediation. Missing tools, bootstrap failures, sandbox denials, and other host failures remain fail-closed and must not be presented as code findings.

19. Revive an exhausted review only after new evidence arrives. `state:skip` and a stuck
    unresolved-thread count are terminal containment, not a reusable NOGO/GO proof. Require a
    new pushed head, replay the implementation handoff against that head, and submit a fresh
    review whose commit binding matches it. A reply that removes unsupported claims without a
    source commit remains evidence for the next review; it is not authorization by itself.
    PR #2602 / issue #2284 followed this path from an earlier no-go/skip state to final-head
    review at `0674b3f2`.

20. Keep review authorization separate from merge readiness when checks have different speeds.
    PR #2602 received its final `state:implementation-go` at `18:59:11Z`, while the unit test
    jobs completed at `19:05:55Z` and `19:06:55Z`; `required-checks-gate` completed at
    `19:07:04Z`, and merge followed at `19:07:16Z`. `merge_wait` must preserve the exact-head
    review proof and wait for the repository's normal merge requirements; the delayed checks
    establish merge eligibility, not source-review authorization.

21. Persist a batched thread-reply handoff as structured identity, not free-form completion prose.
    Record `pr_number`, `head_sha`, a unique `batch_nonce`, the thread-ID-to-reply map, and
    `thread_snapshot_sha256`. Replay only when the live PR, head, and thread snapshot still match.
    PR #2612 used five such batches across changing heads; the final batch remained review input,
    not authorization, until the matching final-head review completed with zero unresolved threads.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Put strict-review proof in CI/CD | A required workflow generated proof artifacts and gated progress. | The automation loop did not control the CI loop, so it could neither make progress nor make a trustworthy decision from that external state. | The component that owns the review decision must execute and persist it; make strict source review an in-loop CI-free operation. |
| Restart from external proof data | `merge_wait` depended on workflow artifacts, leases, and status records. | Those records were external coupling, could be absent or stale after restart, and were unnecessary once loop-owned approval existed. | Restart from the loop-owned approval label and live PR state only. |
| Preserve strict-review ingress fields dynamically | A review pass snapshotted its ingress payload keys and kept those keys after GO. | An unknown alias or forged snapshot entry could preserve review evidence; after a NOGO retry, the stale snapshot could drop fresh implementation context. | At the GO boundary, retain a small fixed set of non-authorizing context keys instead of trying to classify or remember review fields. |
| Treat a repository-wide PR as an issue-less strict review | Direct PR discovery constructed a work item with `issue=None`. | The strict stage requires issue/comment context and rejected the work item as terminal. | For direct PR review, use the PR number as the work-item context unless the design supplies an equivalent explicit context. |
| Trust strict review as the only orphan check | An unlinked direct PR with a stale `state:implementation-go` label was routed around strict review and into merge-wait. | The strict-stage check never ran on that path, so merge-wait could otherwise consume the label without requirements context. | Repeat the invariant at the irreversible side-effect boundary and fail before label consumption or any GitHub mutation. |
| Drop the reviewed-head receipt at the stage transition | GO was treated as sufficient after strict review routed to merge-wait. | A durable label alone cannot prove which head this process reviewed; a restart or push can make it stale. | Preserve the process-local exact-head receipt through merge admission; missing or drifted proof routes to fresh review. |
| Treat `autoMergeRequest` as a post-merge signal | A rerun checked `autoMergeRequest` after the PR had already merged. | GitHub clears `autoMergeRequest` on merged PRs, so the rerun misread a terminal PR as still pending. | Post-merge consumers must check `state` and short-circuit on non-`OPEN` instead of treating `autoMergeRequest` as durable. |
| Mutate a native auto-merge request | The loop tried to disable or adopt an existing request before continuing. | GitHub exposes no conditional disable that proves the loop owns the request; mutation can race with an external actor. | Require explicit absence for advancement and leave a present request blocked and untouched. |
| Duplicate signature enforcement in `pr-policy` | CI rechecked cryptographic commit signatures already required by the active branch ruleset. | Two owners can drift, and the workflow check adds maintenance without adding enforcement. | Keep `required_signatures` in the ruleset; keep `pr-policy` focused on structural PR/commit rules. |
| Keep an advisory auto-merge policy job | A CI job reported whether auto-merge policy looked satisfied but authorized no label or merge action. | The signal appeared authoritative while changing no durable state and granting no permission. | Delete non-authorizing advisory gates; audit the actual GO label, required checks, and merge event instead. |
| Treat a repeated NOGO as an actionable merge retry | PR #2344 / issue #2232 was reviewed repeatedly while the changed integration test had unproven full-suite claims, an ambient Git configuration hazard, and a formatting defect. | The implementation was substantively correct, but the evidence gate and concrete findings still prevented safe authorization; merging on the claim or on a prior head would have bypassed the review contract. | Keep the PR in NOGO/no-go until each finding is fixed, independently review the new head, and require current-head checks before normal merge-wait progression. |
| Assume read-only review execution can validate real temporary-repository integration | The reviewer attempted to run the full/integration evidence in a read-only environment. | The test needed a writable temporary directory, so the claimed test count could not be independently confirmed there. | Treat sandbox-blocked execution as unavailable evidence; do not repeat the claim in PR prose, and use writable local or CI execution for merge evidence. |
| Treat a clean detached worktree as “no fix” | Publication logic relied only on whether `commit_if_changes()` created a commit. | An address agent can commit its own fix, leaving the worktree clean while `HEAD` is ahead of the reviewed remote SHA. | After handling dirty changes, compare `expected_remote_sha..HEAD`; publish the exact detached head whenever it is ahead. |
| Reuse direct-scope reservation cleanup for an adopted PR | A zero-ahead clean review routed `expected_remote_sha` through `delete_reserved_branch_if_unchanged`. | In detached review, that SHA identifies the adopted open PR head; successful cleanup would delete the contributor's remote branch. | Treat the reviewed SHA as a publication lease in detached mode. Zero-ahead means no publication and no deletion. |
| Let a prior GO survive a later review finding | PR #2596 reached GO after the first five findings were addressed, then a later review found a remaining `create_pr` title-normalization gap | The earlier authorization no longer described the complete implementation or final review state | Retract GO to NOGO, address the finding, and run a fresh exact-head review before restoring GO |
| Replay a reply batch from prose alone | A prior remediation comment still looked complete after the PR head or thread set changed. | Its replies described a superseded review snapshot and could skip current-thread disposition. | Bind the journal to PR number, exact head, unique batch nonce, reply map, and thread-snapshot digest; mismatches require a fresh batch. |
| Rewrite accepted ADRs to remove obsolete instructions | Historical ADR text was modified in place. | It obscured the decision record and broke the repository's ADR immutability convention. | Preserve accepted ADRs verbatim; add a superseding ADR and make the index point to the active policy. |
| Reuse the agent's implementation summary in the PR body | A generated handoff copied the implementer's summary, including an absolute worktree path, a stale uncommitted-change claim, and an unsupported full-suite total. | Agent-local state is not a durable fact about the pushed PR head and can mislead the reviewer. | Generate the handoff from host-owned deterministic facts; keep the diff as the source of implementation detail. |
| Run one broad pytest receipt for every changed test path | The review receipt plan used all diff-header paths, including deleted tests and the non-hermetic worker-pool verifier test. | Deleted paths cannot run, and nested verifier execution produces runner failures instead of code evidence. | Parse new-side paths, omit `/dev/null`, and exclude the explicitly non-hermetic verifier suite. |
| Treat every Ruff format exit 1 as a host failure | Output such as `unformatted:` and `3 files would be reformatted` was classified as a runner fault. | The formatter executed successfully and reported an actionable code-format defect; terminalizing the item skipped remediation. | Match known Ruff format diagnostics as `validation`; keep bootstrap and sandbox failures fail-closed. |
| Reuse a review after exhausted rounds or `state:skip` | PR #2602 / issue #2284 had earlier no-go evidence and a stuck unresolved-thread count before the later implementation handoff. | The old review targeted an earlier state and could not authorize the final head; the loop required new evidence and a fresh exact-head review. | Treat skip as containment, require a new pushed head, and bind the next review to that head. |
| Treat a no-commit reply as approval | The implementation reply removed unsupported PR claims without changing source. | It supplied remediation evidence but did not itself authorize the PR; only the subsequent final-head review could produce GO. | Keep no-commit replies reviewable evidence and leave disposition with the reviewer. |
| Merge as soon as GO is labeled | PR #2602 received GO before the slow unit-test jobs and required-checks gate completed. | Merge eligibility was not ready until the final checks finished; merge-wait had to wait before the conditional merge. | Preserve the exact-head proof while waiting; CI readiness gates merging but does not grant review authority. |

## Results & Parameters

| Item | Result |
|------|--------|
| Decision marker | `state:implementation-go` is the sole loop-owned authorization after a current review GO. |
| Prohibited source-review dependencies | CI checks, workflow runs, artifacts, deployments, external status contexts, and review-proof leases cannot grant GO. |
| Review-state handoff | After the label's current-head readback, retain the exact reviewed head only as a process-local merge receipt; discard verdict prose, grades, artifacts, leases, and aliases. |
| Direct-PR correction | Use the PR number as strict-review work-item context rather than `None`. |
| Defense in depth | `merge_wait.on_enter` rejects `issue=None` before consuming labels or making any GitHub mutation. |
| Merge admission | Open `main` PR + explicit `autoMergeRequest:null` + exclusive GO + zero unresolved threads + server conversation protection + exact process-local reviewed head. |
| Merge mutation | One ordinary squash merge conditioned on the reviewed head SHA; native auto-merge is never enabled, disabled, adopted, or polled. |
| Post-merge terminality | PR #2306 / issue #2177 merged at `2026-07-21T01:53:35Z` with `state=MERGED`; `autoMergeRequest` is `null` and `mergeStateStatus` was `UNKNOWN` after merge. Downstream reruns must key off PR state and treat terminal PRs as complete. |
| Review/merge evidence | Issue #2417 required rebase after #2418, exact `Closes #2417`, signed+DCO commits, stage/coordinator regressions, full validation, and the normal `state:implementation-go` label path. |
| NOGO-to-merge evidence | PR #2344 / issue #2232 received repeated NOGO results for unproven full-suite claims, an ambient Git config injection risk, and formatting; after fixes, independently observed required checks passed on the final head and the PR merged at `2026-07-27T09:42:33Z` as `03e31fcd4e0af48ab5ccdbabda2b40c9b460fa3c`. |
| Detached publication decision | With `publish_detached_head=True`, use `git rev-list --count expected_remote_sha..HEAD`: nonzero publishes exact `HEAD` with the expected SHA lease; zero returns false without branch-release cleanup. |
| PR #2508 sequence | Corrected head `55300bb4` passed required checks; final current-head review posted; `state:implementation-go` applied at `2026-07-28T19:18:15Z`; conditional merge completed 12 seconds later as `6a3cb437`. |
| Local validation example | `uv run pytest` over pipeline stage/coordinator and active-documentation/ADR tests: 85 passed; `git diff --check` passed. |
| PR #2596 audit | Reviews crossed exact heads `29b1f41b`, `d5f9307a`, `a4624238`, and `81331951`; the loop recorded NOGO → GO → NOGO → GO. The required-checks gate completed at `02:40:10Z` after final GO at `02:30:28Z`, and merge commit `57468dfe` followed at `02:40:49Z`. |
| CI boundary | Required checks supplied merge eligibility only; review comments and grades remained non-authorizing evidence. |
| Historical-policy migration | Preserve accepted ADRs; record the new label-only rule in a superseding ADR and its index entry. |
| PR handoff summary | Host-generated deterministic summary; never copy agent-local paths, uncommitted-state claims, or unsupported test totals. |
| Changed-test receipt | One bounded `uv run pytest -o addopts= <new-side tests/unit path> -q --tb=short` receipt per changed, non-deleted, hermetic unit-test file. |
| Format classification | Ruff format exit 1 with `unformatted:` or `<N> files would be reformatted` → `validation` → implementation remediation; bootstrap/sandbox faults remain fail-closed. |
| PR #2614 audit | Final head `c1d1b6df`; pre-push receipt `7152 passed, 11 skipped, 5 deselected; 84.74% coverage`; required checks passed; merge commit `375852c3`. |
| PR #2602 / issue #2284 audit | Earlier review/skip state was not reused. Final replies were attached to head `0674b3f2dab6bf5eacc316d83cbe68395ea7539a`; GO was labeled at `18:59:11Z`, slow unit checks completed by `19:06:55Z`, `required-checks-gate` passed at `19:07:04Z`, and conditional merge commit `2232cc42` landed at `19:07:16Z`. |
| PR #2604 / issue #2330 audit | Earlier review head `9275b973` produced NOGO. Final review bound to `76ab9692` at `22:10:45Z`; all 3 threads were resolved; `required-checks-gate` passed at `22:15:13Z`; GO was applied at `22:15:46Z`, NOGO removed at `22:15:48Z`, and conditional merge `b9a27e62` followed at `22:15:59Z`. The active ruleset still enforced `required_signatures` and required `pr-policy`; native auto-merge was absent. |
| Reply batch identity | `pr_number` + `head_sha` + unique `batch_nonce` + thread-ID-to-reply map + `thread_snapshot_sha256`; any PR, head, or snapshot mismatch invalidates replay. |
| PR #2612 / issue #2361 audit | Five head-bound batches covered 31 replies across changing revisions. A same-head follow-up review replaced interim GO with NOGO. The final review targeted `db9bd1ef`; all 37 threads were resolved, GO replaced NOGO at `01:19:49Z` / `01:19:51Z`, `required-checks-gate` completed at `01:26:15Z`, and conditional merge `d9d53fa0` followed at `01:27:15Z`; native auto-merge was null. |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | PR #2280 / issues #2053 and #2276 | CI-free source review and loop-owned `state:implementation-go` authorization. The direct repository-wide PR route now supplies PR context to strict review. Local swarm review then found that dynamic review-payload preservation could retain an aliased proof or survive a NOGO retry; a fixed allowlist removes those fields only after the label's current-head readback. Local verification only; no CI/CD state was queried. |
| ProjectHephaestus | PR #2306 / issue #2177 | Docs PR that reached merged state through the normal review-to-merge path: review GO, loop-owned `state:implementation-go`, and merge_wait. Post-merge `gh pr view` showed `state=MERGED` with `autoMergeRequest=null`, confirming reruns must short-circuit on terminal PR state. |
| ProjectHephaestus | PR #2344 / issue #2232 | Repeated NOGO reviews stayed fail-closed: unproven “6474 passed” prose, read-only inability to run the writable-temp integration test, an ambient Git config injection hazard, and Ruff formatting were treated as blockers. After fixes and current-head required checks passed, the PR completed the normal review/label/merge path at `2026-07-27T09:42:33Z` with merge commit `03e31fcd4e0af48ab5ccdbabda2b40c9b460fa3c`. Verified in CI. |
| ProjectHephaestus | PR #2508 / issue #2507 | A review correction distinguished two clean detached states: unchanged reviewed head (no publish, no branch release) and agent-precommitted head (publish exact `HEAD` through the reviewed-SHA lease). The corrected head `55300bb4` passed required checks, received a fresh current-head review and `state:implementation-go`, then merged as `6a3cb437`. Verified in CI. |
| ProjectHephaestus | Issue #2157 / PR #2596 | Verified-ci iterative review/merge path. Five initial findings produced NOGO; an addressed review produced GO; a later `create_pr` finding retracted GO to NOGO; final exact-head review restored GO, required checks completed, and merge-wait produced `57468dfe`. |
| ProjectHephaestus | Issue #2613 / PR #2614 | Verified-ci handoff path. Host-owned PR text removed agent-local summaries; changed unit-test pytest receipts used new-side diff markers and excluded deleted/non-hermetic tests; Ruff format diagnostics became remediation validation. Final head `c1d1b6df` passed required checks and merge-wait produced `375852c3`. |
| ProjectHephaestus | Issue #2284 / PR #2602 | Verified-ci review revival and merge-wait path. Earlier no-go/skip state and a no-commit reply did not authorize the PR; a fresh review bound GO to final head `0674b3f2`, required checks completed afterward, and merge-wait produced `2232cc42`. |
| ProjectHephaestus | Issue #2330 / PR #2604 | Verified-ci policy-simplification path. Duplicate signature CI and the advisory auto-merge job were removed while the active ruleset retained `required_signatures` and `pr-policy`. Final-head review, complete threads, exclusive GO, required checks, and a SHA-conditional normal merge remained the authoritative sequence. |
| ProjectHephaestus | Issue #2361 / PR #2612 | Verified-ci batched-remediation path. Each issue-comment handoff bound its reply set to the exact head and a thread-snapshot digest. A later same-head review safely reversed transient GO to NOGO. Final head `db9bd1ef8b80311564755e5b5fc5dbf5093603dc` reached zero unresolved threads before exclusive GO and conditional squash merge `d9d53fa0d522ee8799b67686830f81cf1f1ff17c`. |
