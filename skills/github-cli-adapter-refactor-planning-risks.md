---
name: github-cli-adapter-refactor-planning-risks
description: "Plan and review migrations from raw GitHub CLI subprocess calls to a canonical GitHub adapter. Use when: (1) a codebase has scattered subprocess calls to gh, (2) a central gh_call wrapper provides resilience behavior, (3) issue evidence may be stale and reviewers need a fresh executable inventory."
category: architecture
date: 2026-06-26
version: "1.0.0"
verification: unverified
---

# GitHub CLI Adapter Refactor Planning Risks

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Preserve planning and review risks for migrating raw GitHub CLI subprocess calls behind a canonical adapter so they inherit circuit-breaker, throttling, and rate-limit retry behavior without touching unrelated subprocess use. |
| **Outcome** | Planning artifact only. The implementation plan for ProjectHephaestus issue #1400 was not executed end-to-end in this learning capture. |
| **Verification** | unverified |

## When to Use

- Planning or reviewing a refactor that replaces raw `subprocess.run(["gh", ...])` calls with a canonical wrapper such as `gh_call`.
- The adapter is expected to preserve `check`, `timeout`, `stdout`, `stderr`, and exception behavior close enough to `subprocess.CompletedProcess`.
- The module also runs non-GitHub commands such as `git`, `gpg`, shell helpers, or local scripts that must remain as subprocesses.
- An issue, audit, or review comment reports a raw-call count that may be stale because the branch has moved since the issue was filed.
- Tests need to prove behavior at the wrapper seam: repo scoping, dry-run output, timeout forwarding, and exception handling.

<!-- Validator compatibility: ## Verified Workflow -->

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```bash
# 1. Rebuild the current inventory instead of trusting stale issue counts.
python - <<'PY'
import ast
from pathlib import Path

for path in [Path("hephaestus/github/fleet_sync.py"), Path("hephaestus/github/severity_label.py")]:
    tree = ast.parse(path.read_text(), filename=str(path))
    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            target = ast.unparse(node.func)
            if target in {"subprocess.run", "run", "Popen", "check_output", "check_call"}:
                print(f"{path}:{node.lineno}: {ast.unparse(node)[:160]}")
PY

# 2. Verify the adapter contract before planning replacements.
grep -n "def gh_call\\|class GitHubUnavailableError\\|NETWORK_TIMEOUT" \
  hephaestus/github/client.py hephaestus/github/__init__.py hephaestus/utils/helpers.py

# 3. Confirm non-gh subprocesses stay outside the adapter.
grep -RIn "subprocess\\.\\|Popen\\|check_output\\|check_call" \
  hephaestus/github/fleet_sync.py hephaestus/github/severity_label.py
```

### Detailed Steps

1. **Rebuild the inventory from the current branch.**
   Treat issue text and audit counts as historical hints. Before estimating scope, generate a fresh inventory of all subprocess entry points in the target files. Include aliases, imported subprocess functions, `Popen`, `check_output`, `check_call`, variable/list/tuple command construction, command concatenation, and `shell=True` strings. The goal is not just to count calls; it is to classify each command as `gh` versus non-`gh`.

2. **Verify the canonical adapter contract before writing the migration plan.**
   Read the current `gh_call` definition and exports. Confirm whether it expects args without the leading `gh`, accepts `check=...` and `timeout=...`, and returns a CompletedProcess-like object with the exact `stdout` and `stderr` fields that callers use. Do not infer the return shape from similar wrappers or from older plan notes.

3. **Keep command ownership explicit.**
   Only GitHub CLI calls should move behind `gh_call`. Leave `git`, `gpg`, filesystem, and local helper subprocess calls untouched unless a separate issue covers them. The plan should say this directly so an implementer does not route every subprocess through a GitHub-specific adapter.

4. **Test wrapper-seam behavior, not only call-site rewrites.**
   Add focused tests at the helper seam for repo scoping and timeout forwarding. If a fleet wrapper prepends `--repo org/repo`, test both paths: when no repo flag is present and when `--repo` or `-R` already exists. If dry-run returns `stdout="[]"`, verify every current caller either expects a list response or handles that sentinel safely.

5. **Review exception boundaries carefully.**
   If the old path caught `subprocess.CalledProcessError` and returned `False`, decide explicitly whether `GitHubUnavailableError`, circuit-breaker open errors, or other `RuntimeError` subclasses should follow that same path. Confirm the inheritance in code before the plan says `except RuntimeError` is safe.

6. **Use an AST guard as a regression net, not as proof by itself.**
   A guard that searches only for the current code shape creates false assurance. It should detect aliased imports, imported functions, subprocess module calls, string commands, variable commands, and shell execution. It must also avoid failing on intentional non-`gh` commands like `git` and `gpg`.

7. **Make every evidence claim refreshable.**
   File:line citations and raw-call counts should be treated as pre-plan snapshots. The plan should include commands to refresh them on the implementation branch and reviewers should rerun those commands before accepting the plan.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Trust issue raw-call count | Used the issue's claim that there were 13 raw `gh` calls as the scope boundary | Issue evidence can be stale after prior refactors; a correct implementation may look incomplete or may touch unrelated subprocesses to satisfy an old count | Demand a fresh executable inventory from the current branch before planning or reviewing |
| Infer `gh_call` contract from intent | Assumed `gh_call` accepts args without leading `gh`, supports `check` and `timeout`, and returns CompletedProcess-like `stdout`/`stderr` | If the wrapper contract differs, the migration can compile but break text handling, exception handling, or timeout behavior | Read the actual adapter definition and tests before writing call-site steps |
| Route all subprocesses through the adapter | Treated every subprocess in the target files as part of the GitHub CLI migration | Non-`gh` commands such as `git` and `gpg` have different semantics and should not inherit GitHub-specific resilience behavior | Classify commands first; migrate only `gh` invocations unless a separate issue says otherwise |
| Add a narrow raw-call guard | Planned an AST guard that only recognized `subprocess.run(["gh", ...])` in the current shape | Aliases, imported functions, `Popen`, shell strings, and variable command construction can bypass the guard | Make guard coverage broad enough to catch realistic raw `gh` forms while exempting intentional non-`gh` commands |
| Assume adapter failures behave like `CalledProcessError` | Planned to catch circuit-breaker/runtime failures where old code caught `CalledProcessError` without verifying inheritance | The old path may have returned `False`; an uncaught wrapper exception could crash a path that previously degraded gracefully, while overbroad catching could swallow real bugs | Verify the exception hierarchy and preserve caller-visible behavior intentionally |

## Results & Parameters

### Planning Checklist

```text
- [ ] Rebuilt subprocess inventory from the current branch, including aliases and shell strings.
- [ ] Classified every command as gh versus non-gh; non-gh commands remain subprocesses.
- [ ] Verified gh_call signature, return shape, exported import path, and exception hierarchy in code.
- [ ] Verified NETWORK_TIMEOUT still applies to GitHub CLI wrapper calls.
- [ ] Tested repo scoping at the helper seam for no existing repo flag, existing --repo, and existing -R.
- [ ] Tested dry-run sentinel output against every current caller of the helper.
- [ ] Tested timeout forwarding and check=True/check=False behavior at the wrapper seam.
- [ ] Reviewed merge/error paths so circuit-breaker failures are neither swallowed incorrectly nor allowed to crash paths that previously returned False.
- [ ] Added a broad AST guard for raw gh calls without blocking intentional git/gpg subprocesses.
- [ ] Reran file:line and raw-call evidence on the implementation branch before approving.
```

### Unverified Assumptions To Call Out In Plans

| Assumption | Required Verification |
|------------|-----------------------|
| `gh_call` is the canonical adapter | Read `hephaestus/github/client.py` and `hephaestus/github/__init__.py`; confirm exports and local usage. |
| `gh_call` has the needed subprocess-like contract | Inspect signature and tests for args without leading `gh`, `check`, `timeout`, `stdout`, and `stderr`. |
| `NETWORK_TIMEOUT` is the right timeout | Verify helper documentation and current implementation behavior match. |
| Repo scoping via prepending `--repo org/repo` preserves semantics | Test command construction when `--repo` or `-R` is already present. |
| Dry-run `stdout="[]"` is safe for all callers | Inspect every helper caller and add regression tests for any non-list expectation. |
| Adapter exceptions can be handled like subprocess failures | Confirm `GitHubUnavailableError` and circuit-breaker failures inherit from the expected base class and preserve old return behavior. |
| Remaining subprocess calls are only `git`/`gpg` | Rebuild the executable inventory and classify each remaining command. |

### Review Focus

Reviewers should require executable evidence for the inventory and adapter contract. A plan that cites stale issue counts, unverified file:line references, or inferred wrapper semantics should be sent back before implementation starts.
