---
name: zen-review
description: |
  zen-review CLI: multi-reviewer code review (zazen + shodo standards) for PRs,
  branches, local changes. "usage" loads what it is, why, and how to use it.
  "run" loads usage, then runs a review (never posts).
  TRIGGERS:
  - "zen-review", "zen review", "como roda o zen-review", "como usa o zen-review"
  - "review with zen-review", "rode o zen-review", "zen-review run"
  Only "run" (or an explicit "rode/run zen-review") executes. Asking how it runs is usage.
argument-hint: usage | run [pr:N|branch:NAME|current|all] [zen-review flags]
---

# /tools:zen-review

Arguments: $ARGUMENTS

STOP. Read: ~/.claude/tools/prompts/zen-review.md

Then dispatch on the first argument:

| First argument | Action |
|---|---|
| `usage` or empty | Present the guide's answer to the user's question. Execute nothing |
| `run` | Remaining arguments are the target and flags. Follow the run spec |
