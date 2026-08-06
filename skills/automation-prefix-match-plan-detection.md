---
name: automation-prefix-match-plan-detection
description: "Centralize GitHub plan discovery behind a complete, ownership-verified, tri-state comment-journal contract. Use when: (1) boolean or None plan lookups can confuse API failure with absence, (2) bounded or best-effort comment caches gate publication or retries, (3) sibling paths disagree about plan markers or ownership, or (4) reconciliation and verification must retry without mutating state or consuming an absence budget."
category: architecture
date: 2026-08-06
version: "2.0.0"
user-invocable: false
verification: unverified
history: automation-prefix-match-plan-detection.history
tags:
  - automation
  - plan-discovery
  - github-comments
  - complete-read
  - ownership
  - tri-state
  - fail-closed
  - retry-budget
  - journal-reconciliation
  - rest-pagination
  - malformed-payload
  - canonical-selector
---

# Complete, Ownership-Aware Plan Discovery

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-08-06 |
| **Objective** | Ensure that only a complete, validated, actor-owned GitHub comment read can report that no canonical plan exists. |
| **Outcome** | Proposed one shared `FOUND` / `ABSENT` / `READ_ERROR` contract for every plan-discovery path, including planning verification and durable-journal reconciliation. |
| **Verification** | unverified — the architecture and acceptance matrix were reviewed, but the ProjectHephaestus implementation, focused tests, and CI were not run in this learning session. |
| **History** | [changelog](./automation-prefix-match-plan-detection.history) |

Prefix matching remains necessary, but it is only the final selector step. A
trustworthy absence decision also requires a complete paginated read, strict raw
payload validation, authenticated actor identity, consistent ownership derivation,
and an explicit failure state. A boolean or `None` return cannot represent those
conditions safely.

The governing rule is:

> `ABSENT` is evidence produced by a successful complete read. Transport,
> rate-limit, reconciliation, identity, and malformed-payload failures are
> `READ_ERROR`, never synthetic absence.

## When to Use

- A plan lookup returns `bool`, `str | None`, or an empty list after a GitHub API
  failure.
- A write, label transition, routing decision, or retry budget depends on whether
  a plan comment exists.
- Comment discovery uses GraphQL `last: 100`, `gh issue view --comments`, a capped
  cache, or any other source that cannot prove complete enumeration.
- Different consumers derive ownership from `viewerDidAuthor`, author association,
  a path-specific default, or no ownership metadata at all.
- Raw comment bodies are coerced with `str(...)`, allowing `None`, objects, or
  arrays to masquerade as valid non-plan text.
- A prefetched cache treats a missing key as a successful empty journal instead of
  performing a real fallback read.
- Durable journal reconciliation can fail before a stage normalizes labels or
  routes from reconstructed state.
- Verification performs multiple presence reads, publishes after an inconclusive
  lookup, or increments an absence budget after a read error.
- Prefix or substring marker logic is duplicated across planner, reviewer,
  implementer, and queue-pipeline adapters.

## Verified Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat it as a
> hypothesis until the focused implementation tests and CI confirm the contract.

### Quick Reference

```python
class CommentJournalReadError(RuntimeError):
    """A complete, ownership-verifiable comment read was not obtained."""


class PlanDiscoveryStatus(StrEnum):
    FOUND = "found"
    ABSENT = "absent"
    READ_ERROR = "read_error"


@dataclass(frozen=True, slots=True)
class PlanDiscoveryResult:
    status: PlanDiscoveryStatus
    plan_text: str | None = None
    error: str | None = None


def discover_plan(issue_number: int) -> PlanDiscoveryResult:
    try:
        raw = fetch_all_issue_comments_rest(issue_number)  # complete + paginated
        comments = normalize_issue_comments(
            raw,
            viewer_login=require_authenticated_login(),
        )
        return discover_plan_from_comments(comments)
    except CommentJournalReadError as exc:
        return PlanDiscoveryResult(
            PlanDiscoveryStatus.READ_ERROR,
            error=str(exc),
        )
```

```python
lookup = github.discover_plan(issue_number)
if lookup.status is PlanDiscoveryStatus.READ_ERROR:
    return RETRY  # no comment write, label mutation, or absence-budget charge
if lookup.status is PlanDiscoveryStatus.FOUND:
    return ADVANCE

# Only a confirmed ABSENT result may publish a candidate or spend the
# plan-absence budget.
assert lookup.status is PlanDiscoveryStatus.ABSENT
```

### Detailed Steps

1. **Inventory every decision surface.** Search for boolean and optional plan
   APIs, not just marker constants:

   ```bash
   rg -n "has_existing_plan|_has_plan|_get_latest_plan|comments_contain_plan" \
     hephaestus/automation
   ```

   Include protocols, caches, direct CLI paths, worker results, pipeline
   adapters, stage fakes, and tests. A shared helper is not authoritative while a
   sibling path can still invent absence.

2. **Place policy in the owning product layer.** Define the typed read exception,
   tri-state result, strict normalizer, and canonical selector in the automation
   journal-policy module. Keep the dependency direction one-way: product-layer
   orchestration may use shared library helpers, but library packages must not
   import product-layer policy.

3. **Use a complete authoritative reader.** For absence-sensitive decisions, use
   the paginated REST issue-comments endpoint and continue until GitHub supplies no
   next page. Preserve bounded-ingest protections, but reject every non-object page
   element instead of silently dropping it. A cap or failure must produce an error,
   not partial success.

   Bounded GraphQL batches remain useful for non-authoritative review context.
   They are not allowed to prove plan absence because `last: 100`, partial aliases,
   and failure-to-empty fallbacks cannot establish completeness.

4. **Normalize raw comments once and strictly.** Require a non-empty authenticated
   viewer login. For every comment, require an object, a string body, and a
   non-empty author login. Preserve timestamps, URLs, IDs, and author association
   only after validating the ownership-critical fields. Do not coerce an invalid
   body with `str()`.

   ```python
   viewer_did_author = author_login.casefold() == viewer_login.casefold()
   ```

   Derive ownership identically in every REST path. Do not trust path-specific
   defaults, synthetic ownership, or optional viewer metadata.

5. **Select the plan from normalized comments only.** Walk the complete journal
   newest-first, ignore comments not authored by the authenticated actor, exclude
   canonical plan-review comments, and accept only the canonical plan prefix. The
   selector returns `FOUND` with the original body or `ABSENT`; it does not perform
   I/O and cannot return `READ_ERROR` by itself.

6. **Convert transport and shape failures at one boundary.** Wrap REST transport,
   rate-limit, JSON, identity, and malformed-comment failures as
   `CommentJournalReadError`. Consumer-facing discovery methods convert that typed
   exception to `READ_ERROR`. Reconciliation surfaces may propagate it to an
   explicit pipeline retry boundary.

7. **Make caches preserve uncertainty.** Cache successful normalized journals
   separately from per-issue read errors. A missing successful cache key means
   "not read" and triggers a real fallback read; it does not mean "read and empty."
   A cached read error stays `READ_ERROR` until a later prefetch or retry refreshes
   it.

8. **Reuse a reconciled snapshot at stage entry.** Wrap durable-journal loading
   before any entry-label normalization or journal-derived routing. On
   `CommentJournalReadError`, return `RETRY` with the work item, attempts, labels,
   and mutation log unchanged. On success, use the same `JournalSnapshot` to
   fast-forward a restart; do not perform a second plan-presence lookup.

9. **Perform one lookup before publication in verification.** Branch on all three
   statuses before reading or publishing a candidate:

   | Status | Publication | Routing | Absence budget |
   |--------|-------------|---------|----------------|
   | `FOUND` | No duplicate write | Confirm labels, then advance | Unchanged |
   | `ABSENT` | Candidate may be published | Existing publication path | May increment only if still absent |
   | `READ_ERROR` | Forbidden | Stay in verification and retry | Unchanged |

   A required revision may still publish its candidate after a confirmed read,
   even when an older canonical plan is `FOUND`; that is an explicit revision
   rule, not an absence shortcut.

10. **Preserve transaction recovery semantics.** If an idempotent recovery write
    succeeded and its confirming read fails, return `RETRY` so the next pass can
    reconcile durable GitHub state. Keep semantic journal conflicts fail-closed;
    do not relabel them as transient read failures and retry forever.

11. **Update interfaces and fakes, not only implementations.** Replace boolean
    plan-presence methods in protocols with `PlanDiscoveryResult`. Stage fakes
    should inject plan-read and journal-read failures independently. Keep concise
    fixture inputs such as `has_plan=True` only as setup sugar; the callable
    interface must be tri-state.

12. **Test the contract as a matrix and through production adapters.** At minimum,
    cover:

    - empty successful journal -> `ABSENT`;
    - valid actor-owned plan -> `FOUND`;
    - foreign-authored plan marker -> `ABSENT`;
    - transport failure and rate limit -> `READ_ERROR`;
    - missing viewer identity -> `READ_ERROR`;
    - non-object comment, non-string body, or missing author login -> `READ_ERROR`;
    - missing cache key -> fallback read, not absence;
    - reconciliation read error at entry -> retry with no mutation;
    - verification read error -> no publication and no absence-budget increment;
    - raw malformed REST payload passed through the production adapter -> the same
      stage-level no-mutation/no-budget outcome.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Boolean or `None` discovery | `False` or `None` represented both a complete empty journal and any read/parse failure. | Callers published duplicate plans, exited successfully, or spent retry budget without evidence of absence. | Model `FOUND`, `ABSENT`, and `READ_ERROR` explicitly at every interface. |
| Bounded GraphQL cache as absence authority | `last: 100` plus failure-to-empty behavior fed `has_existing_plan()`. | Older plans, capped journals, alias failures, and rate limits all looked empty. | Use complete paginated REST for absence; reserve bounded GraphQL for context optimization. |
| Prefix matching without ownership | A canonical marker from any commenter counted as the automation actor's plan. | Foreign comments could suppress publication or drive restart routing. | Require authenticated viewer identity and compare validated author logins case-insensitively. |
| Path-specific `viewer_did_author` defaults | Some adapters trusted GraphQL metadata while others synthesized ownership. | The same journal produced different results depending on the caller. | Normalize all authoritative REST paths through one ownership contract. |
| Coercing malformed bodies with `str()` | `None`, objects, and arrays became ordinary text. | Invalid journals could still produce a confident `ABSENT`. | Reject non-string bodies before constructing normalized comments. |
| Missing cache key returned `[]` | A prefetch map without an issue key was interpreted as a successful empty read. | Partial batch responses invented absence without a fallback request. | Distinguish missing, empty-success, and cached-error states. |
| Re-reading after reconciliation | Entry reconciliation succeeded, then a separate presence lookup made routing depend on a different observation. | Added latency and a TOCTOU gap; the second read could fail after durable state was already known. | Reuse the reconciled snapshot for restart fast-forward. |
| Catching journal read errors after label normalization | Entry labels were mutated before the stage learned that durable state could not be reconstructed. | Retry was no longer side-effect free. | Put the typed retry boundary before every entry mutation. |
| Retrying semantic journal conflicts | All reconciliation failures were classified as transport errors. | Durable corruption could loop indefinitely instead of failing closed. | Retry typed read failures only; preserve semantic conflict outcomes. |
| Fixing only marker substring logic | Shared prefix matching removed review-comment false positives but retained boolean, partial-read, and ownership ambiguity. | Correct parsing over incomplete or untrusted input still produced unsafe absence. | Treat marker selection as the final step of a complete discovery contract. |

## Results & Parameters

### Result Invariants

```text
complete read + owned canonical plan     -> FOUND(plan_text)
complete read + no owned canonical plan  -> ABSENT
foreign canonical marker only            -> ABSENT
transport/rate-limit failure              -> READ_ERROR
missing authenticated login               -> READ_ERROR
malformed page/comment/body/author         -> READ_ERROR
bounded or partial read                    -> never authoritative ABSENT
READ_ERROR in stage entry or VERIFY        -> RETRY, no mutation, no budget
semantic journal conflict                  -> fail closed, not transient retry
```

### ProjectHephaestus Example Ownership

| Responsibility | Example location |
|----------------|------------------|
| Typed result, strict normalization, actor-owned selector | `hephaestus/automation/review_journal.py` |
| Complete paginated REST ingestion | `hephaestus/automation/github_api/issues.py` |
| Worker lookup and nonzero error result | `hephaestus/automation/plan_reviewer.py` |
| Successful-journal/error caches and fallback read | `hephaestus/automation/state/planner.py` |
| Legacy direct planning phase | `hephaestus/automation/_plan_phase.py` |
| Production pipeline adapter | `hephaestus/automation/pipeline_github_queries.py` |
| Protocol surface | `hephaestus/automation/_interfaces.py`, `pipeline/stages/base.py` |
| Entry reconciliation and verification retry boundaries | `hephaestus/automation/pipeline/stages/planning.py` |

### Proposed Validation Commands

```bash
uv run pytest \
  tests/unit/automation/test_review_journal.py \
  tests/unit/automation/test_interfaces.py -v

uv run pytest \
  tests/unit/automation/test_github_api.py \
  tests/unit/automation/test_plan_reviewer.py \
  tests/unit/automation/state/test_planner.py \
  tests/unit/automation/test_stage_phases.py \
  tests/unit/automation/test_pipeline_github.py \
  tests/unit/automation/pipeline/stages/test_stage_planning.py -v

uv run ruff check \
  hephaestus/automation/review_journal.py \
  hephaestus/automation/github_api/issues.py \
  hephaestus/automation/plan_reviewer.py \
  hephaestus/automation/state/planner.py \
  hephaestus/automation/state/review.py \
  hephaestus/automation/_plan_phase.py \
  hephaestus/automation/_interfaces.py \
  hephaestus/automation/pipeline_github_queries.py \
  hephaestus/automation/pipeline/stages/base.py \
  hephaestus/automation/pipeline/stages/planning.py \
  tests/unit/automation

uv run mypy \
  hephaestus/automation/review_journal.py \
  hephaestus/automation/plan_reviewer.py \
  hephaestus/automation/state/planner.py \
  hephaestus/automation/_plan_phase.py \
  hephaestus/automation/_interfaces.py \
  hephaestus/automation/pipeline_github_queries.py \
  hephaestus/automation/pipeline/stages/base.py \
  hephaestus/automation/pipeline/stages/planning.py
```

### Verification Promotion

Promote this skill to `verified-local` only after the focused result-matrix,
production-adapter, planning-stage, lint, and type commands pass against the
implementation. Promote it to `verified-ci` only after the corresponding
Hephaestus PR's required CI passes. Record exact test counts, the PR, and the
commit SHA in the next history entry.

Rollback requires no schema, label, configuration, dependency, or public comment
format migration; revert the implementation normally while retaining the rule
that partial reads must never be interpreted as absence.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Reviewed ownership-aware `FOUND` / `ABSENT` / `READ_ERROR` plan-discovery design | Proposed only; implementation and CI validation pending. |

## Related Skills

- `automation-graphql-batch-comment-fetch` — bounded GraphQL comment context;
  never use its partial/failure-to-empty result as authoritative plan absence.
- `automation-forced-replanning-journal-recovery` — durable revision publication
  and restart recovery once the comment journal has been read successfully.
- `pipeline-routing-budget-terminal-vs-retry-paths` — budget attribution for
  retry loops versus terminal outcomes.
