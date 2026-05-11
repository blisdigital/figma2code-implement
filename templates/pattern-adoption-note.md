# Pattern-adoption note (B5)

Skill-internal format that Claude reads at B5 when a non-componentized layout requires pattern-search. Output appears in the emit-trace block of the commit message (see `emit-trace.md`).

Not copied to project repo. Never edited by the user.

---

## When B5 fires

B5 triggers only when:
- B4.1 Path A produced no match (no direct component mapping)
- B4.1 Path B produced no match (no fingerprint match via mapping-data)
- The Figma node is a layout composition (ad-hoc page region), not a component candidate

If B5 confirms a similar-context pattern → produce the adoption note. If B5 halts (confidence floor not met) → produce the halt note.

---

## On successful adoption (≥2 similar-context files found)

```
Pattern adopted (B5):
- Source: <path/to/exemplar-file>
- Heuristic match: <route-tree parent | category folder | filename similarity>
- Adopted: <what was concretely taken — structural conventions, styling patterns, layout primitives>
- Confidence: <N> similar-context files found (≥2 required)
- Other candidates considered: <list of the other N-1 files, briefly>
```

### Field rules

- **`<Source>`** — relative path of the file whose pattern was adopted as exemplar. One primary source even when multiple similar files exist.
- **`<Heuristic match>`** — which B5 heuristic level fired (1: same route-tree parent, 2: same category folder, 3: filename similarity). Higher level = stronger match.
- **`<Adopted>`** — describe concretely: e.g., "header-content-footer structure", "max-width wrapper + padded inner", "specific layout primitives used", "naming convention X for child elements".
- **`<Confidence>`** — exact count of similar-context files matched. ≥2 required per B5 confidence floor.
- **`<Other candidates>`** — short list so the reviewer can verify the chosen exemplar is representative, not arbitrary.

---

## On halt (<2 similar-context files found)

```
Pattern search inconclusive (B5):
- Similar-context files found: <N> (need ≥2)
- Heuristic levels exhausted: <route-tree | category | filename>
- Halting per B5 confidence floor

Resolution required: user specifies pattern source OR accepts that no pattern adoption applies — emit will then follow only tokens + auto-layout conventions without inherited structural pattern.
```

This halt-message goes to the user, not to commit (no emit yet).

---

## Why this matters

B5 pattern-search is the most failure-prone step in the workflow: false positives (adopting a pattern from unrelated context) silently degrade output quality. The note serves two purposes:

1. **Traceability** — reviewer can verify "did the skill adopt the right pattern?" by inspecting the source path and heuristic match.
2. **Drift visibility** — if multiple emits keep adopting the same wrong pattern, that surfaces a missing component in the codebase (organism-level extraction opportunity).

## What this is NOT

- **Not a code-generator.** B5 informs how to compose; B4.2-B4.5 still resolve tokens / styling / auto-layout / literal strings on top.
- **Not a fallback for missing mapping.** If B4.1 Path A and B both fail, that's already a halt-and-route case for component-instances. B5 only applies when the Figma node is *legitimately* a layout composition, not a missed component.
- **Not a similarity-scorer.** B5 uses categorical heuristics (route-tree, category, filename), not statistical similarity. Cheap and deterministic.
