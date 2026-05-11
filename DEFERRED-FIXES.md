# Deferred fixes — `figma-to-code-implement`

> Open decisions and design questions that have surfaced during the design conversation
> but are not yet resolved. Parallel to mapping skill's `DEFERRED-FIXES.md`.
>
> **Three tiers:**
> - **🔴 Rule-text-determining** — affects SKILL.md hard rules. Must land before v1.0 rule-tuning.
> - **🟡 Implementation-detail** — affects how a rule operates but not the rule text. Can land during SKILL.md iterations.
> - **🟢 Nice-to-have** — quality-of-life upgrade. Optional, no ship dependency.

Items tracked here have a proposal where one exists; if you decide differently, update the proposal field and bump the relevant section of SKILL.md in the same PR.

---

## 🔴 1. Verify-queue scope — block all emits or only emits with in-scope items?

**Status:** open, proposal documented in SKILL.md rule #7 as "in-scope only".

**Context:** Mapping's `verify-queue.md` holds unconfirmed mappings waiting for the next live-MCP session. If many items pile up, a strict "any verify-queue item blocks any emit" rule deadlocks the user on every implement attempt.

**Proposal (in current rule #7):** verify-queue items block emit *only* if they fall within the emit scope. Items outside scope are noise for this emit.

**Open:** confirm the scope definition. Is "in scope" = same per-component spec? Same atomic-level branch? Same Figma frame tree? Different answers produce different blocking volumes.

**Impact if changed:** rule #7 phrasing, B1 + B6 check #6.

---

## 🔴 2. Rule #2 trigger box — exact list of implement-intent vs auto-clipboard ignores

**Status:** open, proposal documented in SKILL.md rule #2 box.

**Context:** Figma's *Copy Link in Dev mode* puts boilerplate text on the clipboard (*"Implement this design from Figma."*). That sentence must not trigger the skill — it's an artifact, not user intent. Meanwhile, a user typing *"implement this"* is a legitimate trigger.

**Current proposal:**
- **Do treat as trigger:** "build this Figma frame", "implement this design", "generate code for [URL]", "make this component"
- **Do NOT treat as trigger:** pure Figma URL paste, "Implement this design from Figma." (auto-clipboard boilerplate)

**Open:** is this list complete? Does it overlap with skills.sh `figma-implement-design` triggers (which fire on "implement design", "generate code", "implement component")? Need to confirm that overlap doesn't cause dual-triggering when both skills are installed.

**Coupled with:** mapping coordination edit #2 in [mapping#6](https://github.com/blisdigital/figma2code-mapping/pull/6). Both skills must use the same trigger list to avoid divergence.

**Impact if changed:** SKILL.md rule #2 box, frontmatter description, mapping skill rule #2 box.

---

## 🔴 3. Confirmation gate before B7 emit — atom direct vs organism+ gated?

**Status:** open, no current proposal.

**Context:** Mapping rule #7 (*"Ask for confirmation before code or doc changes"*) is a strong norm in the sister skill. Implement, by definition, writes code at B7. A blanket halt-and-ask before every emit creates friction for trivial atom-level emits. No gate at all risks writing large amounts of code without review.

**Three options:**
- **A — Always confirm.** Safest, most friction. Every B7 → user OK before commit.
- **B — Gated by atomic-level.** Atoms/molecules direct emit (B6 is the safety net). Organisms+ require user OK. Reasonable balance.
- **C — Never confirm.** Trust B6's 8-point check fully. Maximum velocity, no safety brake.

**Preliminary leaning:** option B. Matches mapping's "ask before code changes" spirit while not blocking on trivial cases.

**Impact if changed:** B7 description, new rule #13 (if a confirmation rule is added explicitly).

---

## 🟡 4. Cache hash format — what exactly is hashed?

**Status:** open.

**Context:** B3.0 hash-checks the cache against current code-files. Mapping skill writes `spec_synced_with_files_hash` as `sha256:<hex>` (line 113-123). Implement reads this. The exact hash input is undocumented.

**Options:**
- Single file hash (the spec file only)
- Combined hash (spec file + linked component file + dependencies)
- Tree hash (entire component folder)

**Coupled with:** mapping coordination edit #4 in [mapping#6](https://github.com/blisdigital/figma2code-mapping/pull/6) — cache schema formalisation should define this precisely.

**Impact if changed:** B3.0 logic + mapping cache write logic. Coordinated change.

---

## 🟡 5. Hard halt vs `--force` flag

**Status:** open.

**Context:** Three rules halt the emit unconditionally — #5 (component-missing), #7 (verify-queue blocker), #8 (drift in scope). In practice users may want a `--force` escape hatch for known-acceptable cases (e.g., "I know this verify-queue item, emit anyway").

**Two options:**
- **A — No force flag.** Halts are unconditional. Users must resolve in mapping first. Cleaner, slower.
- **B — Allow `--force` on slash-command.** `/figma-to-code-implement --force <link>` skips halts. Faster, risks misuse.

**Open:** worth the implementation complexity? Or document the manual workaround (edit the verify-queue item, then re-run)?

**Impact if changed:** slash command parsing, halt logic in B1 / B4 / B6.

---

## 🟡 6. B5 pattern-search confidence floor

**Status:** open, proposal in SKILL.md B5.

**Context:** B5 pattern search adopts a layout pattern from "similar-context" files. If fewer than 2 similar-context files exist, current proposal is halt-and-ask. The "2" is intuition, not calibrated.

**Open:** does "2" scale with project size? On large projects, "2" is too low (many incidental matches). On small projects, "2" is unreachable and B5 always halts. Possibly should scale (e.g., `>= max(2, ceil(total-matching-context-files / 10))`).

**Impact if changed:** B5 description + heuristic order.

**Calibration plan:** decide during Fase 5 dogfood on a real project.

---

## 🟢 7. Mapping-side classification methodology — suggestion for mapping repo

**Status:** explicit suggestion, **not coupled to implement v0.1 ship.**

**Context:** Path B (fingerprint via mapping-data) works gracefully whether mapping classifies element-vs-component loosely or strictly. A tighter mapping methodology (explicit `element-frame` vs `component-instance` vs `component-missing` classification + verification step) would reduce how often Path B fires.

**Where it lives:** mapping repo's own roadmap, not this repo's. Captured here for traceability — if mapping adds this, implement's Path B becomes near-redundant.

**Coupled with:** mapping repo upgrade (open). Implement should not block on this.

---

## 🟢 8. Per-spec component fingerprint section

**Status:** suggestion for mapping repo.

**Context:** Path B currently scans full per-component specs for matching signals. A condensed "fingerprint" sub-section per spec (token-stempel + variant axes + naming hints in a single block) would make Path B scans cheaper and more deterministic.

**Where it lives:** mapping repo template upgrade.

**Coupled with:** mapping repo, no implement dependency.

---

## Resolved decisions — moved to history

For traceability, decisions that were made during design but later closed:

- **Partial mapping handling** — resolved: halt on first ungated Figma node concretely in scope, route to mapping (PLAN.md). Atomic-order applies. Element-frames without component-equivalent are legitimate via tokens + B5 pattern.
- **Deviation comment in code** — schrapt: redundant with B7 commit traceability and drift-acceptance trail in mapping's `drifts.md`. No in-code comments per accepted drift.
- **Implement fingerprint detection on Figma-side tokens** — schrapt in favor of mapping-data driven matching (Path B uses per-component specs as fingerprint source, no Figma-side detection layer in implement).
- **Implement writes to mapping `drifts.md` / `verify-queue.md`** — schrapt as default; only two narrow propose-to-user exceptions remain (B4.1 Path B accepted → `verify-queue.md`, B4.1 Path C → propose-route-to-mapping without writing).
- **Skill name `figma-to-code-implement`** — kept (parallel naming with `figma-to-code-mapping` outweighs name-similarity with skills.sh `figma-implement-design`).

---

## Process

When resolving an item:
1. Update SKILL.md / README.md / CLAUDE.md as needed in the same PR
2. Move the item from open section to "Resolved decisions" with the resolution noted
3. Reference the resolution PR in this file
4. Bump SKILL.md frontmatter version per CLAUDE.md edit rule #3
