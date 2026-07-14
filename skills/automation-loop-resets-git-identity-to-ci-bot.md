---
name: automation-loop-resets-git-identity-to-ci-bot
description: "The HomericIntelligence automation loop silently overwrites the repo-local git config user.name/user.email to a CI bot (e.g. Odysseus CI Bot <ci@HomericIntelligence.local>); a later manual `git commit -S -s` then records the BOT as author, committer, AND Signed-off-by trailer, failing the pr-policy DCO/authorship gate even though the GPG signature can still be the human's key. Use when: (1) after running hephaestus-automation-loop or its pipeline you make a manual commit and pr-policy rejects it for a DCO/authorship mismatch, (2) a commit LOOKS signed (git log --show-signature is green) but author/committer/Signed-off-by show a bot identity, (3) you need to re-set your identity or amend a bot-authored commit back to the human, (4) any automation process may have mutated repo-local git config before you committed."
category: tooling
date: 2026-07-12
version: "1.0.0"
user-invocable: false
verification: verified-local
tags: [git-identity, automation-loop, dco, ci-bot, committer, pr-policy]
---

# Automation Loop Resets Repo-Local Git Identity to a CI Bot

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-07-12 |
| **Objective** | Prevent manual commits made after an automation-loop run from being silently attributed to a CI bot, which fails the `pr-policy` DCO/authorship CI gate and misattributes the human's work |
| **Outcome** | Successful — identified that `hephaestus-automation-loop` overwrites repo-local git identity; re-setting identity before committing (and amending an already-bot-authored commit) fixes author + committer + DCO trailer |
| **Verification** | verified-local — directly observed the bot identity on a commit this session and fixed it; not yet confirmed via a CI run |

## When to Use

- After running `hephaestus-automation-loop` (or its pipeline) you make a manual commit and `pr-policy` rejects it for a DCO / authorship mismatch.
- A commit LOOKS signed — `git log --show-signature` is green — but `author`, `committer`, and the `Signed-off-by` trailer all show a bot identity instead of yours.
- You need to re-set your git identity, or amend a commit already made with the bot identity back to the human identity.
- Any automation/orchestration process may have mutated repo-local `git config user.name/user.email` before you committed.

## Verified Workflow

The key fact: running the automation loop sets the **repo-local**
`git config user.name` / `user.email` to a bot identity (observed:
`Odysseus CI Bot <ci@HomericIntelligence.local>`). This persists after the loop
exits. A later manual `git commit -S -s` then records the BOT as author,
committer, AND the `Signed-off-by:` trailer. The `pr-policy` gate checks BOTH a
valid GPG signature AND a DCO `Signed-off-by` trailer matching the author — a
bot-authored/bot-signed-off commit fails it. This is easy to miss because the
GPG signature can still be the human's key (key-based), so the commit LOOKS
correctly signed; only the author/committer/DCO identity is wrong.

### Quick Reference

```bash
# 1. After ANY automation-loop run, re-set the repo-local identity BEFORE committing:
git config user.name "Micah Villmow"
git config user.email "4211002+mvillmow@users.noreply.github.com"

# 2. To FIX a commit already made with the bot identity — reset author+committer:
git commit --amend --reset-author -S -s
#    (--reset-author fixes author & committer; -s re-adds YOUR Signed-off-by trailer.
#     If a stale bot Signed-off-by remains, strip it and rewrite the message:)
git log -1 --format=%B | grep -v "Signed-off-by: Odysseus CI Bot" \
  | git commit --amend -S -s -F -

# 3. Verify author, committer, AND sign-off are all the human identity — in one shot:
git log -1 --format="%an <%ae> | %cn <%ce> | %(trailers:key=Signed-off-by,valueonly)"
```

### Detailed Steps

1. **Assume the identity is wrong after any automation run.** Before any manual
   commit that follows a `hephaestus-automation-loop` / pipeline invocation,
   re-set the repo-local identity (step 1 above). Do not trust that it is still
   yours — the loop overwrites it silently.
2. **If you already committed, do not just re-sign.** The GPG signature being
   valid is a red herring; `--amend --reset-author` is required to fix the
   author and committer, and `-s` to re-add the correct DCO trailer.
3. **Strip any bot `Signed-off-by` trailer.** `--amend -s` appends YOUR trailer
   but does not remove a bot one already in the message body — filter it out
   (step 2 above) so the trailer matches the (now-corrected) author.
4. **Verify all three at once** with the `%an / %cn / trailers` format (step 3).
   Author, committer, and Signed-off-by must ALL be the human identity before
   pushing.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| `git commit -S -s` immediately after an automation-loop run | Assumed the repo-local identity was still the human's | The loop had silently overwritten `git config user.name/user.email` to `Odysseus CI Bot <ci@HomericIntelligence.local>`, so author + committer + DCO trailer were all the bot; pr-policy's DCO/authorship check rejected it | Automation processes can mutate repo-local git config; ALWAYS re-set and verify identity before committing after they run |
| Re-ran `git commit --amend -S` to "re-sign" the rejected commit | Thought a fresh GPG signature would satisfy the gate | The GPG signature was already valid (human's key) — the failure was the bot author/committer/trailer, which a plain `--amend -S` does not touch | The signature can be correct while the identity is wrong; use `--amend --reset-author` to fix author+committer, not just re-sign |

## Results & Parameters

- **Repo**: `HomericIntelligence/ProjectHephaestus`
- **Offending identity observed**: `Odysseus CI Bot <ci@HomericIntelligence.local>`
- **Correct human identity** (copy-paste):

  ```bash
  git config user.name "Micah Villmow"
  git config user.email "4211002+mvillmow@users.noreply.github.com"
  ```

- **Fix-already-committed recipe** (copy-paste):

  ```bash
  git commit --amend --reset-author -S -s
  git log -1 --format=%B | grep -v "Signed-off-by: Odysseus CI Bot" \
    | git commit --amend -S -s -F -
  ```

- **Verify-all-three recipe** (copy-paste):

  ```bash
  git log -1 --format="%an <%ae> | %cn <%ce> | %(trailers:key=Signed-off-by,valueonly)"
  ```

  Author, committer, and Signed-off-by must all be `Micah Villmow <4211002+mvillmow@users.noreply.github.com>`.

- **Why this differs from `git-dco-signoff-distinct-from-gpg-sign`**: that skill
  covers a *missing* `-s` trailer (identity correct, trailer absent). THIS skill
  covers a *wrong identity* — the trailer and signature can both be present, but
  point at a bot because the loop overwrote `git config`.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Manual commit after a `hephaestus-automation-loop` run | Commit was authored/committed/signed-off as `Odysseus CI Bot <ci@HomericIntelligence.local>` despite a valid human GPG signature; fixed by re-setting `git config user.name/user.email` and `git commit --amend --reset-author -S -s` (verified-local) |
