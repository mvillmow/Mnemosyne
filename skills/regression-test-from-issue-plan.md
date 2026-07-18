---
name: regression-test-from-issue-plan
description: 'Add missing regression tests by reading the issue''s implementation
  plan. Use when: (1) an issue references a test by name that should exist as a harness,
  (2) a bug fix landed but the named test was never added, (3) issue comments contain
  hand-computed expected values, (4) the code fix is ALREADY MERGED (a half-zombie
  issue whose only live acceptance criterion is the paired regression test) so you
  cannot get a genuine RED from committed code and must demonstrate "fails-before"
  via a reviewer-runnable, NON-COMMITTED source mutation, (5) the guard pins a
  single value in an allowlist/dispatch table and needs a positive + discriminating
  negative pair to avoid passing vacuously.'
category: testing
date: 2026-07-17
version: 1.1.0
user-invocable: false
verification: verified-local
---
## Overview

| Field | Value |
| ------- | ------- |
| **Purpose** | Recover and implement regression tests specified in GitHub issue plans that were never added |
| **Input** | GitHub issue number with a named test in the description or comments |
| **Output** | New test function with verified passing behavior |
| **Risk** | Low — additive only, no production code changes |

## When to Use

- Issue description says "test X will serve as the regression harness" but test X does not exist in the codebase
- A bug fix is already merged to main but the paired regression test is missing
- Issue comments contain hand-computed expected values for specific inputs
- Issue cross-references another issue that planned a test (e.g. "follow-up from #NNNN")
- `grep -r "test_<name>" tests/` returns no matches for a test the issue expects to exist
- **Half-zombie issue**: the referenced fix PR is already MERGED (the source
  behavior is correct on `main`) but the issue's binding acceptance criterion is
  a *focused regression test that fails before and passes after*. You cannot get
  a genuine RED from committed code — the "fails-before" must be demonstrated a
  different way (see the *already-merged* section below).
- The behavior to pin is a single entry in an allowlist / lookup / dispatch
  table (e.g. a valid git-porcelain status pair such as `" T"` inside a
  `frozenset` of accepted pairs). A bare positive assertion can pass vacuously;
  it needs a discriminating negative sibling.

## Verified Workflow

### Quick Reference

```bash
# 1. Find what test name the issue expects
gh issue view <N> | grep "test_"
gh issue view <N> --comments | grep -A5 "test_"

# 2. Check if test exists
grep -r "test_<name>" tests/

# 3. Check if the fix is already in source
git log --oneline -- <file> | head -5

# 4. Read the plan for expected values
gh issue view <REFERENCED_ISSUE> --comments

# 5. Count tests in target file (≤10 per file)
grep -c "^fn test_" tests/path/to/test_file_part2.mojo

# 6. Validate pre-commit before commit
SKIP=mojo-format pixi run pre-commit run --files <file>
```

### Steps

1. **Read the issue** — `gh issue view <N>` — note exact test name mentioned
2. **Search for the test** — `grep -r "test_<name>" tests/` confirms it is missing
3. **Check fix status** — `git log --oneline -- shared/core/<file>.mojo | head -5`
4. **Read the referenced issue plan** — `gh issue view <REFERENCED> --comments` for expected values
5. **Find the right part file** — count tests with `grep -c "^fn test_"` per file, pick one with room
6. **Write the test** — use hand-computed expected values from the plan; manually set strides if testing stride logic
7. **Add to main()** — append the call in the appropriate section
8. **Validate** — `SKIP=mojo-format pixi run pre-commit run --files <file>`
9. **Commit and PR** — `git add <file> && git commit -m "test(scope): add <name> regression test"`

### When the fix is ALREADY MERGED: proving "fails-before" without a real RED

If step 3 (`git log`) shows the source fix already landed, the paired test is
GREEN the moment you write it — there is no pre-fix commit to run it against, so
you cannot demonstrate a genuine RED. Do **not** hand-write or commit a failure
log to fake it (RUNNABLE-EVIDENCE / ADR-014). Instead, prove the guard is
non-vacuous with a **reviewer-runnable, NON-COMMITTED source mutation**:

1. Write the positive test that asserts the fixed behavior; confirm it PASSES on
   unmodified `main`.
2. In the *plan and PR body only*, author the fails-before demonstration as an
   inspectable command sequence: temporarily revert the one-token change that the
   merged fix introduced (e.g. delete `T` from the allowlist generator), re-run
   the new test — it MUST report `1 failed` — then `git checkout --` the file and
   confirm `1 passed` again. The mutation is never committed.
3. Pair the positive test with a **discriminating negative** so the guard cannot
   pass vacuously: assert that a sibling value the allowlist should still REJECT
   raises. Pick negatives that are absent from the set but adjacent to accepted
   entries, so they exercise the reject branch without colliding with a valid
   pair. Verify the choice by enumerating the actual set before asserting:

```bash
# Enumerate the real allowlist so the negative params are provably REJECTed
python3 - <<'PY'
pairs = (
    {"D ", "DD", "AU", "UD", "UA", "DU", "AA", "UU", "??"}
    | {f" {w}" for w in "AMDTRC"}
    | {f"{i}{w}" for i in "MTARC" for w in " MTD"}
)
for s in (" T", "TA", "TR", "TC", "TM", "TD", "T "):
    print(f"{s!r}: {'ACCEPT' if s in pairs else 'REJECT'}")
PY
# → ' T': ACCEPT ; 'TA'/'TR'/'TC': REJECT ; 'TM'/'TD'/'T ': ACCEPT
# so parametrize the negative test over ("TA","TR","TC"), NOT "TM"/"TD"/"T ".
```

```python
def test_accepts_worktree_type_change_status(self) -> None:
    """Regression: a worktree-only type change `" T"` is a valid porcelain pair.

    The fix (already merged) added `" T"` to the accepted set; this pins it so a
    future edit to that allowlist cannot silently reject the entry again.
    """
    assert pr_manager._parse_porcelain_status(" T src/type-changed.py\0") == (
        (" T", "src/type-changed.py"),
    )

@pytest.mark.parametrize("status", ("TA", "TR", "TC"))
def test_rejects_invalid_type_change_status(self, status: str) -> None:
    """Discriminating negative: an unsupported T-pair must still raise."""
    with pytest.raises(RuntimeError, match="Malformed"):
        pr_manager._parse_porcelain_status(f"{status} src/type-changed.py\0")
```

**Why this passes review:** the reviewer NOGOs an already-merged "regression"
test that only re-encodes shipped behavior with no proof it gates anything. The
non-committed mutation names *exactly* the change each test catches, and the
positive+negative pair proves the guard discriminates the accepted value from
its rejected siblings. Confine the diff to the test file — re-editing the
already-correct source is churn (KISS/YAGNI) and re-opens the silent-corruption
class the fix closed.

### Key insight: `_get_float64` vs stride-based indexing

When `as_contiguous()` uses `_get_float64(i)` with a flat index, it reads `index * dtype_size` from
memory — ignoring strides entirely. This means manually mutating `_strides` changes what
`is_contiguous()` reports but does NOT change the flat memory layout. The non-contiguous branch of
`as_contiguous()` that uses stride arithmetic will reorder values; the flat-copy branch preserves them.
Know which branch is exercised before computing expected values.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
| --------- | ---------------- | --------------- | ---------------- |
| Search for test by exact name | `grep -r "test_contiguous_stride_correct_values" tests/` | Returned no matches — test never existed | Absence of a test is exactly the bug to fix |
| Checked existing branch for the fix | Expected `4088-fix-as-contiguous` branch to contain the test | Branch had a different change (pre-commit hook), not the regression test | Branch names can be misleading; use `git diff main..<branch> -- <file>` to confirm |
| Considered adding to original test file | `test_utility.mojo` seemed consistent with issue plan | File had 43 tests — far exceeds the ≤10 fn test_ limit | Always count tests before adding; use numbered part files |
| Assumed the fix still needed implementing | Expected the source fix to be missing | Fix was already in `shape.mojo` from a prior commit | Check `git log` for the source file first — fix and test can land separately |
| Planned to re-edit the source to force a RED | On a half-zombie issue whose fix was already merged (#2208), considered re-adding the parser change to get a genuine fails-before | The value was already in the allowlist; re-editing correct source is churn and re-opens the silent-corruption class the fix closed | When the fix is merged, demonstrate fails-before with a NON-COMMITTED mutation in the plan/PR, and confine the committed diff to the test file |
| Wrote only a positive assertion | `assert _parse_porcelain_status(" T …") == ((" T", …),)` alone | A bare positive re-encodes shipped behavior and can pass vacuously; a reviewer NOGOs "tests encode the contract" with no named mutation | Pair every positive with a discriminating negative over sibling values the set must REJECT (enumerate the set first to pick provably-rejected params) |

## Results & Parameters

**Test template for stride-based bug regression (Mojo)**:

```mojo
fn test_contiguous_stride_correct_values() raises:
    """Regression test: as_contiguous() must use stride-based indexing.

    For shape [2, 3] with strides [1, 2] (column-major), the stride-based
    source offsets produce:
      Output[0,0] = src[0*1 + 0*2] = src[0] = 0.0
      Output[0,1] = src[0*1 + 1*2] = src[2] = 2.0
      Output[0,2] = src[0*1 + 2*2] = src[4] = 4.0
      Output[1,0] = src[1*1 + 0*2] = src[1] = 1.0
      Output[1,1] = src[1*1 + 1*2] = src[3] = 3.0
      Output[1,2] = src[1*1 + 2*2] = src[5] = 5.0
    Flat output: [0.0, 2.0, 4.0, 1.0, 3.0, 5.0]
    """
    var shape = List[Int]()
    shape.append(2)
    shape.append(3)
    var a = arange(0.0, 6.0, 1.0, DType.float32)
    var b = a.reshape(shape)

    # Manually set column-major strides to simulate non-contiguous view
    b._strides[0] = 1
    b._strides[1] = 2

    assert_false(b.is_contiguous(), "Column-major tensor should not be contiguous")
    var c = as_contiguous(b)
    assert_true(c.is_contiguous(), "as_contiguous() result should be contiguous")

    assert_almost_equal(c._get_float64(0), 0.0, 1e-6, "c[0,0] should be 0.0")
    assert_almost_equal(c._get_float64(1), 2.0, 1e-6, "c[0,1] should be 2.0")
    assert_almost_equal(c._get_float64(2), 4.0, 1e-6, "c[0,2] should be 4.0")
    assert_almost_equal(c._get_float64(3), 1.0, 1e-6, "c[1,0] should be 1.0")
    assert_almost_equal(c._get_float64(4), 3.0, 1e-6, "c[1,1] should be 3.0")
    assert_almost_equal(c._get_float64(5), 5.0, 1e-6, "c[1,2] should be 5.0")
```

**Test file constraint**: ≤10 `fn test_` functions per `.mojo` file. Check with:

```bash
grep -c "^fn test_" tests/path/to/file.mojo
```

**Strides for common shapes**:

| Shape | Row-major (contiguous) | Column-major (non-contiguous) |
| ------- | ------------------------ | ------------------------------- |
| (2, 3) | `[3, 1]` | `[1, 2]` |
| (3, 4) | `[4, 1]` | `[1, 3]` |
| (2, 3, 4) | `[12, 4, 1]` | `[1, 2, 6]` |

## Verified On

| Project | Context | Details |
| --------- | --------- | --------- |
| ProjectOdyssey | Issue #4088, PR #4868 | [notes.md](../references/notes.md) |
| ProjectHephaestus | Issue #2228 (half-zombie; fix merged via PR #2208) | Only live acceptance criterion was a focused regression test; fails-before demonstrated via non-committed deletion of `T` from `_PORCELAIN_STATUS_PAIRS` at `hephaestus/automation/pr_manager.py:77`; positive `test_accepts_worktree_type_change_status` + negative `test_rejects_invalid_type_change_status(("TA","TR","TC"))` in `tests/unit/automation/test_pr_manager_commit.py` |

### Related skills

- `testing-non-vacuous-regression-guards-behavior-unchanged` — the mutation
  spot-check / positive+negative discipline for docstring-only fixes; this skill
  applies the same non-vacuity proof to an already-merged *behavioral* fix.
- `automation-moot-issue-regression-guard-pattern` — the *fully* moot case
  (every cited symbol/PR never existed → AST DRY count-guard); use that when the
  premise is false, this one when the fix is real and already shipped.
- `planning-check-already-shipped-before-planning` — detecting the already-merged
  condition before you plan any source change.
