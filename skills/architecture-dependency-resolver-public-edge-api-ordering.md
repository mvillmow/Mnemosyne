---
name: architecture-dependency-resolver-public-edge-api-ordering
description: "Preserve dependency-ordering behavior while removing private graph coupling from pipeline admission. Use when: (1) a pipeline or scheduler reaches through a resolver facade to mutate an internal graph, (2) adding a public edge-insertion API such as add_dependency(issue_number, depends_on), (3) keeping in-set dependency filtering and implementation ordering stable while improving encapsulation."
category: architecture
date: 2026-07-07
version: "1.0.0"
user-invocable: false
verification: unverified
tags:
  - dependency-resolver
  - pipeline-admission
  - encapsulation
  - public-api
  - graph-edge
  - implementation-order
  - projecthephaestus
  - pytest
  - ruff
---

# Dependency Resolver Public Edge API Ordering

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-07-07 |
| **Objective** | Add a public `DependencyResolver.add_dependency(issue_number, depends_on)` API so pipeline admission no longer reaches through `resolver.graph.add_dependency(...)`, while preserving dependency ordering for ProjectHephaestus issue #1915. |
| **Outcome** | Proposed implementation workflow captured. No ProjectHephaestus code was edited or tested in this learn run. |
| **Verification** | `unverified` for the ProjectHephaestus workflow. The Mnemosyne skill-file validation should still be run with `python3 scripts/validate_plugins.py` before commit. |

## When to Use

- A resolver, planner, scheduler, or admission pipeline exposes a high-level object but callers mutate an owned internal graph directly.
- Existing ordering behavior depends on graph edges, but edge insertion should be owned by the resolver facade instead of callers reaching into private structure.
- You need focused regression tests for both the new public API and the caller-side encapsulation boundary.
- You are reviewing a plan that swaps `resolver.graph.add_dependency(...)` for `resolver.add_dependency(...)` and must verify behavior stayed unchanged.

## Verified Workflow

<!--
The repository validator currently requires a "## Verified Workflow" heading.
This skill is intentionally marked unverified; the actionable workflow below is proposed only.
-->

### Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until the ProjectHephaestus code change is implemented and CI confirms it.

### Quick Reference

```bash
# ProjectHephaestus proposed regression commands after implementing the code change:
pixi run pytest tests/unit/automation/test_dependency_resolver.py -k "add_dependency"
pixi run pytest tests/unit/automation/pipeline/test_admission.py -k "order_for_implementation"
! rg -n "resolver\\.graph\\.add_dependency" hephaestus/automation/pipeline/admission.py
pixi run ruff check \
  hephaestus/automation/dependency_resolver.py \
  hephaestus/automation/pipeline/admission.py \
  tests/unit/automation/test_dependency_resolver.py \
  tests/unit/automation/pipeline/test_admission.py
```

### Detailed Steps

1. **Keep edge insertion behind the resolver API.** Add `DependencyResolver.add_dependency(issue_number, depends_on)` in `hephaestus/automation/dependency_resolver.py`. The method should delegate to `self.graph.add_dependency(issue_number, depends_on)` so callers no longer depend on the resolver's private graph structure.

2. **Route existing resolver-owned issue ingestion through the new API.** In `DependencyResolver.add_issue`, add the issue first, then insert every `issue.dependencies` edge by calling `self.add_dependency(issue.number, dep)`. This keeps both issue registration and edge insertion under the resolver's public surface.

3. **Update pipeline admission without changing filtering semantics.** In `order_for_implementation`, keep the existing in-set dependency filtering, but replace `resolver.graph.add_dependency(info.number, dep)` with `resolver.add_dependency(info.number, dep)`.

4. **Pin public API behavior in resolver tests.** Add or extend tests in `tests/unit/automation/test_dependency_resolver.py` to cover the new `add_dependency` method and any existing graph validation behavior that depends on edge insertion.

5. **Pin caller encapsulation in admission tests.** Add or extend tests in `tests/unit/automation/pipeline/test_admission.py` for `order_for_implementation`. A useful structural regression is:

   ```python
   import inspect

   source = inspect.getsource(order_for_implementation)
   assert "resolver.graph.add_dependency" not in source
   assert "resolver.add_dependency" in source
   ```

6. **Run focused behavioral and style verification.** Use the Quick Reference commands above before broadening to larger suites. The source grep is intentionally self-falsifying: a hit means private graph coupling returned.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Mutate `resolver.graph` from pipeline admission | Call `resolver.graph.add_dependency(info.number, dep)` inside `order_for_implementation` | The pipeline became coupled to the resolver's private graph structure, making future resolver internals harder to change safely | Expose a resolver-owned `add_dependency` method and make admission call that public API |
| Only test final ordering | Verify implementation order but skip a structural regression for the caller | Ordering can remain correct while the caller still reaches through `resolver.graph`, so encapsulation can regress silently | Pair behavior tests with an `inspect.getsource` guard that rejects `resolver.graph.add_dependency` in admission |
| Treat the plan as verified | Document the provided workflow as executed | The ProjectHephaestus change was not implemented or tested during this learn run | Mark the workflow `unverified` until the code change and test commands actually run |

## Results & Parameters

### Proposed API Shape

```python
class DependencyResolver:
    def add_dependency(self, issue_number: int, depends_on: int) -> None:
        """Add a dependency edge owned by the resolver facade."""
        self.graph.add_dependency(issue_number, depends_on)
```

### Proposed `add_issue` Pattern

```python
def add_issue(self, issue: IssueInfo) -> None:
    self.graph.add_issue(issue.number)
    for dep in issue.dependencies:
        self.add_dependency(issue.number, dep)
```

### Proposed Admission Pattern

```python
for dep in info.dependencies:
    if dep in issue_numbers:
        resolver.add_dependency(info.number, dep)
```

### Verification Status

- ProjectHephaestus implementation: unverified in this learn run.
- ProjectHephaestus tests and ruff commands: proposed only.
- Mnemosyne skill validation: run `python3 scripts/validate_plugins.py` in ProjectMnemosyne before committing the skill.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1915 implementation plan | Unverified plan capture only; no ProjectHephaestus code change or CI result was produced in this learn run |
