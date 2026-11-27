# Tools

Claude Code skills for Davi's CLI tools. Each tool gets one skill:

- `/tools:<cli> usage`: what it is, why, how to use it. Executes nothing.
- `/tools:<cli> run`: loads usage, then runs the tool under its run rules.

## Tools

| Skill | Tool |
|---|---|
| `/tools:zen-review` | zen-review: multi-reviewer code review against zazen + shodo standards, plus the AI lens on PR and branch targets. Results are analyzed in session; the agent posts only after approval |

## Install

```bash
/plugin install tools@daviguides
```

or `./install.sh`.
