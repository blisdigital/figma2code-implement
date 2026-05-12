# Deferred fixes — `figma-to-code-implement`

> Open decisions and design questions that have surfaced during the design conversation
> but are not yet resolved. Parallel to mapping skill's `DEFERRED-FIXES.md`.
>
> **Three tiers:**
> - **🔴 Rule-text-determining** — affects SKILL.md hard rules. Must land before v1.0 rule-tuning.
> - **🟡 Implementation-detail** — affects how a rule operates but not the rule text. Can land during SKILL.md iterations.
> - **🟢 Nice-to-have** — quality-of-life upgrade. Optional, no ship dependency.

Items tracked here have a proposal where one exists; if you decide differently, update the proposal field and bump the relevant section of SKILL.md in the same PR.

**Current state (v0.3):** all rule-text-determining items resolved. Remaining items are implementation-detail (🟡, 3 items) and nice-to-have mapping-repo suggestions (🟢, 2 items). None are ship-blockers for v1.0; they can resolve during Fase 5 dogfood or later iteration.

---

## 🟡 1. Cache hash format — what exactly is hashed?

**Status:** open.

**Context:** B3.0 hash-checks the cache against current code-files. Mapping skill writes `spec_synced_with_files_hash` as `sha256:<hex>` (line 113-123). Implement reads this. The exact hash input is undocumented.

**Options:**
- Single file hash (the spec file only)
- Combined hash (spec file + linked component file + dependencies)
- Tree hash (entire component folder)

**Coupled with:** mapping skill cache schema. Cache-format formalisation should define this precisely — check current mapping `SKILL.md § Cache + hash check` for state. (Earlier tracked in mapping#6, now closed; mapping v3.3+ may already cover this.)

**Impact if changed:** B3.0 logic + mapping cache write logic. Coordinated change.

---

## 🟡 2. Hard halt vs `--force` flag

**Status:** open.

**Context:** Three rules halt the emit unconditionally — #5 (component-missing), #7 (verify-queue blocker), #8 (drift in scope). In practice users may want a `--force` escape hatch for known-acceptable cases (e.g., "I know this verify-queue item, emit anyway").

**Two options:**
- **A — No force flag.** Halts are unconditional. Users must resolve in mapping first. Cleaner, slower.
- **B — Allow `--force` on slash-command.** `/figma-to-code-implement --force <link>` skips halts. Faster, risks misuse.

**Open:** worth the implementation complexity? Or document the manual workaround (edit the verify-queue item, then re-run)?

**Impact if changed:** slash command parsing, halt logic in B1 / B4 / B6.

---

## 🟡 3. B5 pattern-search confidence floor

**Status:** open, proposal in SKILL.md B5.

**Context:** B5 pattern search adopts a layout pattern from "similar-context" files. If fewer than 2 similar-context files exist, current proposal is halt-and-ask. The "2" is intuition, not calibrated.

**Open:** does "2" scale with project size? On large projects, "2" is too low (many incidental matches). On small projects, "2" is unreachable and B5 always halts. Possibly should scale (e.g., `>= max(2, ceil(total-matching-context-files / 10))`).

**Impact if changed:** B5 description + heuristic order.

**Calibration plan:** decide during Fase 5 dogfood on a real project.

---

## 🟢 4. Mapping-side classification methodology — suggestion for mapping repo

**Status:** explicit suggestion, **not coupled to implement v0.1 ship.**

**Context:** Path B (fingerprint via mapping-data) works gracefully whether mapping classifies element-vs-component loosely or strictly. A tighter mapping methodology (explicit `element-frame` vs `component-instance` vs `component-missing` classification + verification step) would reduce how often Path B fires.

**Where it lives:** mapping repo's own roadmap, not this repo's. Captured here for traceability — if mapping adds this, implement's Path B becomes near-redundant.

**Coupled with:** mapping repo upgrade (open). Implement should not block on this.

---

## 🟢 5. Per-spec component fingerprint section

**Status:** suggestion for mapping repo.

**Context:** Path B currently scans full per-component specs for matching signals. A condensed "fingerprint" sub-section per spec (token-stempel + variant axes + naming hints in a single block) would make Path B scans cheaper and more deterministic.

**Where it lives:** mapping repo template upgrade.

**Coupled with:** mapping repo, no implement dependency.

---

## 🟢 6. Mapping-side fluidity-intent for page-level frames

**Status:** suggestion for mapping repo. Surfaced by pre-implement-skill internal testing (see [LESSONS.md](LESSONS.md) entry 2026-05-08 correction).

**Context:** A Figma frame may be designed at 1440×900 but in code should fill the viewport (100vw / 100vh) — e.g. 404 pages, error layouts, hero sections. Implement rule #4 translates Figma `fill/hug/gap` correctly when documented, but page-level "fill viewport" intent is currently not a documented concept in mapping artifacts. Without a marker, implement defaults to literal Figma pixel dimensions and the emit only fills ~40% of viewport.

**Proposal:** mapping skill extends `tokens.md § Auto-layout conventions` or per-component spec with a "fluidity intent" field — e.g. `viewport-fill: true`, `max-width: <token>`, `fluid-grid: <breakpoint-set>`. Implement reads the field and emits the appropriate fluid expression (`width: 100vw`, container queries, etc.) instead of Figma pixel dimensions.

**Where it lives:** mapping repo. Implement cannot solve this alone — requires intent data that only the designer/dev pair knows.

**Coupled with:** mapping repo upgrade. Not blocking for implement, but the responsiveness gap reported in pre-skill testing remains until mapping documents intent.

---

## Resolved decisions — moved to history

For traceability, decisions that were made during design but later closed:

- **Partial mapping handling** — resolved: halt on first ungated Figma node concretely in scope, route to mapping (PLAN.md). Atomic-order applies. Element-frames without component-equivalent are legitimate via tokens + B5 pattern.
- **Deviation comment in code** — schrapt: redundant with B7 commit traceability and drift-acceptance trail in mapping's `drifts.md`. No in-code comments per accepted drift.
- **Implement fingerprint detection on Figma-side tokens** — schrapt in favor of mapping-data driven matching (Path B uses per-component specs as fingerprint source, no Figma-side detection layer in implement).
- **Implement writes to mapping `drifts.md` / `verify-queue.md`** — schrapt as default; only two narrow propose-to-user exceptions remain (B4.1 Path B accepted → `verify-queue.md`, B4.1 Path C → propose-route-to-mapping without writing).
- **Skill name `figma-to-code-implement`** — kept (parallel naming with `figma-to-code-mapping` outweighs name-similarity with skills.sh `figma-implement-design`).
- **Verify-queue scope** — resolved (v0.3): blocks emit only when the verify-queue item is linked to the **same per-component spec** as the emit-scope. Smallest meaningful blocking scope; broader definitions deadlock the user on every implement run. SKILL.md rule #7 updated.
- **Rule #2 trigger box exact list** — resolved (v0.3): keep current list (*"Build this Figma frame"*, *"Implement this design"*, *"Generate code for [URL]"*, *"Make this component"*) plus add developer-style Figma-MCP invocations *"Use Figma MCP"* and *"Implement this Figma design"* — so devs reach the skill from their normal Figma-MCP workflow. SKILL.md rule #2 trigger box updated.
- **Stand-alone (no-mapping) variant** — resolved: not built. Projects without mapping use [skills.sh `figma-implement-design`](https://skills.sh/figma/mcp-server-guide/figma-implement-design) or bare Figma MCP. README scope-clause clarifies this. Replicating bare-MCP would dilute the strict-mode value proposition.
- **Confirmation gate before B7 emit** — resolved (v0.3): no skill-level confirmation gate. Host environment provides the safety nets — Claude Code's permission-system asks per file-write, and the project's PR-review process catches issues before merge to main. A skill-level halt-and-ask would duplicate those gates without adding safety. SKILL.md B7 makes this explicit.
- **Audit I1: Differential emit op bestaande files** — resolved (v0.4): B7 § File-path determination herzien naar Read-before-write. Bestaande file: prefer extend > replace. Twijfel → halt per regel-range. Voorkomt page-overschrijving regression uit pre-skill testing.
- **Audit I2: Stack-hardcoding refuse-on-mismatch** — resolved (v0.4): rule #3 verhard naar binary halt-on-mismatch. Implement leest `tokens.md § Project styling stack` bij start B4.3, halt direct op tweede styling-API. Voorkomt dual-styling regression uit pre-skill testing.
- **Audit I3: Post-emit screenshot-diff actief** — resolved (v0.4): B8 herzien naar vijf actieve sub-stappen (dev server start, screenshot capture, diff tegen Figma, 7-point check op diff, in-session prompt bij critical mismatch met 3 opties). Procedure-update, geen rule-text bloat.

---

## Process

When resolving an item:
1. Update SKILL.md / README.md / CLAUDE.md as needed in the same PR
2. Move the item from open section to "Resolved decisions" with the resolution noted
3. Reference the resolution PR in this file
4. Bump SKILL.md frontmatter version per CLAUDE.md edit rule #3
