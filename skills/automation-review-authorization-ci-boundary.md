---
name: automation-review-authorization-ci-boundary
description: "Separate automated implementation eligibility, operator exact-head merge authorization, repository readiness, and the irreversible merge mutation. Use when: (1) an automation label is being treated as complete merge authority, (2) a queue-owned merge needs one durable operator approval, (3) GitHub review pagination or provenance must fail closed, (4) restart and head-change behavior must preserve distinct proofs, or (5) native auto-merge has been replaced by a SHA-conditional merge."
category: architecture
date: 2026-08-08
version: "3.1.0"
user-invocable: false
verification: unverified
history: automation-review-authorization-ci-boundary.history
tags:
  - automation-loop
  - pr-review
  - merge-wait
  - implementation-go
  - operator-approval
  - github-review
  - exact-head-authorization
  - conditional-merge
  - capability
  - replay-detection
  - pagination
  - collaborator-permission
  - restart-safety
  - fail-closed
  - toctou
  - native-auto-merge
---

# Automation Review and Operator Merge Authorization

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-08-08 |
| **Objective** | Require one durable, explicit, operator-authored GitHub approval for the exact PR head before an automation queue performs its irreversible merge mutation. |
| **Outcome** | Proposed a marked native `APPROVED` review, deterministic resolver, strict nullable-BigInt normalization, immutable authorization capability, stable bounded GitHub reads that preserve duplicate IDs for replay classification, and initial/final identity checks around readiness. Automated eligibility, operator authorization, repository readiness, and merge execution remain separate authorities. |
| **Verification** | unverified — this is an architecture and implementation design. The resolver, adapters, stage behavior, restart cases, documentation, and CI have not been executed in this learning session. |
| **History** | [changelog](./automation-review-authorization-ci-boundary.history) |

The central correction is that a durable automation label and a process-local reviewed-head proof
are necessary but not sufficient for a queue-owned merge. A distinct trusted operator must
authorize the exact immutable head through a native GitHub review. CI/CD remains readiness
evidence and does not create either automated eligibility or operator authorization. GitHub's
`User` type and repository permission can enforce useful provenance constraints, but
"human-operated" remains a repository-administration assumption rather than a property the API
can prove.

## When to Use

- `state:implementation-go`, a verdict, or another machine-written artifact currently acts as the
  sole durable authority for an automated merge.
- A human must explicitly authorize the exact code revision before a bot calls GitHub's merge API.
- Native auto-merge arming was removed, but an issue or design still discusses gating that obsolete
  path instead of the successor conditional merge operation.
- A GitHub review must be rejected when it is stale, dismissed, edited, duplicated, bot-authored,
  automation-authored, or written by an actor without current repository write permission.
- A restart should reuse durable human authorization without reconstructing the process-local
  automated review proof.
- Review pagination, viewer identity, collaborator permission, or snapshot drift can become
  unavailable and must block the merge before any mutation.
- A final read can minimize but cannot atomically eliminate the race between review dismissal and a
  REST merge request.

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat it as a hypothesis until the
> resolver tests, GitHub adapter tests, pipeline tests, static architecture guards, full automation
> suite, and CI confirm it.

### Quick Reference

```text
automated source review ────────► state:implementation-go eligibility
current process ────────────────► reviewed-head proof
trusted operator GitHub review ─► durable authorization for that exact head
required checks/protection ─────► repository readiness
                                  │
                                  ▼
                       SHA-conditional squash merge
```

```text
<!-- hephaestus-merge-authorization:v1 -->
```

The review must be native GitHub state `APPROVED`; its body must equal the marker exactly; its
`commit.oid` must equal the reviewed head; and it must be unedited. Its actor must have
`author.__typename == "User"`, `viewerDidAuthor == false`, a login distinct case-insensitively
from the authenticated automation actor, no `[bot]` suffix, and current legacy repository
permission `WRITE` or `ADMIN`. The legacy permission endpoint maps role `maintain` to
`write`, so `MAINTAIN` is not a valid response value.

```bash
gh pr view <N> --json state,headRefOid,baseRefName,autoMergeRequest
gh pr review <N> \
  --approve \
  --body '<!-- hephaestus-merge-authorization:v1 -->'
<automation-loop-command> --prs <N> --loops 1 --max-workers 1
```

### Detailed Steps

1. **Gate the operation that actually exists.** Trace the current irreversible production call
   rather than relying on stale issue wording. If native auto-merge arming was replaced by a
   SHA-conditional REST merge, require authorization at every conditional-merge call site and keep
   the architectural test that prevents native auto-merge mutation from returning.

2. **Name four separate authorities.** Automated review writes only implementation eligibility.
   The current process retains an exact reviewed-head proof. A human review supplies durable
   operator authorization. Required checks and branch protection supply repository readiness.
   None substitutes for another, and only the merge adapter performs the mutation.

3. **Use a native review as the creation record.** Select an exact marker body and require GitHub
   review state `APPROVED`. Bind repository and PR through repository-scoped traversal and PR
   nesting; bind code through `commit.oid`. Retain the required GraphQL node ID, normalized nullable
   `fullDatabaseId`, author login and current permission, exact head, submitted/updated timestamps,
   edit fields, and SHA-256 body digest in an immutable capability. Reject
   `includesCreatedEdit=true` or non-null `lastEditedAt` before capability construction.

4. **Resolve candidates with closed precedence.** First establish marker presence, actor provenance,
   and commit binding. Data that prevents any of those facts from being established is unavailable
   service evidence and raises; it is not a replay decision. Once a string-body marked review is
   proven to be from a trusted actor on the current head, invalid or missing candidate metadata is
   `REPLAYED`—including node ID, state, submitted/updated timestamp, `fullDatabaseId`, edit
   indicator, or noncanonical marker body. `REPLAYED` takes precedence even when another valid
   approval exists. Otherwise, more than one active trusted current-head approval is `AMBIGUOUS`;
   exactly one is `AUTHORIZED`, regardless of inert stale, dismissed, or untrusted candidates.
   With no active trusted current approval, prefer `REVOKED` for current dismissed markers, then
   `STALE` for trusted old-head markers, then `UNTRUSTED` for current untrusted markers, then
   `ABSENT` when no marked review exists. Duplicate review node IDs in one snapshot are
   `REPLAYED` input.

5. **Keep trust from becoming a veto.** Reject non-`User` actors, `viewerDidAuthor=true`, the
   authenticated automation actor by case-insensitive login, `[bot]` logins, and actors without
   current `WRITE` or `ADMIN` permission. An untrusted marker reports `UNTRUSTED` only when no
   trusted authorization exists; an attacker cannot invalidate a valid operator approval merely by
   adding another marked review. Document that these checks cannot prove a `User` account is not
   machine-operated; administrators must restrict trusted access and keep the automation identity
   distinct.

6. **Treat reread identity differently from replay.** Re-observing the same review ID for the same
   repository, PR, and head after restart or before a bounded retry is durable recovery. The same
   review on an old head is stale. A duplicated ID within one snapshot or edit metadata on a trusted
   artifact is replay/tampering.

7. **Read one complete, stable GitHub snapshot.** Traverse at most 100 pages and 10,000 reviews with
   unique nonempty cursors, validated connection/page envelopes, repository/PR/head identity,
   `totalCount` agreement, and explicit truncation rejection. Preserve duplicate review IDs and
   stable malformed candidate fields so the resolver—not the transport adapter—can classify replay.
   Select `id`, `fullDatabaseId`, `body`, `state`, `submittedAt`, `updatedAt`,
   `includesCreatedEdit`, `lastEditedAt`, `viewerDidAuthor`, author login/type, and commit OID.
   Normalize only positive integers and canonical positive decimal strings in `fullDatabaseId` to
   `int`; preserve `null` as `None`; booleans, floats, zero, negatives, and noncanonical strings
   remain malformed candidate data. Read the complete normalized snapshot twice and reject drift.
   Fetch current collaborator permission through the repository permission endpoint, mapping only a
   real 404 to `NONE`; transport, JSON, GraphQL, viewer-login, malformed response, or other status
   failures remain unavailable evidence.

8. **Resolve before readiness and immediately before mutation.** Catch every adapter and resolver
   exception inside the closed merge job and classify it as
   `merge_authorization_unavailable`. Require the second immutable authorization value to equal the
   first completely, including normalized database ID, actor permission, timestamps, edit fields,
   and body digest. A non-authorized result exits with its explicit status; identity drift exits as
   `merge_authorization_changed`; neither reaches the merge call.

9. **Make authorization a required capability.** Change the irreversible adapter signature to
   require the immutable authorization object. Validate its runtime type plus repository, PR number,
   and head against the requested merge before logging, dry-run handling, or transport. A missing,
   wrong-type, or mismatched capability returns a malformed result without any observable mutation
   attempt. Keep a static test proving the job runner is the only production caller. A process-local
   HMAC adds no provenance because the same trusted process can construct values; the externally
   sourced, fully bound frozen capability is the meaningful boundary.

10. **Classify retry behavior explicitly.** `ABSENT`, `STALE`, `AMBIGUOUS`, `REPLAYED`, `REVOKED`,
    and `UNTRUSTED` are operator-correctable blocked outcomes and must preserve labels while making
    no merge or auto-merge mutation. Unavailable evidence or changed identity is a terminally
    classified failure for that cycle, not an unclassified worker exception.

11. **Test restart and head movement as different dimensions.** A GitHub authorization review
    survives process restart, but the process-local reviewed-head proof does not. The restarted
    coordinator must perform a fresh automated review before it can reuse the same exact-head human
    authorization. If the head changes, the old human review becomes stale and no merge occurs until
    a new exact-head authorization is submitted.

12. **Document the residual TOCTOU boundary.** GitHub's merge endpoint accepts a head-SHA
    precondition but no review-ID or review-state precondition. The final read minimizes the window,
    and the SHA condition prevents authorization from moving to different code. A dismissal visible
    before the final read blocks. A dismissal racing after it cannot atomically cancel an in-flight
    request; define the review as issuance for an immutable head, not a continuously revocable lease.

13. **Roll out and roll back fail closed.** Existing GO-labelled PRs without a marked approval stop
    as `merge_authorization_absent`, keep their labels, and make no merge or auto-merge mutation. If
    the resolver is defective, stop queue-driven merging and use the normal branch-protected manual
    merge process. Never restore label-only automatic merging as rollback.

14. **Preserve decision history.** Accepted ADRs remain immutable. Add a superseding ADR that
    records the artifact, resolver precedence, exact-head and restart semantics, error outcomes,
    TOCTOU residual risk, migration, and manual rollback; update only the ADR index and active docs.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Treat the implementation-GO label as complete merge authority | Automated review wrote one durable state label and merge-wait consumed it after a process-local head check | A machine can prove its own eligibility decision but cannot provide the required explicit human authorization for the irreversible mutation | Keep automated eligibility and operator authorization as independent mandatory artifacts |
| Gate native auto-merge because the issue names it | The design targeted an `arm_auto_merge` path that had already been removed | The safeguard would protect dead code while the successor `merge_pr_if_head()` mutation remained callable | Reconcile the issue with current architecture and gate every live mutation call site |
| Encode a user-supplied nonce or duplicate repository/PR/SHA in the body | A custom token attempted to establish creation identity | The nonce has no trusted creation record and duplicates identity GitHub already supplies through nesting and `commit.oid` | Use the native review record and one exact canonical marker |
| Accept a canonical body after editing | An edited review currently matched the expected marker | Current text does not prove original creation content; editing destroys the intended creation-record semantics | Reject edit metadata even when the present body is canonical |
| Let any marked review veto a valid operator | An automation actor, bot, or low-permission user added a second marker | Untrusted input could force ambiguity or denial after a trusted operator authorized the head | Ignore untrusted candidates when a trusted authorization exists |
| Fetch only one page or one snapshot | The resolver classified the first 100 reviews from one read | Later pages can hide candidates or duplicate IDs, and a changing snapshot can mix incompatible states | Bound pagination, reject cursor loops and truncation, preserve duplicate IDs for replay classification, and compare two complete reads |
| Reject duplicate IDs during review traversal | The transport adapter enforced ID uniqueness before returning a snapshot | A duplicate is authorization evidence with a deterministic `REPLAYED` meaning, not malformed connection transport | Preserve duplicate nodes in each stable snapshot and let the pure resolver classify them |
| Accept `MAINTAIN` as a collaborator-permission response | The trust predicate modeled repository roles rather than the endpoint's legacy `permission` field | GitHub maps maintain to `write`; accepting a separate value widens the parser beyond its documented wire contract | Accept only `NONE`, `READ`, `WRITE`, and `ADMIN`; trust only `WRITE` and `ADMIN` |
| Coerce every integer-like database ID | Boolean, float, zero, negative, or padded/signed decimal values were accepted as review identities | Python coercion erases malformed service data and can make unstable identities appear canonical | Normalize only positive integers and canonical positive decimal strings; retain `null` and classify all other candidate forms as replayed after provenance is established |
| Collapse malformed candidates into unavailable transport | Any bad marked-review field raised before candidate resolution | A trusted current-head artifact with malformed authorization metadata is materially different from a broken repository/connection snapshot and must veto a valid sibling approval | Raise only when marker/provenance/head facts cannot be established; otherwise resolve the malformed trusted current-head candidate as `REPLAYED` |
| Sign the capability inside the automation process | An HMAC was proposed to distinguish verified capability values from raw Python objects | The trusted process can construct both the value and signature, so the signature proves no external provenance | Use a frozen value bound to the externally sourced review and validate it at the sole irreversible adapter |
| Treat the approval as a continuously revocable lease | The design implied dismissal could cancel a merge atomically at any instant | GitHub cannot condition a merge request on review ID/state, so dismissal can race after the final read | Document issuance-for-head semantics and the unavoidable post-read race |
| Let adapter failures escape the worker | Pagination, viewer, permission, JSON, or GraphQL errors raised out of merge-wait | The queue recorded an unclassified worker failure and obscured whether a mutation was attempted | Catch and classify all authorization-read failures before the conditional PUT |
| Pass raw review data to the merge adapter | Callers could invoke the irreversible method without a verified, head-bound proof | Validation was split across call sites and tests could not prove every mutation was guarded | Require and validate one immutable authorization capability at the adapter boundary |

## Results & Parameters

| Parameter | Required value or behavior |
|-----------|----------------------------|
| Marker | `<!-- hephaestus-merge-authorization:v1 -->` exactly |
| Body identity | SHA-256 digest of the exact canonical marker body |
| Native state | `APPROVED` for active authorization; dismissed current-head markers classify as revoked when no active trusted approval exists |
| Head binding | Review `commit.oid` equals the live reviewed head SHA |
| Trusted actor | GitHub `User`, `viewerDidAuthor=false`, distinct case-insensitively from the automation viewer, login without `[bot]`, and current legacy `WRITE` or `ADMIN` permission |
| Human-operated assumption | API provenance cannot prove a `User` account is not machine-operated; repository administrators own trusted access and identity separation |
| Review database ID | `fullDatabaseId`: positive `int` or canonical positive decimal string normalizes to `int`; `null` remains `None`; all other forms are malformed |
| Edit policy | `includesCreatedEdit` must be false and `lastEditedAt` must be null |
| Authorized cardinality | Exactly one active trusted current-head marked approval |
| Stable read | Two complete normalized bounded paginated snapshots compare equal; duplicate IDs remain present |
| Capability identity | Repository, PR, head, node/normalized database IDs, author and permission, submitted/updated timestamps, edit fields, and body digest |
| Unavailable boundary | Transport/JSON/GraphQL, repository/PR/connection/page identity, cursor/truncation/count/drift, viewer/permission, or provenance-establishment failure raises |
| Replayed boundary | Once marker, trusted actor, and current head are established, malformed authorization metadata—including duplicate IDs—resolves as `REPLAYED` |
| Admission reads | Once before readiness wait and once immediately before the merge request |
| Merge precondition | Conditional request includes the verified head SHA and receives the matching capability |
| Blocked statuses | `absent`, `stale`, `ambiguous`, `replayed`, `revoked`, `untrusted` |
| Failed-cycle statuses | `unavailable`, `changed` |
| Restart rule | Durable review may be reused for the same PR/head only after fresh process-local automated review proof |
| Head-change rule | Old review becomes stale; a new exact-head human approval is required |
| Native auto-merge | Queue never enables, disables, adopts, or polls it |

### Acceptance Tests

```bash
uv run pytest \
  tests/unit/automation/test_merge_authorization.py \
  tests/unit/automation/test_pipeline_github.py \
  tests/unit/automation/pipeline/stages/test_stage_merge_wait.py \
  tests/unit/automation/pipeline/test_timer_heap.py \
  tests/unit/automation/pipeline/test_coordinator.py -v

uv run pytest \
  tests/unit/automation/pipeline/test_pipeline_architecture.py::test_pipeline_stages_have_no_auto_merge_mutation_capability \
  -v

uv run ruff check hephaestus/automation tests/unit/automation
uv run ruff format --check hephaestus/automation tests/unit/automation
uv run mypy hephaestus/automation tests/unit/automation
```

Required behavioral assertions include:

- no conditional PUT for absent, stale, ambiguous, replayed, revoked, untrusted, unavailable, or
  changed authorization;
- a mismatched capability produces no GitHub call;
- duplicate IDs reach the resolver and classify as replayed, while cursor loops, truncation,
  malformed envelopes, and snapshot drift classify as unavailable;
- malformed trusted current-head metadata classifies as replayed even beside a valid approval;
- `fullDatabaseId` normalization accepts only positive integers, canonical positive decimal
  strings, and null;
- same-review same-head restart reuse succeeds only after fresh automated review proof;
- the same review becomes stale after a head change;
- production has one capability-bearing conditional-merge call site and no native auto-merge
  mutation capability.

## Verified Workflow

_Not applicable yet._ The actionable methodology is under **Proposed Workflow**. This placeholder
exists for corpus validation and makes no verification claim. Promote the workflow only after the
implementation and CI pass.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Exact-head operator authorization design | Unverified architecture proposal for gating the queue-owned SHA-conditional merge while retaining the native auto-merge prohibition. |
