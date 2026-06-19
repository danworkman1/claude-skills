---
name: run-hindsight
description: Use when the user asks to run hindsight, review the current branch, or compare branches for a post-implementation code review. Also use when you see "run hindsight" or "hindsight review" in conversation.
---

# Run Hindsight

Hindsight is a post-implementation code review CLI. It diffs the current branch against a base and sends the diff to Claude for review.

## Invocation

**From any repo** (installed globally):
```bash
hindsight
```

**From the hindsight repo itself** (development):
```bash
node dist/index.js
```

**Do NOT** run `pnpm build`, `npm run build`, or any build step. The dist is pre-built and committed.

## Flags

| Flag | Purpose | Default |
|------|---------|---------|
| `--base <ref>` | Diff against `<ref>..HEAD` | `main` |
| `--force` | Bypass triage (always run full review) | off |
| `--path <dir>` | Run as if launched in `<dir>` | cwd |
| `--triage-model <name>` | `haiku`, `sonnet`, `opus`, or raw model id | — |
| `--review-model <name>` | `haiku`, `sonnet`, `opus`, or raw model id | `opus` |

## Common Uses

```bash
hindsight                        # review current branch vs main
hindsight --base develop         # review against develop
hindsight --force                # skip triage, always do full review
hindsight --review-model sonnet  # use sonnet instead of opus
```

## What It Returns

A verdict (`no_issues`, `minor`, `major`, `critical`) plus a prose summary of findings. Output goes to stdout.
