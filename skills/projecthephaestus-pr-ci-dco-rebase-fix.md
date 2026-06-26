---
name: projecthephaestus-pr-ci-dco-rebase-fix
description: "Resolve ProjectHephaestus automation PR CI failures caused by stale main, DCO trailer gaps, and review-helper conflicts. Use when: (1) a ProjectHephaestus automation PR goes red after main moves, (2) pr-policy/DCO fails because an older commit lacks Signed-off-by, (3) rebasing review utility changes must preserve both main and issue-specific helpers."
category: ci-cd
date: 2026-06-26
version: "1.0.0"
user-invocable: false
verification: verified-ci
tags:
  - projecthephaestus
  - ci
  - dco
  - pr-policy
  - rebase
  - review-utils
---

# ProjectHephaestus PR CI DCO Rebase Fix

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture the CI-drive fix for ProjectHephaestus issue #1381 / PR #1612 after the branch fell behind main and carried an unsigned commit in the final PR range. |
| **Outcome** | Successful: PR #1612 was driven to green CI by rebasing onto current `origin/main`, preserving both review utility helpers, fixing the test import style, and replacing the final branch range with a DCO-compliant signed-off commit. |
| **Verification** | verified-ci based on the user's green-CI statement plus local git evidence; GitHub API was unavailable from the sandbox during capture, so PR status was not re-queried locally. |

## When to Use

- ProjectHephaestus automation PR CI fails after `origin/main` has moved and the PR branch is stale or conflicting.
- `pr-policy` or DCO fails because an earlier commit in `origin/main..HEAD` lacks a `Signed-off-by` trailer.
- Rebasing issue-specific automation helper extraction conflicts with new review utility helpers already present on main.
- Tests or lint fail after a helper extraction because the test import style no longer matches the surrounding file.

## Verified Workflow

### Quick Reference

```bash
git fetch origin
git rebase origin/main

# During conflicts in hephaestus/automation/_review_utils.py, keep both helpers:
# - _discover_prs_simple from current main
# - print_worker_summary from the issue branch

git log --format='%h %s%n%b' origin/main..HEAD
# Every commit in the final range must contain Signed-off-by.

# If any commit lacks DCO, rebuild or squash the final PR range and commit with -s.
git reset --soft origin/main
git commit -s -m "fix: Address CI failures for PR ProjectHephaestus#1612"
git push --force-with-lease origin HEAD:<pr-branch>
```

### Detailed Steps

1. Check whether the PR is stale or conflicting before editing code. For PR #1612, the branch `1381-auto-impl` needed to be rebased onto current `origin/main`.
2. During the rebase, resolve `_review_utils.py` by preserving both sides of the conflict:
   - Keep `_discover_prs_simple`, which already existed on main.
   - Keep the issue #1381 extraction, `print_worker_summary`.
3. Keep all call sites aligned with the shared helper. For PR #1612 this meant `address_review.py`, `ci_driver.py`, `plan_reviewer.py`, and `pr_reviewer.py` called `print_worker_summary`.
4. Match test import style to the existing file. In `tests/unit/automation/test_review_utils.py`, import `print_worker_summary` directly instead of adding a module alias such as `import hephaestus.automation._review_utils as review_utils`.
5. Check DCO on the complete final commit range, not just the newest fix commit. The original implementation commit `b94caf0` lacked `Signed-off-by`; leaving it under a signed follow-up still leaves `pr-policy` red.
6. If any commit in the final range lacks DCO, rewrite the branch into a signed-off range with `git commit -s` and push with `--force-with-lease`.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Add a signed follow-up over an unsigned implementation commit | Commit `5cd8f84` had `Signed-off-by`, but earlier commit `b94caf0` did not | DCO evaluates every commit in the PR range | Verify `origin/main..HEAD`; rewrite or squash until every final commit is signed off |
| Resolve the rebase by taking only one side of `_review_utils.py` | Either the main helper or the issue helper can be lost | Main had added `_discover_prs_simple` while issue #1381 needed `print_worker_summary` | Preserve both helpers during conflict resolution |
| Use a module alias in `test_review_utils.py` | `import hephaestus.automation._review_utils as review_utils` diverged from the file's direct-import style | Style/lint review required consistency with existing imports | Import `print_worker_summary` directly |
| Trust local GitHub status checks during capture | `gh pr view 1612` failed with `error connecting to api.github.com` | The sandbox could not reach GitHub API | Use local git evidence and state the API limitation; rely on separately observed green CI only when explicitly provided |

## Results & Parameters

- Repository context: `HomericIntelligence/ProjectHephaestus`, issue #1381, PR #1612, branch `1381-auto-impl`.
- Original implementation commit on the remote branch: `b94caf0 refactor(automation): share worker summary printing`; it extracted `print_worker_summary` and added tests, but had no `Signed-off-by` trailer.
- Follow-up remote commit: `5cd8f84 fix(tests): use direct review utils import`; it changed `tests/unit/automation/test_review_utils.py` to import `print_worker_summary` directly and included `Signed-off-by`.
- CI-drive commit: `aa05e914f7af3ad18ce36f06fe8897a1765d5ff7 fix: Address CI failures for PR ProjectHephaestus#1612`; it rebased onto main, preserved `_discover_prs_simple` plus `print_worker_summary`, updated the automation callers, added worker summary test coverage, and produced a DCO-compliant final branch head.
- Required helper conflict invariant: `_discover_prs_simple` and `print_worker_summary` must coexist in `hephaestus/automation/_review_utils.py` after the rebase.
- Worker summary tests covered empty, successful, mixed, custom PR count noun, and custom failure header cases.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1381 / PR #1612 CI drive | verified-ci by user statement; local evidence showed signed-off commit `aa05e914f7af3ad18ce36f06fe8897a1765d5ff7`, earlier unsigned `b94caf0`, and the direct-import follow-up `5cd8f84`; GitHub API unavailable locally during learn capture |
