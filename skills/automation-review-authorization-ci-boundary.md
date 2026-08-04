---
name: automation-review-authorization-ci-boundary
description: "Keep automation-loop source-review authorization independent of CI/CD and bind direct-PR review to metadata-only admission plus an exact head. Use when: (1) strict review is mistakenly delegated to CI, (2) a direct --prs seed still expects a retired remote diff, (3) review can race with a changed head, (4) review prose or CI is being treated as merge authorization, (5) repeated NOGO rounds need fail-closed evidence handling, or (6) a later finding must retract an earlier GO before exact-head merge-wait."
category: architecture
date: 2026-08-03
version: "2.2.0"
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
  - iterative-review
  - review-batch
---

# Automation Review Authorization: CI Boundary

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-08-03 |
| **Objective** | Keep strict source-review authorization in the automation loop, ensure direct PR admission carries only stable metadata until an exact-head checkout derives the diff, and require each review/address round to re-authorize the final head before merge-wait. |
| **Outcome** | ProjectHephaestus issue #2157 / PR #2596 demonstrated the iterative fail-closed path: five initial major findings led to NOGO, an addressed review led to GO, a later finding retracted GO to NOGO, and a final exact-head review led to GO before merge-wait. |
| **Verification** | verified-ci — the final review targeted head `81331951`; GO was recorded at `02:30:28Z`, NOGO was removed at `02:30:30Z`, required checks completed, and merge commit `57468dfe` landed at `02:40:49Z`. |
| **History** | [changelog](./automation-review-authorization-ci-boundary.history) |

## When to Use

- A strict source review has been added as a required CI check even though the automation loop owns the decision.
- A direct `--prs` seed or adapter still indexes `pr_diff` after the review-context contract became metadata-only.
- A PR diff is fetched remotely while a push may occur, or a reviewer is not bound to a verified checkout at the GitHub head.
- A reviewer grade, comment, status check, artifact, or workflow result is being treated as authorization to advance or merge.
- `merge_wait` must decide whether a label is usable after a refresh, restart, or head change.
- A downstream rerun sees a merged PR and incorrectly expects `autoMergeRequest` or an open-PR review path.
- A direct issue seed can reach planning/review with missing or placeholder title/body context; exact-head review is not sufficient if the artifact was evaluated against the wrong requirements.
- A PR needs multiple review/address rounds and each later finding must invalidate earlier authorization.
- A completed run needs an event audit that separates review evidence, GO/NOGO label transitions, required checks, and the final merge.

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
  5. Treat each address/review batch as a fresh authorization attempt; retract GO to NOGO
     when a later finding or head change invalidates the prior review.
  6. Apply `state:implementation-go` and verify the exclusive label/readback.

exact-head merge path
  7. `merge_wait` consumes only the process-local reviewed-head proof and conditionally
     merges that exact head. It does not mutate native auto-merge.
  8. On head drift, missing proof, or merged state, route/reclassify from fresh live state;
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

3. For direct issue seeds, carry the original issue title and body from discovery into the
   work-item payload and then into initial and amended planning prompts. Do not rely on an ad hoc
   stage refetch or let a correct PR head compensate for missing requirements context. Fence the
   hydrated issue/advice/review text before agent dispatch so the reviewer sees the real task as
   data, not executable instructions.

4. Run strict review as an in-loop, CI-free operation against the checkout-derived evidence.
   If GitHub identity, head, issue linkage, or thread data cannot be verified, record a
   blocked/retry outcome and do not apply an implementation label. Review prose and grades
   are audit evidence, not authorization.

5. Before advancing, re-read the live PR and require it to remain open with the exact head
   that was reviewed. Apply the loop-owned `state:implementation-go` label only after an
   explicit GO and successful structural/thread checks; verify the exclusive label state by
   readback. Required CI checks remain independent validation context.

   A prior GO is not terminal: if a later review cycle finds another major issue, retract
   `state:implementation-go`, apply NOGO, and repeat the addressed-head review. In PR #2596,
   the initial five findings were addressed before GO, then a later `create_pr` finding forced
   another NOGO before final head `81331951` was reviewed and authorized.

6. Keep the reviewed-head proof in active-run memory through the handoff. `merge_wait` must
   reject missing or drifted proof and route back to PR review. With valid proof, use the
   normal conditional merge operation for that exact head; queue stages do not enable,
   disable, adopt, or poll native auto-merge.

7. Treat a merged PR as terminal before looking for open-PR authorization fields. GitHub may
   clear `autoMergeRequest` after merge, so a downstream rerun should short-circuit on
   non-`OPEN` state rather than interpret the cleared field as a new blocker.

8. Validate the contract at the seams: direct-seed routing, hydrated issue context, current metadata-only adapter
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
| Let a prior GO survive a later review finding | PR #2596 reached GO after the first five findings were addressed, then a later review found a remaining `create_pr` title-normalization gap | The earlier authorization no longer described the complete implementation or final review state | Retract GO to NOGO, address the finding, and run a fresh exact-head review before restoring GO |
| Reuse authorization after merge or head drift | Let a rerun consume stale approval fields | Merged PRs are no longer open and GitHub can clear `autoMergeRequest`; a changed head invalidates the old review | Re-read terminal state/head and short-circuit or re-review before any merge action |

## Results & Parameters

| Item | Result |
|------|--------|
| Incident | Hephaestus issue `#2448`, PR `#2449`; direct `--prs` seeding failed on stale `pr_diff` access |
| Correction | Removed `pr_diff` from direct-seed and `SeedEntry` paths; aligned the metadata-only fixture; retained checkout-derived diff and ABA regression coverage |
| Review evidence | Initial API-unreachable review was blocked; the later live-facts review returned `A`/`LGTM` and remained informational |
| Direct-issue context | Hydrate issue title/body at seed time and carry it through `WorkItem.payload` into initial/amended planning prompts; exact-head review does not authorize an artifact built from blank or placeholder requirements |
| Authorization path | Fresh live facts → exact reviewed head → `state:implementation-go` readback → exact-head `merge_wait` conditional merge |
| Merge result | PR #2449 merged at `2026-07-25T21:54:14Z` as `35175ab159c2b3a0e50681be26e9d8ec6855b92b` |
| Validation | `uv run --all-extras --locked pytest tests/unit/automation/pipeline -q --no-cov`; `uv run --all-extras --locked pytest tests/unit/automation/test_pipeline_github.py -q --no-cov`; targeted Ruff; pre-push full suite: 6798 passed, 6 skipped, 85.20% coverage |
| PR #2590 audit | Reviews on `4854a87c`, `76b3420c`, and `717462f5` crossed successive heads. Only the final review matched live head `717462f5`; GO was added at `21:29:07Z`, NOGO was removed at `21:29:09Z`, required checks completed before merge, and merge commit `dad87bbf` landed at `21:35:16Z`. Earlier receipts and CI remained non-authorizing evidence. |
| PR #2596 audit | Four exact-head review records crossed `29b1f41b`, `d5f9307a`, `a4624238`, and `81331951`. The loop recorded NOGO → GO → NOGO → GO; only the final head reached merge-wait, and merge commit `57468dfe` followed at `02:40:49Z`. |
| CI boundary | PR #2596's required-checks gate completed at `02:40:10Z`, after final GO at `02:30:28Z`; checks were merge eligibility, not the source-review authorization. |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #2448 / PR #2449 | Direct metadata-only seed, checkout-derived review diff, live-facts review retry, loop-owned implementation label, and exact-head merge path. Required CI passed and the PR merged. |
| ProjectHephaestus | Issue #1950 / PR #2590 | Verified-ci direct-issue context and exact-head merge path. Seeded issue title/body context was preserved into planning payloads and fenced prompts; successive reviews on superseded heads did not authorize the final revision. The final review matched `717462f5`, the loop-owned GO label followed, the incompatible NOGO label was removed, required checks completed, and merge commit `dad87bbf` followed. |
| ProjectHephaestus | Issue #2157 / PR #2596 | Verified-ci iterative review/merge path. Five initial major findings produced NOGO; one addressed review produced GO; a later `create_pr` finding retracted GO to NOGO; the final review matched `81331951`, GO was restored, required checks completed, and merge-wait produced `57468dfe`. |
