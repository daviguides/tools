# zen-review Run Specification

**Purpose**: Rules an agent MUST follow when running zen-review through `/tools:zen-review run`, analyzing its results, and posting a review.

Commands, flags, YAML fields and lens internals live in the usage guide (`tools/context/guides/zen-review.md`), loaded with this spec.

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
4. Resolve the PR base branch (`gh pr view N --json baseRefName`). It is the `-b` of the AI lens.
5. Pass every flag the user gave verbatim. For flags the user did not give, apply the repo pattern recorded in the project's instructions, then the defaults below.

## Lenses and Profile

- **Profile**: a `pr:` target MUST run with `--profile production` unless the user names another profile. `internal` (the CLI default) only when the user asks for it.
- **Standard lens**: `zen-review review` runs on every target.
- **AI lens**: on every `pr:` and `branch:` target the agent MUST also run `zen-review ai review <target> -r <repo> -b <PR base> --profile production`. The CLI's own predetect decides what is an AI surface; the agent MUST NOT skip the AI lens on its own judgment that the change "has no AI". Only an explicit user instruction skips it.
- The AI lens `-b` MUST be the PR base. Its CLI default is `main`, which is wrong for repos whose PRs target another branch.

## Execution

- The agent MUST append `< /dev/null` to every run. This stores the YAML and NEVER posts.
- The agent MUST NOT pipe input to the confirm prompt (`yes |`, `echo y |`) and MUST NOT run `post`, `re-review` or `--auto-approve`. Posting is done by the agent through the GitHub API (see Posting), never by zen-review.
- The run is long: the agent MUST use a long timeout (>= 60 min) or run it in the background and wait for completion, and MUST NOT start a second run of the same target and lens while one is running.
- Both lenses MAY run in parallel.

## Exit Codes

| Lens | Exit | Meaning for an agent run |
|---|---|---|
| Standard | 1 after `Post this review to GitHub?` with stdin EOF | Expected, when a new YAML exists |
| AI | 0 with "Not an interactive terminal: nothing was posted" | Expected |
| Any | 2 | INCOMPLETE: MUST be reported as INCOMPLETE with the YAML `failures`. NEVER presented as a review |

## Analysis

The agent MUST analyze the stored YAMLs, not relay them. A count of issues is NOT an analysis.

1. Read the newest YAML of each lens (`reviews/<target>/<ts>.yaml`, `reviews/<target>/ai/<ts>.yaml`).
2. Verify every shown issue against the code at the PR head (`git show <head>:<path>`) and, when the claim depends on runtime, against production data the session can read. Record CONFIRMED, PARTIAL or REFUTED with the file:line evidence.
3. Re-open every filtered issue (`filtered_issues`). A filter (`ungrounded`, low relevance) can be wrong; an issue the agent confirms is reported as a finding.
4. Deduplicate across runs and lenses.
5. Classify each finding as **Must** (blocks merge), **Nice** (follow-up) or **Nit**, by real exposure: a defect on a path nothing calls today is Nice, not Must. State the reason when the classification differs from the tool's `severity` or `merge_gate`.
6. Report: YAML paths and exit codes, then the complete list (shown and filtered) with file:line, the tool's severity, and the agent's reading. Nothing is dropped silently.

## Posting

Posting happens ONLY after the user approves the exact text. Message wording follows the project's communication rules and the fala plugin; this spec does not restate them.

- One call: `gh api -X POST repos/<owner>/<repo>/pulls/<N>/reviews --input <payload.json>` with `commit_id`, `event`, `body`, `comments`.
- `event`: `REQUEST_CHANGES` when any Must exists; else `COMMENT`.
- **Inline comments are ALWAYS ranges**: `start_line` + `line`, `start_side` and `side` = `RIGHT`, covering the whole block the comment discusses. A single-line anchor is a VIOLATION.
- Every range MUST lie inside the PR diff. The agent MUST check each range against the `patch` of `pulls/<N>/files` before posting.
- A finding whose file or lines are not in the diff goes to the body, never inline.
- Each inline comment opens with its class: `**Must:**`, `**Nice:**` or `**Nit:**`.
- The body has three sections, in order: **Must**, **Nice**, **Nit**. Each lists every finding of its class with `path:start-end`, including the inline ones (the Nit section is the summary of the inline nits).
- Before posting, the agent MUST confirm the PR head still equals `commit_id`. After posting, MUST confirm the review state and the number of inline comments GitHub recorded.

## After the Author's Fixes

- Reply to each inline comment of the review (`pulls/<N>/comments/<id>/replies`): `Fixed in <sha>: <what>` or `Not in this PR: <why>, follow-up`.
- A Must fixed outside any inline comment is stated in the approval body.
- Approve with `event=APPROVE` on the current head, body = what was fixed (with SHAs) and the follow-ups left open.
- Replies and approval are posted only when the user asked for them.

## Validation Rules

A run is compliant if:
- It was explicitly requested.
- It ran with `< /dev/null` from the right clone and gh account, with `--profile production` on PR targets.
- The AI lens ran on every `pr:`/`branch:` target, with `-b` = PR base, unless the user skipped it.
- Nothing was posted by zen-review; nothing was posted by the agent without the user's approval of the exact text.
- The report cites the stored YAML paths and exit codes and covers shown and filtered issues.
