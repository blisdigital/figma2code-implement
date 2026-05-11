# figma2code-implement — repo context for Claude

When Claude works in this repo, the following meta rules apply (separate from what is in `SKILL.md` — those are rules for *applying* the skill in a project; these are for *editing* the skill itself).

## Context

This repo is the **source of truth** for the `figma-to-code-implement` skill. The skill is installed by developers via a symlink:

```
~/.claude/skills/figma-to-code-implement → ~/Github/figma2code-implement
```

Every change to `SKILL.md` becomes active after `git pull` in every project that uses the symlink. **Changes propagate broadly** — there is no "test in one repo first".

This skill is a **strict consumer** of `figma-to-code-mapping`. Edits to consumption-related sections (mapping contract, cache schema reads, slash commands) must stay aligned with what mapping produces. When in doubt: check the mapping skill's `SKILL.md` for the current schema.

## Edit rules

1. **Branch + PR required.** Always work on a feature branch with a descriptive name (`skill-<what>`, e.g. `skill-fingerprint-confidence-thresholds`).
2. **One type of change per PR.** Split feature additions, readability restructures, and template changes into separate PRs.
3. **Bump version in frontmatter** on every significant change to `SKILL.md`. Patch (0.1 → 0.2) for refinements, minor (0.x → 1.0) on behavior changes that affect existing emits.
4. **Hold the mapping/implementation line.** When considering a new rule, ask: does this enforce something at emit time (implement) or document something (mapping)? Verbs like *consume, refuse, translate, search-and-adopt, halt, emit* belong here. Verbs like *document, detect, inventory, mark, link, capture* belong in mapping. On uncertainty: park as note for mapping skill or DEFERRED-FIXES.
5. **Skills.sh references are design-rationale, not runtime.** Comparisons with skills.sh `figma-implement-design` and `implement-design` skills stay in this `CLAUDE.md` (see below). They do not leak into `SKILL.md` — runtime users do not need that context.
6. **Coordinate breaking mapping-contract changes** with mapping repo. If implement needs a new mapping-output field, add it as a coordination item in mapping's `IMPLEMENT-HANDOFF.md` first; do not silently break the contract.

## Skill vs project mapping

- `SKILL.md` = method + rules for *applying* the skill in a project (loaded in every project where the skill triggers)
- `CLAUDE.md` = edit rules for *this repo* (loaded only when Claude works in this repo)
- `templates/` = starting point for projects (planned for Fase 3 — emit-trace, pre/post-emit checklists, claude-md-snippet)
- `README.md` = developer-facing setup guide on GitHub
- `PLAN.md` = design history with open decisions

When in doubt whether something belongs in `SKILL.md` or `CLAUDE.md`: **does it concern emitting code in a project? → SKILL.md. Does it concern maintaining this repo? → CLAUDE.md.**

## Skills.sh design rationale — what we took, what we rejected

Two reference skills at skills.sh describe a 7-step workflow for bare Figma MCP without a mapping layer: `figma-implement-design` and `implement-design`. We mined them during design but deviated where mapping discipline demanded it.

### Taken (mapped into our skill)

| Element | Source | Our location |
|---|---|---|
| `get_screenshot` as visual source-of-truth | step 3 | B3 (cache miss path) + B8 |
| `get_metadata` fallback on truncation | step 2 | B3 fallback chain |
| Post-emit visual validation | step 7 | B8 (added beyond mapping's A6) |
| Asset discipline (no new icon packages) | step 4 | Rule #9 |

### Rejected (deliberate divergence)

| Element | skills.sh position | Our choice | Why |
|---|---|---|---|
| "Adjust spacing or sizes minimally to match visuals" | step 6 | Rule #11 — never. Mismatch → drift. | Inline pixel-fixes undermine the drift-detection loop that mapping invests in. |
| Bare MCP without mapping-prerequisite | implicit | Rule #1, B1 — mapping required | Without mapping, we provide no value over bare MCP. We are explicitly the strict-mode pipeline on top of mapping. |
| Drift handling | implicit inline-fix | Rule #8 — surface + halt | Same reason as above. |
| Verify-queue | absent | B1 + B6 blockers | Mapping's verify-queue is a first-class concept; we honor it. |
| Pattern for non-componentized | "reuse components" only | B5 search-and-adopt with heuristics + confidence floor | "Reuse components" doesn't cover ad-hoc layout regions. |
| Traceability to mapping sources | absent | B7 commit-trace | We must be able to trace back to the mapping ground-truth we consumed. |

**Summary.** Skills.sh is stronger on live MCP mechanics (screenshot, metadata-fallback, post-emit visual check) — we took those. Our skill is stronger on mapping-discipline (drift, verify-queue, pattern-search, traceability) — we held the line.

## Open decisions

Tracked in `PLAN.md § Open beslissingen — verplaatsen naar DEFERRED-FIXES.md in Fase 1`. Six remain. Three are rule-text-determining and must land before Fase 2 rule-tuning:

1. **Verify-queue scope** — block all emits or only emits with in-scope verify items? (Proposal: only in-scope.)
2. **Rule #2 trigger box** — exact list of implement-intent sentences vs auto-clipboard ignores. Must be mirrored in mapping skill's rule #2 box.
3. **Confirmation gate before B7 emit** — atom/molecule direct emit, organism+ requires user confirmation? Or always confirmation? Or never?

The other three (cache hash format, hard-halt vs `--force` flag, pattern-search confidence floor) are implementation detail and can land during SKILL.md iterations.

## Writing lessons-learned

For every significant lesson at the bottom of `SKILL.md § Lessons learned` (to be added):

```
[LESSON — YYYY-MM-DD] [type: correction | confirmation]
Situation: <what happened, 1 line>
What worked (or did not): <observation, 1-2 lines>
Proposal: <change rule or keep, 1 line>
```

No longer. No vaguer. On overflow: split into two entries or the lesson is not sharply enough formulated.

## Reference

- The symlink is created by the end user (see `README.md`); for debug sessions you can run `ls -la ~/.claude/skills/figma-to-code-implement` to verify the symlink exists.
- Test changes: `git pull` in your own `~/Github/figma2code-implement/`, then trigger Claude in a test project with `/figma-to-code-implement check <Figma-link>` on a node that has mapping.
- Sister skill: [`figma-to-code-mapping`](https://github.com/blisdigital/figma2code-mapping). Coordination doc: [mapping#6](https://github.com/blisdigital/figma2code-mapping/pull/6).
