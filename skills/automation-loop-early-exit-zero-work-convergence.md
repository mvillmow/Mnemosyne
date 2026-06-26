---
name: automation-loop-early-exit-zero-work-convergence
description: "How to wire and test early-exit in hephaestus-automation-loop when a full pass produces zero new work, keep the human-facing per-loop summary consistent with the machine convergence signal, and review plans that centralize work-report write-on-exit behavior. Use when: (1) implementing loop convergence detection in run_loop, (2) adding tests for early-exit stub placeholders, (3) diagnosing whether early-exit scaffold already exists before implementing, (4) auditing a per-loop/iteration summary that contradicts the early-exit message, (5) reviewing a plan that extracts HEPH_WORK_REPORT handling into a context manager."
category: tooling
date: 2026-06-26
version: "1.2.0"
user-invocable: false
verification: unverified
history: automation-loop-early-exit-zero-work-convergence.history
tags: [automation, loop-runner, early-exit, convergence, testing, summary, work-units, work-report, contextmanager, planning-risk, planner]
---

# Automation Loop Early-Exit on Zero-Work Pass

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 (v1.2.0); 2026-06-21 (v1.1.0); 2026-05-28 (v1.0.0) |
| **Objective** | Break `run_loop` when a full pass produces 0 new work, keep the human-facing per-loop summary (`_summarize_loop`) reading the same `work_units` signal the early-exit predicate reads, and review plans that centralize `HEPH_WORK_REPORT` write-on-exit behavior |
| **Outcome** | v1.0.0/v1.1.0 implementation guidance succeeded locally; v1.2.0 adds an unexecuted plan-review risk checklist for issue #1390 |
| **Verification** | unverified for v1.2.0 planning-risk addition; earlier v1.1.0 content was verified-local |
| **History** | [changelog](./automation-loop-early-exit-zero-work-convergence.history) |

## When to Use

- Implementing early-exit / convergence detection in an automation loop driver
- Filling in `TestRunLoopEarlyExit` stub test placeholders that say "Deferred to implementation phase"
- Before implementing: check whether the scaffolding (`PhaseResult.work_units`, `RepoResult.produced_work`, `_CONVERGENCE_PHASES`, `break` block) already exists from a prior issue
- Adding new loop-convergence phases (must add to `_CONVERGENCE_PHASES` AND call `write_work_report`)
- **Auditing a human-facing per-loop / per-iteration summary** (e.g. `_summarize_loop`) that contradicts the early-exit / convergence message — grep for `+= 1` next to a phase-name check; it likely counts "phase ran" instead of "phase did work"
- Deciding whether a fully-filtered issue set should warn or stay silent (explicit `--issues` vs auto-discovery)
- Reviewing or implementing a refactor that extracts repeated `HEPH_WORK_REPORT` handling into a context manager such as `work_report_context(work_units_fn)`
- Checking whether all meaningful planner / plan-reviewer returns, exceptions, rate-limit exits, and `KeyboardInterrupt` paths are inside the report-writing context
- Updating tests that previously patched direct `write_work_report` imports after the symbol moves behind a shared context helper

## Verified Workflow

> **Warning:** The v1.2.0 work-report context-manager review checklist is unverified. It was derived
> from an implementation plan, not from an executed patch or CI run. Treat that checklist as a
> hypothesis until tests and the loop runner validate the behavior.

### Quick Reference

```python
# Check if scaffolding already exists before implementing:
grep -n "produced_work\|work_units\|_CONVERGENCE_PHASES\|early.exit" hephaestus/automation/loop_runner.py

# Pattern: mock process_repo for run_loop tests
from unittest.mock import patch
from hephaestus.automation import loop_runner
from hephaestus.automation.loop_runner import LoopConfig, PhaseResult, RepoResult, run_loop

def _zero_work_result(repo, loop_idx):
    rr = RepoResult(repo=repo, loop_idx=loop_idx)
    rr.phases.append(PhaseResult(name="plan", rc=0, work_units=0))
    rr.phases.append(PhaseResult(name="review-plans", rc=0, work_units=0))
    return rr

def test_early_exit_fires(tmp_path):
    cfg = LoopConfig(loops=5, projects_dir=tmp_path)
    (tmp_path / "r1" / ".git").mkdir(parents=True)
    call_count = 0
    def fake(repo, loop_idx, cfg):
        nonlocal call_count; call_count += 1
        return _zero_work_result(repo, loop_idx)
    with patch.object(loop_runner, "process_repo", side_effect=fake):
        results = run_loop(cfg, repos=["r1"])
    assert max(r.loop_idx for r in results) == 1  # stopped after loop 1
    assert call_count == 1
```

### Detailed Steps

1. **Audit existing scaffold** — Before writing any code, grep for `produced_work`, `work_units`, `_CONVERGENCE_PHASES`, and the `break` block. In ProjectHephaestus the scaffolding was added in issue #613; issue #614 only needed to complete test stubs and clean up the implementation.

2. **Understand `_CONVERGENCE_PHASES`** — Only phases in this frozenset count toward early-exit. Currently `{"plan", "review-plans"}`. New phases that call `write_work_report` must be added here.

3. **The early-exit condition** in `run_loop` (correct form):
   ```python
   if (
       loop_idx < cfg.loops                              # never on final loop
       and not any(r.any_failure for r in loop_results)  # failures block exit
       and not any(r.produced_work for r in loop_results) # all repos zero work
   ):
       LOG.info("Early exit after loop %d/%d ...", loop_idx, cfg.loops, len(repos))
       break
   ```
   Note: `loop_results` is already filtered to the current loop — no re-filter from `all_results` is needed.

4. **Implement the six test scenarios** (see Results & Parameters for all six).

5. **Run checks**:
   ```bash
   pixi run ruff check hephaestus/automation/loop_runner.py
   pixi run pytest tests/unit/automation/ -q --no-cov
   ```

### Proposed Workflow: Work-Report Context Manager Plan Review (v1.2.0, UNVERIFIED)

When a plan centralizes `HEPH_WORK_REPORT` behavior into a context manager such as
`work_report_context(work_units_fn)`, review it as an exit-path contract change, not as a
mechanical DRY cleanup.

### Quick Reference: Work-Report Context Risk Audit

```bash
# 1. Locate every direct report writer/import that should move behind the shared context.
rg -n "HEPH_WORK_REPORT|write_work_report|work_report_context" hephaestus tests pyproject.toml

# 2. For each CLI main(), list returns/exits before approving the context placement.
rg -n "return |sys\\.exit|KeyboardInterrupt|rate.limit|rate-limit" hephaestus/automation/planner.py hephaestus/automation/plan_reviewer.py

# 3. Confirm tests assert the observable report file, not only a patched helper call.
rg -n "HEPH_WORK_REPORT|write_work_report|work_units|work_report" tests/unit/automation

# 4. Type-check the context-manager signature if it uses contextlib.contextmanager.
pixi run mypy hephaestus/
```

### Detailed Review Checklist

1. **No-op semantics must stay exact when `HEPH_WORK_REPORT` is absent.** The context should avoid
   creating files, evaluating `work_units_fn`, or changing control flow unless reporting was active at
   entry.
2. **Context placement must cover every meaningful exit path.** For planner and plan-reviewer CLIs,
   verify issue-discovery rate-limit returns, normal returns, exception paths, and
   `KeyboardInterrupt` handling are all inside the `with work_report_context(...)` block.
3. **`work_units = 0` must be an explicit sentinel decision.** Rate-limit, pre-run failure, and
   interrupt paths can reasonably mean "no completed work" once reporting is active, but the plan must
   state that contract and align it with the loop runner's missing-report = unknown/work convention.
4. **Do not let `work_units_fn` mask the original failure.** If the callback is evaluated in `finally`,
   review whether callback exceptions are caught/logged or otherwise prevented from replacing the
   exception that caused process exit.
5. **Entry-time environment capture is a contract.** Reading `HEPH_WORK_REPORT` once on context entry
   makes the target path stable for the process lifetime. If the environment changes later, that change
   should not silently redirect the report unless the plan explicitly chooses dynamic lookup.
6. **Patch-target tests need a migration audit.** Replacing module-level `write_work_report` imports can
   break tests or downstream monkeypatches that patch `planner.write_work_report` or
   `plan_reviewer.write_work_report`. Grep beyond the two modified modules and update tests to patch
   the new lookup site only when patching is still the right surface.
7. **Prefer observable report-file assertions for regression coverage.** Tests that inspect the
   generated `HEPH_WORK_REPORT` file prove the public loop-runner contract. Patched helper-call
   assertions are still useful for narrow unit tests, but they can overfit to import placement.

## Keep the Human Summary Consistent with the Convergence Signal (v1.1.0)

The early-exit predicate above reads `work_units` / `produced_work`. The
human-facing per-loop summary (`_summarize_loop` in
`hephaestus/automation/loop_runner.py`) MUST read the **same** signal — otherwise
the two silently disagree.

### The bug (real 2026-06-21 output.log)

A run scoped to `--issues 123,456,789,101` (all CLOSED) logged:

```text
loop 1: planned=4 implemented=4 skipped=0
Early exit ... produced 0 new plans
```

Two adjacent lines contradict each other. Nothing actually failed — the planner
correctly filtered all closed issues — but the summary inflated the count,
masking a fully no-op'd / mis-scoped run.

### Root cause

`_summarize_loop` incremented `total_planned += 1` for any non-skipped plan
phase, never consulting `phase.work_units`. The `plan` phase DOES populate
reliable `work_units` via the work-report file
(`planner.py` `write_work_report` → `loop_runner.py` `_read_work_report` →
`PhaseResult.work_units`). The `implement` phase writes NO work-report, so its
`work_units` is always `None`.

### The fix

1. **Count plan by `work_units`, not "ran":**

   ```python
   # _summarize_loop — count actual work, conservatively
   total_planned += phase.work_units if phase.work_units is not None else 1
   ```

   The `is not None → else 1` fallback matches `produced_work`'s "unknown counts
   as work, conservatively" convention (`loop_runner.py:224-225`). Leave
   `implement` as-is (no work-report to count).

2. **Warn on a fully-filtered EXPLICIT issue set** (mis-scoped run made obvious):

   ```python
   # PlannerStateManager.filter (planner_state.py)
   if options.issues_explicit and not remaining:
       LOG.warning("All %d explicitly-requested issues filtered out "
                   "(closed / already-planned) — nothing to plan", n_requested)
   ```

   Gated by a new `PlannerOptions.issues_explicit` flag set in `planner.main()`
   **before** auto-discovery overwrites `args.issues`. Auto-discovery stays quiet
   — a converged repo legitimately empties every pass, and re-introducing a
   warning there would just be loop spam.

### Audit heuristic

When a loop/iteration summary disagrees with the convergence message, grep for
`+= 1` adjacent to a phase-name check. "Phase ran" (`rc == 0`, not skipped) is
NOT "phase did work". Any human-facing summary must read the same `work_units` /
`produced_work` signal as the early-exit predicate.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Redundant loop_results reassignment | `loop_results = [r for r in all_results if r.loop_idx == loop_idx]` was placed after the pool block | `loop_results` already contains only current-loop results; re-filtering from `all_results` is harmless but misleading | Remove the redundant line; rely on the list built during the pool phase |
| Checking `review-prs` in convergence | Assumed `review-prs` should be a convergence phase | `review-prs` is not in `_CONVERGENCE_PHASES` — it is not instrumented with `write_work_report`; adding it would require also calling `write_work_report` in that phase | Only phases that actively call `write_work_report` belong in `_CONVERGENCE_PHASES` |
| Summary counted "phase ran" not "work done" | `_summarize_loop` did `total_planned += 1` for any non-skipped plan phase | Diverged from the convergence predicate, which reads `work_units` — produced `planned=4` directly above `Early exit ... produced 0 new plans` | Any human-facing loop summary must read the SAME signal (`work_units`/`produced_work`) the early-exit predicate reads; grep for `+= 1` next to a phase check when auditing |
| Trusting `work_units` for the `implement` phase | Considered counting `implement` by `work_units` too | `implement` writes no work-report, so its `work_units` is always `None`; counting it by `work_units` would always fall to the conservative `else 1` and tell you nothing | `work_units` reliability is per-phase: only `plan` reports it. Trust `work_units` only for instrumented phases; fall back conservatively otherwise |
| Warning on every empty pass | Considered warning whenever `filter()` returns nothing | A converged repo under AUTO-discovery legitimately empties every pass → re-introduces loop spam | Warn only when an EXPLICIT `--issues` set fully filters out. Capture the explicit-vs-discovered distinction at the CLI boundary (`planner.main`, before discovery overwrites `args.issues`), since by `filter()` time `options.issues` is always populated either way |
| Context starts after an early return | Plan assumes all meaningful early returns happen after `with work_report_context(...)` starts | Any return before the context preserves the old missing-report behavior, which the loop runner treats as unknown/work rather than zero work | Trace every `return` / `sys.exit` in `main()` and prove rate-limit / interrupt / pre-run paths are inside the context before approving |
| Callback failure in `finally` ignored | Plan evaluates `work_units_fn()` during context exit without stating what happens if the callback raises | A callback exception can mask the original exception or convert a clean exit into a failure | Wrap callback/report-write failures deliberately, or document why fail-fast is desired; do not leave masking behavior accidental |
| Patch-call assertions as the only regression surface | Tests patch `write_work_report` and assert the call rather than reading the report file | The public loop-runner contract is the report artifact, not the internal helper binding; import refactors can make patched-call tests brittle | Prefer direct `HEPH_WORK_REPORT` file assertions for CLI/reporting behavior, with patch tests only for narrow helper internals |

## Results & Parameters

### Six Required Test Scenarios for `TestRunLoopEarlyExit`

| Test | Scenario | Key Assertion |
|------|----------|---------------|
| `test_early_exit_fires_on_zero_work_pass` | loops=5, all repos return work_units=0 | `max(r.loop_idx) == 1`, call_count == 1 |
| `test_loops_caps_when_work_continues_every_loop` | loops=3, work_units=3 every iteration | `max(r.loop_idx) == 3`, len(results) == 3 |
| `test_no_early_exit_when_failure_present` | loops=3, phase rc=1 + work_units=0 | all 3 loops run (failure blocks exit) |
| `test_early_exit_skipped_on_final_loop` | loops=1, zero work | exactly 1 result; guard `loop_idx < cfg.loops` prevents exit on final |
| `test_unknown_work_units_prevents_early_exit` | loops=3, work_units=None | all 3 loops run (conservative: None → True) |
| `test_early_exit_multi_repo_requires_all_zero` | loops=5, r1 productive + r2 zero | all 5 loops run because r1 has work |

### Helper Factories (copy-paste into test file)

```python
def _zero_work_result(repo, loop_idx):
    rr = RepoResult(repo=repo, loop_idx=loop_idx)
    rr.phases.append(PhaseResult(name="plan", rc=0, work_units=0))
    rr.phases.append(PhaseResult(name="review-plans", rc=0, work_units=0))
    return rr

def _work_result(repo, loop_idx, work_units=3):
    rr = RepoResult(repo=repo, loop_idx=loop_idx)
    rr.phases.append(PhaseResult(name="plan", rc=0, work_units=work_units))
    rr.phases.append(PhaseResult(name="review-plans", rc=0, work_units=0))
    return rr

def _failed_result(repo, loop_idx):
    rr = RepoResult(repo=repo, loop_idx=loop_idx)
    rr.phases.append(PhaseResult(name="plan", rc=1, work_units=0))
    rr.phases.append(PhaseResult(name="review-plans", rc=0, work_units=0))
    return rr

def _unknown_work_result(repo, loop_idx):
    rr = RepoResult(repo=repo, loop_idx=loop_idx)
    rr.phases.append(PhaseResult(name="plan", rc=0, work_units=None))
    return rr
```

### Net Change (v1.0.0)

- `hephaestus/automation/loop_runner.py`: 14 LOC net (removed redundant reassignment, improved log message with loop/total/repo count)
- `tests/unit/automation/test_loop_runner_early_exit.py`: +137 LOC net (6 stubs → 6 real tests + 4 helper factories)

### Summary/Convergence Consistency (v1.1.0 — PR #1564 / issue #1563)

- `_summarize_loop` (`loop_runner.py`): count plan via
  `total_planned += phase.work_units if phase.work_units is not None else 1`
  (conservative fallback mirrors `produced_work` at `loop_runner.py:224-225`);
  `implement` left as-is.
- `PlannerStateManager.filter` (`planner_state.py`): WARNING when an explicit
  `--issues` set fully filters out; gated by new `PlannerOptions.issues_explicit`
  set in `planner.main()` before auto-discovery overwrites `args.issues`.
- **Verification (verified-local):** `pixi run pytest tests/unit/automation`
  → 1851 passed; `ruff check` + `ruff format --check` + `mypy` (409 files) all
  clean. End-to-end:
  `pixi run hephaestus-automation-loop --dry-run --loops 1 --issues 123,456,789,101 -v`
  now logs `loop 1: planned=0 ...` plus the all-filtered WARNING per issue.
  CI for PR #1564 was still pending at capture time (NOT verified-ci).

### Work-Report Context Manager Plan-Risk Checklist (v1.2.0 — issue #1390 plan review)

**Verification level: unverified.** This was derived from an implementation plan for a small
ProjectHephaestus refactor and was not executed. Use it as a reviewer checklist until a downstream PR
proves the contract.

Most uncertain assumptions to challenge:

- `HEPH_WORK_REPORT` is intentionally read only on context entry, not dynamically at exit time.
- All meaningful planner and plan-reviewer early returns happen after the proposed context starts.
- `work_units = 0` is the right sentinel for rate-limit, `KeyboardInterrupt`, and pre-run failure paths
  once reporting is active.
- Replacing module-level `write_work_report` imports will not break tests or monkeypatch expectations
  outside the call sites found by `rg`.
- Direct report-file assertions are the better regression surface than patched `write_work_report`
  call assertions.

Unverified dependencies / sources from the plan:

- Relied on `rg` search over `hephaestus/`, `tests/`, and `pyproject.toml`; no implementation was run.
- Relied on file paths and line numbers from the checkout at planning time; re-anchor by symbol before
  editing.
- Relied on loop-runner convention that a missing report means unknown/work.
- Listed `pytest`, `ruff`, and `mypy` expectations without running them.
- Did not verify CI behavior or the external GitHub issue context beyond the supplied plan.

Reviewer focus:

```text
- Does the context manager write on exception and return paths while preserving no-op behavior when
  HEPH_WORK_REPORT is absent?
- Is planner issue-discovery rate-limit return inside the reporting context?
- Is plan_reviewer KeyboardInterrupt handling inside the reporting context?
- Can work_units_fn() in finally mask the original exception?
- Is entry-time HEPH_WORK_REPORT capture intentional if the env var mutates mid-run?
- Are there remaining direct imports or patch-based tests targeting a removed write_work_report symbol?
- Does mypy accept contextlib.contextmanager + Callable[[], int] + Iterator[None]?
```

### Related Skills

- [[automation-loop-post-loop-filter-omission]] — another caller-side
  aggregation/filter divergence (per-loop vs post-loop results).
- [[automation-loop-log-driven-layered-debug]] — the output.log methodology that
  surfaced this divergence.

> Context note: the GraphQL batch-fetch single-quote bug from an OLDER
> 2026-06-10 log was already fixed by ProjectHephaestus #1148 (regression test
> present) — not part of this learning.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #614 — early-exit loop on zero-work pass | PR #669; 736 automation tests pass locally; all pre-commit hooks pass |
| ProjectHephaestus | Issue #1563 — summary/convergence divergence (`_summarize_loop` counted "ran" not "work done") | PR #1564; 1851 automation tests pass locally; ruff/mypy clean; CI pending |
| ProjectHephaestus | Issue #1390 — plan for `work_report_context(work_units_fn)` refactor | unverified; plan-review risk capture only, no implementation or CI run |
