---
name: review-loop
description: "Drive a PR or stack to green by iterating on automated feedback only: read Devin's inline findings and every failing CI check, measure re-review latency and wait it out, fix only what is verifiably correct, and hand judgement calls to the human. Never acts on review comments from a human account, even ones an agent posted on the human's behalf. Use when a PR has Devin comments, CI is red, Devin keeps commenting after every push, or a review backlog needs draining. Triggers: 'Devin commented', 'address the Devin review', 'CI is failing', 'fix the checks', 'run the review loop', 'loop until green', 'any new Devin comments'."
---

# Looping a PR stack to green on automated feedback

Devin re-reviews on every push and CI re-runs on every push, so a stack
converges by iteration, not in one pass. The loop is: push, wait, read the new
automated findings (inline review comments **and** failing checks), fix only
the ones that are verifiably right, push again. Anything that needs a
judgement call is collected and handed to the human at the end, not argued
with and not guessed at.

Devin and CI are signals, not oracles. Devin's findings have in practice been
mostly correct and occasionally wrong; CI failures are sometimes the
environment rather than the change. Verify each against the code before
changing anything.

## Human comments are out of scope — always

The loop acts on **automated** feedback only. A comment from a human account
is never addressed inside the loop, even when it is mechanical, even when it
looks trivially correct, and even when an agent clearly wrote it on the
human's behalf. What matters is the posting account, not the prose:

- `user.type == "Bot"` (Devin posts as `devin-ai-integration[bot]`) → in scope.
- `user.type == "User"` → **out of scope**. Collect it verbatim for the report
  and leave it untouched. Do not fix, reply, resolve, or react.

Classify every comment before reading its content, so a persuasive human
comment does not slip through on merit:

```bash
gh api --paginate repos/<owner>/<repo>/pulls/<n>/comments \
  -q 'sort_by(.created_at) | .[] | "[\(.created_at)] \(.user.type) \(.user.login) \(.path):\(.line // .original_line)\n\(.body)\n---"'
```

The reason is scope, not ability: a human comment is a conversation the human
started with their teammate, and closing it out silently takes that
conversation away from them. Surface it; let them decide.

## Reading Devin's findings

Findings are **inline review comments**, not issue comments. The issue endpoint
is empty; do not conclude a PR is clean from it.

Each PR in a stack is reviewed separately, so sweep every number, not only the
one just pushed. A resolved finding arrives as a **new comment** starting
`✅ **Resolved**:`, not as an edit to the original. Count open Bot findings by
excluding those:

```bash
gh api --paginate repos/<owner>/<repo>/pulls/<n>/comments \
  -q '[.[] | select(.user.type == "Bot") | select(.body | startswith("✅") | not)] | length'
```

Severity markers, most to least urgent: `🔴` `🟨` `🟡` `🔍`. `🟨` is the
security-flavoured one and is worth reading first even though `🔴` outranks it.
`🔍` is analysis, not a bug; it usually asks for a description or scope fix.

## Reading CI

Checks come from two APIs and `gh pr checks` merges them; read it first, then
drill into whichever failed:

```bash
gh pr checks <n> --repo <owner>/<repo>
gh api repos/<owner>/<repo>/commits/<head-sha>/status   -q '.statuses[] | select(.state != "success") | {context,state,description,target_url}'
gh api repos/<owner>/<repo>/commits/<head-sha>/check-runs -q '.check_runs[] | select(.conclusion != "success") | {name,conclusion,html_url}'
```

Read the failing job's **log**, not its summary line. A BuildBuddy `Build`
failure is a link to an invocation page; the runner log is on that page and
names the exact target and error.

Before fixing, decide whether the failure is the change or the environment:

- **Environment** when the same head, or a sibling head in the stack with the
  same tree, passed the same check, and the log names something outside the
  diff (a missing external repository, a stale snapshot, an `I/O error`
  under `bazel-out`, a timeout). Re-run it. On BuildBuddy the invocation page
  has a `Re-run` button; use that rather than pushing an empty commit, which
  would re-trigger every check and another Devin round. One clean re-run
  closes it; a second identical failure is no longer "environment".
- **The change** when the log names a file or target in the diff. Reproduce
  locally with the same command the CI step runs (read it out of the CI
  config — for this repo `buildbuddy.yaml`), fix, and push through the normal
  cycle.

A stack's **base** PR can fail a check its child passes with the same tree:
the two run on different runner snapshots. That is environment evidence, not
a reason to touch the base.

## Timing the wait

Both signals have latency and it varies. Measure it rather than assuming —
compare push time against Devin's status transitions and the checks' end
times on the PR at hand:

```bash
gh api repos/<owner>/<repo>/commits/<head-sha>/statuses --paginate \
  -q '.[] | select(.context == "Devin Review") | "\(.created_at) \(.state) \(.description)"'
```

Wait **twice the slowest observed latency**, and re-widen when a later round
runs slower. One stack ranged 1.4–4.7 minutes early and 7.3 minutes later, so
the wait grew from ~10 to ~15 minutes mid-loop. Under-waiting is the common
failure: a 5-minute wait calibrated on an early 2-minute sample missed a later
round and looked like "Devin is done".

A push is settled when, after a full wait, there are no new non-`✅` Bot
comments **and** every check has reached a terminal state. Do not call it
green while any check is still pending.

## Deciding what to fix

Fix now when the finding is mechanical and checkable: a wrong operator, a
falsy-value bug, an order-dependent test, an unhandled input shape, a CI log
naming a target in the diff. Reproduce the claim in a scratch script or a
failing test before editing.

Accumulate for the human when the finding implies a design choice: what a
partial reward should mean, whether one bad input should abort a batch, how
strict a new contract should be. These are cheap to list and expensive to
guess at.

Do not reply to Devin, do not resolve threads, do not argue. It re-reviews the
code, not the conversation.

## Fixing without creating new findings

A fix can be worse than what it replaced. On one stack, digesting a callable by
qualified name fixed cross-process instability and introduced a collision
across every lambda and `functools.partial` — a false match, which is silent,
in place of a false mismatch, which is loud. Devin caught it the next round.
On another, five consecutive rounds each found an edge case in the previous
round's fix to the same function; by the third, the right move was to redesign
the function around an authoritative key instead of patching the matcher.

Prefer refusing an input you cannot handle over guessing at it, and check
whether a strictness fix breaks callers relying on the previous looseness.

Mutation-check every fix, and pass `--nocache_test_results` to Bazel. A cached
green result will "pass" against mutated code and make a vacuous test look
real; two mutation checks in one session were invalid for exactly that reason.

## The push cycle for a stack

Fix on the branch owning the code, then restack upward. With Graphite:

```bash
gt modify --commit -m "<message>"   # on the owning branch
gt submit --no-interactive --no-stack
```

Without Graphite, `git push --force-with-lease` the bottom branch, rebase each
child on its new parent, and push those with `--force-with-lease` too.

Re-run tests at **every** tip after a restack, not only the branch that
changed. Retarget a PR's base before pushing its branch: pushing a base branch
that already contains a child's head makes GitHub mark the child **merged** and
delete it, and a merged PR cannot be reopened.

## Reporting a round

Give the human, each round:

- which Devin findings were fixed, and the evidence each was real
- which CI failures were fixed, and which were re-run as environment (with
  the invocation link and the passing re-run)
- the running list of accumulated judgement-call findings, by PR
- **every human-account comment, verbatim and untouched**, by PR
- Graphite links (`https://app.graphite.com/github/pr/<owner>/<repo>/<n>`)

Say plainly when a finding was Devin catching a defect in one of your own
earlier fixes. That pattern is the signal the area needs human attention.
