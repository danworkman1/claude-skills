---
name: feature-audit
description: Use when a large chunk of merged work needs runtime verification in a live browser — a big PR stack or weeks of features have landed and someone asks to audit, QA, walk through, or verify that shipped functionality actually works. App-agnostic — point it at any app directory in this repo.
---

# Feature Audit

## Overview

Turn recent git history into a test charter, then walk each shipped feature in a
real browser against a live dev server — one feature per iteration, with
file-backed state so the run survives compaction or a fresh session. The audit
is read-only on the codebase: it verifies and records, it never fixes.

History is the charter, main is the subject. Never check out historical commits
to test them.

## Inputs — prompt the user before starting

1. **Target dir** — the app to audit (e.g. `b2c-frontend/`).
2. **Window** — how far back: a ref, PR number, or date ("since #2163 landed").
3. **Server lane** — how to run the app. Default to the project's normal dev
   workflow; check memory and project skills for app-specific gotchas (env
   vars, WARP, ports) before starting it.

## Phase 1 — Charter

Create a workspace at `.claude/audits/<slug>/` (gitignored) holding:

- `features.json` — one entry per shipped feature:
  `{id, ticket, title, source, exercise, verify: [], status, evidence}`.
  `status` is one of `untested | pass | fail | blocked`. Seed it from
  `git log <since>..main -- <dir>` plus merged PR titles.
- `progress.md` — run journal: what's done, where to resume, server quirks
  discovered along the way.
- `findings.md` — the accumulating report.

If the window contains the app itself landing, the PR list understates the
surface: start the server, run a discovery crawl (routes, nav, modals, forms)
and add a feature entry per discovered flow before testing anything.

## Authenticated flows

Guest flows run on the default `chrome-devtools` MCP (headless, isolated).
Flows needing a real account — chat, alert creation, settings, checkout entry —
use the per-client profile server instead: `chrome-${client}-staging` (e.g.
`chrome-b2c-staging`), a headed Chrome on a persistent
`~/.cdp-profiles/<client>` profile that the user has logged into once. Never
type or fill credentials yourself — if the profile isn't logged in, mark those
features `blocked` and ask the user to log in via that browser window. One
session drives a given profile at a time (single-instance lock).

## Phase 2 — Audit loop

Each iteration:

1. Read `progress.md`; health-check the server, restart if dead.
2. Pick the highest-priority `untested` feature.
3. Drive it through the chrome-devtools MCP as a user would.
4. Capture evidence into the workspace: screenshot, console messages, and
   failed network requests — all three, every feature.
5. Update `features.json`, `findings.md`, `progress.md`. One feature per
   iteration, then loop.

`verify` steps are append-only — never weaken a criterion to make it pass. A
feature you cannot reach is `blocked` with a reason, never `pass`.

For a long run, this loop suits a dedicated autonomous session or `/loop`; the
workspace files are the memory, so resuming costs one read.

## Phase 3 — Report

When nothing is `untested`, publish `findings.md` as an artifact: pass/fail
table, each failure with its evidence, blocked items with reasons.

## Common mistakes

- Checking out old commits to test them — rebuilds, schema drift, wasted hours.
- Reaching for the seeded e2e harness — wrong lane; the audit wants real data
  on the dev server. The e2e skills are for deterministic regression specs.
- Fixing a bug mid-audit — record it in `findings.md` and move on; fixes are
  separate, ticketed work.
- Trusting the screenshot alone — half of what's broken only shows in the
  console or a failed request.
