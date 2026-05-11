# Pre-emit checklist (B6)

Skill-internal format that Claude walks at B6 before producing code. In `/figma-to-code-implement check <link>` mode, Claude reports the filled-in checklist back to the user without emitting. In emit mode, Claude halts on any failed check.

Not copied to project repo. Never edited by the user.

---

## The 8-point check

For each item: mark ✅ pass or ❌ fail with a one-line note explaining why.

| # | Check | Rule | Status | Note |
|---|---|---|---|---|
| 1 | Every value emit-ready: token-path or hoisted to stack-appropriate higher-scope construct, never inline raw | #2 | [ ] | |
| 2 | Single styling API used (no parallel paradigms) | #3 | [ ] | |
| 3 | Auto-layout primitives translated per `tokens.md § Auto-layout conventions` | #4 | [ ] | |
| 4 | All components imported from existing locations per `components.md` | #5 | [ ] | |
| 5 | Literal strings match per-component spec § Literal strings (no generic substitutions) | #12 | [ ] | |
| 6 | No verify-queue blockers in emit scope | #7 | [ ] | |
| 7 | Drift notes from per-component specs + `drifts.md` surfaced to user (not silently fixed) | #8 | [ ] | |
| 8 | No new icon packages installed; assets from existing project files or MCP-localhost URLs | #9 | [ ] | |

## How to use

**In `check` mode:** Claude fills the entire table, reports to user, takes no further action. The output is the user's preview of what would block an emit.

**In emit mode:** Claude walks each item in order. First ❌ → halt. Surface the failure as a halt-message including:
- Which check failed
- Why it failed (the Note column content)
- What action would unblock (e.g., "run `/figma-to-code-mapping map X` for missing token", "resolve verify-queue item Y", "ask user how to handle drift Z")

## Output format for halt

When a check fails in emit mode:

```
Halt at B6 check #<N>: <check name>
Reason: <specific failure>
Unblock: <concrete action>
Rule violated: #<rule-number>
```

## Output format for `check` mode

Full table reported with all 8 rows filled. Then:

```
Summary: <N> blocker(s) found. Cannot proceed to B7.
```

or

```
Summary: 0 blockers. Ready for B7 emit.
```

## What this is NOT

- **Not a quality-of-output check.** That is B8 (post-emit visual validation).
- **Not a drift-detection check.** That is mapping skill's drift loop. B6 #7 only surfaces drifts that mapping already detected.
- **Not a code-style check.** TypeScript, JSDoc, ESLint are out of scope (CLAUDE.md edit rule #4).

This check is **pre-emit gate** — does the mapping-data permit a clean emit?
