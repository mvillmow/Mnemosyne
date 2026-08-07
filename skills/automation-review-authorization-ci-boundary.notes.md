# Scoped design note: explicit exact-head operator authorization

This note records the unverified architecture input that caused the v3.0.0 major revision of the
canonical `automation-review-authorization-ci-boundary` skill.

## Architecture reconciliation

ProjectHephaestus previously treated the exclusive `state:implementation-go` label plus a
process-local reviewed-head proof as sufficient loop-owned authorization. The current design adds a
separate human authorization requirement before the queue-owned merge mutation.

The issue's original native auto-merge path is obsolete. ADR-0014 replaced auto-merge arming with a
SHA-conditional REST merge, and the codebase retains an architecture regression proving
`PipelineGitHub` has no `arm_auto_merge`. The authorization gate therefore belongs immediately
around every live `merge_pr_if_head()` call. Native auto-merge remains prohibited.

## Proposed artifact

One native GitHub `APPROVED` review whose body is exactly:

```text
<!-- hephaestus-merge-authorization:v1 -->
```

GitHub repository/PR nesting binds scope, and `commit.oid` binds the review to the exact head. The
author must be a human with current write-or-higher permission and must differ from the authenticated
automation actor. Edited trusted markers, duplicate review IDs in one snapshot, pagination loops,
truncation, malformed data, snapshot drift, and unavailable permission/viewer facts fail closed.

## Restart and race semantics

Repeated observation of the same review for the same repository, PR, and head is durable recovery,
not replay. Restart preserves the GitHub review but loses the process-local automated review proof,
so fresh automated review is still required. A changed head makes the old authorization stale.

GitHub can condition the merge on the head SHA but not atomically on review identity or current review
state. An initial and final authorization read minimize the dismissal race. A dismissal observed on
the final read blocks; one racing after that read cannot cancel the in-flight request. The artifact is
therefore an issuance for an immutable head, not a continuously revocable lease.

## Rollout status

Unverified. Existing GO-labelled PRs without the marker should block without label, merge, or
auto-merge mutation. If validation is defective, stop queue-driven merging and use the repository's
normal branch-protected manual process; do not restore label-only automatic merging.

The prior notes about label-only authority are historical evidence and are preserved in the v2.8.0
snapshot in the skill history rather than remaining active guidance here.
