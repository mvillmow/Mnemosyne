---
name: tooling-cli-batch-terminal-outcome-contracts
description: "Design honest batch CLI outcomes. Use when: (1) a command processes discovered items but returns success after blocked, failed, or unprocessed work, (2) JSON output changes shape on setup errors or interruption, (3) KeyboardInterrupt must stop a batch with exit 130 while ordinary item failures continue."
category: tooling
date: 2026-08-01
version: "1.0.0"
user-invocable: false
verification: unverified
tags:
  - python
  - cli
  - batch-processing
  - exit-codes
  - json
  - keyboard-interrupt
  - terminal-outcomes
  - dry-run
  - github
---

# Batch CLI Terminal Outcome Contracts

## Overview

| Field | Value |
| ------- | ------- |
| **Date** | 2026-08-01 |
| **Objective** | Give every discovered batch item an explicit terminal outcome, derive the command exit code from those outcomes, and emit one stable structured summary on every path |
| **Outcome** | Proposed design for manual batch drivers, including per-item failure isolation, interruption handling, dry-run classification, complete aggregation, and single-line untrusted details |
| **Verification** | unverified — derived from a reviewed ProjectHephaestus `hephaestus-merge-prs` implementation plan; implementation and CI validation are pending |

## When to Use

- A CLI discovers multiple pull requests, jobs, files, or other items and currently processes them with fire-and-forget returns.
- A per-item exception aborts the whole batch, or an exception is logged but the command still exits `0`.
- Some discovered items can be blocked or accidentally left unprocessed, and those conditions must make the command fail.
- `--dry-run` intentionally suppresses an otherwise eligible operation and should count as a successful, explicit outcome rather than missing work.
- `KeyboardInterrupt` must be distinguished from ordinary failures, stop later work, and return conventional exit code `130`.
- Machine consumers require the same JSON fields for success, partial failure, setup failure, and interruption before discovery.
- Provider text or exception messages may contain newlines or control characters that would split log records.

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```python
from dataclasses import dataclass
from enum import Enum


class _ItemStatus(str, Enum):
    SUCCEEDED = "succeeded"
    QUEUED = "queued"
    SKIPPED = "skipped"
    BLOCKED = "blocked"
    FAILED = "failed"
    INTERRUPTED = "interrupted"


def _escape_detail(value: object) -> str:
    return ascii(str(value))[1:-1]


@dataclass(frozen=True)
class _ItemOutcome:
    item_id: int | None
    status: _ItemStatus
    detail: str

    def __post_init__(self) -> None:
        object.__setattr__(self, "detail", _escape_detail(self.detail))


def _exit_code(outcomes: list[_ItemOutcome], requested: int) -> int:
    statuses = {outcome.status for outcome in outcomes}
    if _ItemStatus.INTERRUPTED in statuses:
        return 130
    if len(outcomes) != requested:
        return 1
    if statuses & {_ItemStatus.BLOCKED, _ItemStatus.FAILED}:
        return 1
    return 0
```

Stable JSON envelope on every path:

```json
{
  "status": "error",
  "exit_code": 1,
  "message": "batch complete",
  "results": [],
  "totals": {
    "succeeded": 0,
    "queued": 0,
    "skipped": 0,
    "blocked": 0,
    "failed": 0,
    "interrupted": 0
  },
  "requested": 0,
  "processed": 0
}
```

### Detailed Steps

1. **Keep outcome tracking private unless consumers require a public API.** Define a private string enum and frozen dataclass in the command module. The result object should carry the item identifier, one terminal status, and normalized detail text.

2. **Classify provider responses at the provider boundary.** Convert dict responses and legacy response objects into outcomes immediately. For a merge driver, useful statuses are `merged`, `queued`, `skipped`, `blocked`, `failed`, and `interrupted`; do not force merge-queue enrollment into the same label as a synchronous merge.

3. **Return an outcome on every ordinary item path.** Missing required metadata is `failed`; unmet eligibility is `blocked`; an intentional dry-run suppression is `skipped`; a confirmed operation is `succeeded` or its domain-specific equivalent. Let unexpected operational exceptions reach the batch boundary.

4. **Preserve side-effect order before refactoring return values.** When an option such as `--push-all` requires a side effect even for blocked items, retain the sequence: validate identity, evaluate eligibility, perform the unconditional option-owned side effect, return `blocked` if needed, otherwise avoid duplicate side effects and perform the terminal operation.

5. **Catch failures at the per-item batch boundary.** Catch `Exception`, append `failed` with the exception type and text, and continue to later items. Catch `KeyboardInterrupt` separately, append `interrupted` for the current discovered item, stop the loop, and do not process later items.

6. **Detect incomplete batches independently of named failures.** Exit `1` when `processed != requested`, even if no recorded outcome is explicitly `blocked` or `failed`. This closes silent-success paths caused by premature loop termination or future control-flow regressions.

7. **Use deterministic exit precedence.** Return `130` if setup or item processing was interrupted. Otherwise return `1` for blocked, failed, or incomplete work. Return `0` only when every requested item is in an allowed terminal set such as merged, queued, or intentionally skipped.

8. **Build totals from the complete enum.** Initialize every status to zero before counting. Stable keys matter: a status with no occurrences must still appear in machine-readable output, including failures before discovery.

9. **Emit one result record per item and one aggregate record.** Normalize details once with `ascii(str(value))[1:-1]` before both logging and serialization. This escapes newlines and control characters so untrusted provider or exception text cannot create extra physical log records.

10. **Route every JSON path through one summary emitter.** Always include `status`, `exit_code`, `message`, `results`, `totals`, `requested`, and `processed`. Setup/access/listing failures use an empty complete envelope. A pre-discovery interruption has no item to own an `interrupted` result, so represent it with exit code `130`, an interruption message, empty results, zero totals, and `requested=processed=0`.

11. **Test behavior and sequencing, not only helpers.** Cover all success, partial failure with continuation, all eligible items skipped by dry-run, blocked items, missing metadata, provider exceptions, queue enrollment, item interruption, setup/listing interruption, incomplete-batch detection, exact JSON dictionaries, zero-valued totals, and multiline detail escaping. Record side-effect calls in a list to assert exact order and exactly-once behavior.

12. **Document the public command contract.** State the meanings of exit codes `0`, `1`, and `130`, plus the invariant JSON keys. Users and wrappers should not need to infer semantics from implementation details.

## Verified Workflow

_Not applicable._ This skill was captured from a planning session and is
`unverified`: no implementation was applied, no targeted tests were run, and CI
was not confirmed. The actionable methodology is under **Proposed Workflow**
above and must remain hypothesis-level until those checks pass. This placeholder
exists because `scripts/validate_plugins.py` requires the literal
`## Verified Workflow` heading; it makes no verification claim.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
| ------- | ---------------- | ------------- | -------------- |
| Fire-and-forget item processing | Per-item helpers returned `None` after logging, and the command returned success after the loop | Logs are not a machine-checkable terminal ledger; blocked and failed work can collapse into exit `0` | Return typed outcomes and derive status from the complete batch ledger |
| Catch exceptions around the whole batch | One ordinary provider or push failure aborted later items | Independent items never received outcomes, so the summary was incomplete | Catch ordinary `Exception` at the per-item boundary and continue |
| Treat `KeyboardInterrupt` like an ordinary exception | Interruption could continue later work or become generic exit `1` | Operator cancellation has distinct stop and exit semantics | Catch it separately, record the current item when discovered, stop, and return `130` |
| Emit special-case JSON for early failures | Setup and listing errors returned only `status`, `exit_code`, and `message` | Consumers needed path-dependent parsing and could not rely on aggregate fields | Use one additive summary emitter and an empty complete envelope before discovery |
| Count only observed statuses | Totals omitted keys when a status count was zero | The schema changed from run to run | Initialize totals from the enum so all keys are always present |
| Log raw provider or exception details | Newlines and control characters created multiple physical log lines | Untrusted text could forge or corrupt summary records | Normalize once with `ascii(str(value))[1:-1]` before logs and JSON |
| Infer completeness only from failure statuses | A prematurely stopped loop could contain no explicit failure yet process fewer items than requested | Missing outcomes remained invisible | Make `processed != requested` an independent exit-`1` condition |

## Results & Parameters

Recommended outcome semantics for a pull-request merge driver:

| Status | Meaning | Success for command exit? |
| ------ | ------- | ------------------------- |
| `merged` | Provider confirms synchronous merge | Yes |
| `queued` | Provider confirms merge-queue enrollment | Yes |
| `skipped` | Dry-run intentionally suppresses an otherwise eligible merge | Yes |
| `blocked` | Checks or legacy status do not permit merging | No |
| `failed` | Metadata, push, provider API, or ordinary per-item operation failed | No |
| `interrupted` | Operator interrupted current batch item | No; exit `130` |

Exit-code truth table:

| Condition | Exit code |
| --------- | --------- |
| Setup or batch interruption | `130` |
| Any `blocked` or `failed` result | `1` |
| `processed != requested` without interruption | `1` |
| Every requested item is merged, queued, or intentionally skipped | `0` |

Required structured fields:

```text
status, exit_code, message, results, totals, requested, processed
```

For a pre-discovery failure or interruption:

```text
results=[]
requested=0
processed=0
totals=<every declared status mapped to 0>
```

Suggested focused verification:

```bash
pytest tests/unit/path/to/test_batch_cli.py -k "outcome or incomplete or interrupt or json_summary or multiline" -v
ruff check path/to/batch_cli.py tests/unit/path/to/test_batch_cli.py
ruff format --check path/to/batch_cli.py tests/unit/path/to/test_batch_cli.py
mypy path/to/batch_cli.py tests/unit/path/to/test_batch_cli.py
```

## Verified On

| Project | Context | Details |
| ------- | ------- | ------- |
| ProjectHephaestus | Reviewed implementation plan for hardening `hephaestus-merge-prs` | Unverified until implementation tests and CI pass |
