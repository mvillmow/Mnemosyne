---
name: automation-review-authorization-ci-boundary
description: "Keep automation-loop source-review authorization independent of CI/CD and bind direct-PR review to metadata-only admission plus an exact head. Use when: (1) strict review is mistakenly delegated to CI, (2) a direct --prs seed still expects a retired remote diff, (3) review can race with a changed head, (4) review prose or CI is being treated as merge authorization, or (5) a downstream rerun sees a merged PR."
category: architecture
date: 2026-07-25
version: "2.0.0"
user-invocable: false
verification: verified-ci
history: automation-review-authorization-ci-boundary.history
tags:
  - automation-loop
  - pr-review
  - ci-independent
  - authorization-boundary
  - direct-pr-seed
  - metadata-only-seed
  - mutable-diff
  - implementation-go
  - reviewed-head
  - checkout-verification
  - merge-wait
  - exact-head-merge
  - fail-closed
---

# Automation Review Authorization: CI Boundary

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-07-25 |
| **Objective** | Keep strict source-review authorization in the automation loop, ensure direct PR admission carries only stable metadata until an exact-head checkout derives the diff, and bind merge-wait to that reviewed head. |
| **Outcome** | ProjectHephaestus issue #2448 / PR #2449 fixed a stale direct-seed lookup of the retired `pr_diff` field. The merged path was metadata-only admission, exact-head checkout-derived review evidence, fresh live review facts, `state:implementation-go`, and exact-head merge-wait. |
| **Verification** | verified-ci — PR #2449's required checks passed, including `pr-policy`, unit/integration tests, lint, build, and security checks; it merged as `35175ab1` on 2026-07-25. |
| **History** | [changelog](./automation-review-authorization-ci-boundary.history) |

## When to Use

- A strict source review has been added as a required CI check even though the automation loop owns the decision.
- A direct `--prs` seed or adapter still indexes `pr_diff` after the review-context contract became metadata-only.
- A PR diff is fetched remotely while a push may occur, or a reviewer is not bound to a verified checkout at the GitHub head.
- A reviewer grade, comment, status check, artifact, or workflow result is being treated as authorization to advance or merge.
- `merge_wait` must decide whether a label is usable after a refresh, restart, or head change.
- A downstream rerun sees a merged PR and incorrectly expects `autoMergeRequest` or an open-PR review path.

## Verified Workflow

### Quick Reference

```text
direct-PR admission
  1. Seed only stable PR metadata; never include a mutable remote `pr_diff`.
  2. Prove a clean checkout at the exact head SHA, then derive the review diff locally.

loop-owned review authorization
  3. Run strict source review in the loop; unavailable GitHub facts are a block/retry,
     not a label decision.
  4. Require explicit GO plus fresh live PR state and an exact reviewed-head match.
  5. Apply `state:implementation-go` and verify the exclusive label/readback.

exact-head merge path
  6. `merge_wait` consumes only the process-local reviewed-head proof and conditionally
     merges that exact head. It does not mutate native auto-merge.
  7. On head drift, missing proof, or merged state, route/reclassify from fresh live state;
     never reuse stale review evidence.

CI/CD remains independent: checks may validate the PR, but they do not authorize the loop.
```

### Detailed Steps

1. Make direct-PR admission deterministic. A `--prs` selector should obtain the current PR
   identity, state, base, head, and requirements metadata only. Do not make `SeedEntry` or a
   fixture carry `pr_diff`; after the adapter stopped returning that field, the stale lookup
   caused a pre-review `KeyError` in issue #2448.

2. Establish the checkout barrier before review. Synchronize a clean worktree to the named
   head SHA, prove `HEAD` equals it, and derive the diff from the verified base/head pair.
   A remotely fetched diff is mutable evidence and can create an ABA mismatch even when a
   head-before/head-after comparison appears stable.

3. Run strict review as an in-loop, CI-free operation against the checkout-derived evidence.
   If GitHub identity, head, issue linkage, or thread data cannot be verified, record a
   blocked/retry outcome and do not apply an implementation label. Review prose and grades
   are audit evidence, not authorization.

4. Before advancing, re-read the live PR and require it to remain open with the exact head
   that was reviewed. Apply the loop-owned `state:implementation-go` label only after an
   explicit GO and successful structural/thread checks; verify the exclusive label state by
   readback. Required CI checks remain independent validation context.

5. Keep the reviewed-head proof in active-run memory through the handoff. `merge_wait` must
   reject missing or drifted proof and route back to PR review. With valid proof, use the
   normal conditional merge operation for that exact head; queue stages do not enable,
   disable, adopt, or poll native auto-merge.

6. Treat a merged PR as terminal before looking for open-PR authorization fields. GitHub may
   clear `autoMergeRequest` after merge, so a downstream rerun should short-circuit on
   non-`OPEN` state rather than interpret the cleared field as a new blocker.

7. Validate the contract at the seams: direct-seed routing, current metadata-only adapter
   fixtures, checkout-derived diff/ABA protection, label readback, exact-head merge-wait,
   and post-merge terminalization. Run the targeted tests and the repository-required gate;
   record whether evidence is local or CI-backed.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Retain `pr_diff` in direct seeding | Indexed `pr_review_context()["pr_diff"]` and preserved the old fixture shape | The adapter contract intentionally became metadata-only; direct `--prs` failed before review with `KeyError: 'pr_diff'` | Keep direct seeds metadata-only and derive mutable review evidence only after exact-head checkout |
| Review a remotely fetched diff | Passed a mutable remote diff into review before proving the checkout | A concurrent push can pair a stale/intermediate diff with a restored head | The checkout barrier and local base/head diff are the sole review-evidence source |
| Treat the first review grade as authorization | Accepted a review after GitHub API identity/head facts were unavailable | The first PR #2449 review was an informational `F` because live facts could not be verified; it could not authorize merge | Block or retry on unavailable live facts; only a fresh structural audit can authorize the label |
| Treat CI or review prose as the merge decision | Used successful checks or an `A` comment as the authority | CI and reviewer prose are evidence; the loop-owned label plus exact reviewed-head proof owns advancement | Keep authorization in the loop and use CI as independent validation |
| Reuse authorization after merge or head drift | Let a rerun consume stale approval fields | Merged PRs are no longer open and GitHub can clear `autoMergeRequest`; a changed head invalidates the old review | Re-read terminal state/head and short-circuit or re-review before any merge action |

## Results & Parameters

| Item | Result |
|------|--------|
| Incident | Hephaestus issue `#2448`, PR `#2449`; direct `--prs` seeding failed on stale `pr_diff` access |
| Correction | Removed `pr_diff` from direct-seed and `SeedEntry` paths; aligned the metadata-only fixture; retained checkout-derived diff and ABA regression coverage |
| Review evidence | Initial API-unreachable review was blocked; the later live-facts review returned `A`/`LGTM` and remained informational |
| Authorization path | Fresh live facts → exact reviewed head → `state:implementation-go` readback → exact-head `merge_wait` conditional merge |
| Merge result | PR #2449 merged at `2026-07-25T21:54:14Z` as `35175ab159c2b3a0e50681be26e9d8ec6855b92b` |
| Validation | `uv run --all-extras --locked pytest tests/unit/automation/pipeline -q --no-cov`; `uv run --all-extras --locked pytest tests/unit/automation/test_pipeline_github.py -q --no-cov`; targeted Ruff; pre-push full suite: 6798 passed, 6 skipped, 85.20% coverage |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #2448 / PR #2449 | Direct metadata-only seed, checkout-derived review diff, live-facts review retry, loop-owned implementation label, and exact-head merge path. Required CI passed and the PR merged. |
