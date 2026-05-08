# Proposal — `figma-to-code-implement` skill

Sister skill to `figma-to-code-mapping`. Builds on the mapping-output produced by this skill. **Not yet implemented.** This document captures the design so we don't lose context when we start.

---

## Why a separate skill

`figma-to-code-mapping` (this repo) has scope: **document the relationship between Figma and existing code**. Verbs: document, detect, inventory, mark, link, capture.

What developers also need (per real-world Figma sessions and developer-test feedback): **enforce that relationship at code-emit time**. Verbs: refuse, translate, apply, search-and-adopt, hoist.

Mixing both in one skill caused expectation-failures — developers assumed figma2code would handle implementation; it doesn't. Renaming the mapping skill to `figma-to-code-mapping` (v3.0) signals the half-of-pipeline intent. The other half lives here.

**Pattern:** mapping enables, implementation enforces. Mapping is the data layer; implementation is the policy layer that consumes it.

---

## Scope

### In scope (implement skill)

| Concern | Behavior at emit time |
|---|---|
| **Tokens** | Refuse to emit raw values where mapping has a token-path. If raw is necessary (no matching token), hoist to a page-level CSS variable instead of inlining. Source: `tokens.md` token-verdict column from mapping. |
| **Components** | Refuse to generate a new component when an existing one fits. Use mapping's components.md as authoritative inventory. |
| **Styling stack** | Refuse to introduce parallel styling APIs. Use the API documented in mapping's "Project styling stack" section. No mixing Emotion + className in one element. |
| **Auto-layout** | Translate Figma fill/hug/gap to the project's code expression using mapping's "Auto-layout conventions" table. Preserve responsiveness intent — never emit fixed pixel widths where Figma was fill. |
| **Patterns for non-componentized layouts** | Search the codebase for similar context (other error pages, other detail pages) and adopt the pattern. If no similar context exists, stop and ask user. |
| **Drift surfacing** | Read mapping's drifts.md before emit. Surface drifts as design decisions, not silently resolve. |
| **Verify-queue blockers** | If a needed mapping is in verify-queue (not confirmed), pause and ask user — don't improvise. |
| **Literal strings** | Use mapping's documented `aria-label`, `alt`, `placeholder` values. Don't generate generic ones. |

### Out of scope (implement skill)

- Building Figma frames from text descriptions (that's `figma-generate-design`).
- Fetching Figma data without mapping context (route to `figma-to-code-mapping` first).
- Code Connect mappings (that's `figma-code-connect`).
- Mapping itself — implement skill consumes mapping; it does not produce it.

---

## Workflow sketch (B1-B7, parallel to mapping's A1-A6)

### B1. Mapping presence check

Before any emit: verify mapping output exists for the requested Figma scope. If not, stop and route to `figma-to-code-mapping` first. Do not improvise emit without ground-truth.

### B2. Read mapping output

In order:

1. `tokens.md` — full token table + verdict per row + styling stack + auto-layout conventions
2. `components.md` — atomic-design index, all levels organic
3. Per-component specs for components in scope
4. `drifts.md` — known divergences not yet decided
5. `verify-queue.md` — unconfirmed mappings that block emit

### B3. Live MCP fetch on the target Figma frame

`get_design_context(nodeId)` for the frame to be implemented. Cross-reference returned data against mapping cache; if cache is stale (hash mismatch with code), refresh before proceeding.

### B4. Per-element resolution

For each Figma element in the frame:

1. **Component lookup.** Use master-id or instance-id → components.md → existing code-component. If no match, halt with `component-missing` drift.
2. **Token lookup.** Each Figma value → tokens.md → token-path. If raw-token-available verdict in mapping → emit token, never raw. If raw-legitimate verdict → emit as page-level variable.
3. **Styling-stack adherence.** Emit using the documented API only. No parallel paradigms.
4. **Auto-layout translation.** Apply auto-layout-conventions table to translate fill/hug/gap/direction. Preserve responsiveness.
5. **Literal strings.** Pull aria-label/alt/placeholder from mapping. No generic substitutions.

### B5. Pattern search for non-componentized regions

If the frame contains a layout that is not bound to an existing code-component (e.g., ad-hoc page composition):

1. Search codebase for similar-context files (same route-tree parent, same error/detail/dashboard category).
2. Adopt their pattern (className convention, layout-primitive choice, spacing utilities).
3. No similar context found → halt, ask user. Do not improvise a new pattern parallel to existing conventions.

### B6. Validation pre-emit

8-point check before producing code:

| # | Check |
|---|---|
| 1 | Every value emit-ready: token-path or page-variable, never inline raw |
| 2 | Single styling-API used (no className alongside Emotion, etc.) |
| 3 | Auto-layout primitives translated per conventions table |
| 4 | All components imported from existing locations |
| 5 | Literal strings match mapping (no generic substitutions) |
| 6 | No verify-queue blockers in emit scope |
| 7 | New drift discovered? Surfaced to user, not silently fixed |
| 8 | No new icon packages installed; assets from MCP-localhost or existing |

### B7. Emit

Produce code. Document in commit/PR what mapping sources were consumed (e.g., "Implements 16599:2645 per docs/components/confirmation-modal.md and docs/tokens.md"). Allows traceability back to the mapping ground-truth.

---

## Hard rules sketch (parallel to mapping's eleven)

1. **Read mapping first.** No emit without consulting tokens.md + components.md + relevant per-component specs + drifts.md + verify-queue.md.
2. **Never emit raw where token exists.** If `tokens.md` has a path for the value, use it. If raw is necessary, hoist to page-level variable.
3. **Single styling API.** Use the API documented in mapping's "Project styling stack" section. Refuse to introduce a parallel one.
4. **Translate auto-layout via conventions table.** No fixed pixel widths where Figma is fill/hug.
5. **Consume existing components — never emit a new version.** When a component fits, import it. When none fits → component-missing drift, halt.
6. **Pattern reference for non-componentized layouts.** Search codebase for similar context before introducing new pattern.
7. **Verify-queue blocks emit.** Pause on unconfirmed mappings; ask user.
8. **Never silently resolve drift.** Surface to designer/dev.
9. **Asset discipline.** Existing project assets first, then MCP-localhost. Never new packages, never placeholders.
10. **No write outside emit-scope.** Implement skill only emits component/page/route code in the project's source tree. No test files, no docs, no config — unless user explicitly asks.

---

## Repo and naming

- **Repo name:** `figma2code-implement` (sibling to `figma2code` that hosts mapping skill)
- **Skill name in frontmatter:** `figma-to-code-implement`
- **Slash commands:**
  - `/figma-to-code-implement <Figma-link>` — emit code for the Figma frame using mapping
  - `/figma-to-code-implement check` — pre-emit validation (run B1-B6 without emitting)
- **Symlink convention:** `~/.claude/skills/figma-to-code-implement → ~/Github/figma2code-implement`

---

## Triggers (when to invoke implement skill vs mapping skill)

| User says | Skill |
|---|---|
| "Document this component" | mapping |
| "Map this Figma frame" | mapping |
| "Update tokens for X" | mapping |
| "Build this Figma frame" / "Implement this design" | implement |
| "Generate code for [Figma URL]" | implement |
| Pastes a Figma URL with implementation intent | implement |
| Pastes a Figma URL without intent | ask user — likely mapping if no mapping yet, implement if mapping exists |

---

## Migration path

Before implement skill exists, projects rely on:
- Mapping skill for documentation
- Manual implementation by developer (consuming mapping output mentally)
- Bare Figma MCP for per-request emit (the figma2code old-name behavior — full pipeline mixed)

After implement skill ships:
- Mapping skill still produces mapping output
- Implement skill consumes it for emit
- Bare Figma MCP continues to exist for users who don't run mapping (lower fidelity)

**Boundary with bare MCP:** implement skill REQUIRES mapping to exist. Bare MCP works without mapping but produces lower-quality output (the gap that motivated this whole project).

---

## Risks and unknowns

1. **Search-and-adopt for non-componentized layouts** — pattern-matching across files is non-trivial. May produce false positives (adopting a pattern from an unrelated context). Need heuristics: file-naming similarity, route-path similarity, parent-folder context.
2. **Pattern detection in codebases without conventions** — if the project genuinely has ad-hoc patterns everywhere, "search-and-adopt" finds nothing useful. Implement skill must gracefully halt and ask user, not pick something arbitrary.
3. **Performance** — reading entire `tokens.md`, `components.md`, plus several per-component specs adds context overhead. Consider: implement skill loads only the components in scope of the requested Figma frame.
4. **Verify-queue locks emit** — if many items are in verify-queue, the user gets blocked on every implement attempt. Rule could be relaxed: verify-queue blocks only items in the requested emit scope.
5. **Conflict between mapping and live Figma** — what if Figma changed since mapping was last refreshed? Implement skill should refresh cache (per A4a fallback chain) before emit, not rely on stale mapping.

---

## Status

**Not started.** This document is the entry point when we begin. Living document — update as design questions get resolved.

**Estimated effort to first usable version:** 2-3 design sessions + 1 implementation pass on a real project. ~Same scale as the v3.0 mapping skill is now.

**Owner:** TBD.
