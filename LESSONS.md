# Lessons learned

Record what worked and what did not during the skill's evolution. Each entry teaches the skill about itself — confirmations to deliberately repeat, corrections to fix.

> **Format and discipline:** see `CLAUDE.md § Edit rules` and `§ Writing lessons-learned`. Strict 5-line format, append-only, one rule per entry.

---

[LESSON — 2026-05-08] [confirmation]
Situation: Pre-implement-skill internal testing ran the same 404 page in two configs — bare Figma MCP, and mapping skill alone. Documented six failure modes: bare MCP deleted existing 404 page and emitted a new one; both modes added raw CSS props without consulting tokens; mapping skill introduced className alongside Emotion (parallel styling); random colors used where tokens existed; cache file ended up in git repo unnoticed; generic back-button copy "probeer opnieuw" instead of mapping-documented "ga terug".
What worked: v0.3 rules cover all six modes — rule #5 + B7 (file-path edit-existing), rule #2 + #12 (token-verdict + no-improvising), rule #3 (single styling API), rule #2 hoist-via-stack, mapping `setup` adds `figma-context/` to `.gitignore`, B4.5 + rule #12 (literal strings). Six-of-six failure modes addressed before first dogfood.
Proposal: Keep current rule set. Dogfood-run on the same 404 page validates that v0.3 actually prevents the observed failures — first real test target for Fase 5.

[LESSON — 2026-05-08] [correction]
Situation: Pre-skill testing reported the bare-MCP and mapping-only output filled only ~40% of viewport instead of 100vw, despite the Figma frame being a full-page 1440×900 design. Implement rule #4 covers Figma `fill/hug/gap` translation, but page-level "fill viewport" intent is not explicitly captured in mapping artifacts.
What did not work: Mapping skill currently has no convention for marking "this page-level frame is a full-viewport intent, emit 100vw/100vh not 1440px". Implement consumes whatever mapping documents — without an intent marker, emit defaults to literal Figma pixels.
Proposal: Mapping-side enhancement (added as suggestion in DEFERRED-FIXES.md) — extend `tokens.md § Auto-layout conventions` or per-component spec with a "fluidity intent" field for page-level frames. Cannot be solved in implement alone.

[LESSON — 2026-05-12] [confirmation]
Situation: Audit-based v0.4 fixes from skills.sh comparison + pre-skill internal testing observations. Three fixes landed: B7 differential emit (Read-before-write to prevent page-overschrijving), Rule #3 binair halt-on-mismatch (dual-styling regression prevention), B8 actief screenshot-diff (post-emit verification in-session).
What worked: Each fix has Wat-dit-toevoegt rationale + minimal text-investering. B7 is workflow-safeguard for proven regression. Rule #3 hardening makes "refuse" actionable as halt. B8 active diff closes the post-emit-blindness gap before PR-review (workflow-context: Claude Code session = primary safety net, not PR).
Proposal: Dogfood v0.4 on a real implement-pass before adding v0.5 (design-fidelity) or v1.0 (drift loop closing via drifts-implement.md). Confirm fixes prevent observed failures; only then expand scope.

[LESSON — 2026-05-12] [confirmation]
Situation: v0.4 audit-fixes (B7 read-before-write, rule #3 binary halt, B8 active diff) sluit niet de drift loop-closing gap — drifts blijven in mapping's `drifts.md` archief zonder structureel beslispunt. Architectuur-analyse toont: detectie + logging klaar, surfacing aan beslisser + decision-routing + status-update ontbreken.
What worked: Rule #8 uitgebreid naar drift-summary in chat na emit met decision-prompt per drift. Rule #10 uitzondering voor `drifts-mapping.md` (implement-owned, parallel aan mapping's `drifts.md`). § Write contract uitgebreid. B8.5 prompt + nieuwe B8.6 drift-summary step. Implement is self-sufficient — geen mapping-coordination nodig voor v0.5 ship.
Proposal: Bump v0.4 → v0.5 met loop-closing erin. Originally v1.0 scope, maar onafhankelijk van mapping coordination via aparte file. Dogfood-pass valideert of de loop in praktijk werkt.
