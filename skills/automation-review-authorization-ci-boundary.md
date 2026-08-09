---
name: automation-review-authorization-ci-boundary
description: "Separate automated implementation eligibility, operator exact-head merge authorization, repository readiness, and the irreversible merge mutation. Use when: (1) an automation label is being treated as complete merge authority, (2) a queue-owned merge needs one durable operator approval, (3) GitHub review pagination or provenance must fail closed, (4) restart and head-change behavior must preserve distinct proofs, (5) native auto-merge has been replaced by a SHA-conditional merge, (6) a later same-head review can revoke a prior GO, or (7) a final merge audit must distinguish the latest exact-head review from earlier records."
category: architecture
date: 2026-08-09
version: "3.7.0"
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
  - review-convergence
  - api-compatibility
  - runtime-boundary
---

# Automation Review and Operator Merge Authorization

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-08-08 |
| **Objective** | Require one durable, explicit, operator-authored GitHub approval for the exact PR head before an automation queue performs its irreversible merge mutation. |
| **Outcome** | Proposed a marked native `APPROVED` review, deterministic resolver, strict nullable-BigInt normalization, immutable authorization capability, stable bounded GitHub reads that preserve duplicate IDs for replay classification, and initial/final identity checks around readiness. Automated eligibility, operator authorization, repository readiness, and merge execution remain separate authorities. Added verified direct-implementation audits from ProjectHephaestus issues #2374 / PR #2717 and #2392 / PR #2672; PR #2672 demonstrates that a later same-head review can revoke an earlier GO before the final exact-head review and conditional merge. PR #2686 / issue #2404 adds the host-verification recovery case: intermediate host failures kept the loop at NOGO, while the final exact-head review and clean live checks preceded GO and a SHA-conditional merge with native auto-merge unset. PR #2644 / issue #2367 and PR #2646 / issue #2369 add the inline/local-review audits: the loop emitted no GitHub review or issue-comment object at all, and the review-to-merge path was still audited from the exact head, loop-owned GO label, live required checks, and terminal conditional squash merge. PR #2661 / issue #2380 adds the generated-prose audit: the generated body reported testing was not run, but live required checks on the exact head still established merge readiness, and the loop-owned GO label and exact-head conditional merge remained authoritative. |
| **Verification** | unverified for the proposed operator-authorization design; the appended PR #2717 direct-implementation audit is verified-ci. |
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
- A PR can briefly reach GO and then receive a later same-head blocking finding; the latest
  exclusive label transition and final exact-head review must control merge admission.
- A review is exercising malformed-input runtime boundaries or changing a helper API behind a CLI;
  compiler exceptions, unresolved references, and return-type compatibility are review findings,
  not optional polish.
- A completed ProjectHephaestus direct-implementation run may emit only non-authorizing `COMMENTED`
  review records and no issue/comment object, so its review-to-merge path must be audited from the
  exact head, loop-owned state label, live required checks, and terminal conditional merge event.
- An inline/local review may emit zero GitHub review, review-comment, and issue-comment objects; the
  review-to-merge path is still audited from the exact head, loop-owned GO label, live required checks,
  and terminal conditional squash-merge event.
- A generated PR body reports testing was not run while the live exact-head required-check rollup passes;
  the prose is pipeline context, not merge authority, and must be reconciled with the live check rollup.
- Host-side review verification fails on an intermediate head or for an environmental reason, and
  the loop must preserve NOGO until a fresh final-head review succeeds instead of treating the
  failed verifier as source-review authorization.

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
| Reuse an earlier non-authorizing review after remediation | PR #2717 first had an empty-body `COMMENTED` review on head `cd0de8a5`, then a corrected final head `f36d6c57` | The earlier review was historical evidence for a superseded head; it could not support GO for the final revision | Re-review the final exact head, then audit the exclusive GO/NOGO label transition, required-check completion, and conditional merge event |
| Pass raw review data to the merge adapter | Callers could invoke the irreversible method without a verified, head-bound proof | Validation was split across call sites and tests could not prove every mutation was guarded | Require and validate one immutable authorization capability at the adapter boundary |
| Treat the first clean re-review as final authorization | PR #2672 briefly reached GO after a remediation pass, but a later review on the same head found an API-compatibility break and replaced GO with NOGO. | Review convergence is not monotonic; a later exact-head finding can revoke an earlier label before the implementation changes again. | Treat the latest exclusive GO/NOGO state and its matching exact-head review as authoritative; merge only after the final replacement transition. |
| Validate only nominal malformed-input behavior | PR #2672's initial implementation caught ordinary `re.error` but still let regex `OverflowError` escape, and `check_schema()` did not catch unresolved `$ref` failures during iteration. | Runtime boundary behavior differed from the nominal validation path, so malformed input could still produce a traceback. | Exercise exception paths and reference resolution through the real CLI boundary in every output mode before allowing GO. |
| Change a helper return type to simplify CLI diagnostics | PR #2672 temporarily changed `check_files()` from its public `SchemaCheckResult` contract to a tuple. | Existing callers using attributes such as `exit_code`, counts, and `error_count` would break even though the issue targeted CLI diagnostics. | Preserve established helper return contracts; add diagnostics compatibly or introduce a separate API, then re-review the corrected head. |
| Infer review authorization from GitHub review objects or generated PR prose | ProjectHephaestus PR #2654 had only non-authorizing `COMMENTED` review records and no issue/comment object, while its generated body said tests were not run, although the loop completed its direct-implementation review path | Those surfaces were incomplete or informational and could misclassify a successfully reviewed and merged PR | Audit the exact final head, loop-owned GO label transition, live required-check completion, and terminal merge event separately |
| Treat a host-verification failure as a source-review verdict | ProjectHephaestus PR #2686 / issue #2404 encountered host quota, tool-availability, typing, and disk-capacity failures on intermediate review heads | A host verifier can fail before source review and cannot authorize the code; advancing from that evidence would conflate runner health with review state | Keep the loop NOGO, publish the failure classification, and require a fresh exact-head review before writing GO |
| Treat generated PR-body testing text as the merge gate | PR #2661's generated body said `Testing: Not run by the automation pipeline` while the live required-check rollup for head `21939a6f` passed | The body described pipeline execution context; it was not a source-review verdict or current repository merge-readiness fact | Treat generated prose as context only; read the exact head, loop-owned label, and live required checks before merge-wait |

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
| Direct implementation audit | For issue #2373 / PR #2654, final head `6689d076d04b824a7915b239083fdbf0f3f9f6a5` received only non-authorizing `COMMENTED` review records and no issue/comment object usable as authorization; `state:implementation-go` was recorded at `09:47:30Z`, NOGO was removed at `09:47:32Z`, `required-checks-gate` completed at `10:00:17Z`, and `merge_wait` conditionally merged commit `3ff588ca6419adf2da4618186e82d380ef2cc1ac` at `10:03:47Z` with native auto-merge null. This is an audit of the current queue path, not validation of the proposed operator-authorization design above. |
| Direct implementation NOGO-to-GO audit | For issue #2374 / PR #2717, the first exact-head `COMMENTED` review targeted `cd0de8a52a8a5785022342e0e343606c36d0e05f` and NOGO was recorded at `10:48:03Z`; a fresh review matched final head `f36d6c5752d100e00bb5db6e26483e5a6348c77f`, GO was recorded at `11:16:04Z`, NOGO was removed at `11:16:07Z`, `required-checks-gate` completed at `11:23:50Z`, and `merge_wait` conditionally merged `57d2619942670851dd95e1c6c7e615f4d609f62b` at `11:24:13Z` with `autoMergeRequest` null. This is an audit of the current queue path, not validation of the proposed operator-authorization design above. |
| Direct implementation multi-cycle audit | For issue #2392 / PR #2672, the first exact-head review found uncaught regex `OverflowError`; a later review found unresolved JSON-Schema references and then a `check_files()` return-contract break. The loop recorded GO at `11:15:55Z`, later replaced it with NOGO at `19:28:42Z`, then matched final head `5c1311f904df952b6f60730deee17d56f3f5a957` with a final `COMMENTED` review at `19:50:21Z`; GO was recorded at `19:52:10Z`, `required-checks-gate` completed at `20:01:03Z`, and `merge_wait` conditionally merged `95f9d24cb7cd7ef2a8ca52682a48ffa3a1c03b9b` at `20:01:16Z` with native auto-merge unset. This is an audit of the current queue path, not validation of the proposed operator-authorization design above. |
| Direct implementation host-verification recovery audit | For issue #2404 / PR #2686, intermediate host-verification failures kept the run at NOGO. The final review matched head `e1b43fe7c3606e69ad442cd47f3b48f325106c75` at `02:30:34Z`; GO was recorded at `02:38:16Z`, NOGO was removed at `02:38:18Z`, `required-checks-gate` completed at `02:40:40Z`, and `merge_wait` conditionally merged `957c906e2b038da7147630cf67c241351ae36243` at `02:41:23Z` with native auto-merge unset. This is an audit of the current queue path, not validation of the proposed operator-authorization design above. |
| Direct implementation inline/local review audit | For issue #2367 / PR #2644, no GitHub review or issue-comment object was emitted; final head `c79325cf` received `state:implementation-go` at `23:40:47Z`, `unit-tests` completed at `23:43:27Z`, `required-checks-gate` at `23:43:33Z`, and `merge_wait` conditionally merged `248e6efb` at `23:44:34Z` with native auto-merge null. This is an audit of the current queue path, not validation of the proposed operator-authorization design above. |
| Direct implementation inline/local review repeat audit | For issue #2369 / PR #2646, zero GitHub review, review-comment, and issue-comment objects were emitted; final head `310290abc2603111db9a546507ccd0c6f6f9e95f` received `state:implementation-go` at `23:53:34Z`, `unit-tests` completed at `00:00:44Z`, `required-checks-gate` at `00:00:49Z`, and `merge_wait` conditionally merged `5ed02735` at `00:01:33Z` with native auto-merge null. This is an audit of the current queue path, not validation of the proposed operator-authorization design above. |
| Direct implementation generated-prose audit | For issue #2380 / PR #2661, the generated body said testing was not run by the automation pipeline, but live `unit-tests` and `required-checks-gate` passed on final head `21939a6f`. The loop reviewed the exact head, applied `state:implementation-go`, and `merge_wait` conditionally squash-merged `f61ca9ed` with `autoMergeRequest` null and no GitHub approval review required. This is an audit of the current queue path, not validation of the proposed operator-authorization design above. |

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
| ProjectHephaestus | Issue #2373 / PR #2654 | verified-ci direct-implementation audit. The final head was `6689d076`; GitHub exposed only non-authorizing `COMMENTED` review records and no issue/comment object, the loop-owned GO label preceded the later required-check gate, and `merge_wait` conditionally merged `3ff588ca` with native auto-merge unset. The audit relied on head, labels, checks, and the merge event rather than PR prose. |
| ProjectHephaestus | Issue #2374 / PR #2717 | verified-ci direct-implementation NOGO-to-GO audit. The first empty-body `COMMENTED` review was bound to superseded head `cd0de8a5`; only the fresh review of final head `f36d6c57` supported GO. The required-checks gate completed before `merge_wait` conditionally merged `57d26199`; native auto-merge remained unset. |
| ProjectHephaestus | Issue #2392 / PR #2672 | verified-ci multi-cycle review audit. The loop held the PR at NOGO across runtime-boundary and API-compatibility findings, briefly reached GO, correctly reverted to NOGO after a later same-head review, then authorized the final head `5c1311f9`; the required-checks gate completed before conditional merge `95f9d24c`, with native auto-merge unset. |
| ProjectHephaestus | Issue #2404 / PR #2686 | verified-ci host-verification recovery audit. Intermediate host failures did not authorize advancement; the final-head review, exclusive GO transition, required-checks completion, and conditional merge were separately observed, with native auto-merge unset. |
| ProjectHephaestus | Issue #2367 / PR #2644 | verified-ci inline/local review and merge audit. No GitHub review or issue-comment object was emitted; the loop authorized final head `c79325cf` with `state:implementation-go`, required checks completed, and merge-wait conditionally squash-merged `248e6efb` with native auto-merge null. |
| ProjectHephaestus | Issue #2369 / PR #2646 | verified-ci repeat of the inline/local review path. Zero GitHub review, review-comment, and issue-comment objects were emitted; final head `310290ab` reached `state:implementation-go`, required checks completed, and merge-wait conditionally squash-merged `5ed02735` with native auto-merge null. |
| ProjectHephaestus | Issue #2380 / PR #2661 | verified-ci generated-prose audit. The generated body reported testing was not run, but live required checks on final head `21939a6f` established merge readiness; the loop-owned GO label, required-check completion, and conditional squash-merge `f61ca9ed` followed with native auto-merge null. |
