---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) deleting a deprecated public wrapper while preserving an internal alias or test patch seam, (7) an issue title conflicts with the issue body/evidence."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, wrapper-removal, public-api, observability]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions that invalidate DRY refactoring plans, including module consolidation, shim delegation, and deleting deprecated wrapper APIs while preserving intended compatibility boundaries |
| **Outcome** | v2.2.0 adds ProjectHephaestus issue #1402 planning risks: issue-title/body mismatch, static evidence limits, alias-vs-wrapper compatibility, observability loss, and plan-time line-number/test-coverage drift |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history). v1.0.0: initial 5-assumption capture. v2.0.0: revised with concrete fix patterns for signature collision and test delegation. v2.1.0: add R2 findings — DOTALL regex crosses TOML sections, wrong test count stated in plan. v2.2.0: add wrapper-deletion and issue-intent review risks from issue #1402. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Deleting a deprecated public wrapper while keeping an internal alias for callers/tests
- An issue title mentions one symbol but the body, motivation, and affected files point to another
- A plan relies on `rg`, AST parsing, exact line numbers, GitHub issue context, or current test coverage without re-reading the branch immediately before implementation

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

# 7. If the issue title/body disagree, capture that as an explicit assumption
gh issue view <issue> --json title,body --jq '{title, body}'
rg -n "<title_symbol>|<body_symbol>" hephaestus tests docs scripts -g "*.py" -g "*.md"

# 8. Wrapper deletion: look beyond static first-party imports
rg -n "from .* import <wrapper>|<module>\\.<wrapper>|patch\\(|monkeypatch|<alias>" hephaestus tests docs scripts -g "*.py" -g "*.md"

# 9. Observability check: deleting a delegating wrapper may delete useful logs
rg -n "<wrapper>|logger\\.(debug|info|warning|error)" hephaestus/<path>/<module>.py
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

6. **When an issue title conflicts with body/evidence, make the mismatch reviewable.**
   Do not silently scope to the part that looks technically correct. State the assumption:
   "The title mentions `<title_symbol>`, but the body, motivation, affected files, and current grep evidence scope the change to `<body_symbol>`."
   Ask the reviewer to verify issue intent, not only code mechanics. The implementation can be correct and still solve the wrong request if the title was the real user intent.

7. **Static grep/AST evidence is useful but not a public-API proof.**
   An AST regression test can prevent first-party static reintroduction of a wrapper, but it does not prove external consumers are safe.
   Before deleting a wrapper, reviewers should check dynamic imports, re-exports, fixture monkeypatches, docs, scripts, non-Python callers, release notes, and any advertised API surface.
   Treat "no first-party `rg` hits" as a scoped claim, not as compatibility proof.

8. **Preserving an internal alias while deleting a public wrapper is a compatibility decision.**
   Keeping `io_write_secure`-style aliases can be correct when internal call sites and tests patch the alias path deliberately.
   The reviewer should verify whether those tests encode intended behavior or only historical implementation coupling.
   If tests patch both the old wrapper and the retained alias, separate the public API deletion from the internal test seam before approving.

9. **Delegating wrappers may still carry observability.**
   A wrapper that "only delegates" may still log, normalize inputs, preserve stack context, or anchor operational search terms.
   Removing it can change debugging behavior even when persisted bytes stay identical.
   Reviewers should explicitly decide whether the wrapper's log line was operationally meaningful, and whether the canonical helper needs equivalent instrumentation.

10. **Reverify plan-time line numbers, coverage claims, and GitHub context on the branch.**
    Exact file line numbers, "already covered by test X", and GitHub issue-body readings are snapshots.
    Re-run the grep/test discovery on the actual branch before implementation, especially after rebases or adjacent automation changes.

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
| Issue title/body mismatch hidden in scope decision | Plan for issue #1402 scoped to `write_secure` because body/evidence supported it while the title mentioned `retry_with_exponential_backoff` | Reviewer may approve a mechanically sound wrapper deletion that ignores the title's intended target | Put the mismatch in the assumptions section and ask reviewer to confirm intent before implementation |
| Treating `rg` + AST as full compatibility proof | Plan proposed static import checks and an AST regression test for first-party Python code | Static checks miss dynamic imports, re-exports, monkeypatch fixtures, docs, scripts, non-Python callers, and external users | Phrase static evidence as first-party scope only; add public-surface and test-seam review steps |
| Deleting a wrapper but keeping its alias without naming the boundary | Plan removed `github_api.write_secure` while preserving `github_api.io_write_secure` because production/tests patch it | The retained alias may be intended compatibility or accidental test coupling; reviewers need to decide which | Review alias patch paths separately from public wrapper deletion |
| Dropping wrapper logging as "just delegation" | Wrapper delegated to the canonical helper but also logged a debug line | Removing the wrapper can remove an operational breadcrumb even if persistence behavior is unchanged | Check whether the log message is relied on; add equivalent canonical logging only if it is operationally meaningful |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] If issue title/body disagree, reviewer explicitly confirmed which symbol the issue intends to change
- [ ] Static `rg`/AST evidence is labeled first-party only; dynamic imports, re-exports, monkeypatches, docs, scripts, and external API expectations were checked or intentionally deferred
- [ ] Public wrapper deletion and retained internal alias/test seam are reviewed as separate compatibility boundaries
- [ ] Any wrapper logging/observability removed by the refactor is called out and accepted
- [ ] Exact line numbers, current coverage claims, and GitHub issue context were reverified on the implementation branch
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1402 Specific Findings

| Assumption | Status | Reviewer Focus |
|------------|--------|----------------|
| Issue title says `retry_with_exponential_backoff`, but body/evidence scopes to `write_secure` | UNVERIFIED | Confirm issue intent before implementation; a body-scoped fix may not satisfy a title-scoped request |
| `hephaestus.io.utils.write_secure` is canonical and covered by `tests/unit/io/test_utils.py::TestWriteSecure` | PLAN-TIME SNAPSHOT | Re-read the branch and rerun focused tests; exact line numbers and coverage claims may drift |
| `github_api.write_secure` can be deleted because first-party static consumers are only state modules/tests | SCOPED CLAIM | Check dynamic imports, re-exports, fixture monkeypatches, docs, scripts, non-Python callers, and external public-surface expectations |
| `github_api.io_write_secure` should remain as an alias because production/tests patch it | COMPATIBILITY DECISION | Decide whether alias patch paths are intended API/test seams or accidental coupling |
| Static regression test prevents reintroduction | PARTIAL | It guards first-party static imports only; it does not prove external consumers are safe |
| Wrapper logging can disappear with no behavior change | UNVERIFIED | Confirm whether the debug line was operationally useful before deleting it |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1402 (`write_secure` wrapper deletion) | v2.2.0 captures plan-review risks from a planning-only draft: title/body mismatch, alias-vs-wrapper boundary, static evidence limits, observability loss, and plan-time evidence drift. Implementation pending. |
