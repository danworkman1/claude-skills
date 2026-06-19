---
name: dedupe
description: Find duplicated code left behind by large refactors or AI-generated output — parallel type/contract definitions, copy-pasted logic, repeated constants, and near-identical components — then plan its removal. Use when the user mentions de-duping, removing duplication, DRYing up code, consolidating repeated code, or cleaning AI-generated code toward production. Detects mechanically (jscpd + grep), recommends consolidate-vs-leave per the meaning-not-shape rule, requires a human decision on each, then hands the approved list off to a standalone md doc to action later. Does not edit code in the same session.
---

# De-dupe

AI-generated code and big refactors leave two failure modes side by side: **real duplication left in place**, and the over-correction — **incidental similarity collapsed into a leaky abstraction**. This skill catches the first without causing the second.

The whole skill is one rule: **consolidate shared *meaning*, not shared *shape*.** Two blocks that look identical today but change for different reasons are not duplicates — they're coincidence. Two blocks that must always change together are one concept typed twice.

## Workflow

This skill **plans** the de-dupe; it doesn't do it in the same breath. It ends by writing the approved list to a standalone doc the user picks up whenever they like.

1. **Detect mechanically** — find candidates, don't eyeball.
2. **Triage each candidate** — apply the decision rule to form a *recommendation*, not a verdict.
3. **Get the human's call** — present the candidates, let them decide what's consolidated and what's left. **This gate is mandatory.**
4. **Build the approved list, then hand it off** — write the approved consolidations to a separate md file via the handoff skill. **This is the deliverable — stop here.**
5. **(On pickup) Consolidate + verify** — when the doc is later picked up, work the list by type (below) and verify.

Do not start consolidating in this session. The point of the handoff is that the user runs the actual edits whenever they choose — possibly a fresh agent following the doc.

## 1. Detect

Run the copy-paste detector across the area you refactored:

```bash
pnpm dlx jscpd --min-tokens 50 --reporters consoleFull --gitignore <path>
```

Lower `--min-tokens` to ~30 to catch smaller blocks; raise it to cut noise. Then grep for the categories jscpd misses:

- **Constants/strings** — `grep -rn "the-literal-value"` for URLs, keys, magic numbers, status strings re-typed across files.
- **Parallel types** — search for a field name unique to a shape (`grep -rn "loyaltyTier"`) to find the same object redeclared as a separate `interface`/`type`/zod schema.
- **Component shells** — grep for repeated JSX (`grep -rn "className=\"card"`), repeated prop names, near-identical hooks.

jscpd finds character-level copies; grep finds the same *idea* spelled slightly differently. You need both.

## 2. Triage — the decision rule

For each candidate, before touching it:

| Keep duplicated (leave it) | Consolidate |
|---|---|
| Only **two** occurrences and no third in sight — wait (rule of three) | **Three+** occurrences, or two that demonstrably change in lockstep |
| Looks alike but changes for **different reasons** | Changes for the **same reason**, always together |
| Sharing would need a `mode`/`variant`/`isX` flag to paper over differences | Differences are pure data (values), not behaviour |
| Different domains that coincidentally match | Same domain concept, one source of truth |

If consolidating forces a boolean/enum parameter that switches the body's behaviour, **stop** — that's two functions wearing a trenchcoat. Leave them apart.

## 3. Hand off the approved list

Once the human has signed off, invoke the **handoff** skill (`Skill` tool, `handoff`) to write the approved set to a standalone markdown doc. Pass it a brief: *"de-dupe plan — approved consolidations to action when picked up."*

The doc must capture, per approved item:

- **What** — the duplicate concept (e.g. "loyalty-tier payload schema declared 3×").
- **Where** — every file:line occurrence, and the chosen canonical home.
- **How** — the consolidation type from §4 and any non-obvious decision (e.g. "derive TS type via `z.infer`, delete the hand-written interface").
- **Verify** — the exact lint/typecheck/test commands for the touched packages.

Also record what was **deliberately left duplicated** and why — so the next pass doesn't re-litigate it.

The handoff skill saves to the OS temp dir; tell the user the path. If they want it kept long-term, offer to move it into the repo (e.g. `docs/`). Build the TodoWrite list from the approved items only — never add a candidate before it's approved. Then **stop**: don't edit code this session.

## 4. Consolidate by type *(on pickup)*

- **Parallel type/contract defs** → one canonical schema, derive the rest. With zod, define the schema once and `z.infer` the TS type; reuse it across client and worker. Never let a hand-written `interface` shadow a zod schema for the same payload.
- **Copy-pasted logic** → extract a function whose *parameters are the data that varied*. If what varied is behaviour, it's not one function — see the rule above.
- **Repeated constants/config** → a single `const`/enum in the nearest shared module. Co-locate related constants; export, don't re-type.
- **Similar components/JSX** → prefer **composition** (shared child, `children`, slots) over a mega-component with conditional branches. Extract the shared sub-component; let callers compose. A prop that swaps whole layouts is a smell — split instead.

In a pnpm workspace, put genuinely cross-package shared code in `packages/*`, not copied into each consumer. Watch import direction — shared code must not depend back on its consumers.

## 5. Verify *(on pickup)*

Consolidation is a refactor: same behaviour, less code. Confirm it:

```bash
pnpm -w turbo run lint typecheck   # adjust to the repo's actual tasks
pnpm --filter <pkg> test           # scope tests to touched packages
```

Then re-run jscpd to confirm the duplication count actually dropped. Don't claim it's done until the detector agrees and tests pass.

## Red flags you're over-doing it

- You're adding a config object/flag to a helper so it can serve two callers → leave them separate.
- The "shared" abstraction lives in a `utils`/`common` grab-bag with no coherent owner → wrong home.
- You're consolidating across domain boundaries because the code *looks* similar → coincidence, not duplication.
- The diff makes call sites harder to read than the duplication did → net negative, revert it.
