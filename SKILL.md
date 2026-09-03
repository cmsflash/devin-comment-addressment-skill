---
name: devin-comment-addressment
description: "Work through Devin's automated review comments on a PR or stack until they run out: read the inline findings, measure Devin's re-review latency and wait it out, fix only what is verifiably correct, and accumulate judgement calls for the human instead of guessing or replying. Use when a PR has Devin comments, when Devin keeps commenting after every push, or when draining a review backlog. Triggers: 'Devin commented', 'check Devin's comments', 'address the Devin review', 'Devin raised', 'any new Devin comments', 'fix what Devin found'."
---

# Draining Devin's review comments on a PR stack

Devin re-reviews on every push, so a stack converges by iteration rather than
in one pass. The loop is: push, wait, read the new comments, fix only the ones
that are obviously right, push again. Comments needing a judgement call are
collected and handed to the human at the end, not argued with.

Devin is a reviewer, not an oracle. Its findings have in practice been mostly
correct and occasionally wrong, so verify each against the code before changing
anything.

## Reading the comments

Findings are **inline review comments**, not issue comments. The issue endpoint
is empty; do not conclude a PR is clean from it.

```bash
gh api repos/<owner>/<repo>/pulls/<n>/comments \
  -q 'sort_by(.created_at) | .[] | "[\(.created_at)] \(.path):\(.line // .original_line)\n\(.body)"'
```

Each PR in a stack is reviewed separately, so sweep every number, not only the
one just pushed.

A resolved finding arrives as a **new comment** starting `✅ **Resolved**:`,
not as an edit to the original. Count open findings by excluding those:

```bash
gh api repos/<owner>/<repo>/pulls/<n>/comments \
  -q '[.[] | select(.body | startswith("✅") | not)] | length'
```

Severity markers, most to least urgent: `🔴` `🟨` `🟡` `🔍`. `🟨` is the
security-flavoured one and is worth reading first even though `🔴` outranks it.

## Timing the wait

Devin's latency is a few minutes and varies. Measure it rather than assuming —
compare push times against review submission times on the PR at hand:

```bash
gh api repos/<owner>/<repo>/pulls/<n>/reviews -q '.[] | "\(.submitted_at)  \(.user.login)"'
```

Wait **twice the slowest observed latency**. One stack ranged 1.4–4.7 minutes,
so ~10 minutes. Under-waiting is the common failure: a 5-minute wait calibrated
on an early 2-minute sample missed a later round and looked like "Devin is
done".

Devin is done with a PR when a full wait after a push yields no new non-`✅`
comments.

## Deciding what to fix

Fix now when the finding is mechanical and checkable: a wrong operator, a
falsy-value bug, an order-dependent test, an unhandled input shape. Reproduce
the claim in a scratch script or a failing test before editing.

Accumulate for the human when the finding implies a design choice: what a
partial reward should mean, whether one bad input should abort a batch, how
strict a new contract should be. These are cheap to list and expensive to guess
at.

Do not reply to Devin, do not resolve threads, do not argue. It re-reviews the
code, not the conversation.

## Fixing without creating new findings

A fix can be worse than what it replaced. On one stack, digesting a callable by
qualified name fixed cross-process instability and introduced a collision
across every lambda and `functools.partial` — a false match, which is silent,
in place of a false mismatch, which is loud. Devin caught it the next round.

Prefer refusing an input you cannot handle over guessing at it, and check
whether a strictness fix breaks callers relying on the previous looseness.

Mutation-check every fix, and pass `--nocache_test_results`. A cached green
result will "pass" against mutated code and make a vacuous test look real; two
mutation checks in one session were invalid for exactly that reason.

## The push cycle for a stack

Fix on the branch owning the code, then restack upward:

```bash
git push --force-with-lease origin <bottom-branch>
cd <worktree-of-next> && git rebase <bottom-branch>
cd <worktree-of-top>  && git rebase <next-branch>
git push --force-with-lease origin <next-branch> <top-branch>
```

Re-run tests at **every** tip after a restack, not only the branch that
changed. Retarget a PR's base before pushing its branch: pushing a base branch
that already contains a child's head makes GitHub mark the child **merged** and
delete it, and a merged PR cannot be reopened.

## Reporting a round

Give the human, each round:

- which findings were fixed, and the evidence each was real
- the running list of accumulated judgement-call findings, by PR
- Graphite links (`https://app.graphite.com/github/pr/<owner>/<repo>/<n>`)

Say plainly when a finding was Devin catching a defect in one of your own
earlier fixes. That pattern is the signal the area needs human attention.
