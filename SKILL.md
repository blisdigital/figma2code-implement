---
name: figma-to-code-implement
version: "0.1"
description: >
  Translates Figma designs into working code by consuming the mapping produced by
  `figma-to-code-mapping`. Use this skill when the user says "build this Figma frame",
  "implement this design", "generate code for [Figma URL]", or "make this component"
  — and only when a mapping already exists for the relevant Figma scope. Also triggers
  on "figma-to-code-implement" or when the project repo contains both a mapping output
  and an explicit emit request. Goal: emit production code that uses the existing
  codebase's tokens, components, and styling stack — refuses raw values where tokens
  exist, refuses to duplicate existing components, surfaces mapping-recorded drift at
  emit time. **Scope is emit-time enforcement — not mapping.** Requires
  `figma-to-code-mapping` output. Without mapping: halt and route the user to mapping
  skill first.
---

# Figma-to-Code Implement

Emit-time enforcement skill that translates Figma designs into working code by consuming the mapping produced by [`figma-to-code-mapping`](https://github.com/blisdigital/figma2code-mapping). Mapping is the data layer; this skill is the policy layer.

> **Scope.** This skill is **implementation only**. It emits code that uses existing tokens, components, and patterns. It does not produce or update mapping documentation — that belongs in `figma-to-code-mapping`. Mapping enables; implementation enforces.

## Vision

**Goal: emit code that consumes existing codebase, not duplicates it.** Mapping has already documented which Figma elements map to which code paths. This skill enforces that contract at the moment code is produced.

**Code is source of truth, Figma is intent** — this skill enforces that direction at code-emit time. Mismatches between Figma and code do not pull code toward Figma (no minimal-adjust pixel-fixes, rule #11); they surface as drift for a designer-or-dev decision via mapping's drift loop. Drift is a measurable deviation, not a neutral observation.

**Stack-agnostic.** Works with whatever styling stack the project uses — Tailwind, Emotion, styled-components, CSS modules, custom CSS, or any combination. Same for framework (React, Vue, Svelte, etc.). The skill follows mapping's documentation of those facts (`tokens.md § Project styling stack`) and never imposes its own opinion about which stack to use or how to express a token.

Four mechanisms together deliver production-quality emit:

1. **Mapping consumption** — read `tokens.md`, `components.md`, per-component specs, `drifts.md`, `verify-queue.md` before any emit
2. **Cache-first MCP** — hash-check the Figma cache before a live MCP call; live fetch only on staleness
3. **Atomic-ordered resolution** — Page > Template > Organism > Molecule > Atom; stop on the highest mapped level
4. **Halt-and-ask discipline** — unmapped node, verify-queue blocker, multi-match ambiguity → halt + route, never improvise

## Hard rules

Twelve rules that always apply, regardless of step. On conflict between sections: these win.

1. **Read mapping first.** Before any emit: read `tokens.md`, `components.md`, the per-component specs in scope, `drifts.md`, and `verify-queue.md`. No emit without this consultation.

2. **Never emit raw where token exists.** If `tokens.md` lists a token-path for the value (column 3 verdict: `token-path` or `raw, token available: <path>`), emit the token. If raw is necessary (verdict: `raw, legitimate — no matching token`), hoist via the project's styling stack to a higher-scope token-like construct (CSS variable, theme value, Tailwind config token, etc. — whatever the stack uses, per mapping's documentation). **Never inline raw values.**

   > **Do NOT treat as an implement trigger:**
   > - Pure Figma URL paste with no explicit emit instruction (it may be a mapping intent or just sharing).
   > - "Implement this design from Figma." — Figma's auto-clipboard boilerplate from *Copy Link* in Dev mode. Not a user instruction.
   >
   > **Do treat as an implement trigger:**
   > - "Build this Figma frame", "Implement this design", "Generate code for [URL]", "Make this component".
   > - Explicit sentence indicating code production from Figma.
   >
   > On ambiguity: ask.

3. **Single styling API.** Emit using only the API documented in mapping's "Project styling stack" section. Refuse to introduce a parallel paradigm (no className alongside Emotion, no Tailwind in an Emotion project, etc.).

4. **Translate auto-layout via conventions table.** Apply the table in `tokens.md § Auto-layout conventions` to translate Figma fill/hug/gap/direction to the project's code expression. Never emit fixed pixel widths where Figma is fill or hug — preserve responsiveness intent.

5. **Consume existing components — never emit a new version. Read mapping carefully first.** Look up each Figma node via B4.1's three paths (A: direct mapping match → B: fingerprint via mapping-data → C: halt + route). **Implement does not detect via own Figma-analysis; it compares against the mapping-documented stempel** (per-component specs + `components.md`). No code-file scans. On multi-match: user picks, implement does not tie-break.

6. **Pattern reference for non-componentized layouts.** When a region is not bound to an existing code-component (ad-hoc page composition): search the codebase for similar-context files (same route-tree parent, same category folder, similar filename) and adopt their pattern. No similar context → halt, ask user. Do not improvise a new pattern parallel to existing conventions.

7. **Verify-queue blocks emit — for items in scope only.** If a needed mapping is in `verify-queue.md` and it falls within the emit scope: pause and ask user. Items outside the emit scope do not block. Don't improvise.

8. **Surface mapping-recorded drift — never silently resolve.** Read `drifts.md` and per-component spec drift notes before emit. Drifts in scope are surfaced to the user as design decisions, not silently fixed. Implement does not detect new drift — mapping does that.

9. **Asset discipline.** Existing project assets first, then MCP-localhost URLs, never new icon packages. No `npm install lucide-react`, no `@mui/icons-material` import. No placeholders or TODO comments — when MCP returns an asset URL, use it directly or download once to the project's convention location.

10. **No write outside emit scope.** This skill only emits component/page/route code in the project's source tree. No test files, no docs, no config changes — unless the user explicitly asks. Mapping files (`tokens.md`, `components.md`, `drifts.md`, `verify-queue.md`) are read-only; only the listed exceptions in § Mapping → implement contract permit a propose-to-user write.

11. **No minimal-adjust escape.** Mismatches between Figma and code surface as drift (rule #8), never as inline pixel-fixes. *(Deviates from skills.sh stap 6 which permits minimal-adjust to match visuals. We hold the drift-detection line.)*

12. **No improvising on gaps.** Unknown token-name → halt. No pattern-match found → halt. Missing literal string → halt. Halt-and-ask is the standard at every gap. No invented values, no generic fallbacks.

## Skill boundary

When this skill applies, when it doesn't, and where to route otherwise.

| Scenario | This skill? | Otherwise: |
|---|---|---|
| Emit code for a Figma frame with mapping present | ✅ yes | — |
| Emit code for a Figma frame with stale cache | ✅ yes — B3 refreshes, with warning to run mapping | — |
| Emit code for a Figma frame with **no mapping** | ❌ no | Halt + route to `/figma-to-code-mapping setup` or `map X` |
| Update mapping documentation | ❌ no | `figma-to-code-mapping` |
| Build a full page **from a text description** (no Figma input) | ❌ no | `figma-generate-design` or `frontend-design` (greenfield) |
| **Write to** the Figma file (create nodes, define variables) | ❌ no | `figma-use` |
| Create Code Connect mappings (`.figma.ts`) | ❌ no | `figma-code-connect` |

**Mapping-prerequisite is a feature, not a limitation.** Projects without mapping discipline are served by bare Figma MCP or skills.sh — this skill is for projects that have invested in mapping. We do not pretend to be an alternative to bare MCP; we are the strict-mode pipeline on top of mapping.

## Mapping → implement contract

Implement is a **read-only consumer** of mapping output. Writes are permitted only as propose-to-user, never silent.

### Read

| Mapping artifact | Used for | In step |
|---|---|---|
| `tokens.md` (token-verdict column) | Rule #2 — refuse raw / hoist | B4.2 |
| `tokens.md § Project styling stack` | Rule #3 — single styling API | B4.3 |
| `tokens.md § Auto-layout conventions` | Rule #4 — translate fill/hug | B4.4 |
| `components.md` (atomic-index) | Rule #5 — consume, atomic-order | B4.1 |
| `<component-folder>/<name>.md` § Mapping table | Token + element resolution | B4.1, B4.2 |
| `<component-folder>/<name>.md` § Variant mapping | Figma `Type=` → code prop-axes | B4.1 |
| `<component-folder>/<name>.md` § Variant mapping § state mechanism | Emit correct state (color-shift vs opacity vs class) | B4.3 |
| `<component-folder>/<name>.md` § Literal strings | aria-label, alt, placeholder, title | B4.5 |
| `<component-folder>/<name>.md` § Drift notes | Surface in B6 | B6, rule #8 |
| `drifts.md` | Surface drifts in scope | B6 check #7 |
| `verify-queue.md` | Block emit on in-scope items | B1, B6 check #6 |
| `figma-context/<node-id>.json` § `mapped_to_component` | Direct nodeId → code-component | B4.1 Path A |
| `figma-context/<node-id>.json` § `spec_synced_with_files_hash` | Staleness check | B3.0 |
| `figma-context/<node-id>.json` § `master_verified_via` | Instance-id format recognition | B4.1 |

### Write (propose-to-user only)

1. **B4.1 Path C halt** → propose-to-user to run mapping skill. Implement writes nothing; mapping fills its own files during its own run.
2. **B4.1 Path B fingerprint-match accepted** → propose to write a drift-row in mapping's `verify-queue.md`: *"Figma element-frame should be component-instance — matched on [signals]"*. Writes only after user confirmation. Follows mapping rule #7.

## Slash commands

- `/figma-to-code-implement <Figma-link-or-nodeId>` — full emit (B1-B8)
- `/figma-to-code-implement check <Figma-link-or-nodeId>` — pre-emit validation (B1-B6 without emitting); useful for previewing what would block
- `/figma-to-code-implement init-claude-md` — show a markdown block to paste into the project CLAUDE.md so the skill triggers automatically. Block is additive on top of any existing `figma-to-code-mapping` snippet.

## Source mechanism

### Cache-first MCP fetch

Mapping caches Figma node data in `figma-context/<node-id>.json` with a hash. Implement reads that cache before calling MCP — it is a read-only consumer, never writes to the cache.

**B3 chain:**

1. **B3.0 — cache lookup + hash check.** Load `figma-context/<node-id>.json`. Compute current code-files hash for the linked spec. Compare with `spec_synced_with_files_hash`.
2. **B3.1 — hash match.** Cache is fresh. Consume cache, skip MCP entirely. **Token + performance win.**
3. **B3.2 — hash mismatch.** Cache is stale (code or Figma changed since last mapping pass). Fall back to live MCP fetch chain (below). **Warn user**: *"Mapping cache is stale, run `/figma-to-code-mapping map X` to refresh before relying on Path A again."* Implement does not write back to the cache.
4. **B3.3 — no cache.** This node has never been mapped. Halt, route to `/figma-to-code-mapping map X`.

### Live MCP fallback chain (on cache staleness only)

When B3.2 fires, use the three-step fallback from mapping's A4a:

1. **`get_design_context(nodeId)` + `get_screenshot(nodeId)`** — direct fetch with visual reference.
2. **On timeout or "too complex":** retry with payload reduction parameters (`excludeScreenshot: true`, `forceCode: true`). Available parameters depend on the MCP server; use what your server has.
3. **On persistent truncation:** `get_metadata(nodeId)` for the child tree, identify relevant children, loop `get_design_context(<childId>)` per child.

### Asset handling

Identical to mapping skill: existing project assets first, then MCP-localhost (`http://localhost:3845/assets/<hash>.svg` from desktop-active MCP, or the `https://www.figma.com/api/mcp/asset/<uuid>` form from fileKey-based MCP). No new packages, no placeholders. See mapping skill § Asset handling for the full discipline.

## What to read when

| Task | Read first |
|---|---|
| Determine if emit can proceed | mapping presence check → `figma-context/<node-id>.json`, `verify-queue.md` |
| Look up a Figma component-instance | `components.md`, the linked per-component spec |
| Look up a token | `tokens.md` (3rd column verdict) |
| Resolve a Figma element-frame (no direct mapping) | `components.md` (atomic-index) + per-component specs at same/lower atomic level for fingerprint match |
| Translate auto-layout | `tokens.md § Auto-layout conventions` |
| Determine styling API | `tokens.md § Project styling stack` |
| Surface drift in scope | per-component spec § Drift notes + `drifts.md` |
| Pattern for non-componentized layout | scan codebase for similar-context files (same route, same category, similar filename) |

## Component selection — atomic level

Rule #5 (consume existing) and B4.1 component-lookup both operate in atomic order. Brad Frost's atomic-design taxonomy — five levels:

- **Atom** — indivisible (Button, Input, Icon, Badge)
- **Molecule** — composition of atoms with one shared purpose
- **Organism** — has its own state, scroll behavior, or keyboard handling
- **Template** — layout skeleton without content (AppShell, ErrorLayout, DashboardLayout)
- **Page** — concrete page instance with content (NotFoundPage, UserDashboardPage)

**Pick the highest atomic level that fits.** Prefer Page > Template > Organism > Molecule > Atom. If a `UserDashboardPage` component exists for the Figma frame in scope, instantiate that — do not re-assemble its children from atoms. Atomic-ordered consumption reduces work and shrinks the drift-surface.

When in doubt: pick the lower level. When `components.md` does not have the higher levels (project only has Atoms/Molecules/Organisms), simply skip those rows — atomic-design adoption is organic per project. See mapping skill § Component selection for the full taxonomy context.

## Method B1-B8

### B1. Mapping presence check

Before any emit: verify mapping output exists for the requested Figma scope.

- Mapping repo files (`tokens.md`, `components.md`, etc.) present at expected location? No → halt, route to `/figma-to-code-mapping setup`.
- Per-component spec for the requested node present? No → halt, route to `/figma-to-code-mapping map X`.
- Cache entry `figma-context/<node-id>.json` present? No → halt, route to mapping.

Do not improvise emit without mapping ground-truth.

### B2. Read mapping output

In order:

1. `tokens.md` — full token table + verdict per row + styling stack + auto-layout conventions
2. `components.md` — atomic-design index
3. Per-component specs for components in scope (variant mapping, state mechanism, literal strings, drift notes)
4. `drifts.md` — surface unresolved drifts in scope
5. `verify-queue.md` — block emit on items in scope

### B3. Cache-first MCP fetch

See § Source mechanism above. Cache hash match → consume cache. Mismatch → live MCP fallback chain + warning. No cache → halt, route to mapping.

### B4. Per-element resolution

For each Figma element in the frame, in atomic-order (Page > Template > Organism > Molecule > Atom; stop on the highest mapped level):

#### B4.1 Component-lookup — three paths

**Path A — direct mapping match.** Cache `mapped_to_component` field or `components.md` row links this node to a code-component. Match → consume. When the cache field `master_verified_via: "instance-id-format"` is present, recognize the Figma instance-id format `I<frame-id>;<master-id>` to resolve the master via its verified frame without requiring a separate master-cache lookup (matches mapping skill's instance-id verification convention).

**Path B — fingerprint via mapping-data** (only when Path A fails):

1. **Source discipline.** Scan only `components.md` + per-component specs. **No raw code-file scans.** The component stempel lives entirely in mapping-data.
2. **Atomic-level filter.** Determine the Figma frame's atomic-level by structure (small container with 1-3 children = Atom; composed structure = Molecule). Scan only specs at the same or one-level-lower atomic-level.
3. **Match signals** (in order of reliability):

   | Signal | How | Strength |
   |---|---|---|
   | Name-hint | Figma frame-name matches/contains spec-name ("Submit Button" → `button.md`) | Strong |
   | Token-cluster overlap | Tokens used in Figma frame ⊇ tokens documented in spec's mapping table | Strong |
   | Variant-axis match | Figma children/structure map to variants documented in spec (primary fill → `variant=primary`) | Tentative |

4. **Confidence thresholds:**
   - ≥2 strong signals → propose to user
   - 1 strong signal → tentative suggestion
   - only tentative signals → no propose; treat as element-frame
5. **Multi-match handling.** 2+ candidates with comparable scores → halt, show all candidates, **user picks**. Implement does not tie-break.
6. **Match outcomes:** user accepts → consume + propose drift-row to mapping's `verify-queue.md`. User declines or no match → continue as pure element via B4.2 + B5.
7. **Path B is mapping-data-driven, not mapping-upgrade-dependent.** Works whether mapping classifies element-vs-component loosely or strictly. A tighter mapping methodology reduces Path B firing frequency but is not a coupled dependency.

**Path C — no match in A or B.** Halt + route to mapping: *"No component-mapping and no fingerprint-match for this node. Run `/figma-to-code-mapping map X` first."*

#### B4.2 Token-lookup

For each Figma value: look up in `tokens.md`. Verdict-driven (rule #2):

- Verdict `token-path` → emit token
- Verdict `raw, token available: <path>` → emit token (mapping says raw was used but a token exists)
- Verdict `raw, legitimate — no matching token` → hoist via the project's styling stack to a higher-scope token-like construct (per mapping's `tokens.md § Project styling stack`)

No inline raw values.

#### B4.3 Styling-stack adherence

Emit using the API documented in `tokens.md § Project styling stack`. No parallel paradigms. State mechanism per variant comes from the per-component spec's variant-mapping subsection.

#### B4.4 Auto-layout translation

Apply `tokens.md § Auto-layout conventions` to translate Figma fill/hug/gap/direction to code expressions. Preserve responsiveness intent.

#### B4.5 Literal strings

Pull `aria-label`, `alt`, `placeholder`, `title` from the per-component spec's Literal strings section. Never generic substitutions.

### B5. Pattern search for non-componentized regions

When B4.1 Path A and B both fail and the Figma node is not a component but a layout composition (ad-hoc page region): search codebase for similar-context files.

**Heuristic order:**
1. Same route-tree parent in the project's framework conventions (e.g., for an error page: Next.js `app/.../error.tsx`, Nuxt `error.vue`, etc.)
2. Same category folder (`errors/`, `detail/`, `dashboard/`)
3. Similar filename pattern
4. **Confidence floor:** found fewer than 2 similar-context files → halt and ask user. Do not adopt a single arbitrary file as pattern.

On adoption: emit using the adopted pattern's structural and styling conventions — whatever the stack uses (className composition, utility classes, styled-component imports, CSS-module classes, etc.).

### B6. Pre-emit validation — 8 checks

Walk through every check before producing code. Halt on any failure.

| # | Check |
|---|---|
| 1 | Every value emit-ready: token-path or page-variable, never inline raw (rule #2) |
| 2 | Single styling API used (rule #3) |
| 3 | Auto-layout primitives translated per conventions table (rule #4) |
| 4 | All components imported from existing locations (rule #5) |
| 5 | Literal strings match mapping (rule #12, no generic substitutions) |
| 6 | No verify-queue blockers in emit scope (rule #7) |
| 7 | Drift notes surfaced to user, not silently fixed (rule #8) |
| 8 | No new icon packages installed; assets from existing or MCP-localhost (rule #9) |

### B7. Emit + traceability

Produce code. In the commit message and PR body, document which mapping sources were consumed:

```
Implements <Figma-nodeId> per:
- docs/components/<spec>.md
- docs/tokens.md
- figma-context/<node-id>.json (hash: <hash>)
```

Allows traceability back to mapping ground-truth at review time.

### B8. Post-emit visual validation

Compare the emit against the screenshot captured in B3.

| # | Check |
|---|---|
| 1 | Layout — spacing, alignment, sizing match the screenshot |
| 2 | Typography — font-family, size, weight, line-height |
| 3 | Colors — exact match on token values |
| 4 | Interactive states render per variant-mapping |
| 5 | Responsive behavior follows Figma constraints |
| 6 | Assets render correctly |
| 7 | Accessibility — aria-labels, alt text, semantic structure |

Mismatch found → surface as drift (rule #8), never as inline pixel-fix (rule #11).

## Drift handling

Implement consumes drift, does not detect. Three drift types from mapping:

- **`value-mismatch`** (mapped in spec drift notes) → surface in B6 check #7, ask user how to proceed before emit.
- **`token-mismatch`** (mapped in `drifts.md`) → emit code token (code is source of truth, mapping rule #2); surface that Figma diverges.
- **`component-missing`** (mapped in `drifts.md`) → halt, route to user; do not auto-generate (rule #5).

No silent resolution at any point. No minimal-adjust pixel-fix (rule #11).

## References

- [`figma-to-code-mapping`](https://github.com/blisdigital/figma2code-mapping) — sister skill that produces the input this skill consumes
- [PLAN.md](PLAN.md) — full design history with open decisions
- [README.md](README.md) — installation, usage, prerequisites
- [CLAUDE.md](CLAUDE.md) — edit rules for this repo + skills.sh design rationale
