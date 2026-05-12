# Drifts — implement-owned

> Owned by `figma-to-code-implement` skill. Mapping skill leest read-only.
>
> Mapping's eigen drifts leven in `drifts.md` (parallel file, mapping-owned).
> Implement schrijft hier alleen na user-decision in chat (per rule #8).

---

## Format

Eén regel per drift-decision:

```
[YYYY-MM-DD] [Severity][Owner] <Origin> — <description>. Decision: <action>. Status: <STATUS>.
```

**Velden:**

| Veld | Waardes | Toelichting |
|---|---|---|
| Date | `YYYY-MM-DD` | Datum van user-decision |
| Severity | `Critical` / `Major` / `Minor` | Per mapping's drift-severity classificatie |
| Owner | `DEV` / `DESIGNER` / `DEV+DESIGNER` | Wie moet handelen op de drift |
| Origin | `B8` / `mid-emit` / `mapping-surface` | Detectie-moment in implement-pass |
| Description | One-liner | Wat de drift is (max 80 char) |
| Decision | `revert` / `accept` / `update-code` / `update-mapping` | User's keuze in B8.5/B8.6 prompt |
| Status | `OPEN` / `ACCEPTED` / `IGNORED` / `SCHEDULED` / `RESOLVED` | Resolution-status na decision |

---

## Drift-entries

<!-- Append-only. Most recent at top. Format: zie boven. -->

(geen entries; implement schrijft hier na de eerste emit-run met drift-decisions)

---

## Schema-versie

`v0.5` — initieel format. Wijzigingen volgen via `LESSONS.md` entry + PR.
