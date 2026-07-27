# Goals — measurable objectives per improvement iteration

> Working method (agreed 2026-07-27): every improvement iteration starts from **jointly agreed, measurable objectives** — extracted from an article, lesson, or dogfood run — before any SKILL.md edit lands. A goal is fixed *before* its first test run; results append to the run log; lessons go to [LESSONS.md](LESSONS.md). Test-emit code itself is throwaway — the scorecard result is the artifact.

---

## Goal 1 — 404 fixture: 0/6 historical failure modes

**Origin:** Orbit (Polar) article mining (2026-07) × LESSONS 2026-05-08 (six failure modes from pre-skill internal testing). The article contributes the method — every objective is scored by a deterministic check, not by self-assessment ("Docs are a suggestion. CI is a contract."). The lessons contribute the test cases — real failures observed on this exact fixture.

**Goal statement:** On the 404-page fixture, an implement-pass with the current skill reproduces **0 of the 6** historically observed failure modes, scored by the deterministic checks below. Objectives O0–O4 gate; O5 is tracked (known-fail pending mapping fluidity-intent, DEFERRED-FIXES 🟢 #6).

**Dropped:** a suppression-tracking objective (Orbit's `eslint-disable`-as-bug) — zero support in LESSONS history; parked as future watch-item.

### Fixture

| Fact | Value |
|---|---|
| Figma node | `14961:2761` — "Pagina niet beschikbaar" (404), file `RFmWghsnS3kQn5fDZwZw58` (eBlinqx-WorQX) |
| Project | `~/Github/WorQX/Figma-to-code/` |
| Mapping | `figma-to-code-mapping/` (tokens.md, components.md, drifts.md, verify-queue.md, 6 per-component specs) |
| MCP cache | `figma-mcp-context/` — **no entry for 14961:2761 at baseline** |
| Codebase | `codebase/worqx-codebase-2026-04-28/` — React + Emotion + MUI theme (`src/theme/tokens.ts`) |
| Pre-existing target | `src/pages/error-pages.tsx` — the file the pre-skill test destroyed |
| Known schema gap | fixture `tokens.md` predates current mapping schema: no `§ Project styling stack`, no `§ Auto-layout conventions` (stack implied in prose). Not pre-fixed — gaps the skill hits are data. |

### Protocol

1. **Phase 0 — unmapped-node halt.** Request emit for `14961:2761` with no mapping present. Scores O0.
2. **Fixture prep (unscored).** Run `figma-to-code-mapping` to map the 404 node (per-component spec + cache entry). This is the pipeline as designed, not part of implement's score.
3. **Phase 1 — mapped emit.** Full implement-pass (B1–B8). Scores O1–O5.
4. **Teardown.** Score, append to run log, write LESSONS entries, `git restore` the WorQX test emit.

### Scorecard

| # | Objective | Gate? | Historical failure (LESSONS 2026-05-08) | Deterministic check |
|---|---|---|---|---|
| O0 | Halt on unmapped node — no improvised emit | gate | bare MCP emitted without mapping ground-truth | Phase 0: skill halts + routes to mapping; `git status` in WorQX shows **0 files written** |
| O1 | Zero raw style values | gate | raw CSS props without consulting tokens; random colors where tokens existed | on the emit diff: color literals (`#hex`, `rgb(`, `hsl(`) = **0**; every numeric px literal either matches a `raw, legitimate` verdict in tokens.md or is hoisted — unexplained px = fail |
| O2 | Single styling surface | gate | `className` alongside Emotion | on the emit diff: `className=`, `style={{`, Tailwind-utility strings = **0** occurrences |
| O3 | Literal fidelity to mapping | gate | generic "probeer opnieuw" instead of documented copy | every user-facing string in the emit matches spec § Literal strings exactly; mismatches = **0** |
| O4 | Non-destructive, in-scope writes | gate | existing 404 page deleted + replaced; cache file committed to git | `git diff` in WorQX: no deleted exports/lines in pre-existing code beyond the intended edit-range; changed files = intended target(s) only; nothing staged from `figma-mcp-context/` |
| O5 | Visual fidelity deltas | tracked | emit filled ~40% of viewport instead of 100vw | B8 7-point diff: count ✗/⚠ deltas. Viewport-fill ✗ is **expected** until mapping documents fluidity-intent — track, don't gate |

### Run log

| Date | Skill version | O0 | O1 | O2 | O3 | O4 | O5 (✗/⚠) | Notes |
|---|---|---|---|---|---|---|---|---|
| _pre-skill_ | none (bare MCP / mapping-only) | ✗ | ✗ | ✗ | ✗ | ✗ | n/a | the 2026-05-08 baseline: 6/6 failure modes observed |
