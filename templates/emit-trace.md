# emit-trace template

Skill-internal format that Claude reads at B7 (emit + traceability). Claude fills the placeholders dynamically per emit. Output lands in the commit message and PR body.

Not copied to project repo. Never edited by the user.

---

## The block to produce

```
Implements <Figma-nodeId> per:
- docs/components/<spec-path>.md
- docs/tokens.md  (verdict summary: <X tokens consumed, Y raw-token-available hoisted, Z raw-legitimate hoisted>)
- figma-context/<node-id>.json (hash: <hash-at-emit-time>)

Emitted to: <file-path-or-edited-file>

Component-lookup path: <A direct | B fingerprint | C halt-routed>

Drifts surfaced in scope:
- <drift-row from drifts.md if any — copy verbatim>
(none) if no drifts in scope

Verify-queue items honored in scope:
- <verify-queue items from verify-queue.md if any — copy verbatim>
(none) if no verify items in scope

Pattern adopted (B5, if applicable):
- See pattern-adoption-note for source + reason
(omit if no B5 pattern adoption)
```

## Field rules

- **`<Figma-nodeId>`** — the Figma node ID for the emitted scope (e.g., `16599:2645`). One ID per emit-trace block. If multiple atoms emitted in a single PR (consumable-bites), produce one block per emit.
- **`<spec-path>`** — relative path of consumed per-component spec from project root. List all specs touched (often more than one when atomic-order resolves a Page that consumes Organisms).
- **`<verdict summary>`** — short token-verdict tally; consume from tokens.md column 3.
- **`<hash-at-emit-time>`** — `spec_synced_with_files_hash` from cache at the moment of emit. Required so reviewer can verify cache freshness post-hoc.
- **`<file-path-or-edited-file>`** — exact file path emitted to or edited. For component edits: existing file path. For new pages: user-confirmed path.
- **`<component-lookup path>`** — pick one: `A` (direct mapping match), `B` (fingerprint via mapping-data, user accepted), `C` would have halted (use only if recovered by user re-running with mapping).

## When to omit blocks

- `Drifts surfaced` block → write "(none)" rather than omitting; explicit reassurance.
- `Verify-queue items` block → same.
- `Pattern adopted` block → omit entirely if B5 did not fire; otherwise reference the pattern-adoption-note block.

## Why this matters

The trace lets a code reviewer six months later answer three questions without opening Figma:
1. Where did this code come from? → `Figma-nodeId` + `spec-path`
2. Was the mapping fresh at emit time? → `hash`
3. Did emit honor mapping-recorded drift? → `Drifts surfaced` block

Without the trace, those questions become archaeology.
