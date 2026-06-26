---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation and delegation-shim reduction plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) replacing explicit compatibility wrappers with dynamic delegation through __getattr__."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, dynamic-delegation, __getattr__, patch-object, provider-boundary, allowlist]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture the hidden assumptions that invalidated parts of the plan for consolidating `hephaestus/scripts_lib/check_python_version_consistency.py` into `hephaestus/validation/python_version.py` (issue #1189) |
| **Outcome** | Plan produced; NOGO on first version; revised plan addresses all 5 failure modes; v2.2.0 adds dynamic-delegation wrapper-collapse review risks from issue #1389 planning |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history) |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Replacing many explicit compatibility wrappers with a dynamic `__getattr__` delegation allowlist
- A refactor plan relies on grep/AST counts, patch seams, callback lookup behavior, or line-specific source facts that were not rerun during implementation
- Reviewer focus is whether thin-wrapper deletion preserved provider-boundary seams, instance-level patch behavior, direct assignment/deletion restoration, and introspection/tooling expectations

## Verified Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```bash
# Before writing the plan, run these audits:

# 1. Check __all__ in affected __init__.py files
grep -n "__all__" hephaestus/<package>/__init__.py

# 2. Find early-exit paths in the target main()
grep -n "return\|sys.exit" hephaestus/<module>.py | head -30

# 3. Count test classes in the file being replaced/shimmed
grep -n "^class Test" tests/unit/<path>/test_<module>.py | wc -l

# 4. Verify dependency is declared
grep "packaging" pyproject.toml

# 5. Find all callers of the function being renamed/consolidated
grep -rn "from hephaestus.scripts_lib import\|from hephaestus.validation import" hephaestus/ tests/ scripts/

# 6. Find same-name functions across both modules
grep -rn "^def <function_name>" hephaestus/<module_a>.py hephaestus/<module_b>.py

# 7. For dynamic delegation reductions, recompute wrapper purity and patch-surface facts
python - <<'PY'
import ast
from pathlib import Path

source = Path("hephaestus/automation/implementer.py").read_text()
module = ast.parse(source)
cls = next(n for n in module.body if isinstance(n, ast.ClassDef) and n.name == "IssueImplementer")
methods = {n.name for n in cls.body if isinstance(n, ast.FunctionDef)}
namespace: dict[str, object] = {}
exec("from hephaestus.automation.implementer import IssueImplementer\nnames = IssueImplementer._PHASE_RUNNER_DYNAMIC_DELEGATES", namespace)
names = set(namespace["names"])
assert not (names & methods)
assert {"_run_claude_code", "_run_claude_impl_session", "_run_codex_code"} <= methods
PY
rg -n 'patch\.object\(IssueImplementer|patch\.object\([^,]+IssueImplementer' tests hephaestus -g '*.py'
rg -n 'IssueImplementer\.__dict__|hasattr\(.*IssueImplementer|dir\(.*IssueImplementer|monkeypatch' tests hephaestus -g '*.py'
```

### Detailed Steps

1. **Audit `__all__` in every `__init__.py` that re-exports from the canonical module.**
   Before adding functions to `validation/python_version.py`, read `hephaestus/validation/__init__.py` in full.
   If it has an explicit `__all__`, the new symbols MUST be added or `from hephaestus.validation import new_function` will raise `AttributeError`.

2. **Trace every early-return path in `main()` before extending it.**
   Open the target `main()` function and list every `return` / `sys.exit` statement.
   Any new sub-checks added must not be gated behind an early exit that already existed (e.g., `if args.json: ... return 0`).
   Fix: move ALL check calls before the format-branching block, then use results in both JSON and text paths.
   Each output-mode branch (JSON, plain text, quiet) must invoke the same set of checks.

3. **Test delegation shims must import test CLASSES, not just symbols.**
   If replacing a test file with an import-only shim, the shim must import the test **classes** themselves:
   ```python
   # CORRECT: pytest re-discovers imported test classes at module scope
   from tests.unit.validation.test_python_version import TestFoo, TestBar

   # WRONG: pytest collects zero tests from import-only shims of non-test symbols
   from hephaestus.scripts_lib.check_python_version_consistency import extract_pyproject_versions
   ```
   Count `grep "^class Test" | wc -l` in the source test file; verify they all exist in the destination before shimming.

4. **Verify runtime dependencies before adding new imports.**
   For any `import X` inside a new function, confirm `X` appears in `[project.dependencies]` in `pyproject.toml`.
   `packaging` is common in the Python ecosystem but not universal — check before assuming.

5. **When two modules share a function name with incompatible signatures, add a new name — do not alias.**
   If the canonical module has `extract_pyproject_versions(path: Path)` and the source module has
   `extract_pyproject_versions(content: str)`, a shim alias re-exporting the path version under the
   string name causes silent behavioral regression: `"".is_file()` returns `False`, so every
   string-content call returns `{}`.

   Fix pattern:
   - Keep `extract_pyproject_versions(path: Path)` unchanged for existing callers in the canonical module.
   - Extract a private helper `_extract_versions_from_text(content: str)` that both implementations delegate to.
   - Add a new public function `extract_pyproject_versions_str(content: str)` in the canonical module that
     calls `_extract_versions_from_text`.
   - In the shim, alias `extract_pyproject_versions_str as extract_pyproject_versions` so the source module's
     callers get the string-based version without renaming their calls.
   - Add both names to `__all__` in `validation/__init__.py`.

   ```python
   # canonical module (validation/python_version.py)
   def _extract_versions_from_text(content: str) -> dict[str, ...]:
       """Shared implementation — used by both public entry points."""
       ...

   def extract_pyproject_versions(path: Path) -> dict[str, ...]:
       """Existing callers use this; unchanged."""
       content = path.read_text()
       return _extract_versions_from_text(content)

   def extract_pyproject_versions_str(content: str) -> dict[str, ...]:
       """New entry point for callers that already have the file content."""
       return _extract_versions_from_text(content)

   # shim (scripts_lib/check_python_version_consistency.py)
   from hephaestus.validation.python_version import (
       extract_pyproject_versions_str as extract_pyproject_versions,  # preserve caller contract
       ...
   )
   ```

6. **For dynamic `__getattr__` delegation, rerun the proof at implementation time.**
   A plan may correctly identify current-source one-line wrappers, but that evidence stales quickly.
   Before deleting explicit methods, rerun the AST audit against the exact implementation commit and
   assert all of the following:
   - The allowlist contains only pure `return self.phase_runner._same(...)` accessors.
   - Provider-boundary wrappers, selected-agent execution wrappers, and local state/collaborator methods remain explicit.
   - No class-level patch or introspection consumer depends on a method living in `IssueImplementer.__dict__`.
   - Runtime callbacks still resolve `impl._name` at call time, so instance-level `patch.object(impl, "_name")` continues to intercept.
   - Regression tests cover `patch.object`, direct assignment, deletion restoration, and `dir()` discovery for at least one dynamic delegate.

7. **Treat grep and AST evidence in the plan as inputs, not proof.**
   Line numbers, wrapper counts, and patch-site grep output written during planning are external facts from a
   snapshot. The implementer must refresh them, and the reviewer should reject plans that rely on stale
   counts without a verification command that fails closed when the source changed.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Plan claimed "no signature change needed" | Shim re-exported `extract_pyproject_versions(path)` under the same name as the source module's `extract_pyproject_versions(content)` | `"".is_file()` returns `False`; every string-content caller received `{}` silently | When same-name collision exists, a shim alias at the wrong layer is a silent regression — add a new name instead |
| Import-only test shim | Replaced test file body with `from hephaestus.scripts_lib import extract_pyproject_versions` | pytest collects zero tests; 400+ lines of test classes do not teleport via non-test symbol imports | Test delegation shims must import test classes: `from tests.unit.validation.test_python_version import TestFoo, TestBar` |
| Added new sub-checks after `if args.json:` | Placed new `check_*` calls after an existing `if args.json: ...; return 0` block | JSON callers exit before reaching the new checks; CI never sees the new coverage | List all early exits in `main()` first; move check calls above the format-branching block |
| Did not update `validation/__init__.py __all__` | Added 8+ new functions to `python_version.py` without updating the package's `__init__.py` | `from hephaestus.validation import new_function` raises `AttributeError` even though the function exists | Grep all `__init__.py` files that import from the modified module; add every new symbol to `__all__` |
| Assumed `packaging` is a declared dependency | Used `from packaging.version import Version` in a new function | `packaging` may not be in `[project.dependencies]`; runtime `ImportError` on CI | `grep packaging pyproject.toml` before adding the import |
| R2 — DOTALL regex crosses TOML sections | Ported `_extract_versions_from_text` inherited `re.DOTALL` from `_extract_via_regex:113`. With DOTALL, `\[tool\.mypy\].*?python_version` lazy-matches past blank lines and `[tool.other]` headers — `test_mypy_version_not_crossed_from_other_section` fails. | Fix: use scripts_lib's section-bounded negative-lookahead `\[tool\.mypy\]\n(?:(?!\[).+\n)*?python_version` instead. `_extract_via_regex` now delegates to `_extract_versions_from_text`, eliminating the DOTALL regex entirely. | Always use section-bounded negative-lookahead `(?:(?!\[).+\n)*?` for TOML section extraction — never `re.DOTALL` across sections. |
| R0/R1 — Wrong test count stated as "44" | Plans stated "44 test functions" but actual scripts_lib test file has 35 functions / 9 classes. | Count test functions by direct grep before writing the plan (`grep -c "def test_" file`). | Verify counts by reading the actual file before stating them in a plan. |
| Dynamic delegation plan treated wrapper counts as fixed | Plan for issue #1389 cited an AST scan with 28 pure wrappers, 25 delete candidates, and 3 provider-boundary exclusions | That count was not durable evidence; any concurrent edit to `IssueImplementer` or `ImplementationPhaseRunner` changes the safe delete set | Rerun the AST audit immediately before deletion and make the allowlist-vs-class-method assertion part of verification |
| Grep evidence scoped only to expected patch forms | Plan relied on no class-level `patch.object(IssueImplementer, ...)` hits and several instance-level examples | Grep patterns miss dynamic patch targets, helper aliases, and non-`patch.object` introspection consumers | Require broad searches for class-level patch, monkeypatch, `IssueImplementer.__dict__`, `hasattr`, `dir`, and callback lookup usage before approving `__getattr__` compatibility shims |
| Provider-boundary wrappers almost joined the dynamic allowlist | The refactor shape made `_run_claude_code`, `_run_claude_impl_session`, and `_run_codex_code` look like mechanical delegates | These methods are selected-agent execution seams, not pure compatibility surface; moving them behind `__getattr__` hides a policy boundary and weakens reviewability | Keep provider-boundary and local state/collaborator wrappers explicit even when their bodies delegate today |
| Assignment/deletion restoration was assumed from Python lookup rules | Plan asserted instance-level assignment and `patch.object` would shadow dynamic delegates and restore after deletion | Correct in principle, but easy to break with descriptors, cached bound methods, or future custom `__setattr__`/`__delattr__` | Add regression tests for `patch.object`, direct assignment, deletion restoration, and post-restore `.__self__ is phase_runner` |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] Dynamic delegation audit rerun on current source: pure wrapper count, delete set, explicit provider-boundary exclusions
- [ ] Patch surface checked beyond expected `patch.object` forms: class-level patch, monkeypatch, `IssueImplementer.__dict__`, `hasattr`, `dir`, callback lookup
- [ ] Tests prove instance-level patch, direct assignment, deletion restoration, and introspection for at least one dynamic delegate
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1389 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| AST audit found exactly 28 pure wrappers, 25 delete candidates, and 3 provider-boundary exclusions | UNVERIFIED SNAPSHOT | Rerun the AST scan immediately before editing; fail if the allowlist overlaps `IssueImplementer.__dict__` or if explicit agent wrappers disappear |
| No class-level patch targets exist | PARTIAL GREP ONLY | Search broader than `patch.object(IssueImplementer, ...)`: monkeypatch, `IssueImplementer.__dict__`, `dir`, `hasattr`, helper aliases, and dynamically constructed patch targets |
| Runner callback paths resolve through `impl._name` at runtime | UNVERIFIED SNAPSHOT | Re-open every callback site and add regression coverage proving `patch.object(impl, "_collect_diff")` or a similar dynamic delegate still intercepts |
| Dynamic delegates preserve assignment/deletion rollback | HYPOTHESIS UNTIL TESTED | Add explicit tests for direct assignment shadowing, deletion restoration, and restored bound method ownership |
| `_run_claude_code`, `_run_claude_impl_session`, and `_run_codex_code` can be treated as ordinary wrappers | WRONG | Keep selected-agent execution wrappers explicit; they are provider-boundary seams even if their body delegates today |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1389 (replace explicit `IssueImplementer` phase-runner wrappers with dynamic delegation) | Plan produced, NOT executed; this amendment records the stale-proof requirements and reviewer risk checks before implementation. |
