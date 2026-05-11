# Lessons learned

Record what worked and what did not during the skill's evolution. Each entry teaches the skill about itself — confirmations to deliberately repeat, corrections to fix.

> **Format and discipline:** see `CLAUDE.md § Edit rules` and `§ Writing lessons-learned`. Strict 5-line format, append-only, one rule per entry.

---

[LESSON — 2026-05-08] [confirmation]
Situation: Pelle ran the same 404 page in two configs before implement skill existed — bare Figma MCP, and mapping skill alone. Documented six failure modes: bare MCP deleted existing 404 page and emitted a new one; both modes added raw CSS props without consulting tokens; mapping skill introduced className alongside Emotion (parallel styling); random colors used where tokens existed; cache file ended up in git repo unnoticed; generic back-button copy "probeer opnieuw" instead of mapping-documented "ga terug".
What worked: v0.3 rules cover all six modes — rule #5 + B7 (file-path edit-existing), rule #2 + #12 (token-verdict + no-improvising), rule #3 (single styling API), rule #2 hoist-via-stack, mapping `setup` adds `figma-context/` to `.gitignore`, B4.5 + rule #12 (literal strings). Six-of-six failure modes addressed before first dogfood.
Proposal: Keep current rule set. Dogfood-run on the same 404 page validates that v0.3 actually prevents what Pelle observed — first real test target for Fase 5.

[LESSON — 2026-05-08] [correction]
Situation: Pelle reported the bare-MCP and mapping-only output filled only ~40% of viewport instead of 100vw, despite the Figma frame being a full-page 1440×900 design. Implement rule #4 covers Figma `fill/hug/gap` translation, but page-level "fill viewport" intent is not explicitly captured in mapping artifacts.
What did not work: Mapping skill currently has no convention for marking "this page-level frame is a full-viewport intent, emit 100vw/100vh not 1440px". Implement consumes whatever mapping documents — without an intent marker, emit defaults to literal Figma pixels.
Proposal: Mapping-side enhancement (added as suggestion in DEFERRED-FIXES.md) — extend `tokens.md § Auto-layout conventions` or per-component spec with a "fluidity intent" field for page-level frames. Cannot be solved in implement alone.
