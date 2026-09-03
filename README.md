# devin-comment-addressment

Devin re-reviews a PR on every push, so a stack converges by iteration: push,
wait, read the new comments, fix what is verifiably right, push again. This
skill captures that loop — where the comments actually live in the API, how
long to wait before believing there are none, and which findings to fix now
versus save for a human.

The parts that are easy to get wrong:

- Findings are **inline** review comments (`pulls/N/comments`). The issue
  endpoint is empty, so a PR can look clean when it is not.
- A resolved finding is a **new** `✅ Resolved` comment, not an edit — count
  open ones by excluding those.
- Wait **twice** the slowest measured latency. A wait calibrated on an early
  fast sample will miss a later round and read as "Devin is done".
- Mutation-check fixes with `--nocache_test_results`, or a cached green result
  will pass against mutated code and a vacuous test will look real.

## Usage

Ask an agent to work the review:

```
check Devin's comments
address the Devin review on #4197
any new Devin comments?
```

Read the findings on one PR directly:

```bash
gh api repos/<owner>/<repo>/pulls/<n>/comments \
  -q 'sort_by(.created_at) | .[] | "[\(.created_at)] \(.path)\n\(.body)"'
```

## Install

Symlinked into the central skill store:

```bash
ln -s /Users/zhuoran/Programs/skills/devin-comment-addressment-skill ~/.agents/skills/devin-comment-addressment
```

One link, not two: `~/.claude/skills` is itself a symlink to `~/.agents/skills`
on this machine, so a second `ln -s` into it overwrites the first and leaves a
self-referential loop.
