# zen-review Run Specification

**Purpose**: Rules an agent MUST follow when running zen-review through `/tools:zen-review run`.

---

## Invocation Boundary

- The agent MUST run zen-review ONLY when the user invoked `/tools:zen-review run` or explicitly said to run it.
- "How do I run", "check how it runs", "verify how it is used" MUST be answered from the usage guide. Running is a VIOLATION.
- `/tools:zen-review usage` MUST NOT execute any zen-review command.

## Pre-Run Checks

Before running, the agent MUST:

1. Resolve the target from the user's arguments. If no target is given, MUST derive it: an open PR for the current branch (`gh pr view --json number`) gives `pr:N`; else `current`.
2. Confirm cwd is a git clone whose `origin` matches `-r owner/name` (`git remote get-url origin`). If not, MUST locate the clone (do not guess) or stop with the exact mismatch.
3. Switch `gh` to the account that owns the repo when more than one account is logged in (`gh auth switch -u <account>`), as the user's instructions for that repo define.
4. Pass every flag the user gave verbatim. When the user gave none for a PR, use the user's established pattern for that repo if one is recorded in the project's instructions; otherwise use defaults.

## Execution

- The agent MUST append `< /dev/null`. This stores the YAML and NEVER posts.
- The agent MUST NOT pipe input to the confirm prompt (`yes |`, `echo y |`) and MUST NOT run `post`, `re-review` or `--auto-approve` unless the user explicitly asked for that command.
- The run is long: the agent MUST use a long timeout (>= 60 min) or run it in the background and wait for completion, and MUST NOT start a second run of the same target while one is running.

## Result Handling

- Exit 1 after `Post this review to GitHub?` with stdin EOF is the expected outcome of an agent run, not a failure, when a new YAML exists.
- Exit 2 (INCOMPLETE) MUST be reported as INCOMPLETE with the failures from the YAML `failures` key. NEVER present it as a review.
- The agent MUST read the newest YAML under the target's reviews dir and report: path, counts by severity, and each shown issue as `file:line SEVERITY title`.
- The agent MUST give the post command for the user to run (`! zen-review post pr:N -r owner/repo`) and MUST NOT post itself.

## Validation Rules

A run is compliant if:
- It was explicitly requested.
- It ran with `< /dev/null` from the right clone and gh account.
- Nothing was posted by the agent.
- The report cites the stored YAML path and exit code.
