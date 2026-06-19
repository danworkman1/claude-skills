# claude-skills

Personal collection of [Claude Code](https://claude.com/claude-code) skills, shared so colleagues can use them too.

## Skills

| Skill | What it does |
|---|---|
| [`dedupe`](skills/dedupe/SKILL.md) | Find duplicated code left by big refactors or AI output, then plan its removal. Detects mechanically (jscpd + grep), recommends consolidate-vs-leave per the meaning-not-shape rule, and hands the approved list off to a doc to action later. Plans only — doesn't edit code in the same session. |

## Install

Each skill is a directory with a `SKILL.md`. Drop the ones you want into your personal skills dir:

```bash
git clone https://github.com/danworkman1/claude-skills.git
cp -r claude-skills/skills/dedupe ~/.claude/skills/dedupe
```

Skills in `~/.claude/skills/` are picked up automatically — invoke from any project with `/dedupe` (or just describe the task and Claude will trigger it).

To stay up to date, `git pull` and re-copy.
