# skillrepro

**Does your agent skill behave the same way every time?**

skillrepro checks the *behavioral reproducibility* of agent skills (Claude Code / Agent Skills).
It runs a skill many times across models and tells you **which step** the runs diverged at —
not just whether the final output passed.

To make that possible, skills are written as structured YAML (`skill.yaml`) and compiled to `SKILL.md`.
Structure lives only where review and observation need it; everything inside is plain natural language.

> Status: design stage. The specification is in [docs/specs](docs/specs/2026-09-25-skillrepro-design.md) (Japanese).

## Why

Skills get copied and shared, and then they quietly stop working:

- **Behavior drifts** — the same request skips a step on one run, or takes a different route on a smaller model.
- **Environments differ** — the skill silently depends on the author's paths, CLIs, MCP servers, env vars, and subagent definitions.

Static linters check `SKILL.md` without running it. Eval tools report pass rates, but not where the runs split.

## What it looks like

```yaml
spec: skillrepro/v0.1
name: weekly-digest
description: |
  Summarizes this week's meeting notes and posts them to Slack.
  Use when: the user asks for a weekly digest of meetings.
steps:
  - name: collect
    description: Collect this week's meeting notes.
    run: python3 scripts/list_notes.py --dir {{paths.notes_dir}} --days 7
    freedom: low
  - name: summarize
    description: Extract decisions and action items, citing the source note for each.
  - name: post
    description: Post the summary to Slack.
    call: { capability: slack.post, args: { channel: "$SLACK_CHANNEL" } }
    gate: confirm
```

```text
weekly-digest  eval: normal-week  N=5
              pass   path-agreement  first divergence
  haiku       2/5         40%        summarize (3 runs: skipped verify, never reached post)
  sonnet      5/5        100%        -
```

## Planned commands

| Command | Purpose |
| --- | --- |
| `skillrepro compile` | Generate `SKILL.md` (production or trace mode) from `skill.yaml` |
| `skillrepro run` | Run evals × models × N times with all external calls stubbed, and report divergence per step |
| `skillrepro lint` | Static checks, including undeclared environment dependencies |
| `skillrepro doctor` | Check whether this machine satisfies the skill's declared requirements |

## 日本語

エージェントのスキルが「何回実行しても、どのモデルでも、同じように振る舞うか」を検査するツールです。
スキルを YAML で手順ごとに構造化して書き、実行のブレを手順単位で特定します。
設計は [仕様書](docs/specs/2026-09-25-skillrepro-design.md) を参照してください。

## License

MIT
