# zen-review Usage Guide

**For run rules (normative)**: @~/.claude/tools/spec/zen-review-run.md

zen-review source: `~/work/sources/continuum/tools/context/zen-review` (README.md, CLAUDE.md hold the full internals). This guide is what a session needs to use it without searching.

---

## What It Is

Python CLI (Typer, v0.2.0) that runs an automated multi-reviewer code review against **zazen** (universal + zen) and **shodo** standards: python, rust, java, typescript (javascript uses typescript standards).

- Engine: Claude Agent SDK sessions. Model pins from `~/work/sources/continuum/tools/data/model_pins.json`: reviewers `claude-sonnet-5-5[1m]`, security `claude-sonnet-5-5`, dedup judge Haiku.
- 6 reviewers per shard, in parallel: Zen, Security, Performance, Architecture (SOLID), Style, Testing.
- Reviewers get a manifest (paths, +/- lines, language), pull diffs themselves, read code in a temp worktree of the reviewed revision. The user's checkout is never switched.
- Shards: one shard when <= 4000 changed lines and <= 25 files, else split by language. Generated/vendored files excluded and listed.
- Dedup: same file +-10 lines or embedding similarity >= 0.82, then Haiku judge.
- Security lens: deterministic predetect (removed-defense detector, attack surface delta, evasion detector) plus external scanners (opengrep, osv-scanner, trufflehog, gitleaks, zizmor, actionlint, checkov, hadolint, cargo-audit, ruff S, guarddog).

## Why It Exists

One command reviews a PR, branch, local change or whole project against Davi's standards, stores the result as YAML, and posts inline comments to GitHub only after confirmation. Fail-closed: any failed reviewer run makes the review INCOMPLETE; an INCOMPLETE review is never posted and never approves.

---

## Install and Update

- Installed as a uv tool, **editable** from the source dir. Source edits apply immediately.
- Binary: `~/.local/bin/zen-review` -> `~/.local/share/uv/tools/zen-review/bin/zen-review`.
- Reinstall only when dependencies change: `uv tool install --editable --force ~/work/sources/continuum/tools/context/zen-review`
- Requires git >= 2.42, `gh`, uv.
- Scanner binaries: `zen-review deps check`, `zen-review deps install` (installed under `~/.local/share/zen-review`, checksum verified, 7-day minimum release age).

---

## Targets

| Target | Meaning |
|---|---|
| `pr:N` | GitHub PR. Base comes from the PR itself (`baseRefName`); `-b` is ignored |
| `current` | Uncommitted changes, staged + unstaged (default target) |
| `branch:NAME` | Branch vs `-b` base (default `main`) |
| `all` | Entire repository |

## Commands

### `review [target]`: full pipeline

| Flag | Meaning |
|---|---|
| `-r/--repo owner/name` | Repository; must match cwd `origin` |
| `-b/--base-branch` | Base for `branch:` and `current` (default main) |
| `--profile` | `challenge` (exercise code, hardening never blocks), `internal` (default, hardening LOW), `production` (hardening may be MEDIUM, HIGH capped to MEDIUM) |
| `--min-relevance 1-5` | Default 3. Lower issues stored, not shown/posted |
| `-l/--language` | Force one language's standards; files not filtered |
| `-e/--extensions .py,.ts` | Filter files |
| `-s/--storage DIR` | Override result dir (needed when repo is not in dao registry) |
| `--token-budget N` | Max fresh tokens; past it no new session, review INCOMPLETE |
| `--security-token-budget N` | Separate budget for the security pool |
| `--auto-approve` | Approve when a COMPLETE review finds no issues |

### Other commands

| Command | Purpose |
|---|---|
| `collect [target] [-r] [-b] [-o json\|text] [-e]` | Context only, no LLM |
| `post pr:N [-r] [-f file.yaml]` | Post latest stored review (no reviewers, no LLM). Confirm, default No |
| `re-review pr:N [-r] [-f]` | After author replied: classify each issue fixed / justified / no-response, resolve threads, reply, optionally approve. Two confirms, default No |
| `ai review {pr:N\|branch:X} [-r] [-b] [--profile] ...` | AI lens (see below) |
| `security predetect [pr:N\|branch:X] [-r] [-b] [-f yaml\|text]` | Deterministic security checks, no LLM |
| `security scan {pr:N\|branch:X} [-r] [-b] [--json]` | Scanners + lockfile analysis + dependency cards |
| `deps check\|install\|update\|rollback [--tool X]` | Scanner binaries |
| `bench ...` | Internal eval harness (precision, label, run, report). Not for normal use |

### `ai review {pr:N|branch:X}`: AI lens

Reviews what models read, call and are judged by: prompts and docs a model reads, tool/MCP code, evals, retrieval, agent config, model call sites. A separate run from `review`, with its own YAML. A full PR review is both lenses.

| Flag | Meaning |
|---|---|
| `-r/--repo` | Repository |
| `-b/--base-branch` | Base for the comparison. Default `main`, NOT taken from the PR: pass the PR base |
| `--profile` | `challenge`, `internal` (default), `production` |
| `--min-relevance 1-5` | Default 3 |
| `--models opus,agy,codex` | Models that review (default all three) |
| `--quorum N` | Successful models required (default all) |
| `--from FILE...` | Consolidate reviews already run (zen-review YAML, runner JSON, markdown) |
| `--resume` | Reuse runs recorded for the same head and prompt |
| `--ai-token-budget N` | Max fresh tokens for the AI sessions |
| `-s/--storage` | Result dir (an `ai/` subfolder is used) |
| `--auto-approve` | APPROVE when there are no findings |

Pipeline:

```
collect -> predetect (no LLM) -> [0 surfaces: "Continue with models anyway?" default Yes;
non-TTY takes the default] -> model reviewers -> consolidate (grounding, relevance, dedup,
agreement) -> verifier on Must candidates -> merge gate -> STORE ai/<ts>.yaml -> post prompt
```

- **Predetect** (`zen_review/ai/predetect.py`): a changed file is an AI surface when an added line imports a model SDK, when it matches the built-in globs (model-read text, tool/MCP, evals, retrieval, agent config), or when it matches an `ai` path in the project's routing file read at the BASE revision (`docs/team/routing.yml` or `.zen-review/ai.yml`). It also greps model-read text for terms the non-AI diff changed. There is no standalone AI predetect command: `ai review` always goes on to the models.
- **Merge gate**: `must` only when the verifier returned CONFIRMED; PLAUSIBLE or no verification caps it at `nice` (`gate_override_reason`).
- **Filtering**: an issue the grounding check cannot tie to code is filtered with `filter_reason: ungrounded`. The filter can drop real issues.

## Pipeline and Exit Codes

```
collect -> manifest -> temp worktree -> shards -> per shard: reader session
 -> 6 reviewer forks -> dedup (cluster + Haiku judge) -> profile -> relevance
 -> summary -> STORE YAML -> confirm "Post this review to GitHub?" (default No) -> post
```

| Exit | Meaning |
|---|---|
| 0 | Review complete |
| 1 | Error (includes post prompt aborted by stdin EOF, after the YAML was stored) |
| 2 | INCOMPLETE: a reviewer failed, a file was not covered, or budget hit. Stored, not posted |

The AI lens without a TTY skips the post prompt, prints "Not an interactive terminal: nothing was posted" and exits 0.

## Storage

- Path: `~/work/projects/<owner>/<project>/reviews/<pr-N|branch-NAME|current|all>/<YYYY-MM-DD-HHMMSS>.yaml`
- Project resolved from cwd `origin` via dao registry `~/work/projects/data/registry.yaml`, before any LLM spend. Unregistered repo: exit 1; register with `dao project init` or pass `--storage`.
- YAML top-level keys: `issues`, `summary` {high, medium, low, total}, `filtered_issues`, `failures`, `excluded_files`, `dedup_stats`, `usage`, `pr_number`, `repo`, `head_sha`, `base_sha`, `base_branch`, `profile`, `file_count`, `security`.
- Issue keys used most: `file`, `line`, `severity` (HIGH/MEDIUM/LOW), `category`, `title`, `problem`, `suggested_fix`, `relevance`, `reviewer`.
- AI lens path: `.../reviews/<target>/ai/<YYYY-MM-DD-HHMMSS>.yaml`.
- AI lens issue keys, besides the above: `merge_gate` (must/nice/nit), `verify_verdict` (CONFIRMED/PLAUSIBLE), `verify_evidence`, `gate_override_reason`, `merge_impact` (immediate/future), `agreement` (consensus/single-source), `found_by`, `filter_reason`, `code_truth`, `model_text`. Top level adds `lens: ai`, `ai_predetect` (surfaces, terms, hits).
- Latest review: `ls -t ~/work/projects/<owner>/<project>/reviews/pr-N/ | head -1` (AI lens: `.../pr-N/ai/`)

## Environment Variables

Prefix `ZEN_REVIEW_`, also read from `.env` in cwd.

| Var | Default |
|---|---|
| `MAX_CONCURRENCY` | 16 concurrent reviewer sessions |
| `TOKEN_BUDGET` / `SECURITY_TOKEN_BUDGET` | unlimited |
| `MAX_THINKING_TOKENS` / `SECURITY_MAX_THINKING_TOKENS` | 8000 / 12000 |
| `FORK_WARMUP_SECONDS` / `LEAD_FORK_TIMEOUT_SECONDS` | 3 / 30 |
| `DEDUP_THRESHOLD` | 0.82 |
| `SCANNERS` | auto (`off` or comma list) |
| `SCANNER_ON_MISSING` | warn (fail, warn, skip) |

---

## Gotchas

1. Run from a clone whose `origin` is the target repo. The remote name `origin` is hardcoded.
2. `gh` must be on the account that can read the repo: `gh auth switch -u <account>`. "Repository not found" means wrong account.
3. Missing scanners print a banner (`trufflehog missing, ...`); the review still runs with reduced security coverage.
4. Big PRs take minutes to tens of minutes and up to 16 concurrent Sonnet sessions. Cap with `--token-budget`.
5. `pr:` targets require UTF-8 content.
6. Results survive a lost terminal: read the YAML.

## Recipes

```bash
# PR (from a clone of the repo, gh on the right account)
zen-review review pr:123 -r owner/repo --profile production

# Agent run: stores YAML, never posts
zen-review review pr:123 -r owner/repo --profile production < /dev/null

# AI lens, PR base passed explicitly
zen-review ai review pr:123 -r owner/repo -b staging --profile production < /dev/null

# Post after reading the YAML (human, in the terminal)
zen-review post pr:123 -r owner/repo

# After the author replied
zen-review re-review pr:123 -r owner/repo

# Pre-PR / local
zen-review review branch:feature/foo -b main
zen-review review current

# Repo not in dao registry
zen-review review pr:123 -r owner/repo --storage <dir>

# Force language
zen-review review pr:9 -r owner/repo -l java

# No LLM
zen-review collect pr:123 -r owner/repo -o json
zen-review security predetect pr:123 -r owner/repo -f text
```
