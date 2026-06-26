---
name: refactor-extraction-plan-unverified-assumptions
description: "The uncertain assumptions and reviewer risks baked into a large refactor PLAN that extracts functions, replaces a module with a package facade, or standardizes CLI parser construction while preserving compatibility. Use when: (1) reviewing or authoring a plan that moves a function cluster out of a god-module and re-exports moved names, (2) replacing `module.py` with `module/` while keeping imports and console entrypoints stable, (3) migrating direct `argparse.ArgumentParser(...)` sites to a shared helper, (4) the plan claims existing mock patch paths or `old.module:main` entrypoints keep working, (5) the plan cites exact counts or line numbers from planning-context discovery, (6) the plan asserts no circular imports, no stale patch targets, or unchanged subprocess defaults by static reasoning."
category: architecture
date: 2026-06-26
version: "1.1.0"
user-invocable: false
verification: unverified
history: refactor-extraction-plan-unverified-assumptions.history
tags: [refactoring, extraction, cluster-extraction, backward-compat, re-export, mock-patch-path, planning, reviewer-risks, line-number-drift, circular-import, assumptions, argparse, parser-standardization, package-conversion, console-entrypoint, facade, issue-acceptance, subprocess-defaults]
---

# Refactor Extraction Plan — Parser, Package, and Compatibility Risks

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture the uncertain assumptions a refactor PLAN makes when it combines compatibility-sensitive extraction with parser standardization or module-to-package conversion, so a reviewer knows what must be revalidated before implementation. |
| **Outcome** | Plan-review guidance produced; NOT executed. These are assumptions a reviewer must verify and an implementer must not take on faith. |
| **Verification** | unverified — planning artifact only; no implementation, import smoke test, parser migration, or CI run was executed |
| **History** | [changelog](./refactor-extraction-plan-unverified-assumptions.history) |

> This skill is about the **PLANNING-RISK** angle, not the mechanics of *how* to extract a cluster.
> For the how-to mechanics see `python-module-decomposition-and-refactor-patterns` and
> `testing-module-patch-target-after-extraction`. For DRY *consolidation* (two modules → one)
> see `dry-refactoring-plan-assumption-audit`. This skill covers compatibility-sensitive extraction
> and adjacent plan-review risks: parser-helper migrations, `module.py` → `module/` package facades,
> entrypoint compatibility, stale counts, and hidden test patch targets.

## When to Use

- Reviewing or authoring a plan that extracts a cluster of functions out of a large module into a new sibling module.
- The plan keeps the original module importable by adding `from .new_module import _fn as _fn` re-exports (backward-compat shim) and claims existing `unittest.mock.patch("...old_module._fn")` calls still work.
- The plan replaces `package/module.py` with `package/module/` and assumes `package.module:main` console entrypoints and `import package.module` compatibility still work.
- The plan standardizes many direct `argparse.ArgumentParser(...)` sites behind a shared `create_parser` helper and assumes custom help, usage, epilog, formatter, and `add_help` behavior will be preserved.
- The plan combines two or more acceptance surfaces from an issue title/body, and those surfaces were inferred from prior context rather than re-read live from the issue.
- The plan cites exact line numbers (`module.py:566-942`, `pyproject.toml:263`) for a sequence of edits that delete/insert large blocks.
- The plan cites exact population counts (for example, "18 direct parser sites across 17 files") from planning-context discovery.
- The plan depends on a "frozen" list or magic number (e.g. an omit/coverage allowlist "frozen at N modules") that must be bumped in multiple places.
- The plan asserts "introduces no circular import" or "this import is now unused and can be removed" by static reasoning rather than by actually importing/linting.
- The plan extracts a shared subprocess or git helper and assumes timeout, logging, environment, `check`, `capture_output`, or `text` defaults remain behavior-preserving across both old callers.

## Proposed Workflow

<!-- Validator compatibility: scripts/validate_plugins.py currently requires the literal token
"## Verified Workflow" even for unverified skills. This section is intentionally Proposed Workflow. -->

> **Warning:** This workflow has not been validated end-to-end. No code was written or run. It is the
> reviewer/author checklist distilled from an *unexecuted* plan. Treat every item as a hypothesis
> until CI confirms.

### Quick Reference

```bash
# === Reviewer / implementer pre-flight for a compatibility-sensitive refactor plan ===
# Replace OLD_MODULE with the dotted path being extracted FROM (e.g. hephaestus.automation.loop_runner)
# and run from the repo root.

OLD_MODULE="hephaestus.automation.loop_runner"   # the module losing functions
OLD_PATH="hephaestus/automation/loop_runner.py"  # its file
NEW_MODULE="loop_repo_manager"                    # the new sibling (bare name as imported within the pkg)
ISSUE="1406"                                      # replace with the live issue number
VALIDATION_ROOT="hephaestus/validation"           # replace with the parser-migration tree
ENTRYPOINT="hephaestus-fleet-sync = hephaestus.github.fleet_sync:main"

# 0. ACCEPTANCE-SURFACE CHECK — re-read the live issue, not the plan summary.
gh issue view "${ISSUE}" --json title,body,comments --jq '{title: .title, body: .body}'

# 1. PARSER POPULATION CHECK — re-count direct parser sites before implementing.
rg -n "argparse\.ArgumentParser\(" "${VALIDATION_ROOT}" tests scripts

# 2. PARSER BEHAVIOR CHECK — preserve custom kwargs and version flags.
rg -n "formatter_class|epilog|usage|add_help|description" "${VALIDATION_ROOT}" tests scripts
rg -n '"-V"|"--version"|action=["'\'']version["'\'']' "${VALIDATION_ROOT}" tests scripts

# 3. CENTRAL CHECK — does ANY module import the moved private names directly (not via the namespace)?
#    Re-exports only preserve patch paths for callers that look up the name through OLD_MODULE
#    at call time. A `from OLD_MODULE import _fn` anywhere ELSE binds a separate name that
#    patching OLD_MODULE._fn will NOT affect. Grep the WHOLE repo, not just OLD_PATH + its test.
grep -rn "from ${OLD_MODULE} import" hephaestus/ tests/ scripts/
#    Inspect every hit: any moved symbol imported by name into another module = a patch path that breaks.

# 4. ENTRYPOINT / IMPORT CHECK — prove a module-to-package facade imports and exposes main.
rg -n "${ENTRYPOINT}" pyproject.toml setup.cfg setup.py
python -c "import importlib; m = importlib.import_module('${OLD_MODULE}'); assert callable(getattr(m, 'main', None)); print('entrypoint import OK')"

# 5. PATCH-TARGET CHECK — locate tests that patch old module paths before moving symbols.
rg -n "patch\(.*${OLD_MODULE}|monkeypatch\.setattr\(.*${OLD_MODULE}|from ${OLD_MODULE} import" tests hephaestus scripts

# 6. SHARED SUBPROCESS / GIT HELPER CHECK — preserve caller-specific defaults.
rg -n "subprocess\.(run|check_call|check_output)|timeout=|capture_output=|check=|text=|env=" "${OLD_PATH}" hephaestus/ tests/

# 7. MAGIC-NUMBER / FROZEN-LIST CHECK — find EVERY occurrence of the invariant count/list literal,
#    then READ the assertion body (is it `len(...) == 16` literal, or set membership?). Don't trust the comment.
grep -rn "16" pyproject.toml | grep -i "omit\|allowlist\|module"   # adjust literal/keyword
grep -rn "allowlist\|omit" tests/ pyproject.toml
#    Open the test and read whether it asserts a count literal vs a set membership before bumping anything.

# 8. NO-CYCLE CHECK — prove by EXECUTION, not by reading comments.
python -c "import ${OLD_MODULE}; from hephaestus.automation import ${NEW_MODULE}; print('import OK')"

# 9. RE-EXPORT IDENTITY SMOKE TEST — prove the shim actually re-binds the moved object.
python -c "import ${OLD_MODULE} as o; from hephaestus.automation import ${NEW_MODULE} as n; \
assert o._gh_list_repos is n._gh_list_repos, 'identity broken'; print('re-export identity OK')"

# 10. UNUSED-IMPORT CHECK — let ruff be the source of truth, do NOT hand-judge.
ruff check ${OLD_PATH} --select F401   # then `ruff check --fix` only after the deletion

# 11. LINE-NUMBER DRIFT REMINDER — after the FIRST block deletion, every later cited line number is stale.
#    Re-derive targets by stable marker, not by the plan's pre-edit numbers:
grep -n "# Repo discovery\|^def _gh_list_repos\|^def _gh_" ${OLD_PATH}
```

### Detailed Steps

1. **Re-read the live issue title/body before grading acceptance coverage.**
   A plan can quietly treat the issue title as one acceptance criterion and the body as another. That may be
   correct, but it is not verified until the reviewer fetches the live issue and checks that the plan covers
   both surfaces explicitly. If the title says "standardize validation parsers" and the body says "split a
   fleet sync module," require separate acceptance mapping and separate verification commands.

2. **Recompute planning-context counts and line numbers on the current checkout.**
   Counts such as "18 direct `argparse.ArgumentParser(...)` sites across 17 files" are snapshots. They drift
   with parallel PRs and rebases. Treat them as search seeds, not facts. Re-run the exact population query
   before approving the edit order, and prefer stable names/markers over line numbers.

3. **For parser helper migrations, preserve per-site behavior before deleting direct constructors.**
   A shared `create_parser` helper is only behavior-preserving if it carries through every site-specific
   `formatter_class`, `usage`, `epilog`, `description`, and `add_help` detail. Also grep existing `-V` and
   `--version` flags before adding a helper default, because duplicate version options fail at parser
   construction time. Do not remove `argparse` imports blindly; annotations or `argparse.Namespace` references
   may still need them.

4. **For `module.py` → `module/` conversions, prove the old import and entrypoint surfaces.**
   Replacing a file module with a package is more invasive than moving functions to a sibling module. A facade
   `__init__.py` must expose the same public names, and `console_scripts = old.module:main` must still resolve.
   Add an import smoke test and an entrypoint smoke test before trusting the package conversion.

5. **Verify re-export patch-path preservation against the WHOLE repo (the central assumption).**
   The plan's load-bearing claim is that `patch("...loop_runner._gh_list_repos")` keeps working after
   the function moves, because the re-export makes the name a real attribute of `loop_runner` and
   callers resolve it through the `loop_runner` namespace at call time. This is **only true** when
   every internal caller does a bare global lookup in `loop_runner` (or `loop_runner._gh_list_repos`).
   It **breaks** if any other module did `from ...loop_runner import _gh_list_repos`: that binding is a
   separate name in the other module, and patching `loop_runner._gh_list_repos` will not touch it.
   Grep `from <old_module> import` across **`hephaestus/`, `tests/`, and `scripts/`** — not just the
   module and its own test file.

6. **Scan symbol boundaries for hidden state and stale patch targets.**
   A symbol scan can find functions that look independent while missing shared globals, module-level caches,
   default arguments, logger state, or tests patching old paths. Before approving module boundaries, grep for
   each moved symbol, its globals, and every `patch(...)` or `monkeypatch.setattr(...)` target that names the
   old module.

7. **When extracting shared git/subprocess helpers, lock down default options.**
   Helper extraction is not automatically DRY-safe. A shared `run_git(...)` can change behavior if it
   standardizes `timeout`, `check`, `capture_output`, `text`, `cwd`, `env`, or logging defaults that differed
   between callers. Diff caller expectations first, then make tests assert the exact subprocess kwargs for
   the high-risk call paths.

8. **Anchor multi-edit instructions to stable markers, not absolute line numbers.**
   In a sequential multi-edit plan, deleting the first block (e.g. `:566-942`) shifts every later cited
   line. Use function names, unique strings, or section banners (`# Repo discovery`) as anchors, OR state
   explicitly in the plan that line numbers are pre-edit and must be re-derived after each deletion.

9. **Enumerate every place the frozen invariant appears, and read the assertion body.**
   When the plan bumps a "frozen-at-16" allowlist to 17, grep the literal `16` and the keyword across
   `pyproject.toml` and `tests/`. Open the test and confirm whether it is a hardcoded count
   (`len(expected) == 16`) or a set-membership check — the required edits differ. The comment count
   and the asserted count can live in different files; miss one and CI fails.

10. **Prove "no circular import" by execution.**
   "New module → ci_driver introduces no cycle because ci_driver only mentions loop_runner in comments"
   is static reasoning. ci_driver may import another automation module that transitively loads
   loop_runner at import time. The claim is only proven when the import smoke test actually runs.
   Flag it as "verify by execution," never "verified by reading."

11. **Defer conditional import removals to ruff, gated by per-symbol grep.**
   "Drop `urlparse`/`gh_cli_timeout` only if no other references remain" is exactly where hand-judgement
   slips. Removing a still-used import breaks the module; leaving an unused one fails ruff F401. After the
   deletion, grep each symbol in the file and let `ruff check --select F401` (then `--fix`) be the source
   of truth instead of eyeballing it.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Treat issue title/body interpretation as verified | Plan framed the issue as two acceptance surfaces without fetching the live issue text in the planning turn | The plan may be right, but the reviewer cannot know whether the title and body really encode separate requirements until the live issue is re-read | Fetch the issue title/body before approving acceptance mapping; do not let prior planning context stand in for the source of truth |
| Trust parser-site counts from prior discovery | Plan cited 18 direct `argparse.ArgumentParser(...)` sites across 17 validation files from earlier discovery | Counts and line numbers drift with parallel edits; a stale count makes implementation incomplete or causes churn in files that no longer match | Re-run the exact parser-site query on the implementation checkout and treat old counts as hints only |
| Standardize parsers without preserving custom constructor behavior | Plan assumed a shared `create_parser` helper could replace direct parser construction everywhere | Custom `formatter_class`, `usage`, `epilog`, `description`, `add_help`, and version flags are behavior, not decoration; defaults can change help output or create duplicate `-V/--version` conflicts | Before migration, inventory constructor kwargs and existing version flags; helper APIs must pass through or intentionally preserve every observed behavior |
| Replace a file module with a package facade without smoke tests | Plan assumed `module.py` could become `module/` while old imports and `module:main` console entrypoints stayed compatible | Python import resolution and packaging entrypoints usually support this shape, but facade exports, `__all__`, relative imports, and cycles can still break in the target repo | Add import, entrypoint, and facade export parity smoke tests before trusting a file-to-package conversion |
| Draw module boundaries from a symbol scan only | Plan grouped functions by apparent symbol ownership and static references | Static scans can miss shared globals, logger state, module caches, default args, and tests that patch old module paths | Scan moved symbols plus their globals and every `patch` or `monkeypatch` target before finalizing boundaries |
| Extract a shared git/subprocess helper without pinning caller defaults | Plan assumed common git-command code could be shared across workflows | Different callers may rely on different `timeout`, logging, `check`, `capture_output`, `text`, `cwd`, or `env` defaults; helper cleanup can silently alter retry and failure semantics | Write tests that assert subprocess kwargs for each caller before and after helper extraction |
| Assume re-export preserves all mock patch paths | Plan claimed `patch("...loop_runner._gh_list_repos")` keeps working post-move because the `import X as X` re-export makes the name a real attribute, and only grepped within `loop_runner.py` + `test_loop_runner.py` | True only if every caller resolves the name through the `loop_runner` namespace at call time; a `from loop_runner import _gh_list_repos` in ANY other module is a separate binding that patching `loop_runner` does not affect — and that scope was never grepped | Before claiming re-exports preserve patch paths, grep the ENTIRE repo (`hephaestus/` AND `tests/` AND `scripts/`) for `from <old_module> import _<name>`, not just the two files you happened to read |
| Cite exact line numbers for a sequential multi-edit | Plan referenced `loop_runner.py:566-942`, `:1198`, `:1359`, `pyproject.toml:263`, `test_omit_allowlist.py:40-53` read at plan time | An implementer edits sequentially; deleting the `566-942` block shifts every later line, so literal line-number targeting after step 1 hits the wrong lines | Anchor multi-edit instructions to stable markers (function names, section banners, unique strings), or explicitly state line numbers are pre-edit and must be re-derived after each deletion |
| Trust the "frozen at 16 modules" comment | Plan asserted the omit allowlist is frozen at 16 and only the comment + one test need bumping to 17, without running the test or reading its assertion body | The "16" figure and the assertion mechanism (count literal vs set membership) were read, not verified; a hardcoded `len(...) == 16` elsewhere, or a third copy of the count, would be missed and fail CI | When a plan depends on a frozen-list/magic-number invariant, READ the actual assertion body (don't trust the comment) and grep the literal to enumerate EVERY place it appears |
| Reason about circular imports statically | Plan claimed `loop_repo_manager → ci_driver` adds no cycle because ci_driver references loop_runner only in comments | Static reasoning; ci_driver could import another automation module that transitively imports loop_runner at module load, creating a cycle the comment-scan never sees | The import-graph claim is only proven by the smoke-test step actually executing — label it "verify by execution," not "verified by reading" |
| Hand-judge conditional unused-import removal | Plan said to drop `urlparse` and `gh_cli_timeout` from loop_runner "only if no other references remain," left to manual judgement | Removing a still-used import breaks the module; leaving a genuinely-unused one fails ruff F401 — both directions of hand-judgement are wrong | Gate each import removal on a per-symbol grep AFTER the deletion and make `ruff check --select F401` / `--fix` the source of truth, not eyeballing |

## Results & Parameters

### What the plan got RIGHT (keep these strengths)

- **Read the actual code before planning** — confirmed the issue's line numbers were approximately right and that the 12 functions are genuinely self-contained pure helpers safe to move verbatim.
- **Enumerated call sites and patch sites with grep before asserting re-exports are safe** — correct instinct; the only defect was incomplete scope (see assumption #1).
- **Added an identity smoke test** (`assert loop_runner.X is loop_repo_manager.X`) to prove the re-export wiring binds the same object.
- **Mapped a per-criterion verification command to each acceptance criterion** so the plan is checkable rather than narrative.
- **Separated distinct acceptance surfaces** — a combined parser-standardization plus module-split plan is reviewable only if each surface gets its own source-of-truth check and verification command.

### Reviewer focus (the things to check hardest in such a plan)

```
## Compatibility-sensitive refactor plan review checklist

- [ ] Did the reviewer fetch the live issue title/body and map every acceptance surface?
- [ ] Were parser-site counts, file counts, and line numbers recomputed on the implementation checkout?
- [ ] Does the parser helper preserve formatter, usage, epilog, description, add_help, and version flag behavior?
- [ ] Did the plan check for duplicate -V/--version flags before adding a helper default?
- [ ] Does module.py -> module/ keep import, facade export, __all__, and console entrypoint compatibility?
- [ ] Did patch-target grep cover the WHOLE repo (source + tests + scripts), not just the obvious test file?
- [ ] Were module boundaries checked for shared globals, caches, default args, and logger state?
- [ ] Are shared git/subprocess helper defaults pinned with tests per caller?
- [ ] Is the "no circular import" claim validated by an EXECUTED import smoke test?
- [ ] Are conditional import removals deferred to ruff F401, not hand-judged?
```

### Issue #1406 specific findings

| Assumption in the plan | Status | What a reviewer must do |
|------------------------|--------|-------------------------|
| Issue title/body encode two acceptance surfaces: validation parser standardization and `fleet_sync.py` split | UNVERIFIED | Fetch GitHub issue #1406 title/body and confirm both requirements are explicit before grading coverage |
| 18 direct `argparse.ArgumentParser(...)` sites across 17 validation files | UNVERIFIED SNAPSHOT | Re-run the direct parser-site query on the implementation checkout and update the plan if counts drifted |
| `create_parser` extension is behavior-preserving | UNVERIFIED | Inventory every custom parser kwarg plus existing `-V/--version` flags; add help-output and duplicate-flag regression tests |
| `hephaestus.github.fleet_sync` can change from file module to package while `hephaestus-fleet-sync = hephaestus.github.fleet_sync:main` still works | REASONED, NOT RUN | Add import and console-entrypoint smoke tests, plus facade export parity checks |
| Proposed module boundaries are safe because symbol scans grouped the functions | UNVERIFIED | Scan for cross-function state, globals, caches, logger usage, and patch targets before moving code |
| Shared git helper extraction preserves tidy and fleet behavior | UNVERIFIED | Assert subprocess kwargs and timeout/logging behavior for both old caller families |

### Issue #1360 specific findings

| Assumption in the plan | Status | What a reviewer must do |
|------------------------|--------|-------------------------|
| Re-export preserves all `patch("...loop_runner._fn")` paths | UNVERIFIED (scope too narrow) | Grep `from hephaestus.automation.loop_runner import` across hephaestus/, tests/, scripts/ — any direct private-name importer breaks |
| Cited line numbers (`:566-942`, `:1198`, `:1359`, `pyproject.toml:263`) are actionable as-is | FRAGILE | Re-derive by marker after the first block deletion; numbers are pre-edit snapshots |
| Omit allowlist "frozen at 16," bump in pyproject comment + one test | UNVERIFIED | Read the assertion body (count literal vs set membership); grep `16` for a third location |
| `loop_repo_manager → ci_driver` adds no cycle | REASONED, NOT RUN | Run `python -c "import ...loop_runner; from ...automation import loop_repo_manager"` |
| `urlparse` / `gh_cli_timeout` are safely removable | CONDITIONAL | Per-symbol grep after deletion; let `ruff --select F401` decide |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1406 (validation parser standardization plus `fleet_sync.py` module-to-package split while preserving imports and entrypoints) | Plan produced, NOT executed; this skill records unverified assumptions and reviewer risks. Implementation pending. |
| ProjectHephaestus | Planning phase for issue #1360 (extract 12-function repo-management cluster from `automation/loop_runner.py` into `loop_repo_manager.py` with backward-compat re-exports) | Plan produced, NOT executed; this skill records the unverified assumptions and reviewer risks. Implementation pending. |
