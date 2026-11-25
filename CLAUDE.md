# Tools - Claude Code Project Instructions

## Project Overview

**Tools** gives a session everything it needs to use Davi's CLI tools without searching: what each tool is, why it exists, how to use it, and how an agent runs it safely.

**Purpose**: one skill per CLI, dispatched by argument: `/tools:<cli> usage` (explain, execute nothing) and `/tools:<cli> run` (load usage, then run under the tool's run spec).

First tool: **zen-review** (`~/work/sources/continuum/tools/context/zen-review`).

---

## Structure

```
tools/
├── .claude-plugin/plugin.json
├── tools/                              # Bundle, installed to ~/.claude/tools/
│   ├── spec/zen-review-run.md          # Normative run rules
│   ├── context/guides/zen-review.md    # Usage guide (what, why, how)
│   └── prompts/zen-review.md           # Loader: @ guide + @ spec
├── skills/zen-review/SKILL.md          # /tools:zen-review usage|run
├── CLAUDE.md
├── README.md
├── install.sh
└── LICENSE
```

### Adding a tool

1. `tools/context/guides/<cli>.md`: what, why, install, commands, gotchas, recipes. Source of truth stays in the CLI's own repo; the guide holds what a session needs to use it.
2. `tools/spec/<cli>-run.md`: when the agent may run it, pre-run checks, execution, result handling.
3. `tools/prompts/<cli>.md`: @ guide + @ spec.
4. `skills/<cli>/SKILL.md` with `argument-hint: usage | run ...`.
5. Add the skill to the table below and to install.sh next steps. Bump minor.

---

## Skills

| Skill | Triggers | Purpose |
|---|---|---|
| `/tools:zen-review usage` | "zen-review", "como roda/usa o zen-review" | Explain zen-review. Executes nothing |
| `/tools:zen-review run [target] [flags]` | "rode o zen-review", "zen-review run" | Run a review, store YAML, never post |

---

## Related Plugins

- **baliza**: security review beacons consumed by zen-review's security lens. Tools runs the CLI; baliza is the knowledge it uses.
- **zazen**, **shodo**: the standards zen-review checks against. Tools does not restate them.
- **consult**: second opinions across models. Unrelated to running CLIs.

---

## Releasing — mandatory workflow

Every plugin modification MUST follow this sequence. No exceptions.

### 1. Bump version

Patch for fixes/tweaks, minor for new skills or tools: `.claude-plugin/plugin.json` and the install.sh banner.

### 2. Commit and push

```bash
git add -A && git commit -m "bump: vX.Y.Z — <what changed>"
git tag -a vX.Y.Z -m "<what changed>"
git push && git push origin vX.Y.Z
```

### 3. Run install.sh

```bash
~/work/sources/continuum/gradients/tools/install.sh
```

install.sh clones from the GitHub remote, so the push in step 2 must land first.

### 4. Verify cache is not stale

```bash
diff <(ls -lR ~/.claude/tools/) <(ls -lR tools/)
ls ~/.claude/plugins/cache/daviguides/tools/

# If stale
rm -rf ~/.claude/plugins/cache/daviguides/tools/ ~/.claude/tools/
./install.sh
```

Do NOT move to the next task with a stale cache.
