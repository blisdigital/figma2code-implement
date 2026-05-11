# Figma-to-Code Implement skill

> **v0.1 — initial skeleton.** Emit-time enforcement skill that consumes the mapping
> produced by [`figma-to-code-mapping`](https://github.com/blisdigital/figma2code-mapping).
> Mapping enables; implementation enforces.

Implementation skill for developers who want Figma MCP code generation to emit code that consumes the existing codebase — its tokens, components, and styling stack — rather than producing duplicates or hardcoded values. Requires a `figma-to-code-mapping` output to exist for the relevant Figma scope.

## Vision

**Goal: emit code that uses the existing codebase, not duplicates it.** Mapping has already documented which Figma elements map to which code paths. This skill enforces that contract at the moment code is produced, via four mechanisms:

1. **Mapping consumption** — reads `tokens.md`, `components.md`, per-component specs, `drifts.md`, `verify-queue.md` before any emit
2. **Cache-first MCP** — hash-checks the Figma cache before a live MCP call; live fetch only on staleness
3. **Atomic-ordered resolution** — Page > Template > Organism > Molecule > Atom; stops on the highest mapped level
4. **Halt-and-ask discipline** — unmapped node, verify-queue blocker, multi-match ambiguity → halt + route, never improvise

Concrete effect: emit uses the right import paths, applies the documented styling API exclusively, refuses raw values where tokens exist, and surfaces drift instead of silently fixing it.

## For whom

This skill is a **developer tool** that **requires the mapping skill**. Audience: developers working in a React/Vue/Svelte/etc. codebase with a design system in Figma, who have already run `figma-to-code-mapping` to document the relationship, and now want Claude Code to emit production code from that mapping.

**Without mapping output:** this skill halts and routes to mapping. It is not an alternative to bare Figma MCP or skills.sh — those work for projects without mapping discipline. This skill is the strict-mode pipeline **on top of** mapping.

**Two-layer setup:**

| Layer | Where | What |
|---|---|---|
| **Skill (this repo)** | `~/.claude/skills/figma-to-code-implement/` (via symlink) | The method itself — automatically loaded by Claude Code |
| **Input (per project)** | `<your-project>/docs/` (from mapping skill) | The mapping output that this skill consumes |
| **Output (per project)** | `<your-project>/src/...` (or framework's source tree) | The emitted code |

The skill is project-agnostic and installed at user level. Input lives in each project repo separately (produced by the mapping skill). Output lands in the project's source tree.

## What it does

When triggered in a project repo with mapping output present:

- Validates that mapping exists for the requested Figma scope (halt + route otherwise)
- Reads mapping ground-truth (tokens, components, per-component specs, drifts, verify-queue)
- Performs a cache-first Figma MCP fetch — skips live MCP when cache is fresh
- Resolves each Figma element via three paths: direct mapping match → fingerprint-via-mapping-data → halt
- Validates the emit against an 8-point pre-emit check + 7-point post-emit visual check
- Emits code that uses existing tokens, components, and styling stack
- Records traceability back to mapping sources in commit/PR

Full method and twelve hard rules: see [SKILL.md](SKILL.md).

## Installation

The recommended approach is a **symlink** from `~/.claude/skills/figma-to-code-implement/` to this repo.

```bash
# Clone this repo
git clone https://github.com/blisdigital/figma2code-implement.git ~/Github/figma2code-implement

# Symlink into ~/.claude/skills/
ln -s ~/Github/figma2code-implement ~/.claude/skills/figma-to-code-implement

# Verify
claude
> /skills
# Should show figma-to-code-implement
```

To update:
```bash
cd ~/Github/figma2code-implement && git pull
```

## Usage

### Prerequisite — mapping skill installed and mapping output present

This skill **requires** `figma-to-code-mapping` to be installed and to have produced output for the relevant Figma scope. Install mapping first:

```bash
git clone https://github.com/blisdigital/figma2code-mapping.git ~/Github/figma2code-mapping
ln -s ~/Github/figma2code-mapping ~/.claude/skills/figma-to-code-mapping
```

Then in your project repo, run mapping setup:

```
$ cd /path/to/project
$ claude
> /figma-to-code-mapping setup
> /figma-to-code-mapping map <component>
```

Once mapping output exists, you can implement.

### Implement a Figma frame

```
> /figma-to-code-implement <Figma-link-or-nodeId>
```

Or natural language:

```
> Build this Figma frame: <URL>
> Implement this design: <URL>
> Generate code for <URL>
```

The skill walks through B1-B8: mapping presence check, read mapping, cache-first MCP, per-element resolution (three paths), pattern search if needed, 8-point pre-emit check, emit with traceability, 7-point post-emit visual check.

### Pre-emit check without emitting

```
> /figma-to-code-implement check <Figma-link-or-nodeId>
```

Runs B1-B6 and reports blockers (missing mapping, verify-queue items in scope, multi-match ambiguities) without producing code. Useful for previewing emit-readiness.

### Permanently activate in project repo

```
> /figma-to-code-implement init-claude-md
```

Shows a markdown block to paste into the project `CLAUDE.md`. From then on the skill triggers automatically on implement-intent sentences — no slash command needed. The block is additive on top of any existing `figma-to-code-mapping` snippet.

## File structure

```
figma2code-implement/
├── SKILL.md                            ← method + twelve hard rules + B1-B8 workflow
├── README.md                           ← this file (humans)
├── CLAUDE.md                           ← edit conventions for Claude in this repo
├── PLAN.md                             ← design history with open decisions
└── templates/                          ← (planned for Fase 3) skill-resident files in two subgroups:
                                            • 4 ephemeral workflow-formats (emit-trace, pre/post-emit checklists, pattern-adoption-note) — Claude reads per run, output to commit/PR/chat
                                            • 1 one-time paste-block (claude-md-snippet) — Claude shows via init-claude-md, user pastes once into project CLAUDE.md
                                            No setup command in either case.
```

## Sister skill — figma-to-code-mapping

Mapping is the data layer; implementation is the policy layer. They are one pipeline:

```
User: build this Figma frame
  → implement skill triggers
    → B1 mapping presence check
      → mapping exists → consume + emit
      → no mapping → halt + route to /figma-to-code-mapping map X
```

Routing is bi-directional: implement halts and routes to mapping on gaps; mapping skill handles routing of implement-intent sentences as it evolves — check the current [mapping `SKILL.md`](https://github.com/blisdigital/figma2code-mapping/blob/main/SKILL.md) for state.

**Use both together.** Map first, then implement. Mapping documents the relationship; implementation enforces it at code-emit time.

## Prerequisites — outside this skill

**Mapping must be produced for the relevant Figma scope.** This is the hard prerequisite. Without it, the skill halts and routes.

**Figma hygiene matters.** Clear frame names, semantic auto-layout, consistent variant decomposition. Messy Figma → messy mapping → messy emit, regardless of how well this skill enforces. Cleaning up Figma is upstream design-ops work.

**Existing codebase with tokens and component primitives.** Greenfield projects without a styleguide are out of scope. Build the styleguide first, then map, then implement.

## What this is not

- **Not a mapping tool.** It consumes mapping; it does not produce or update it. Mapping changes happen in `figma-to-code-mapping`.
- **Not a bare-MCP alternative.** Projects without mapping should use bare Figma MCP or [skills.sh `figma-implement-design`](https://skills.sh/figma/mcp-server-guide/figma-implement-design) — this skill is the strict-mode pipeline *on top of* mapping. We do not maintain a stand-alone variant and have no plans to: replicating bare-MCP for cold-start projects would dilute the value proposition (drift discipline, verify-queue, traceability) that exists only because mapping invests in it.
- **Not a minimal-adjust pixel-fixer.** This skill refuses inline pixel-fixes to match Figma exactly. Mismatches surface as drift, mapping's drift loop decides resolution. (Bewuste afwijking van skills.sh stap 6.)
- **Not a writer of mapping files.** Mapping files are read-only with two narrow exceptions (Path C halt routing, Path B fingerprint-match propose-to-user write to `verify-queue.md`).

Full skill boundary including routing to sister skills: see [SKILL.md § Skill boundary](SKILL.md#skill-boundary).
