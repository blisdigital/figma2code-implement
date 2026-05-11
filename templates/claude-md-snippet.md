# Claude-md snippet for the project repo

When the user invokes `/figma-to-code-implement init-claude-md`, show the following markdown block plus instructions. **Do not write to CLAUDE.md yourself.** The user pastes.

This is the one template that produces a persistent project artifact — but via user-action, not skill-write.

---

## What to show the user

A short intro:

> Here is the markdown block you can paste into `CLAUDE.md` of your project repo (root).
> It is **additive** on top of any existing `figma-to-code-mapping` snippet — keep both blocks side by side if mapping is also installed (it almost certainly is, because implement requires it).
>
> After pasting: the figma-to-code-implement skill triggers automatically on implement-intent sentences in this project — without typing the slash command every time.
>
> **Placement:**
> - If `CLAUDE.md` does not yet exist in the project-repo root: create it with this block.
> - If `CLAUDE.md` exists with a mapping snippet: append this block below it.
> - If `CLAUDE.md` exists without a mapping snippet: install the mapping snippet first via `/figma-to-code-mapping init-claude-md`, then come back here.

Then the block in a markdown code fence:

````markdown
## Figma-to-code emit

This project uses the figma-to-code-implement skill for emit-time enforcement
on top of figma-to-code-mapping. The skill triggers on implement-intent
sentences ("build this Figma frame", "implement this design",
"generate code for [URL]", "make this component").

**Method in short:**
- Read mapping output first (tokens.md, components.md, per-component specs,
  drifts.md, verify-queue.md)
- Consume existing components — never emit a new version
- Never emit raw where token exists; hoist via the project's styling stack
- Surface drift; never silently resolve or inline pixel-fix
- Cache-first MCP — skip live calls when mapping cache is fresh
- Halt + route to mapping skill when a node has no component-link

**Full method:** `~/.claude/skills/figma-to-code-implement/SKILL.md`

**Slash commands:**
- `/figma-to-code-implement <Figma-link>` — full emit (B1-B8)
- `/figma-to-code-implement check <Figma-link>` — pre-emit dry-run (B1-B6)
````

## What the user does next

1. Copies the block
2. Opens `CLAUDE.md` in the project-repo root (or creates it)
3. Pastes the block — at the end if mapping snippet already present, otherwise as initial content
4. Commits to git so it works team-wide
5. On the next chat in this project, the implement skill triggers automatically on implement-intent sentences

## What you do NOT do

- Do not write to `CLAUDE.md` in the project repo yourself
- Do not run `git add` or `git commit`
- Do not assume whether `CLAUDE.md` already exists — let the user check
- Do not assume the mapping snippet is already there — additive composition works either way (the skill itself halts and routes to mapping if mapping output is missing)
- Do not modify the mapping snippet — if user wants to update it, route to `/figma-to-code-mapping init-claude-md`

## Why additive composition (not replace)

Mapping and implement skills are one pipeline but separate skills. Each owns its own paste-block. If a user wants both auto-triggering, the project CLAUDE.md needs both blocks. Replacing mapping's block with implement's would break mapping triggering for non-implement sentences ("document this component", etc.).
