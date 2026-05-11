# Post-emit visual check (B8)

Skill-internal format that Claude walks at B8 after emit. Compares produced code against the screenshot captured in B3 (or refreshed at emit time).

Mismatch found → surface as drift (rule #8). **Never** apply inline pixel-fix (rule #11).

Not copied to project repo. Never edited by the user.

---

## The 7-point check

For each item: mark ✅ match or ❌ mismatch with a one-line note. Capture the comparison evidence (which element, what differs, by how much).

| # | Check | Status | Note |
|---|---|---|---|
| 1 | Layout — spacing, alignment, sizing match the screenshot | [ ] | |
| 2 | Typography — font-family, size, weight, line-height | [ ] | |
| 3 | Colors — exact match on token values consumed | [ ] | |
| 4 | Interactive states render per per-component spec variant-mapping | [ ] | |
| 5 | Responsive behavior follows Figma constraints (fill/hug per auto-layout) | [ ] | |
| 6 | Assets render correctly — icons, images, SVGs | [ ] | |
| 7 | Accessibility — aria-labels, alt text, semantic structure match per-component spec literal-strings | [ ] | |

## Compare against

Primary reference: `get_screenshot(<nodeId>)` from B3.
Secondary reference: per-component spec § Mapping table (for state mechanism, variant prop wiring).

## Output format

Full table reported. Then a summary block:

**On full match (0 mismatches):**
```
✅ B8 visual validation passed. <N>/7 checks confirmed.
```

**On mismatch (1+ mismatches):**
```
⚠️ B8 visual validation: <X> mismatch(es) detected.

Mismatches:
- #<check-N>: <specific divergence, e.g. "padding 16px in code vs 18px in screenshot">

Resolution: surface as drift per rule #8. Do NOT apply inline pixel-fix (rule #11).

Next step: user decides — accept code value (figma updates needed) or update mapping
(run /figma-to-code-mapping map X to reflect screenshot reality and re-emit).
```

## Why NOT auto-fix

Rule #11 explicitly forbids minimal-adjust pixel-fixes. Reason: pulling code toward Figma on every mismatch defeats the drift-detection loop that mapping invests in. Mapping marks drifts as decision points with severity + owner — pixel-fixes silently resolve those without owner involvement.

The implement skill surfaces; the mapping skill decides; the developer or designer resolves.

## What this is NOT

- **Not a drift-detection step in mapping's sense.** Mapping's drift test fires on every mapping pass. B8 fires only at emit time and only on screenshot vs. code comparison.
- **Not a re-run of B6.** B6 is pre-emit gate over mapping-data. B8 is post-emit visual proof.
- **Not a test runner.** No interaction testing, no accessibility audit. Visual + structural only.
