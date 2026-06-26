---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation and CLI-boilerplate extraction plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) extracting repeated argparse/repo-root/version/logging boilerplate into shared CLI helpers."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, cli-boilerplate, argparse, validation-cli, version-cli, repo-root, logging, gh-call, circuit-breaker, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture the hidden assumptions that invalidate DRY consolidation plans before implementation, including module consolidation and validation/version CLI boilerplate extraction. |
| **Outcome** | Plans produced; findings are reviewer checklists, not executed implementation evidence. v2.2.0 adds CLI-helper extraction review guidance from ProjectHephaestus issue #1419. |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history). v2.2.0: add CLI boilerplate extraction review checklist, including argparse/version duplication, special CLI variants, stale `rg` call-site lists, and `gh_call` circuit-breaker routing assumptions. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Extracting duplicated validation/version CLI parser, repo-root, logging, or version-option boilerplate into shared helpers
- Reviewing a plan that cites `rg` output, exact source lines, or external helper/API behavior that was not re-run in the review request
- Migrating CLIs with special parse behavior (`parse_known_args`, positional paths, default config files, JSON output, or root-relative defaults)
- Touching GitHub CLI call paths where bare `subprocess.run(["gh", ...])` would bypass a shared resilience/circuit-breaker adapter

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

# 7. For CLI-boilerplate extraction, re-run the call-site inventory from current HEAD.
rg -n "ArgumentParser|add_json_arg|add_version_arg|--repo-root|parse_known_args|configure.*logging|subprocess\\.run\\(\\[\"gh\"" \
  hephaestus/ scripts/ tests/

# 8. Prove the proposed shared parser does not duplicate --version.
rg -n "def create_.*parser|add_version_arg|ArgumentParser" hephaestus/cli hephaestus/validation scripts

# 9. Verify every special CLI variant before asserting behavior is preserved.
rg -n "parse_known_args|type_alias|default=.*repo_root|--config|--repo-root|json" hephaestus/ scripts/ tests/

# 10. Verify GitHub calls stay routed through the shared adapter, not bare subprocess.
rg -n "gh_call|subprocess\\.run\\(\\[\"gh\"" hephaestus/automation hephaestus/github tests/
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

6. **Re-run the CLI call-site inventory from current HEAD before scoping the migration.**
   A plan that says "the issue-listed `rg` call-site list is complete" is only a hypothesis unless the
   reviewer or implementer re-runs the search in the checked-out tree. Use broad structural terms first
   (`ArgumentParser`, `add_json_arg`, `add_version_arg`, `--repo-root`, `parse_known_args`) and then read
   each hit. Structural greps are allowed to over-match; the review task is to classify hits, not blindly
   migrate every occurrence.

7. **Build shared validation parsers directly when version flags are already composed elsewhere.**
   If the proposed helper is `create_validation_parser(description)`, verify whether the existing
   `create_parser()` already adds `--version`. Reusing a helper that already calls `add_version_arg()`
   and then calling `add_version_arg()` again can create duplicate-option errors or inconsistent help.
   For this pattern, the safer plan is usually: construct `argparse.ArgumentParser` directly, then call
   the shared `add_json_arg()` and `add_version_arg()` exactly once. Confirm by reading the helper body,
   not by trusting a cited line number.

8. **Preserve special CLI semantics before deduplicating boilerplate.**
   Treat each non-standard entry point as its own contract:
   - `parse_known_args` pass-through must keep unknown args available for the downstream tool.
   - Positional path CLIs must keep their positional arguments and ordering.
   - `--repo-root` defaults must still resolve root-relative files such as ignore lists.
   - `--config` defaults may become root-relative only for defaults; explicit user-supplied paths must
     not be rewritten.
   - JSON/status output must not be polluted by logging changes.

9. **Verify resilience wrappers before approving GitHub call cleanup.**
   If a plan says `ensure_state_labels.py` already imports `gh_call`, and that `gh_call` is
   circuit-breaker wrapped in `hephaestus/github/client.py`, verify both facts in the current tree:
   read the imports/call sites in `ensure_state_labels.py`, read the adapter implementation, and grep
   for any remaining `subprocess.run(["gh", ...])` in the target automation path. Do not accept exact
   file:line/API claims from the plan unless they were re-opened after checkout.

10. **Make logging extraction opt-in and output-safe.**
    A shared `configure_cli_logging(verbose=False)` helper can silently change formatting, stderr/stdout
    routing, or default levels. For validation/version CLIs, require behavior tests that cover quiet,
    verbose, JSON, and status-output modes. Especially verify scripts that emit machine-readable JSON:
    logging belongs on stderr or must be suppressed so stdout remains parseable.

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
| CLI helper line cited but not re-opened | Plan cited `hephaestus/cli/utils.py:95` for helper behavior and assumed reusing `create_parser()` would be safe | Line numbers drift and helper bodies may already add `--version`; composing parser helpers blindly can duplicate options | Open the helper body in current HEAD and prove `add_json_arg` / `add_version_arg` are called exactly once per parser |
| Trust the issue-listed `rg` call-site list | Plan accepted a previously produced call-site inventory for validation/version CLIs as complete | The list was planned search output, not a current-tree fact; stale or narrow `rg` patterns miss semantically equivalent duplicates, while broad patterns catch intentional local variants | Re-run `rg` from current HEAD and classify each hit before scoping the migration |
| Flatten special CLIs into the generic helper | Planned migration assumed `mypy_per_file`, `type_aliases`, audit ignore defaults, and coverage config defaults could share the same parser/root behavior | These variants have observable contracts: pass-through args, positional paths, root-relative default files, and explicit config paths | Write behavior tests for each special variant before replacing boilerplate |
| Trust `gh_call` / circuit-breaker claims from plan text | Plan said `ensure_state_labels.py` already uses `gh_call` and that `gh_call` is circuit-breaker wrapped, with specific cited lines | The learn request did not re-open those files; if wrong, a refactor could preserve or introduce bare `gh` subprocess bypasses | Verify imports, call sites, adapter implementation, and a negative grep for bare `subprocess.run(["gh", ...])` in the automation path |
| Add shared logging without output tests | Introduced `configure_cli_logging(verbose=False)` as a shared helper | Logging level/format/stdout routing can subtly break JSON or status output even when parser tests pass | Test JSON stdout, status output, quiet/default verbosity, and verbose mode for migrated CLIs |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] Re-ran current-head `rg` for CLI parser/version/root/logging call sites; classified broad matches manually
- [ ] Confirmed parser helper composition calls `add_json_arg` / `add_version_arg` exactly once, with no duplicate `--version`
- [ ] Covered special CLI variants: `parse_known_args`, positional path CLIs, default ignore/config root behavior, explicit config preservation
- [ ] Verified GitHub CLI calls route through shared `gh_call` / circuit-breaker adapter; no bare `subprocess.run(["gh", ...])` remains in target path
- [ ] Tested behavior, not just helper presence: help/prog/usage, JSON output, repo-root defaults, pass-through args, version output, and logging streams
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1419 Specific Findings

| Assumption | Status | Reviewer Action |
|------------|--------|-----------------|
| `create_validation_parser(description)` can reuse the current `create_parser()` safely | UNVERIFIED | Re-open `hephaestus/cli/utils.py` and verify whether `create_parser()` already calls `add_version_arg`; build the new helper with `ArgumentParser` directly if reuse would duplicate `--version` |
| The issue-listed validation/version CLI call-site list is complete | UNVERIFIED | Re-run `rg` in current HEAD for parser/version/root/logging patterns and classify every hit before scoping |
| Special variants preserve behavior under generic helpers | UNVERIFIED | Test `mypy_per_file` pass-through, `type_aliases` positional paths, audit default ignore file via `--repo-root`, coverage default config via `--repo-root`, and explicit `--config` preservation |
| `ensure_state_labels.py` already uses `gh_call`, and `gh_call` is circuit-breaker wrapped | UNVERIFIED | Verify imports/call sites and the adapter implementation in `hephaestus/github/client.py`; add a regression check for no bare `gh` subprocess bypass |
| `configure_cli_logging(verbose=False)` is behavior-neutral | RISK | Verify stderr/stdout routing and default/verbose levels for JSON and status-output commands |

### CLI Boilerplate Refactor Review Checklist

```text
- [ ] Current HEAD was searched, not just issue/plan text.
- [ ] Shared helper composition cannot duplicate argparse options.
- [ ] Help/prog/usage/epilog snapshots or assertions cover migrated entry points where users rely on CLI UX.
- [ ] JSON stdout remains parseable; logging goes to stderr or stays suppressed.
- [ ] `--repo-root` changes affect only defaults that are meant to be root-relative.
- [ ] Explicit `--config` / path arguments are not rewritten unexpectedly.
- [ ] `parse_known_args` CLIs still pass unknown args through unchanged.
- [ ] Version entry points keep the same plain/JSON/root behavior.
- [ ] GitHub CLI calls use the shared resilience adapter; bare subprocess bypasses are rejected by tests or structural checks.
```

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning review for issue #1419 (extract validation/version CLI boilerplate into shared `hephaestus.cli.utils` helpers) | unverified — plan-review checklist captured from an unexecuted plan; reviewer must verify cited source lines, call-site inventory, special CLI variants, logging output, and `gh_call` routing before approval |
