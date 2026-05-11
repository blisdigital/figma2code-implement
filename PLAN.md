# Plan — `figma-to-code-implement` skill

> Werkdocument naast [`IMPLEMENT-SKILL-PROPOSAL.md`](IMPLEMENT-SKILL-PROPOSAL.md).
> Het proposal beschrijft het *wat* en *waarom*; dit document beschrijft het *hoe* en de *volgorde van bouwen*.
> Update naar mate beslissingen vallen.

---

## Uitgangspunt

Mapping is de **datalaag**, implement is de **policylaag** die hem afdwingt op emit-time. De skill leeft in deze repo (`figma2code-implement`), gesymlinkt naar `~/.claude/skills/figma-to-code-implement/`. Zelfde structuur als `figma-to-code-mapping`: `SKILL.md`, `README.md`, `CLAUDE.md`, `templates/`.

**Pattern:** mapping enables, implement enforces. Mapping is read-only voor implement (op één uitzondering na: drift-rapportage terug naar de mapping repo — zie open beslissing #4).

---

## Bouwfases

### Fase 1 — Skeleton & frontmatter (1 sessie)

**Doel:** repo bootstrappen zodat hij installeerbaar is, ook als de inhoud nog dun is.

Deliverables:
- `SKILL.md` met frontmatter (`name: figma-to-code-implement`, `version: "0.1"`, description met expliciete triggers: "build this Figma frame", "implement this design", "generate code for [Figma URL]")
- `README.md` met installatie + symlinkconventie + relatie tot mapping skill
- `CLAUDE.md` snippet met "implement requires mapping output present"
- Lege `templates/` directory
- `DEFERRED-FIXES.md` voor open vragen (parallel aan mapping repo)

**Acceptatie:** symlink `~/.claude/skills/figma-to-code-implement → ~/Github/figma2code-implement` werkt; skill verschijnt in skill-list.

---

### Fase 2 — Hard rules & workflow B1-B8 uitschrijven (1 sessie)

**Doel:** policy expliciet maken, parallel aan de elf hard rules van de mapping skill.

Deliverables in `SKILL.md`:

**12 hard rules** (10 uit proposal + 1 uit skills.sh-challenge + 1 uit mapping rule #6 parallel), geformuleerd in emit-verbs (refuse / translate / hoist / halt), niet mapping-verbs:

1. Read mapping first (tokens.md + components.md + per-component specs + drifts.md + verify-queue.md)
2. Never emit raw where token exists; hoist to page-variable when raw is necessary
3. Single styling API only (per mapping's "Project styling stack")
4. Translate auto-layout via conventions table — no fixed pixels where Figma is fill/hug
5. **Consume existing components — never emit a new version. Read mapping carefully first.** Voor elke Figma node in scope: zoek via B4.1's drie paden (A: direct mapping match → B: fingerprint via mapping-data → C: halt + route). **Implement detecteert niet via eigen Figma-analysis; hij vergelijkt tegen mapping-gedocumenteerde stempel** (per-component specs + components.md). Geen code-file scans. Bij multi-match: user kiest, implement tie-breakt niet zelf.
6. Pattern reference for non-componentized layouts (search before invent)
7. Verify-queue blocks emit *for items in scope* (not all)
8. Surface mapping-recorded drift — never silently resolve. *(Implement consumes `drifts.md`; it does not run the drift-test. Mapping detects; implement surfaces.)*
9. Asset discipline (existing → MCP-localhost → never new)
10. No write outside emit-scope (only component/page/route code, no tests/docs/config)
11. **No minimal-adjust escape.** Mismatches surface as drift, never as inline pixel-fixes. *(Bewuste afwijking van skills.sh stap 6.)*
12. **No improvising on gaps.** Onbekende token-naam → halt. Geen pattern-match → halt. Missing literal string → halt. Geen verzonnen waarden of generieke fallbacks. *(Parallel aan mapping rule #6.)*

**Workflow B1-B8:**
- B1 mapping presence check
- B2 read mapping output (volgorde + welke bestanden)
- B3 **cache-first** chain:
  - B3.0 cache lookup + hash-check (`figma-context/<node-id>.json` vs current code-files)
  - B3.1 hash match → consume cache, geen MCP call
  - B3.2 hash mismatch → live MCP fallback chain: `get_design_context` + `get_screenshot` → payload reduction (`excludeScreenshot: true`, `forceCode: true`) → `get_metadata` + per-child `get_design_context`. Read-only — schrijft niet terug naar cache; waarschuwt user dat mapping refresh nodig is.
  - B3.3 geen cache → halt, route naar `/figma-to-code-mapping map X`
- B4 per-element resolution:
  - **B4.1 component-lookup — atomic-ordered** (Page > Template > Organism > Molecule > Atom). Drie paden in volgorde:
    - **Path A — direct mapping match.** Cache (`mapped_to_component`) of `components.md` linkt deze node aan een code-component. Match → consume.
    - **Path B — fingerprint-match via mapping-data** (alleen wanneer Path A geen match levert):
      1. **Bron-discipline:** uitsluitend `components.md` + per-component specs scannen. **Geen rechtstreekse code-file scans.** Component-stempel is volledig in mapping-data.
      2. **Atomic-level filter:** bepaal Figma frame-niveau op basis van structuur (klein container met 1-3 children = Atom; samengestelde structuur = Molecule). Scan alleen specs op zelfde of één-niveau-lager atomic-level.
      3. **Match-signalen** (in volgorde van betrouwbaarheid):

         | Signaal | Hoe | Sterkte |
         |---|---|---|
         | Naam-hint | Figma frame-naam matched/contains spec-naam ("Submit Button" → button.md) | Sterk |
         | Token-cluster overlap | Tokens in Figma frame ⊇ tokens gedocumenteerd in spec's mapping-tabel | Sterk |
         | Variant-axis match | Figma children/structure mapt op variants gedocumenteerd in spec (primary fill → variant=primary) | Tentatief |

      4. **Confidence-thresholds:**
         - ≥2 sterke signalen → propose to user
         - 1 sterk signaal → tentative suggestion
         - alleen tentatieve signalen → geen propose, behandel als element-frame
      5. **Multi-match handling:** bij 2+ kandidaten met vergelijkbare score → halt en toon alle kandidaten. **User kiest**, implement tie-breakt niet zelf.
      6. **Match outcomes:** user accepteert → consume + propose-write drift naar mapping (`verify-queue.md`: "Figma element-frame should be component-instance"). User weigert OR geen match → continue als pure element via B4.2 + B5.
      7. **Path B is mapping-data-driven, geen mapping-upgrade-afhankelijkheid.** Path B leest per-component specs zoals ze vandaag bestaan — werkt of mapping skill loose of strikte element-vs-component classification toepast. Een strakkere mapping-side discipline (expliciete classification methodology + verification step) reduceert hoe vaak Path B vuurt, maar is een **suggestie voor de mapping repo**, geen gekoppelde dependency. Hoort in mapping's eigen roadmap, niet in de coordination PR.
    - **Path C — geen match in A of B.** Halt + route naar mapping: *"Geen component-mapping en geen fingerprint-match voor deze node. Run `/figma-to-code-mapping map X` eerst."*
  - **B4.2 token-lookup** — voor alle Figma values (uitvoerend zodra B4.1 een match of expliciete element-classificatie heeft)
  - **B4.3 styling-stack adherence**
  - **B4.4 auto-layout translation**
  - **B4.5 literal strings**
- B5 pattern search voor non-componentized regio's (heuristieken in Fase 4)
- B6 8-point pre-emit validation
- B7 emit + traceability comment in PR/commit
- B8 post-emit visual validation tegen screenshot

**Slash commands:**
- `/figma-to-code-implement <link>` — full emit
- `/figma-to-code-implement check` — B1-B6 zonder emit

**Trigger-tabel** uit proposal: wanneer mapping vs implement vs ask.

**Acceptatie:** een ontwikkelaar kan SKILL.md lezen en weten wanneer hij moet stoppen vs doorgaan zonder ergens te improviseren.

---

### Fase 3 — Templates & decision artifacts (1 sessie)

**Doel:** de output van een implement-run reproduceerbaar maken, zodat traceability terug naar mapping intact blijft.

**Templates leven in skill repo, geen `setup` command kopieert ze naar project.** Anders dan mapping skill, dat per project `tokens.md` / `components.md` / specs in projectroot schrijft, hebben wij twee subgroepen:

1. **Ephemeral workflow-formats (4 files)** — Claude leest bij elke run, vult dynamisch in, output landt in **commit message / PR body / chat**. Niet persistent in project repo: `emit-trace.md`, `pre-emit-checklist.md`, `post-emit-visual-check.md`, `pattern-adoption-note.md`.
2. **One-time paste-block (1 file)** — Claude toont via `/figma-to-code-implement init-claude-md`, user paste éénmalig in project `CLAUDE.md` (committed via git, team-wide). Wel persistent in project repo, maar als user-actie, niet als skill-write: `claude-md-snippet.md` (additive op mapping skill snippet).

Geen `/figma-to-code-implement setup` command.

Deliverables in `templates/`:
- `emit-trace.md` — template voor het commit/PR-blok (welke mapping-bronnen, welke nodeIds, welke drifts gesurfaced)
- `pre-emit-checklist.md` — de 8-point check als invulbaar lijstje voor `check`-mode
- `post-emit-visual-check.md` — checklist voor B8 (layout, typo, kleur, states, responsive, assets, a11y)
- `claude-md-snippet.md` — paste-block voor project CLAUDE.md (additive op mapping skill snippet)
- `pattern-adoption-note.md` — vorm van het "ik heb pattern X uit file Y geadopteerd"-notitie voor B5

**Niet meer in lijst (vroeger gepland, nu geschrapt):**
- ~~`component-missing-drift.md`~~ — implement schrijft nooit zelf naar mapping's `drifts.md`. Bij Path C halt route je naar mapping skill; die schrijft de drift-row in zijn eigen `drifts.md` template-format. Onze versie zou duplicaat zijn.

**Acceptatie:** elke halt-conditie (verify-queue blocker, geen pattern gevonden) heeft een gestandaardiseerd output-formaat; B7 emit + traceability heeft een herhaalbaar commit-format; B8 visual-check is reproduceerbaar.

---

### Fase 4 — Heuristieken voor B5 pattern-search (1 sessie)

**Doel:** het grootste risico uit het proposal (false positives bij pattern-adoption) afdekken.

Deliverables:
- Heuristiek-volgorde in SKILL.md:
  1. zelfde route-tree parent
  2. zelfde category-mapje (errors/, detail/, dashboard/)
  3. bestandsnaam-similariteit
  4. halt
- Concreet voorbeeld met paden (gebruik `app/(routes)/orders/[id]/error.tsx` als exemplar voor andere error pages)
- Rule: confidence-floor — als <2 vergelijkbare files → halt, niet adopteren

**Acceptatie:** het proposal-risico "search-and-adopt produces false positives" is geadresseerd met expliciete grenzen.

---

### Fase 5 — Validatie op echt project (1-2 sessies)

**Doel:** dogfood op WorQX of Pelle's project.

- Draai `check` mode op een Figma frame waarvoor mapping al bestaat → verifieer dat de 8-point check juiste blockers vindt
- Draai full emit op één klein, geïsoleerd component → vergelijk output met handmatig gemaakte versie → meet hoeveel raw values, dubbele componenten, en parallelle styling-API's de skill nu daadwerkelijk weigert
- Update `DEFERRED-FIXES.md` met gevonden gaps

**Acceptatie:** één voorbeeldcomponent succesvol via implement skill in productiekwaliteit gegenereerd, met traceability comment in commit.

---

## Content placement — wat hoort in welke file

Vier files in deze skill-repo hebben elk een eigen lezer en eigen moment van laden. Plaatsing fout = Claude leest het verkeerde op het verkeerde moment.

| File | Wie leest | Wanneer geladen | Wat hoort hier |
|---|---|---|---|
| `SKILL.md` | Claude in een project waar de skill triggert | Bij elke triggermatch | Method, hard rules, B1-B8 procedure, slash commands, "what to read when"-tabel, skill boundary, references |
| `README.md` | Mens op GitHub | Niet door Claude | Installatie, symlink-stappen, usage-voorbeeld, prerequisites, "what this is not" |
| `CLAUDE.md` | Claude wanneer hij in **deze repo** werkt | Alleen tijdens skill-onderhoud | Edit-rules voor de skill zelf, branch/PR conventie, design-rationale (incl. skills.sh-challenge), version-bump policy, skill-vs-implementation lijn |
| `templates/claude-md-snippet.md` | Claude toont aan user, die paste naar projectroot CLAUDE.md | Bij `/figma-to-code-implement init-claude-md` | Paste-block, additive op mapping skill snippet |
| `templates/<rest>` | Claude leest bij elke run, vult dynamisch in, output gaat naar commit/PR/chat | Bij elke `/figma-to-code-implement` run (geen aparte setup) | Skill-interne workflow-formats voor consistente output (emit-trace, pre/post-emit checklists, pattern-adoption-note). **Niet** gekopieerd naar project repo — implement skill heeft geen `setup` command, anders dan mapping skill. |
| `DEFERRED-FIXES.md` | Skill-maintainer | Bij maintenance-sessies | Open beslissingen, TBD-items, ideeën die nog niet rijp zijn |

### SKILL.md table of contents (concept, naar voorbeeld van mapping skill)

```
1. Frontmatter (name, version, description met triggers)
2. Intro + scope-grens (mapping → implement boundary)
3. Vision (8 mechanismen die ≥90% match opleveren)
4. Hard rules (11 stuks, zelfde nummering als plan)
   ├─ Rule #2 box: "Do treat / Do NOT treat as implement-trigger"
   └─ Rule #11 box: bewuste afwijking van skills.sh stap 6
5. Skill boundary (wanneer wel/niet, routering naar zusterskills)
6. Mapping → implementation contract (welke kolommen consumeren we)
7. Slash commands
8. Source mechanism
   ├─ Cache reading (read-only contract, hash-check)
   ├─ MCP tools (get_design_context + get_screenshot + get_metadata-fallback)
   └─ Asset handling (existing → MCP-localhost → never new)
9. The documents (welke output-templates landen waar in project)
10. What to read when (per-taak tabel)
11. Method B1-B8 (elke stap zoals A1-A6 in mapping)
12. Drift handling (component-missing, value-mismatch surfaces)
13. References
```

### Wat NIET in SKILL.md komt
- **Skills.sh-vergelijkingstabel** — design-rationale, hoort in CLAUDE.md van deze repo
- **Bouwfases 1-5** — alleen hier in PLAN.md (voor onszelf)
- **Open beslissingen** — `DEFERRED-FIXES.md`
- **Versiehistorie / changelog** — `LESSONS.md` (parallel aan mapping)

---

## Skills.sh-challenge: wat we overnemen, wat we afwijken

> **Plek in repo:** deze tabel landt in `CLAUDE.md` van deze repo, niet in `SKILL.md` — design-rationale voor maintainers, geen runtime-info.

Twee referentieskills (`figma-implement-design` en `implement-design` op skills.sh) beschrijven dezelfde 7-staps workflow voor bare Figma MCP zonder mapping-laag.

### Overgenomen
| Element | Bron skills.sh | Onze plek |
|---|---|---|
| `get_screenshot` als visual source-of-truth | stap 3 | B3 + B8 |
| `get_metadata` fallback bij truncation | stap 2 | B3 |
| Post-emit visual validation | stap 7 | B8 (nieuw) |
| Asset discipline (geen nieuwe icon-packages) | stap 4 | rule #9 (al aanwezig) |

### Bewust afwijkend
| Element | Skills.sh | Onze keuze |
|---|---|---|
| "Adjust spacing or sizes minimally to match visuals" | stap 6 | **rule #11** — never. Mismatch → drift. |
| Bare MCP zonder mapping-prerequisite | impliciet | **B1** — mapping required. |
| Drift-omgang | impliciet inline-fix | drift-flag + halt of explicit decision |
| Verify-queue | ontbreekt | B1 + B6 blockers |
| Pattern voor non-componentized | "reuse components" alleen | **B5** search-and-adopt met heuristieken |
| Traceability | ontbreekt | **B7** commit-trace naar mapping bronnen |

**Samenvatting:** skills.sh is steviger op live MCP-mechanica (screenshot, metadata-fallback, post-emit check) — overgenomen. Onze skill is strenger op mapping-discipline (drift, verify-queue, pattern-search, traceability) — bewust strenger gehouden.

---

## Koppeling met mapping skill — naadloosheid & routering

Implement en mapping zijn één pipeline. Drie checks:

### Routering tussen skills

| Richting | Trigger | Geregeld? |
|---|---|---|
| **Implement → mapping** | Geen mapping bestaat | ✓ B1 halt + `/figma-to-code-mapping setup` |
| **Implement → mapping** | Node niet gemapt | ✓ B4.1 halt + `/figma-to-code-mapping map X` |
| **Implement → mapping** | Cache stale (hash mismatch) | ✓ B3.2 waarschuwing + `/figma-to-code-mapping map X` |
| **Mapping → implement** | User vraagt om code generation tijdens mapping | ⚠ Vereist mapping-side updates (zie deferred fixes) |
| **Mapping → implement** | Na `/figma-to-code-mapping map X` completion | ⚠ Optionele auto-handoff (zie deferred fixes) |

### Coördinatie-items (geen overlap, wel afstemming)

1. **`claude-md-snippet.md`** — implement's snippet moet *additive* op mapping snippet zijn, niet duplicaat. `init-claude-md` checkt of mapping snippet aanwezig is en toont alleen het implement-blok ernaast.
2. **README cross-links** — beide README's krijgen "Sister skill" sectie met 3-zins flow-uitleg en link naar de ander.
3. **"Implement this design from Figma" semantiek** — beide skills moeten dezelfde "do treat / do not treat" tekst hanteren (open beslissing #6 is hier het anker).

### Deferred fixes in mapping repo (`figma2code-mapping`)

Track in `DEFERRED-FIXES.md` van **mapping** repo, niet hier. Coördineer met v3.x bump van mapping:

1. **Skill boundary tabel update** (mapping SKILL.md line 67-79) — vervang *"on the roadmap, belongs in a separate skill"* door concrete routering: *"out of scope → use `figma-to-code-implement`"*.
2. **Rule #2 box uitbreiding** — voeg implement-triggers toe: *"Implement-intent sentences (`build this Figma frame`, `implement this design`, `generate code for [URL]`) → route to `figma-to-code-implement`"*.
3. **Optionele auto-handoff na `map X` completion** — *"Mapping complete for X. Run `/figma-to-code-implement X` now?"* — parallel aan A5 recursive Uses pattern.
4. **Cache-schema formaliseren** — mapping SKILL.md line 113-123 is nu één voorbeeldblok. Implement consumeert dit strikt. Mapping documenteert schema-velden expliciet (required vs optional, types) zodat implement een stabiel contract heeft.

---

## Mapping → implement contract (cruciaal — leg vast in Fase 2)

Implement skill is **read-only consument** van mapping-output. Geen schrijven naar mapping-files behalve één uitzondering (zie #5).

| Mapping-artifact | Implement gebruikt voor | Lees-/schrijfrichting |
|---|---|---|
| `tokens.md` (3e kolom: token-verdict) | Rule #2 — refuse raw / hoist | read-only |
| `tokens.md § Project styling stack` | Rule #3 — single styling API | read-only |
| `tokens.md § Auto-layout conventions` | Rule #4 / B4.4 | read-only |
| `components.md` | Rule #5 — consume existing, **atomic-order: Page > Template > Organism > Molecule > Atom** | read-only |
| `<component-folder>/<name>.md` § Variant mapping table | B4.1 — Figma `Type=` → code prop-axes decomposition | read-only |
| `<component-folder>/<name>.md` § Variant mapping § state mechanism | B4.3 — emit correct state mechanism (color-shift vs opacity vs class) | read-only |
| `<component-folder>/<name>.md` § Literal strings (aria-label, alt, placeholder, title) | B4.5 — emit documented values, never generic | read-only |
| `<component-folder>/<name>.md` § Drift notes | B6 check #7 — surface unresolved drifts in scope | read-only |
| `drifts.md` | B6 check #7 — surface unresolved drifts in scope. Geen drift-detectie hier — die zit in mapping. | read-only (zie uitzondering hieronder) |
| `verify-queue.md` | B1+B6 — block emit on items in scope | read-only |
| `figma-context/<node-id>.json` § `mapped_to_component` | B4.1 — directe nodeId → code-component sprong | read-only |
| `figma-context/<node-id>.json` § hash | B3 — staleness check before MCP refresh | read-only voor cache; MCP refresh schrijft, dat is mapping-werk |
| `figma-context/<node-id>.json` § `master_verified_via: "instance-id-format"` | B4.1 — instance-id `I<frame>;<master>` herkenning | read-only |

**Uitzondering — wanneer implement WEL (propose-to-user) schrijft naar mapping:**
1. B4.1 Path C halt → propose-to-user om route naar mapping te volgen. Implement schrijft niets, mapping-skill vult zijn eigen files tijdens een eigen run.
2. B4.1 Path B fingerprint-match geaccepteerd → propose-to-user om drift-row naar mapping's `verify-queue.md` te schrijven (*"Figma element-frame should be component-instance — matched on [signalen]"*). Schrijft alleen na user-bevestiging. Volgt mapping rule #7.

---

## Open beslissingen — verplaatsen naar `DEFERRED-FIXES.md` in Fase 1

Deze landen niet in SKILL.md. Antwoord vóór Fase 2 om verlamming te voorkomen.

1. **Verify-queue scope:** blokkeert verify-queue *alle* emits of alleen items in de huidige scope? *Voorstel: alleen scope* (proposal-risico #4).
2. **Cache-staleness check:** hoe ziet "hash mismatch" er concreet uit? Per-component spec krijgt een `last-mcp-hash` veld, of vergelijking op tokens-snapshot?
3. **Wanneer halt vs warn:** rules #5 (component-missing), #7 (verify-queue), #8 (drift) zijn nu allemaal hard halts. Werkbaar in praktijk, of moet er een `--force` flag zijn?
4. **Output-locatie:** schrijft implement skill ooit terug naar de mapping repo (bijv. nieuw `component-missing` drift)? Zo ja, hoe — directe edit, voorstel-naar-user, of alleen log? *Voorstel staat hierboven onder "uitzondering": propose-to-user, na bevestiging.*
5. **Confidence-floor B5:** is "<2 vergelijkbare files → halt" het juiste getal, of moet dit per project-grootte schalen?
6. **"Do treat / Do NOT treat" box rule #2:** welke zinnen tellen wel als implement-trigger? Voorstel: "build this Figma frame", "implement this design", "generate code for [URL]", "make this component". NIET: kale Figma-URL plak, "Implement this design from Figma" auto-clipboard.
7. **Confirmation gate vóór B7 emit?** Parallel aan mapping rule #7. Implement schrijft per definitie code. Halt-en-vraag bij elke emit is sleur voor atoms/molecules; geen confirmation bij templates/pages is risicovol. *Voorstel: gate aan vanaf Organism-niveau en hoger; atoms/molecules direct emitten met B6 als enige check.*
8. **Partial mapping handling — beslist.** Resolution volgt atomic-order: Page → Template → Organism → Molecule → Atom. Stop op hoogste gemapt niveau. Halt op eerste ungated Figma node die concreet in scope geïnstantieerd zou worden — route naar mapping om te bepalen of het een component-link is of een element-frame. Implement maakt die call niet zelf.
9. ~~Deviation-comment in code bij geaccepteerde drift.~~ **Geschrapt** — dubbel met B7 traceability en drift-acceptance hoort in mapping's `drifts.md`, niet in elke gebruikssite.

---

## Status

**Fase 0 (planning) — in progress.** Dit document + proposal zijn de entry points wanneer we beginnen aan Fase 1.

**Owner:** Kevin Rutten.

**Estimated effort to first usable version:** 5-7 sessies + dogfood-pass op een echt project. Schaal vergelijkbaar met v3.0 mapping skill.
